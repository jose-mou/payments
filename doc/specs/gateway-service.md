# gateway-service Specification

- **Status:** Draft — pending review
- **Date:** 2026-07-17
- **Scope:** Per-service specification of `gateway-service`: the public API surface (behind `pan-proxy-service`), structural validation, the canonical transaction record and state machine, event production/consumption, the notification-secret derivation, and the active-database lifecycle. Behavior constraints come from the application requirements spec (§4–§8, FR-A*, FR-L*, FR-D*, NFR-*); event contracts from the event catalog spec.
- **References:** `doc/ARCHITECTURE.md`, `doc/specs/application-requirements.md`, `doc/specs/event-catalog.md`.

## 1. Responsibility (bounded context)

Public entry point and **owner of the canonical payment record and state machine** (§5 of the requirements). Accepts tokenized operation requests, validates structurally, persists transaction + event atomically (outbox), and keeps the canonical state updated by consuming result events. It never runs business validation (that belongs to the method services) and never sees raw card data (PCI — tokenized requests only).

## 2. API

All endpoints are versioned under `/api/v1`, JWT-authenticated per vendor (FR-A5, JWKS from `vendor-service`) except where noted. Reached through `pan-proxy-service`.

### 2.1 Submit initial operation — `POST /api/v1/payments`

Creates `AUTHENTICATE`, `AUTHORISE` or `PAYMENT` transactions, discriminated by `operationType`. Follow-ups are **sub-resources of the original transaction** (§2.2) — decided 2026-07-17, aligning with prevailing PSP API practice (Adyen `/payments/{ref}/refunds`, Stripe PaymentIntent actions + Refunds, Braintree per-operation mutations) instead of a single endpoint for all five types. Mandatory `Idempotency-Key` header (FR-A4).

Request body (per-type mandatory fields per FR-O1–FR-O3, FR-A8):

| Field | Required for |
|---|---|
| `operationType` | all — `AUTHENTICATE` \| `AUTHORISE` \| `PAYMENT` |
| `vendorReference` | all |
| `amount`, `currency` | all |
| `paymentMethod` + method payload (card data / wallet token) | all. Card fields arrive already tokenized by `pan-proxy-service` |
| `threeDsAuthenticationId` | `CARD` `AUTHORISE` (FR-O2) |
| `returnUrl` | `AUTHENTICATE`, `CARD` `PAYMENT` (FR-A8) |
| `threeDsPreference` | optional on `AUTHENTICATE` / `CARD` `PAYMENT` (FR-O13) |

Responses:
- `202 Accepted` + `Location: /api/v1/payments/{transactionId}`; body `{ transactionId, notificationSecret }` (FR-A10) — the only transmission of the secret ever.
- `400` structural validation failure: machine-readable error list `[{ field, code, message }]` (FR-A2).
- `401` invalid/expired JWT; `404` foreign/unknown `threeDsAuthenticationId` (FR-O9 — never `403`); `409` idempotency-key reuse with different payload (FR-A4).

### 2.2 Follow-up operations — `POST /api/v1/payments/{transactionId}/voids` · `POST /api/v1/payments/{transactionId}/refunds`

Create a `VOID` / `REFUND` transaction against the original identified in the path — a follow-up without an original is impossible by construction, and FR-O9's `404` on foreign/unknown ids falls out naturally. Follow-ups never carry card data (FR-O11) and pass through `pan-proxy-service` untouched. Mandatory `Idempotency-Key` header.

- `voids`: empty body — full amount, at most once (FR-O6).
- `refunds`: body `{ amount }` — minor units, ≤ remaining unrefunded amount (business-validated downstream, FR-O8); currency is the original's (FR-O10 zero/negative rejected structurally). Multiple refunds are naturally modeled as the growing sub-collection.

Responses: same contract as §2.1 (`202` + `Location` and `{ transactionId, notificationSecret }` for the new follow-up transaction; `400`/`401`/`409`; `404` when `{transactionId}` is unknown or foreign).

Structural validation (synchronous, complete list — anything beyond this is business validation and happens downstream): required fields and formats per type; amount > 0 integer minor units + ISO 4217 currency (FR-O10); method enabled for vendor and vendor `ACTIVE` (FR-M5, from the local vendor replica); referenced transaction exists and belongs to the vendor (FR-O9); `returnUrl`/`webhookUrl` well-formed `https`.

### 2.3 Status polling — `GET /api/v1/payments/{transactionId}`

Returns the canonical snapshot (FR-D1 minus `notificationSecret`, which never appears — FR-A10), including `challengeUrl` while `CHALLENGE_REQUIRED` (FR-A7). Served from read replicas (staleness ≤ ~1 s acceptable, FR-A3). `404` on foreign/unknown ids (FR-O9).

### 2.4 Challenge callback — `POST /api/v1/payments/{transactionId}/3ds-challenge-result` (public, unauthenticated)

Receives the ACS browser redirect (FR-A9). Validates the single-use, time-bound challenge-session correlation token (NFR-10; missing/expired/reused → rejected, state unchanged), publishes `threeds-challenge.callback-received`, and redirects the browser (`303`) to the transaction's `returnUrl` with outcome parameters signed with the transaction's `notificationSecret` (FR-A9/FR-N2).

### 2.5 Internal API — `GET /internal/v1/vendors/{vendorId}/non-terminal-count`

Internal-only (mTLS, service allowlist — never exposed through `pan-proxy-service`): returns the number of non-terminal transactions for a vendor, read from the primary. Sole consumer: `vendor-service`, to gate `acquirerId` changes (its spec §3 — a justified synchronous exception, decided 2026-07-17).

