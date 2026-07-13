# Application Requirements Specification

- **Status:** Draft — pending review
- **Date:** 2026-07-13
- **Scope:** Platform-wide functional and non-functional requirements. This document defines *what* the platform does: operations, transaction lifecycle, canonical data model, business outcomes, and the constraints that shape all contracts. It deliberately does not define per-service internals (covered by per-service specs) nor concrete event schemas (covered by the event catalog spec, which is derived from this document).
- **References:** `doc/ARCHITECTURE.md` (how the platform is structured). Where this document and the architecture overlap, the architecture governs structure and this document governs behavior.

## 1. Purpose

A payments platform that lets registered vendors (merchants) accept card and wallet payments through a single asynchronous REST API, processes each operation through an event-driven pipeline against the vendor's configured acquirer bank, and delivers results via status polling and webhooks. Target volume: ~5 million transactions/day.

## 2. Actors

| Actor | Role |
|---|---|
| **Vendor** | Merchant integrated with the public API. Owns transactions; authenticated per request (JWT). |
| **Payer** | The cardholder. Never interacts with the platform directly (no hosted checkout in scope — see §10). |
| **Acquirer bank** | External bank API that authorizes/declines operations. One integration service per bank. |
| **Card networks** | Visa/Mastercard network tokenization (VTS/MDES) for stored-credential operations. |
| **Wallet providers** | Apple Pay / Google Pay: source of encrypted wallet payloads. |
| **Platform operator** | Runs the platform; consumes reporting and settlement outputs. |

## 3. Payment methods

- **FR-M1** — `CARD`: direct card payments (PAN + expiry + CVV, tokenized at the edge by `pan-proxy-service`).
- **FR-M2** — `APPLE_PAY`: encrypted Apple Pay payment token, decrypted only inside `card-vault-service`, then processed on the card rail.
- **FR-M3** — `GOOGLE_PAY`: same pattern as Apple Pay.
- **FR-M4** — Stored credential: when a network token exists for a card (provisioned after first authorization), subsequent operations on that card are routed through the network token. This is a routing concern, not a distinct vendor-facing method.
- **FR-M5** — A vendor may only use payment methods enabled in its configuration (`vendor-service`); operations with a disabled method are rejected structurally (`400`).

## 4. Operations

All operations are submitted via the public API and processed asynchronously (§7). Operations are either **initial** (create a transaction) or **follow-up** (reference an existing transaction by id and are only accepted when the current state allows them — see §5).

### Initial operations

- **FR-O1 — `authenticate`**: verifies that the card is valid and active **without reserving or moving funds** (zero-amount account verification against the acquirer). Terminal outcome: `AUTHORIZED` (verification passed) / `DECLINED` / `FAILED`. Not eligible for capture, void, or cancel.
- **FR-O2 — `authorise`**: reserves funds on the card for the given amount without capturing them. Outcome `AUTHORIZED` reserves the funds; the reservation expires per acquirer rules (default validity window: 7 calendar days, configurable per vendor/acquirer) if not captured.
- **FR-O3 — `payment`** (sale): authorization + capture in a single operation. Outcome `AUTHORIZED` is immediately followed by `CAPTURED` (single acquirer call; both state transitions recorded).

### Follow-up operations

- **FR-O4 — `deferred`** (capture): captures a previous `authorise`. Allowed only when the referenced transaction is `AUTHORIZED` and within its validity window. Capture amount must be ≤ authorized amount (partial capture allowed, **at most one capture** per authorization — multi-capture is out of scope).
- **FR-O5 — `cancel`**: releases a previous `authorise` that will not be captured. Allowed only in state `AUTHORIZED` (of an `authorise`). Terminal state: `CANCELLED`.
- **FR-O6 — `void`**: reverses a captured, not-yet-settled money movement (`payment` or `deferred`) — allowed only before the transaction is included in a settlement batch for its acquirer (cutoff, §9). Terminal state: `VOIDED`.
- **FR-O7** — Follow-up operations run through the same asynchronous pipeline (validation → TID → acquirer) and are themselves recorded as operations linked to the original transaction.
- **FR-O8** — Refunds (returning funds after settlement) are **out of scope for v1** (§10). `settlement-service` must nevertheless model money movement direction so refunds can be added without a file-format redesign.

