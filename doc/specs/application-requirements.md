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
| **Card networks** | Visa/Mastercard network tokenization (VTS/MDES) for stored-credential operations, and the EMV 3DS Directory Server / issuer ACS for `authenticate`. |
| **Wallet providers** | Apple Pay / Google Pay: source of encrypted wallet payloads. |
| **Platform operator** | Runs the platform; consumes reporting and settlement outputs. |

## 3. Payment methods

- **FR-M1** — `CARD`: direct card payments (PAN + expiry + CVV, tokenized at the edge by `pan-proxy-service`).
- **FR-M2** — `APPLE_PAY`: encrypted Apple Pay payment token. The decrypted payload **never contains the real PAN** — it always carries a device-specific network token (DPAN) plus a single-use cryptogram; Apple's wallet specification has no "raw PAN" variant. `applepay-service` decrypts the payload itself (holding the Apple merchant identity certificate) and forwards the DPAN directly onto the card rail — `card-vault-service` is not involved in this flow.
- **FR-M3** — `GOOGLE_PAY`: encrypted Google Pay payment token. Unlike Apple Pay, the decrypted content depends on the **card auth method the vendor is configured to accept** (Google Pay `allowedCardAuthMethods`): `CRYPTOGRAM_3DS` (device token + cryptogram, same shape as Apple Pay) or `PAN_ONLY` (the **real PAN**, still wallet-encrypted in transit). Because the same deployed `googlepay-service` may serve vendors configured either way, it never decrypts the payload itself: it always forwards the still-encrypted envelope to `card-vault-service`, which decrypts and tokenizes it regardless of which auth method it turns out to be, returning only a token. This removes any risk of a configuration bug routing a `PAN_ONLY` payload down a path that bypasses the vault.
- **FR-M4** — Stored credential: when a network token exists for a card (provisioned after first authorization), subsequent operations on that card are routed through the network token. This is a routing concern, not a distinct vendor-facing method.
- **FR-M5** — A vendor may only use payment methods enabled in its configuration (`vendor-service`); operations with a disabled method are rejected structurally (`400`).
- **FR-M6** — For vendors enabling `GOOGLE_PAY`, `vendor-service` additionally stores the **allowed card auth method(s)** (`CRYPTOGRAM_3DS`, `PAN_ONLY`, or both). This setting only affects vendor-facing configuration and validation; it never changes where decryption happens (always `card-vault-service`, per FR-M3).

## 4. Operations

All operations are submitted via the public API and processed asynchronously (§7). Operations are either **initial** (create a transaction) or **follow-up** (reference an existing transaction by id and are only accepted when the current state allows them — see §5).

### Initial operations

- **FR-O1 — `authenticate`**: performs EMV 3-D Secure (3DS) cardholder authentication via `threeds-service`, confirming with the card's issuer that the request originates from the legitimate cardholder, **without reserving or moving funds**. Optional: a vendor calls it independently, before an `authorise`/`payment`, when it wants 3DS liability-shift protection. Two possible outcomes, decided by the issuer (ARes):
  - **Frictionless** — the issuer authenticates silently; the operation resolves directly to a terminal outcome (`AUTHORIZED` / `DECLINED` / `FAILED`).
  - **Challenge** — the issuer requires cardholder interaction (OTP, biometric, etc.); the operation enters the non-terminal `CHALLENGE_REQUIRED` state (§5), exposing a `challengeUrl` (FR-D1) for the vendor to redirect the payer's browser to. Resolves to a terminal outcome once the payer completes the challenge, or `FAILED_CHALLENGE_TIMEOUT` if the challenge window elapses (default 10 minutes).

  Not eligible for capture, void, or cancel. A successful `authenticate` produces authentication evidence (ECI, authentication value, `threeDsTransactionId`) that a subsequent `authorise`/`payment` can reference and forward to the acquirer (FR-O12).
- **FR-O2 — `authorise`**: reserves funds on the card for the given amount without capturing them. Outcome `AUTHORIZED` reserves the funds; the reservation expires per acquirer rules (default validity window: 7 calendar days, configurable per vendor/acquirer) if not captured. **v1 note:** capturing or releasing an `authorise` (`deferred`, `cancel`) is not yet implemented (see below) — in this reduced scope, an `AUTHORIZED` `authorise` can only resolve to `EXPIRED`.
- **FR-O3 — `payment`** (sale): authorization + capture in a single operation. Outcome `AUTHORIZED` is immediately followed by `CAPTURED` (single acquirer call; both state transitions recorded).

### Follow-up operations

