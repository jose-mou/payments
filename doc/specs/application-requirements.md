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

**Scope note (2026-07-13):** the `authenticate` ↔ `authorise`/`payment` relationship described below (FR-O1–FR-O3) applies to `CARD` only. `APPLE_PAY` and `GOOGLE_PAY` are exempt — the wallet's own device-level cardholder verification (biometrics/passcode + cryptogram) already fulfills what 3DS establishes for direct card entry, so their `authorise`/`payment` flow is unchanged (no reference to, or execution of, `authenticate`).

### Initial operations

- **FR-O1 — `authenticate`** (`CARD` only): performs EMV 3-D Secure (3DS) cardholder authentication via `threeds-service`, confirming with the card's issuer that the request originates from the legitimate cardholder, **without reserving or moving funds**. Carries an `amount`/`currency` (FR-O10) — not itself a reservation, but the **cumulative ceiling** available to one or more subsequent `authorise` transactions that reference it (FR-O2, FR-O12). Two possible outcomes, decided by the issuer (ARes):
  - **Frictionless** — the issuer authenticates silently; the operation resolves directly to a terminal outcome (`AUTHORIZED` / `DECLINED` / `FAILED`).
  - **Challenge** — the issuer requires cardholder interaction (OTP, biometric, etc.); the operation enters the non-terminal `CHALLENGE_REQUIRED` state (§5), exposing a `challengeUrl` (FR-D1) for the vendor to redirect the payer's browser to. Resolves to a terminal outcome once the payer completes the challenge, or `FAILED_CHALLENGE_TIMEOUT` if the challenge window elapses (default 10 minutes).

  A successful (`AUTHORIZED`) `authenticate` transaction may be referenced by **one or more** subsequent `authorise` transactions (FR-O2, FR-O12), as long as their cumulative amount does not exceed it and its validity window hasn't elapsed — it is not itself capturable, voidable, or refundable.
- **FR-O2 — `authorise`**: reserves funds on the card for the given amount without capturing them. For `CARD`, **always references a prior successful `authenticate` transaction** (`threeDsAuthenticationId`, FR-D1, **same currency** as the referenced `authenticate`) — the cardholder must already be 3DS-verified before funds can be reserved, so `authorise` never runs 3DS itself and skips straight to TID assignment (§5). The referenced `authenticate` transaction must (FR-O12): belong to the same vendor, be in state `AUTHORIZED`, be referenced within its validity window (default 15 minutes from authentication — reflecting typical 3DS session validity, shorter than `authorise`'s own reservation window below), and this `authorise`'s amount plus the sum of all other currently-`AUTHORIZED` `authorise` amounts already referencing the same `authenticate` must not exceed the `authenticate`'s own amount (a **voided** `authorise`, FR-O6, releases its amount back to this cumulative total — see below). Only successful (`AUTHORIZED`) `authorise` attempts count toward the cumulative total; declined/failed/rejected attempts do not consume any of it. Violations are rejected structurally (`400`) or as `REJECTED_INVALID_STATE`, per FR-A2. For `APPLE_PAY`/`GOOGLE_PAY`, no `authenticate` reference applies (scope note above). Outcome `AUTHORIZED` reserves the funds; the reservation has a validity window (default 7 calendar days, configurable per vendor/acquirer). In v1 (no `deferred` capture — see `Resolved`), an `AUTHORIZED` `authorise` resolves only via `void` (releasing it, FR-O6 — which also frees its amount back to the referenced `authenticate`'s remaining ceiling) or `EXPIRED` if neither voided nor otherwise resolved within its window.
- **FR-O3 — `payment`** (sale): for `CARD`, performs 3DS authentication and authorization, followed by immediate capture, **all as part of the same transaction** — a single `transactionId`, not a reference to a separately submitted `authenticate` (contrast with `authorise`, FR-O2). Internally: `SENT_TO_THREEDS` → (`CHALLENGE_REQUIRED` →) `THREEDS_AUTHENTICATED` → `TID_ASSIGNED` → `SENT_TO_ACQUIRER` → `AUTHORIZED` → `CAPTURED` (§5). Requires `returnUrl` (FR-A8) for the same reason `authenticate` does: the issuer's frictionless-vs-challenge decision is unknown until the request is sent. For `APPLE_PAY`/`GOOGLE_PAY`, `payment` has no embedded 3DS step (scope note above): authorization + capture in a single acquirer call, as before.

### Follow-up operations