### Cross-cutting operation rules

- **FR-O9** — Every operation belongs to exactly one vendor; a vendor can only reference its own transactions (enforced from the JWT, returns `404` on foreign ids — never `403`, to avoid id-space probing).
- **FR-O10** — Amounts are integers in **minor units** (e.g. cents); currency is **ISO 4217 alphabetic**. Zero/negative amounts are structurally invalid except `authenticate` (amount must be absent or zero).
- **FR-O11** — Follow-up operations never carry card data; they reference the original transaction's token.

## 5. Transaction lifecycle

State machine owned by `gateway-service` (authoritative; consistent with `ARCHITECTURE.md`):

```
REGISTERED ──► VALIDATED ──► TID_ASSIGNED ──► SENT_TO_ACQUIRER ──┬─► AUTHORIZED
     │              │                                            ├─► DECLINED   (terminal)
     └─► REJECTED   └─► REJECTED                                 └─► FAILED     (terminal)
        (terminal)     (terminal)

AUTHORIZED ──► CAPTURED     (payment: immediately; authorise: via deferred)
AUTHORIZED ──► CANCELLED    (terminal — via cancel, authorise only)
AUTHORIZED ──► EXPIRED      (terminal — validity window elapsed, no capture)
CAPTURED   ──► VOIDED       (terminal — via void, before settlement cutoff)
CAPTURED   ──► SETTLED      (terminal — included in a confirmed settlement batch)
```

- **FR-L1** — Transitions and their triggers:

| From | To | Trigger |
|---|---|---|
| — | `REGISTERED` | Gateway accepts the request (`202`) |
| `REGISTERED` | `VALIDATED` | Method service business validation passes |
| `REGISTERED`/`VALIDATED` | `REJECTED` | Business validation fails (method service) |
| `VALIDATED` | `TID_ASSIGNED` | TID allocated for vendor/acquirer |
| `TID_ASSIGNED` | `SENT_TO_ACQUIRER` | Acquirer adapter sends the request |
| `SENT_TO_ACQUIRER` | `AUTHORIZED` / `DECLINED` / `FAILED` | Acquirer result |
| `AUTHORIZED` | `CAPTURED` | Sale auto-capture or successful `deferred` |
| `AUTHORIZED` | `CANCELLED` | Successful `cancel` |
| `AUTHORIZED` | `EXPIRED` | Validity window elapses without capture |
| `CAPTURED` | `VOIDED` | Successful `void` before cutoff |
| `CAPTURED` | `SETTLED` | Settlement batch confirmed for its cutoff |

- **FR-L2** — Every transition is recorded with a timestamp and the id of the causing event; the transition history is part of the canonical record (and of reporting).
- **FR-L3** — A follow-up operation that fails (`DECLINED`/`FAILED`/`REJECTED`) does **not** change the state of the original transaction; the follow-up records its own outcome.
- **FR-L4** — Every service re-validates the transaction state on event consumption; a consumed event is a snapshot, not current state.

## 6. Canonical transaction model

The minimal data set that identifies and describes a transaction platform-wide. Events carry this snapshot (event-carried state transfer); the event catalog spec turns it into concrete schemas.

- **FR-D1** — Core fields:

| Field | Notes |
|---|---|
| `transactionId` | UUID, assigned by the gateway; the platform-wide correlation id and Kafka partition key |
| `operationType` | `AUTHENTICATE` \| `AUTHORISE` \| `PAYMENT` \| `DEFERRED` \| `CANCEL` \| `VOID` |
| `originalTransactionId` | Present on follow-ups only |
| `vendorId` | Owning vendor |
| `vendorReference` | Vendor's own order/reference id (opaque string, ≤ 64 chars) |
| `paymentMethod` | `CARD` \| `APPLE_PAY` \| `GOOGLE_PAY` |
| `amount`, `currency` | Minor units + ISO 4217 (absent for `authenticate`) |
| `cardToken` | Vault token — **never** raw PAN |
| `maskedCard` | BIN, last4, expiry month/year, scheme |
| `networkTokenUsed` | Whether routing used a network token |
| `acquirerId` | Resolved from vendor configuration |
| `tid` | Terminal id, present from `TID_ASSIGNED` onward |
| `state` | Current lifecycle state (§5) |
| `result` | Outcome detail (§8): platform result code + acquirer response code |
| `createdAt`, `updatedAt` | UTC instants |

- **FR-D2** — Raw PAN, CVV, decrypted wallet payloads, and DPANs must **never** appear in this model, in any event, log, or API response. Card data outside `card-vault-service` is restricted to token + masked fields.
- **FR-D3** — `transactionId` is the Kafka partition key for all transaction lifecycle events, guaranteeing per-transaction ordering.

## 7. Public API behavior

- **FR-A1** — Submission is asynchronous: the gateway validates structurally, persists, and answers `202 Accepted` with a `Location` header → `GET /api/v1/payments/{transactionId}`. No operation result is ever returned synchronously.
- **FR-A2** — Structural validation failures return `400` with a machine-readable error list (field, code, message). Structural = required fields, formats, amounts, currency, method enabled, vendor active, referenced transaction exists and belongs to the vendor.
- **FR-A3** — Status polling returns the canonical transaction snapshot (§6) including current state and result. Polling is served from read replicas; staleness of up to ~1 s is acceptable.
- **FR-A4 — Idempotency**: every state-creating request carries a vendor-supplied `Idempotency-Key` header (unique per vendor). Retries with the same key return the original `202` and the same `transactionId` without creating a duplicate. Keys are retained for at least 24 h. A reused key with a different payload returns `409`.
- **FR-A5** — Authentication: JWT per request, scoped to one vendor. Invalid/expired token → `401`; valid token but disabled vendor → structural rejection.
- **FR-A6** — API is versioned under `/api/v1/`; breaking changes require a new version prefix.

## 8. Business outcomes and result codes

- **FR-R1** — Every terminal outcome carries a **platform result code** (stable, documented enum — the vendor-facing contract) plus the **raw acquirer response code** (informational, per-bank).
- **FR-R2** — Platform result code families:

| Family | Meaning | Examples |
|---|---|---|
| `REJECTED_*` | Failed business validation, never reached a bank | `REJECTED_DUPLICATE`, `REJECTED_INVALID_STATE`, `REJECTED_CARD_RULES`, `REJECTED_EXPIRED_CARD` |
| `DECLINED_*` | Bank answered negatively | `DECLINED_GENERIC`, `DECLINED_INSUFFICIENT_FUNDS`, `DECLINED_SUSPECTED_FRAUD`, `DECLINED_DO_NOT_HONOUR` |
| `FAILED_*` | Technical failure, outcome not obtained | `FAILED_ACQUIRER_TIMEOUT`, `FAILED_ACQUIRER_UNAVAILABLE`, `FAILED_INTERNAL` |
| `APPROVED` | Success for the requested operation | — |

- **FR-R3** — Each acquirer adapter owns the mapping from its bank's response codes to platform result codes; the mapping is part of that adapter's spec and tests.
- **FR-R4** — A `FAILED_ACQUIRER_TIMEOUT` means the real outcome is **unknown**; such transactions are flagged for reconciliation and are never silently retried with money movement (no automatic re-send after an ambiguous result).

## 9. Notifications and settlement (behavioral requirements)