- **FR-O6 — `void`**: reverses a captured, not-yet-settled `payment` — allowed only before the transaction is included in a settlement batch for its acquirer (cutoff, §9). Terminal state: `VOIDED`.

> **Removed for v1 (2026-07-13):** `deferred` (capture) and `cancel`, previously `FR-O4`/`FR-O5`, are descoped — they will be reintroduced later with different behavior than previously specified here. The identifiers `FR-O4`/`FR-O5` are retired and must not be reused for unrelated requirements. See `Resolved` below.
- **FR-O7** — Follow-up operations run through the same asynchronous pipeline (validation → TID → acquirer) and are themselves recorded as operations linked to the original transaction.
- **FR-O8** — Refunds (returning funds after settlement) are **out of scope for v1** (§10). `settlement-service` must nevertheless model money movement direction so refunds can be added without a file-format redesign.

### Cross-cutting operation rules

- **FR-O9** — Every operation belongs to exactly one vendor; a vendor can only reference its own transactions (enforced from the JWT, returns `404` on foreign ids — never `403`, to avoid id-space probing).
- **FR-O10** — Amounts are integers in **minor units** (e.g. cents); currency is **ISO 4217 alphabetic**. Zero/negative amounts are structurally invalid except `authenticate` (amount must be absent or zero).
- **FR-O11** — Follow-up operations never carry card data; they reference the original transaction's token.
- **FR-O12** — `authorise` and `payment` may optionally reference a prior successful `authenticate` transaction via `threeDsAuthenticationId` (FR-D1); when present, the authentication evidence produced by that transaction (ECI, authentication value, `threeDsTransactionId`) is forwarded to the acquirer with the authorization request, for 3DS liability-shift purposes.

## 5. Transaction lifecycle

State machine owned by `gateway-service` (authoritative; consistent with `ARCHITECTURE.md`):

```
REGISTERED ──► VALIDATED ──► TID_ASSIGNED ──► SENT_TO_ACQUIRER ──┬─► AUTHORIZED
     │              │                                            ├─► DECLINED   (terminal)
     └─► REJECTED   └─► REJECTED                                 └─► FAILED     (terminal)
        (terminal)     (terminal)

AUTHORIZED ──► CAPTURED     (payment: immediately — authorise has no capture path in v1, see FR-O2)
AUTHORIZED ──► EXPIRED      (terminal — validity window elapsed, no capture)
CAPTURED   ──► VOIDED       (terminal — via void, before settlement cutoff)
CAPTURED   ──► SETTLED      (terminal — included in a confirmed settlement batch)
```