## 3. State machine ownership

Implements §5 of the requirements verbatim (authoritative diagrams there). Transitions are applied by consuming the events listed in §4, guarded by the current state: an event arriving for a transaction whose state does not admit that transition (stale/out-of-order snapshot, FR-L4/EV-D2) is recorded as an anomaly and does **not** change state. Every applied transition stores `occurredAt`, the causing `eventId` (FR-L2, `causationId` chain), and is appended to a `transaction_transition` history table (part of the canonical record and of reporting's feed).

The **`EXPIRED` scheduler** (only state transition originated here): a periodic job transitions `AUTHORISED` `authorise` transactions whose reservation window (default 7 days, per vendor/acquirer — FR-O2) elapsed without `void`, publishing `gateway-service.transaction.expired`.

## 4. Events

Exact contracts in the event catalog (§gateway-service and consumers lists there); summary:

- **Produces:** `card-operation.registered`, `threeds-authentication.registered`, `applepay-operation.registered`, `googlepay-operation.registered` (routing per rail decided at acceptance), `threeds-challenge.callback-received`, `transaction.expired`.
- **Consumes (state updates):** wallet/card rejections; `threeds-service` authentication outcomes, challenge-required, payment-authentication outcomes; `tid.assigned`; acquirer results (pattern subscription); `card-payment.refund-recorded` (drives `refundedAmount` and `SETTLED` → `REFUNDED`); `settlement-service.transaction.settled`; `vendor-service.vendor.*` (local replica for structural validation).

All consumption is idempotent by `eventId` (NFR-3).

## 5. Notification secret (FR-A10 / NFR-12)

At registration the service derives `notificationSecret = HKDF(masterKey[version], transactionId)`, returns it in the `202` body, and stores only `hmacKeyVersion` on the transaction. The same derivation signs the challenge-redirect parameters (§2.3). The master key lives in KMS; only this service and `notification-service` may read it. Rotation: new registrations pick up the newest version; verification of in-flight transactions uses their stored version.

## 6. Persistence and data lifecycle

Own PostgreSQL:

- `transaction` — canonical record (FR-D1), **partitioned by day** (creation date); `transaction_transition` history; `outbox` (Debezium); `idempotency_key` (vendor + key, ≥ 24 h retention, FR-A4); `challenge_session` (correlation tokens, NFR-10); `vendor_replica` (from vendor events).
- **Purge** (requirements NFR-7, `ARCHITECTURE.md` §Data Lifecycle): day partitions are dropped once every row is terminal, outside its operational window (including the refund-eligibility window for `SETTLED` `payment`s), and verified present in reporting (batch reconciliation by id — purge stalls if reporting lags). Rows still in-window are relocated before the drop. The purge threshold for `SETTLED` `payment`s is this service's own configuration and **must be kept ≥ `card-transactions-service`'s refund window by operational discipline** (decided 2026-07-17 — no automatic safeguard in v1).
- Polling reads go to **read replicas**; writes and event consumption to the primary.

## 7. Configuration

| Property | Default | Notes |
|---|---|---|
| `gateway.purge.settled-payment-retention` | 200 days | Must be ≥ refund window (180 d) — operational discipline, NFR-7 |
| `gateway.authorise-reservation-window` | 7 days | Per vendor/acquirer override (FR-O2); drives the `EXPIRED` scheduler |
| `gateway.challenge-window` | 10 min | `CHALLENGE_REQUIRED` timeout input (FR-O1; enforced by `threeds-service`, mirrored here for session TTL) |
| `gateway.idempotency-key-retention` | 48 h | ≥ 24 h (FR-A4) |
| `gateway.hmac-master-key-ref` | — | KMS reference + active version (NFR-12) |

## 8. Testing notes

- Domain: state machine exhaustively unit-tested — every legal transition, every illegal transition rejected, stale-event guards (TDD).
- Integration (Testcontainers PostgreSQL + Kafka): 202 flow with outbox → Kafka assertion; idempotent re-delivery of every consumed event type; idempotency-key replay/conflict; challenge-session single-use; partition-drop purge with reconciliation stall.
- NFR-2 latency budgets asserted in performance tests (p99 500 ms submit, 200 ms polling) — environment-gated, not in CI's blocking suite.

## 9. Open questions for review

1. **Vendor replica staleness at acceptance:** a vendor disabled milliseconds ago may still pass structural validation here (replica lag). Downstream re-validation (FR-L4) catches it, but the vendor receives a `202` for an operation that will be rejected — acceptable, or should submission double-check replica freshness? (v1 position: acceptable — document it.)
2. **`vendorReference` uniqueness:** FR-D1 treats it as opaque; should the platform enforce per-vendor uniqueness (helps vendors' reconciliation, costs an index and a `409` path)? v1 position: not enforced.

## Resolved

- **Endpoint shape (§2.1–§2.2, 2026-07-17):** hybrid, following prevailing PSP practice (Adyen Checkout sub-resource modifications, Stripe PaymentIntent actions + Refunds resource, Braintree per-operation mutations): `POST /api/v1/payments` creates initial operations only (`AUTHENTICATE`/`AUTHORISE`/`PAYMENT`, discriminated by `operationType`), while follow-ups are sub-resources of the original transaction (`POST /api/v1/payments/{id}/voids`, `POST /api/v1/payments/{id}/refunds`) — a follow-up without an original is impossible by construction, FR-O9's `404` falls out naturally, and repeatable partial refunds read as a growing sub-collection. Replaces the earlier single-endpoint-for-all-five-types draft.