- **FR-N1** — On every terminal outcome (and on `CAPTURED`), `notification-service` delivers a webhook to the vendor's configured callback URL carrying the canonical snapshot. Delivery is at-least-once with exponential-backoff retries and a dead-letter state visible in backoffice; vendors must treat notifications idempotently (by `transactionId` + state).
- **FR-N2** — Webhooks are signed (HMAC with a per-vendor secret) so vendors can authenticate the origin.
- **FR-S1** — Settlement: per acquirer, all money-moving operations (`CAPTURED` not `VOIDED`) of a cutoff window are written to that bank's settlement file and delivered over SFTP at the bank's cutoff time/timezone. Inclusion in a confirmed batch transitions the transaction to `SETTLED`.
- **FR-S2** — The `void` window for a transaction closes when its settlement batch generation starts (per-bank cutoff); a `void` arriving later is rejected with `REJECTED_INVALID_STATE`.

## 10. Out of scope (v1)

Refunds; hosted checkout; 3DS / redirect challenge flows; APMs (PayPal, etc.); fraud screening; multi-capture; multi-currency settlement per vendor account. See `ARCHITECTURE.md` §Planned Extensions.

## 11. Non-functional requirements

- **NFR-1 — Volume:** ~5 M transactions/day sustained (~58 tps average); design for peak **600 tps** (10× average) on the submission path.
- **NFR-2 — Latency:** `202` response (including edge tokenization) p99 ≤ 500 ms; status polling p99 ≤ 200 ms; end-to-end submission→terminal-state p95 ≤ 5 s with a healthy acquirer.
- **NFR-3 — Delivery semantics:** all Kafka consumers are idempotent (at-least-once delivery); dedup key = `eventId`. Per-transaction ordering via partition key (`transactionId`, FR-D3).
- **NFR-4 — Event contract:** events are versioned, backward-compatible, schema-registry-enforced snapshots carrying the canonical model (§6). Every event carries an envelope: `eventId` (UUID), `eventType`, `occurredAt`, `schemaVersion`, plus the snapshot.
- **NFR-5 — Durability:** no accepted operation may be lost — state change and event are atomic (transactional outbox, `ARCHITECTURE.md`).
- **NFR-6 — PCI:** CDE boundary and data-handling rules per `ARCHITECTURE.md` §PCI DSS Compliance; FR-D2 applies to every artifact in this repo including test fixtures (use test PANs only inside vault/proxy/adapter test code).
- **NFR-7 — Retention:** historical store ≥ 13 months (scheme dispute window); active database only in-flight + operational window; idempotency keys ≥ 24 h.
- **NFR-8 — Availability:** submission path (proxy, gateway, vault tokenization) target 99.95%; asynchronous processing tolerates downstream outages by buffering in Kafka — a degraded acquirer must not affect other acquirers (per-bank isolation, `ARCHITECTURE.md`).
- **NFR-9 — Auditability:** every detokenization call is audited (caller, transaction, timestamp); every state transition is traceable to its causing event (FR-L2).

## 12. Open questions for review

1. **`void` vs `cancel` semantics (FR-O5/O6):** this spec defines `cancel` = release authorization, `void` = reverse unsettled capture. Confirm this matches the intended vendor-facing meaning, since `ARCHITECTURE.md` lists both without distinguishing them.
2. **Refunds:** `ARCHITECTURE.md` (settlement section) mentions refunds among money-moving operations, but no refund operation exists in the gateway API. This spec declares refunds out of scope for v1 (FR-O8) — confirm, or add a `refund` operation now.
3. **`authenticate` semantics (FR-O1):** defined here as zero-amount account verification. If it is instead meant as the future 3DS authentication step, it should move to Planned Extensions and out of v1.
4. **`EXPIRED` and `SETTLED` states:** added by this spec (not in the `ARCHITECTURE.md` diagram) because authorization expiry and settlement confirmation are real lifecycle facts. Confirm, and update `ARCHITECTURE.md` accordingly.
5. **Partial capture (FR-O4):** allowed (≤ authorized amount, single capture). Confirm.
6. **Peak factor (NFR-1):** 10× average is an assumption; adjust if traffic is spikier (e.g. retail events).