- **FR-O6 — `void`**: reverses a transaction that has **not yet been settled** — either an `authorise` still in state `AUTHORIZED` (releasing the reservation, and its amount back to the referenced `authenticate`'s remaining ceiling, FR-O2) or a `payment` in state `CAPTURED` but not yet included in a settlement batch for its acquirer (cutoff, §9). Full amount only, and **at most one `void` per transaction** — unlike `refund` (FR-O8), it is not partial and not repeatable. Terminal state: `VOIDED`.
- **FR-O8 — `refund`**: reverses a `payment` that has **already been settled** — the post-settlement counterpart of `void` (FR-O6). Unlike `void`, **partial and multiple `refund`s are allowed** against the same `payment`: one or more `refund` transactions may reference it, as long as their cumulative amount does not exceed the original `payment`'s amount. Only successful (`AUTHORIZED`) `refund` attempts count toward this cumulative total; declined/failed/rejected attempts do not consume any of it. A `refund` is itself a new acquirer-facing money movement: it runs through the same pipeline as any other operation (validation → TID → acquirer, FR-O7) since funds must actually be returned rather than simply withheld, and settles in its own right, in the opposite direction (`settlement-service` models this — no file-format change needed). Once the cumulative refunded amount reaches the original `payment`'s full amount, it transitions `SETTLED` → `REFUNDED` (§5); partially refunded, it remains `SETTLED` (further `refund`s still possible up to the remaining amount).
- **FR-O7** — Follow-up operations run through the same asynchronous pipeline (validation → TID → acquirer) and are themselves recorded as operations linked to the original transaction.

> **Removed for v1 (2026-07-13):** `deferred` (capture) and `cancel`, previously `FR-O4`/`FR-O5`, are descoped — `void` now covers releasing an unsettled `authorise` (FR-O6), which was `cancel`'s role. They may be reintroduced later with different behavior than previously specified here. The identifiers `FR-O4`/`FR-O5` are retired and must not be reused for unrelated requirements. See `Resolved` below.

### Cross-cutting operation rules

- **FR-O9** — Every operation belongs to exactly one vendor; a vendor can only reference its own transactions — including a referenced `authenticate` (FR-O2) or the original transaction for `void`/`refund` — enforced from the JWT, returns `404` on foreign ids (never `403`, to avoid id-space probing).
- **FR-O10** — Amounts are integers in **minor units** (e.g. cents); currency is **ISO 4217 alphabetic**. Zero/negative amounts are structurally invalid for every operation, including `authenticate` (FR-O1), whose amount is the cumulative ceiling for the `authorise`(s) that will reference it.
- **FR-O11** — Follow-up operations never carry card data; they reference the original transaction's token.
- **FR-O12** — For `CARD`, `authorise`'s reference to its prerequisite `authenticate` transaction (FR-O2) is **mandatory but not single-use**: a given `authenticate` may be referenced by multiple `authorise` transactions, gated by the cumulative-amount rule (FR-O2/FR-O1) rather than a reuse-count restriction; a `void` on one of those `authorise` transactions (FR-O6) frees its amount back to the ceiling, making it available to a further `authorise` within the `authenticate`'s validity window. `payment` does not reference an external `authenticate` transaction — its embedded 3DS step (FR-O3) produces evidence used only for its own acquirer call, not shared with other transactions.

## 5. Transaction lifecycle

State machine owned by `gateway-service` (authoritative; consistent with `ARCHITECTURE.md`). `authenticate`, `authorise` and `payment` diverge from `VALIDATED` onward — each has its own path, shown below (`CARD`; see the scope note in §4 for `APPLE_PAY`/`GOOGLE_PAY`, which follow the `authorise`/`payment` diagrams with the 3DS-related states simply absent).

### `authenticate`

Verifies the cardholder via 3DS; never moves or reserves funds, but carries an amount that acts as a cumulative ceiling for the `authorise`(s) that reference it. Terminal here — one or more later `authorise` transactions reference it (FR-O2) but do not continue its state machine; it is not itself capturable, voidable, or refundable.

```
REGISTERED ──► VALIDATED ──► SENT_TO_THREEDS ──┬─► AUTHORIZED   (frictionless, terminal)
     │              │                          ├─► DECLINED     (terminal)
     └─► REJECTED   └─► REJECTED               ├─► FAILED       (terminal)
        (terminal)     (terminal)              └─► CHALLENGE_REQUIRED ──┬─► AUTHORIZED
                                                                          ├─► DECLINED
                                                                          ├─► FAILED
                                                                          └─► FAILED (challenge timeout)
```

### `authorise`

For `CARD`, always references a prior successful `authenticate` (FR-O2) — 3DS is never repeated here; validation includes checking that reference and its remaining cumulative ceiling (FR-O12), then the flow skips straight to TID assignment. Multiple `authorise` transactions may reference the same `authenticate`, gated by amount, not by count. (`APPLE_PAY`/`GOOGLE_PAY`: same diagram, no `authenticate` reference involved.)

```
REGISTERED ──► VALIDATED ──► TID_ASSIGNED ──► SENT_TO_ACQUIRER ──┬─► AUTHORIZED
     │              │                                            ├─► DECLINED   (terminal)
     └─► REJECTED   └─► REJECTED                                 └─► FAILED     (terminal)
        (terminal)     (terminal)

AUTHORIZED ──► VOIDED    (terminal — via void, releasing the reservation, FR-O6)
AUTHORIZED ──► EXPIRED   (terminal — validity window elapsed, not voided)
```

### `payment`

For `CARD`, performs 3DS authentication and authorization, then immediate capture, as part of the same transaction (FR-O3): `THREEDS_AUTHENTICATED` marks a successful embedded 3DS step (frictionless or post-challenge) before continuing to TID assignment. (`APPLE_PAY`/`GOOGLE_PAY`: no embedded 3DS — starts directly at `TID_ASSIGNED`.)

```
REGISTERED ──► VALIDATED ──► SENT_TO_THREEDS ──┬─► THREEDS_AUTHENTICATED ──► TID_ASSIGNED ──► SENT_TO_ACQUIRER ──┬─► AUTHORIZED ──► CAPTURED
     │              │                          ├─► DECLINED (terminal)                                          ├─► DECLINED (terminal)
     └─► REJECTED   └─► REJECTED               ├─► FAILED   (terminal)                                          └─► FAILED   (terminal)
        (terminal)     (terminal)              └─► CHALLENGE_REQUIRED ──┬─► THREEDS_AUTHENTICATED (continues above)
                                                                          ├─► DECLINED (terminal)
                                                                          ├─► FAILED   (terminal)
                                                                          └─► FAILED   (challenge timeout, terminal)

CAPTURED ──► VOIDED     (terminal — via void, before settlement cutoff, FR-O6)
CAPTURED ──► SETTLED    (settlement batch confirmed for its cutoff)
SETTLED  ──► REFUNDED   (terminal — via refund, FR-O8)
```

- **FR-L1** — Notable transition triggers not fully evident from the diagrams above:

| From | To | Trigger |
|---|---|---|
| `REGISTERED`/`VALIDATED` | `REJECTED` | Business validation fails (method service) — for `authorise` (`CARD`), this includes the referenced `authenticate` failing any check in FR-O12 (wrong vendor, not `AUTHORIZED`, reference expired, or this `authorise`'s amount would exceed the `authenticate`'s remaining cumulative ceiling) |
| `CHALLENGE_REQUIRED` | terminal outcome / `THREEDS_AUTHENTICATED` | Issuer's challenge result (RRes) via the public callback (FR-A9), or `FAILED_CHALLENGE_TIMEOUT` if the challenge window elapses (default 10 minutes) |
| `AUTHORIZED` (`authorise`) | `VOIDED` | Successful `void` (FR-O6) — also frees the `authorise`'s amount back to its referenced `authenticate`'s remaining ceiling |
| `CAPTURED` (`payment`) | `VOIDED` | Successful `void` before the settlement cutoff (FR-O6, FR-S2) |
| `CAPTURED` | `SETTLED` | Settlement batch confirmed for its cutoff (FR-S1) |
| `SETTLED` | `REFUNDED` | Successful `refund` (FR-O8) |

- **FR-L2** — Every transition is recorded with a timestamp and the id of the causing event; the transition history is part of the canonical record (and of reporting).
- **FR-L3** — A follow-up operation that fails (`DECLINED`/`FAILED`/`REJECTED`) does **not** change the state of the original transaction; the follow-up records its own outcome.
- **FR-L4** — Every service re-validates the transaction state on event consumption; a consumed event is a snapshot, not current state.

## 6. Canonical transaction model

The minimal data set that identifies and describes a transaction platform-wide. Events carry this snapshot (event-carried state transfer); the event catalog spec turns it into concrete schemas.

- **FR-D1** — Core fields:

| Field | Notes |
|---|---|
| `transactionId` | UUID, assigned by the gateway; the platform-wide correlation id and Kafka partition key |
| `operationType` | `AUTHENTICATE` \| `AUTHORISE` \| `PAYMENT` \| `VOID` \| `REFUND` |
| `originalTransactionId` | Present on follow-ups only |
| `vendorId` | Owning vendor |
| `vendorReference` | Vendor's own order/reference id (opaque string, ≤ 64 chars) |
| `paymentMethod` | `CARD` \| `APPLE_PAY` \| `GOOGLE_PAY` |
| `amount`, `currency` | Minor units + ISO 4217. For `authenticate` (`CARD`): the cumulative ceiling available to referencing `authorise`s (FR-O1). For `void`: always equal to the original transaction's amount (full-amount only, FR-O6). For `refund`: up to the original `payment`'s remaining unrefunded amount (partial allowed, FR-O8) |
| `cardToken` | Opaque card identifier — **never** raw PAN. For `CARD` and `GOOGLE_PAY`: a `card-vault-service`-minted token. For `APPLE_PAY`: the wallet-issued network token (DPAN) itself, since the vault is not involved in that flow (FR-M2) |
| `maskedCard` | BIN, last4, expiry month/year, scheme |
| `networkTokenUsed` | Whether routing used a network token |
| `acquirerId` | Resolved from vendor configuration |
| `tid` | Terminal id, present from `TID_ASSIGNED` onward (not `authenticate`) |
| `threeDsAuthenticationId` | `CARD` `authorise` only: mandatory link to its prerequisite `authenticate` transaction (FR-O2, FR-O12) — the same `authenticate` may be shared by several `authorise` transactions. Not used by `payment`, which performs 3DS internally as part of the same transaction, nor by `APPLE_PAY`/`GOOGLE_PAY` |
| `challengeUrl` | Present only while `state = CHALLENGE_REQUIRED` (`authenticate` or `CARD` `payment`); URL the vendor redirects the payer's browser to |
| `returnUrl` | Vendor-supplied at submission of `authenticate` or `CARD` `payment` (mandatory, FR-A8); where the platform redirects the browser once a challenge completes |
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
- **FR-A8** — `returnUrl` is a **mandatory** field on `authenticate` requests and on `CARD` `payment` requests (structural validation, extends FR-A2): the issuer's frictionless-vs-challenge decision cannot be predicted by the vendor at submission time, so it must always be supplied. Not required for `authorise` (3DS already resolved via its referenced `authenticate`, FR-O2) nor for `APPLE_PAY`/`GOOGLE_PAY` `payment` (no embedded 3DS step).
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
- **FR-S1** — Settlement: per acquirer, all money-moving operations (`CAPTURED` `payment`s not `VOIDED`, and successful `refund`s) of a cutoff window are written to that bank's settlement file and delivered over SFTP at the bank's cutoff time/timezone. Inclusion in a confirmed batch transitions a `payment` to `SETTLED`; a `refund` settles in the opposite direction and, on inclusion, its original `payment` transitions to `REFUNDED` only once the cumulative refunded amount reaches the `payment`'s full amount (FR-O8) — otherwise it remains `SETTLED` (partially refunded; further `refund`s still possible; see open question below on whether a distinct partially-refunded state is needed instead).
- **FR-S2** — The `void` window for a transaction closes when its settlement batch generation starts (per-bank cutoff); a `void` arriving later is rejected with `REJECTED_INVALID_STATE` (a `refund` is the correct operation past that point, FR-O8).

## 10. Out of scope (v1)

Hosted checkout; APMs (PayPal, etc.); fraud screening; multi-capture; multi-currency settlement per vendor account; 3DS / redirect challenge flows **for APMs** (cards are covered in v1 via `authenticate` + `threeds-service`, §3–§7); **`deferred` (capture) and `cancel`** — descoped, to be reintroduced with different behavior than previously specified (see `Resolved`; `void`, FR-O6, now covers releasing an unsettled `authorise`). See `ARCHITECTURE.md` §Planned Extensions.

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

1. **Peak factor (NFR-1):** 10× average is an assumption; adjust if traffic is spikier (e.g. retail events).
2. **`authenticate` reference validity window (FR-O2/FR-O12):** default 15 minutes for how long a successful `authenticate` may still be referenced by an `authorise`, chosen to reflect typical 3DS session validity — an assumption, no value was given; confirm or adjust.
3. **Partially-refunded state (FR-O8/FR-S1):** a `payment` with `refund`s still below its full amount is modeled as staying `SETTLED` (no distinct state) — only the fully-refunded case transitions to `REFUNDED`. Confirm, or introduce a `PARTIALLY_REFUNDED` state if vendors/reporting need to distinguish it from a not-yet-refunded `SETTLED` transaction.
4. **Race between concurrent `authorise`/`refund` requests against the same ceiling:** the cumulative-amount checks (FR-O2 against `authenticate`; FR-O8 against `payment`) must be atomic per referenced transaction to avoid two concurrent requests both passing a check that, combined, exceeds the ceiling. This is a per-service implementation concern (`threeds-service` for `authorise`/`authenticate`; `card-transactions-service` for `refund`/`payment`) rather than a business-rule question, but is flagged here since it wasn't previously a requirement.

## Resolved

- **Wallet decryption split (FR-M2/FR-M3, 2026-07-13):** confirmed that `applepay-service` decrypts payloads itself (no real PAN ever possible, per Apple's wallet spec) while `googlepay-service` always routes through `card-vault-service` regardless of the vendor's configured auth method, since Google Pay's `PAN_ONLY` mode can carry the real PAN. `ARCHITECTURE.md` updated accordingly (`applepay-service`, `googlepay-service`, `card-vault-service`, `vendor-service`, PCI DSS Compliance sections).
- **`authenticate` = 3DS authentication, with challenge support (FR-O1/FR-O12, 2026-07-13):** confirmed `authenticate` performs EMV 3DS cardholder authentication via a new `threeds-service`, with full challenge-redirect support in v1 (not deferred, unlike the original `ARCHITECTURE.md` Planned Extensions note). Adds the `CHALLENGE_REQUIRED` state, `threeDsAuthenticationId` / `challengeUrl` / `returnUrl` fields, a public challenge-callback endpoint (FR-A9), and puts `threeds-service` in the PCI CDE (detokenizes to route the AReq to the card scheme's Directory Server). `ARCHITECTURE.md` updated (`threeds-service` section, CDE list, Planned Extensions, Payment Flow, Repository Layout, `bank-simulator-service`).
- **`deferred` and `cancel` descoped from v1 (former FR-O4/FR-O5, 2026-07-13):** removed from this pass — will be reintroduced later with different behavior than previously specified. The identifiers `FR-O4` and `FR-O5` are retired and must not be reused.
- **`authorise` always requires a prior `authenticate`; `payment` bundles both; `void` releases `authorise`; `refund` added (FR-O1–FR-O3, FR-O6, FR-O8, FR-O12, 2026-07-13):** for `CARD` only (`APPLE_PAY`/`GOOGLE_PAY` exempt — the wallet's own device-level cardholder verification already covers this, confirmed explicitly). `authorise` now mandatorily references a prior successful, unexpired `authenticate` transaction and never runs 3DS itself. `payment` performs 3DS + authorization + capture as one transaction (new `THREEDS_AUTHENTICATED` state), not by referencing a separate `authenticate`. This resolves the `authorise` dead-end concern raised previously: `void` (FR-O6) now also releases an unsettled `AUTHORIZED` `authorise`, taking over `cancel`'s old role. `refund` (FR-O8, repurposing the FR-O8 slot previously marking refunds out of scope) reverses an already-`SETTLED` `payment`, symmetric to `void` but post-settlement. New terminal state `REFUNDED` (`SETTLED` → `REFUNDED`). `SETTLED` is hereby confirmed as an intentional, real state, resolving the earlier open question about it; `EXPIRED` remains as previously defined. `ARCHITECTURE.md` updated to match (gateway operation list, state machine, Payment Flow, Planned Extensions).
- **`authenticate` carries an amount (cumulative ceiling); `authorise` and `refund` become multi-reference, cumulative-capped; `void` stays single-shot (FR-O1/FR-O2/FR-O6/FR-O8/FR-O12, 2026-07-13):** `authenticate` now carries `amount`/`currency` (reversing the earlier "no amount" assumption) — this is the maximum cumulative amount available to the `authorise`(s) that reference it, not a fund reservation itself. Multiple `authorise` transactions may reference the same `authenticate` as long as their cumulative amount doesn't exceed it (reversing the earlier "single-use" rule); same currency required between them. Voiding an `authorise` frees its amount back to the `authenticate`'s remaining ceiling (confirmed explicitly — `threeds-service` must therefore also consume `void` events, not just `authorise`-creation events). Symmetrically, `refund` (FR-O8) now allows partial amounts and multiple `refund`s against the same `payment`, capped at its cumulative amount (reversing the earlier "full-amount-only" assumption) — `void` (FR-O6) explicitly does **not** follow this pattern: full amount only, at most one per transaction. Only successful (`AUTHORIZED`) attempts count toward either cumulative total. `ARCHITECTURE.md` updated (`threeds-service` now tracks/owns the `authenticate` ceiling including `void` consumption, `card-transactions-service` tracks the `payment`/`refund` ceiling, Payment Flow, state diagrams).