`authenticate` only — bypasses `TID_ASSIGNED` / `SENT_TO_ACQUIRER` (3DS is scheme-level, resolved by the card network's Directory Server and the issuer, not the acquirer bank):

```
REGISTERED ──► VALIDATED ──► SENT_TO_THREEDS ──┬─► AUTHORIZED   (frictionless)
     │              │                          ├─► DECLINED     (terminal)
     └─► REJECTED   └─► REJECTED               ├─► FAILED       (terminal)
        (terminal)     (terminal)              └─► CHALLENGE_REQUIRED ──┬─► AUTHORIZED
                                                                          ├─► DECLINED
                                                                          ├─► FAILED
                                                                          └─► FAILED (challenge timeout)
```

- **FR-L1** — Transitions and their triggers:

| From | To | Trigger |
|---|---|---|
| — | `REGISTERED` | Gateway accepts the request (`202`) |
| `REGISTERED` | `VALIDATED` | Method service business validation passes |
| `REGISTERED`/`VALIDATED` | `REJECTED` | Business validation fails (method service) |
| `VALIDATED` | `TID_ASSIGNED` | TID allocated for vendor/acquirer (not `authenticate`) |
| `TID_ASSIGNED` | `SENT_TO_ACQUIRER` | Acquirer adapter sends the request |
| `SENT_TO_ACQUIRER` | `AUTHORIZED` / `DECLINED` / `FAILED` | Acquirer result |
| `VALIDATED` | `SENT_TO_THREEDS` | 3DS authentication request sent (`authenticate` only) |
| `SENT_TO_THREEDS` | `AUTHORIZED` / `DECLINED` / `FAILED` | Frictionless issuer decision (ARes) |
| `SENT_TO_THREEDS` | `CHALLENGE_REQUIRED` | Issuer requires cardholder challenge (ARes) |
| `CHALLENGE_REQUIRED` | `AUTHORIZED` / `DECLINED` / `FAILED` | Challenge completed (issuer RRes via callback, FR-A9) |
| `CHALLENGE_REQUIRED` | `FAILED` | Challenge window elapses without completion (default 10 min) |
| `AUTHORIZED` | `CAPTURED` | Sale (`payment`) auto-capture |
| `AUTHORIZED` | `EXPIRED` | Validity window elapses without capture (only path out of `AUTHORIZED` for `authorise` in v1) |
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
| `operationType` | `AUTHENTICATE` \| `AUTHORISE` \| `PAYMENT` \| `VOID` |
| `originalTransactionId` | Present on follow-ups only |
| `vendorId` | Owning vendor |
| `vendorReference` | Vendor's own order/reference id (opaque string, ≤ 64 chars) |
| `paymentMethod` | `CARD` \| `APPLE_PAY` \| `GOOGLE_PAY` |
| `amount`, `currency` | Minor units + ISO 4217 (absent for `authenticate`) |
| `cardToken` | Opaque card identifier — **never** raw PAN. For `CARD` and `GOOGLE_PAY`: a `card-vault-service`-minted token. For `APPLE_PAY`: the wallet-issued network token (DPAN) itself, since the vault is not involved in that flow (FR-M2) |
| `maskedCard` | BIN, last4, expiry month/year, scheme |
| `networkTokenUsed` | Whether routing used a network token |
| `acquirerId` | Resolved from vendor configuration |
| `tid` | Terminal id, present from `TID_ASSIGNED` onward (not `authenticate`) |
| `threeDsAuthenticationId` | Optional; links an `authorise`/`payment` to a prior successful `authenticate` transaction (FR-O12) |
| `challengeUrl` | Present only while `state = CHALLENGE_REQUIRED`; URL the vendor redirects the payer's browser to |
| `returnUrl` | Vendor-supplied at submission of `authenticate` (mandatory, FR-A8); where the platform redirects the browser once the challenge completes |
| `state` | Current lifecycle state (§5) |
| `result` | Outcome detail (§8): platform result code + acquirer response code |
| `createdAt`, `updatedAt` | UTC instants |

- **FR-D2** — Raw PAN, CVV, and full decrypted wallet payloads (beyond the DPAN + cryptogram, `APPLE_PAY` only) must **never** appear in this model, in any event, log, or API response outside `card-vault-service`. A DPAN is exempt from this restriction and may appear as `cardToken` (FR-D1) for `APPLE_PAY`: it is a network-issued, device- and merchant-scoped, single-use token with no path back to the real PAN outside the card network, and is safe to route through the platform's internal Kafka topics under the same handling as any other token. Card data outside `card-vault-service` is otherwise restricted to token + masked fields.
- **FR-D3** — `transactionId` is the Kafka partition key for all transaction lifecycle events, guaranteeing per-transaction ordering.

## 7. Public API behavior

- **FR-A1** — Submission is asynchronous: the gateway validates structurally, persists, and answers `202 Accepted` with a `Location` header → `GET /api/v1/payments/{transactionId}`. No operation result is ever returned synchronously.
- **FR-A2** — Structural validation failures return `400` with a machine-readable error list (field, code, message). Structural = required fields, formats, amounts, currency, method enabled, vendor active, referenced transaction exists and belongs to the vendor.
- **FR-A3** — Status polling returns the canonical transaction snapshot (§6) including current state and result. Polling is served from read replicas; staleness of up to ~1 s is acceptable.
- **FR-A4 — Idempotency**: every state-creating request carries a vendor-supplied `Idempotency-Key` header (unique per vendor). Retries with the same key return the original `202` and the same `transactionId` without creating a duplicate. Keys are retained for at least 24 h. A reused key with a different payload returns `409`.
- **FR-A5** — Authentication: JWT per request, scoped to one vendor. Invalid/expired token → `401`; valid token but disabled vendor → structural rejection.
- **FR-A6** — API is versioned under `/api/v1/`; breaking changes require a new version prefix.
- **FR-A7** — While `state = CHALLENGE_REQUIRED`, the status response (FR-A3) includes `challengeUrl` (FR-D1) so the vendor can redirect the payer's browser to complete the 3DS challenge.
- **FR-A8** — `returnUrl` is a **mandatory** field on `authenticate` requests (structural validation, extends FR-A2): the issuer's frictionless-vs-challenge decision cannot be predicted by the vendor at submission time, so it must always be supplied.
- **FR-A9** — The platform exposes a public, **unauthenticated** endpoint, `POST /api/v1/payments/{transactionId}/3ds-challenge-result`, that receives the issuer ACS's challenge-completion callback — a browser redirect as part of the EMV 3DS challenge flow, not a vendor-authenticated API call. It is validated against the pending challenge session opened when `CHALLENGE_REQUIRED` began (single-use, time-bound correlation token — see NFR-10), never the vendor JWT, and never carries card data. After processing, the platform redirects the browser to the vendor's `returnUrl`.

## 8. Business outcomes and result codes

- **FR-R1** — Every terminal outcome carries a **platform result code** (stable, documented enum — the vendor-facing contract) plus the **raw acquirer response code** (informational, per-bank).
- **FR-R2** — Platform result code families:

| Family | Meaning | Examples |
|---|---|---|
| `REJECTED_*` | Failed business validation, never reached a bank | `REJECTED_DUPLICATE`, `REJECTED_INVALID_STATE`, `REJECTED_CARD_RULES`, `REJECTED_EXPIRED_CARD` |
| `DECLINED_*` | Bank/issuer answered negatively | `DECLINED_GENERIC`, `DECLINED_INSUFFICIENT_FUNDS`, `DECLINED_SUSPECTED_FRAUD`, `DECLINED_DO_NOT_HONOUR`, `DECLINED_AUTHENTICATION_FAILED` (issuer explicitly rejected 3DS authentication, e.g. wrong OTP) |
| `FAILED_*` | Technical failure, outcome not obtained | `FAILED_ACQUIRER_TIMEOUT`, `FAILED_ACQUIRER_UNAVAILABLE`, `FAILED_INTERNAL`, `FAILED_CHALLENGE_TIMEOUT` (3DS challenge window elapsed without completion) |
| `APPROVED` | Success for the requested operation | — |

- **FR-R3** — Each acquirer adapter owns the mapping from its bank's response codes to platform result codes; the mapping is part of that adapter's spec and tests.
- **FR-R4** — A `FAILED_ACQUIRER_TIMEOUT` means the real outcome is **unknown**; such transactions are flagged for reconciliation and are never silently retried with money movement (no automatic re-send after an ambiguous result).

## 9. Notifications and settlement (behavioral requirements)

- **FR-N1** — On every terminal outcome (and on `CAPTURED`), `notification-service` delivers a webhook to the vendor's configured callback URL carrying the canonical snapshot. Delivery is at-least-once with exponential-backoff retries and a dead-letter state visible in backoffice; vendors must treat notifications idempotently (by `transactionId` + state).
- **FR-N2** — Webhooks are signed (HMAC with a per-vendor secret) so vendors can authenticate the origin.
- **FR-S1** — Settlement: per acquirer, all money-moving operations (`CAPTURED` not `VOIDED`) of a cutoff window are written to that bank's settlement file and delivered over SFTP at the bank's cutoff time/timezone. Inclusion in a confirmed batch transitions the transaction to `SETTLED`.
- **FR-S2** — The `void` window for a transaction closes when its settlement batch generation starts (per-bank cutoff); a `void` arriving later is rejected with `REJECTED_INVALID_STATE`.

## 10. Out of scope (v1)

Refunds; hosted checkout; APMs (PayPal, etc.); fraud screening; multi-capture; multi-currency settlement per vendor account; 3DS / redirect challenge flows **for APMs** (cards are covered in v1 via `authenticate` + `threeds-service`, §3–§7); **`deferred` (capture) and `cancel`** — descoped, to be reintroduced with different behavior than previously specified (see `Resolved`). In this reduced v1, `authorise` transactions can only resolve via `EXPIRED`. See `ARCHITECTURE.md` §Planned Extensions.

## 11. Non-functional requirements

- **NFR-1 — Volume:** ~5 M transactions/day sustained (~58 tps average); design for peak **600 tps** (10× average) on the submission path.
- **NFR-2 — Latency:** `202` response (including edge tokenization) p99 ≤ 500 ms; status polling p99 ≤ 200 ms; end-to-end submission→terminal-state p95 ≤ 5 s with a healthy acquirer.
- **NFR-3 — Delivery semantics:** all Kafka consumers are idempotent (at-least-once delivery); dedup key = `eventId`. Per-transaction ordering via partition key (`transactionId`, FR-D3).
- **NFR-4 — Event contract:** events are versioned, backward-compatible, schema-registry-enforced snapshots carrying the canonical model (§6). Every event carries an envelope: `eventId` (UUID), `eventType`, `occurredAt`, `schemaVersion`, plus the snapshot.
- **NFR-5 — Durability:** no accepted operation may be lost — state change and event are atomic (transactional outbox, `ARCHITECTURE.md`).
- **NFR-6 — PCI:** CDE boundary and data-handling rules per `ARCHITECTURE.md` §PCI DSS Compliance; FR-D2 applies to every artifact in this repo including test fixtures (use test PANs only inside vault/proxy/adapter test code). `threeds-service` is in the CDE: it detokenizes at the last hop before sending the authentication request to the card scheme's Directory Server, the same pattern as `<acquirer-bank>-tx-service`.
- **NFR-7 — Retention:** historical store ≥ 13 months (scheme dispute window); active database only in-flight + operational window; idempotency keys ≥ 24 h.
- **NFR-8 — Availability:** submission path (proxy, gateway, vault tokenization) target 99.95%; asynchronous processing tolerates downstream outages by buffering in Kafka — a degraded acquirer must not affect other acquirers (per-bank isolation, `ARCHITECTURE.md`).
- **NFR-9 — Auditability:** every detokenization call is audited (caller, transaction, timestamp); every state transition is traceable to its causing event (FR-L2).
- **NFR-10 — Challenge session integrity:** the public 3DS challenge-callback endpoint (FR-A9) is unauthenticated by necessity (hit by the payer's browser, not the vendor) and must be protected against replay/forgery by a single-use, time-bound correlation token issued when `CHALLENGE_REQUIRED` began; a missing, expired, or reused token is rejected without changing transaction state.

## 12. Open questions for review

1. **Refunds:** `ARCHITECTURE.md` (settlement section) mentions refunds among money-moving operations, but no refund operation exists in the gateway API. This spec declares refunds out of scope for v1 (FR-O8) — confirm, or add a `refund` operation now.
2. **`EXPIRED` and `SETTLED` states:** added by this spec (not in the `ARCHITECTURE.md` diagram) because authorization expiry and settlement confirmation are real lifecycle facts. Confirm, and update `ARCHITECTURE.md` accordingly.
3. **Peak factor (NFR-1):** 10× average is an assumption; adjust if traffic is spikier (e.g. retail events).
4. **3DS mandate (FR-O1/FR-O12):** `authenticate` is modeled as optional (vendor chooses whether to call it). If a regulatory mandate (e.g. PSD2/SCA) should make it compulsory for certain transactions/regions, that rule needs to be added explicitly — out of scope for this pass.
5. **`deferred`/`cancel` redesign:** descoped for v1 (see `Resolved`). Until they return, `authorise` is effectively a dead end (only resolves via `EXPIRED`) — confirm whether `authorise` should also be temporarily removed from the vendor-facing API rather than exposing an operation with no useful completion path.

## Resolved

- **Wallet decryption split (FR-M2/FR-M3, 2026-07-13):** confirmed that `applepay-service` decrypts payloads itself (no real PAN ever possible, per Apple's wallet spec) while `googlepay-service` always routes through `card-vault-service` regardless of the vendor's configured auth method, since Google Pay's `PAN_ONLY` mode can carry the real PAN. `ARCHITECTURE.md` updated accordingly (`applepay-service`, `googlepay-service`, `card-vault-service`, `vendor-service`, PCI DSS Compliance sections).
- **`authenticate` = 3DS authentication, with challenge support (FR-O1/FR-O12, 2026-07-13):** confirmed `authenticate` performs EMV 3DS cardholder authentication via a new `threeds-service`, with full challenge-redirect support in v1 (not deferred, unlike the original `ARCHITECTURE.md` Planned Extensions note). Adds the `CHALLENGE_REQUIRED` state, `threeDsAuthenticationId` / `challengeUrl` / `returnUrl` fields, a public challenge-callback endpoint (FR-A9), and puts `threeds-service` in the PCI CDE (detokenizes to route the AReq to the card scheme's Directory Server). `ARCHITECTURE.md` updated (`threeds-service` section, CDE list, Planned Extensions, Payment Flow, Repository Layout, `bank-simulator-service`).
- **`deferred` and `cancel` descoped from v1 (former FR-O4/FR-O5, 2026-07-13):** removed from this pass — will be reintroduced later with different behavior than previously specified (the earlier definitions, capture-an-authorise and release-an-authorise, are not carried forward as assumptions). The identifiers `FR-O4` and `FR-O5` are retired and must not be reused. Consequences applied: `void` (FR-O6) now only reverses `payment` captures (the `deferred` case removed); the `CANCELLED` state and its transition are removed from the lifecycle; `authorise` (FR-O2) can currently only resolve via `EXPIRED`, flagged as open question 5 above. `ARCHITECTURE.md` updated (`gateway-service` operation list, state machine diagram, Payment Flow follow-up note).
