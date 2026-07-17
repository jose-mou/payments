# Architecture

## Overview

Payments App is a payments management platform built as a set of event-driven microservices. Vendors (merchants) call a single public REST API: a thin tokenizing proxy (`pan-proxy-service`) fronts the **gateway** so raw card data never enters the core platform; behind it, method-specific and bank-specific services process each operation asynchronously through Kafka. Each microservice is an independent Spring Boot application living in its own top-level directory and owning its own PostgreSQL database.

```
┌─────────┐  REST + JWT   ┌───────────────────┐  tokenized   ┌─────────────────┐
│ Vendors │ ────────────► │ pan-proxy-service │ ───────────► │ gateway-service │ ◄── status polling (202 + Location)
└─────────┘   (PAN/CVV)   └───────────────────┘   request    └────────┬────────┘
                                                               outbox │ (Debezium relay)
                                                                      ▼
      ┌───────────────── Kafka (token + masked data only) ───────────────────┐
      │                                                                      │
      ▼                                                                      │
┌───────────────────────────┐   ┌────────────────────────┐   ┌────────────────────────────┐
│ card-transactions-service │ ─►│ tid-assignment-service │ ─►│ <acquirer-bank>-tx-service │ ─► bank API
└───────────────────────────┘   └────────────────────────┘   │ (one per acquirer)         │    (simulator in dev)
      │ rejection                                            └─────────────┬──────────────┘
      ▼                                                                    │ result
┌──────────────────────┐                                                   ▼
│ notification-service │ ◄──────────────────── result events ──────────────┴──► gateway-service (state update)
└──────────────────────┘                                                   └──► tid-assignment-service (TID release)

CARD only: card-transactions-service also publishes to / consumes from threeds-service — to
check the `authenticate` reference (cumulative ceiling, also consuming `void` to free it back up)
before an `authorise`, or to run payment's embedded 3DS step — before continuing to
tid-assignment-service (see threeds-service, card-transactions-service).

┌────────────────────┐  tokenize ◄── pan-proxy-service │ detokenize ◄── <acquirer-bank>-tx-service,
│ card-vault-service │  sole store of encrypted PAN; CVV transient (short TTL)      threeds-service
└────────────────────┘  Google Pay payload decryption (Google certs) — always, any auth method

┌──────────────────┐  ┌───────────────────┐  applepay: decrypts itself (own cert), DPAN never
│ applepay-service │  │ googlepay-service │  is real PAN. googlepay: never decrypts, always
└──────────────────┘  └───────────────────┘  via vault. Both join the card rail at
                                              card-transactions-service (dev: *-simulator-service)
                                              — no 3DS involved for these methods.

┌─────────────────┐  EMV 3DS for CARD only: standalone `authenticate` (owns its cumulative
│ threeds-service │  ceiling — multiple authorise/void events adjust it), and payment's
└─────────────────┘  embedded 3DS step. Detokenizes via card-vault-service (last hop, PCI
                      scope) to route the AReq to the scheme's Directory Server.
                      Frictionless → continue/terminal; challenge → redirect payer, callback resumes.

┌────────────────────┐  ┌──────────────────────────┐  network tokens (VTS/MDES) provisioned
│ visa-token-service │  │ mastercard-token-service │  after first authorization, used for
└────────────────────┘  └──────────────────────────┘  stored-credential operations

┌────────────────┐
│ vendor-service │  vendor onboarding & configuration (replicated via events)
└────────────────┘

┌───────────────────┐  all lifecycle events   ┌───────────────────────────┐
│ reporting-service │ ◄─────── Kafka          │ Aurora PostgreSQL (hist.) │ ◄── backoffice
│                   │ ──────────────────────► │ + OpenSearch (search)     │     (vendor UI)
└───────────────────┘                         └───────────────────────────┘

┌────────────────────┐  money-moving events        daily settlement files
│ settlement-service │ ◄─────── Kafka          ──────────────────────────► bank SFTP
└────────────────────┘                              (per-bank cutoff)
```

## Architectural Principles

1. **Event-driven communication.** Services communicate asynchronously by publishing and consuming domain events on Kafka whenever possible. Synchronous REST calls between services are an exception that must be explicitly justified.
2. **Transactional outbox.** State changes and their events are persisted in the same database transaction; a Debezium relay publishes the outbox to Kafka. Events are never lost or published without their state change.
3. **Database per service.** Each microservice owns its data and schema. Data needed from other services is obtained by consuming their events (e.g. the gateway keeps a local replica of vendor configuration).
4. **DDD + Hexagonal architecture.** Each microservice maps to a bounded context with `domain` / `application` / `infrastructure` layers; dependencies point inward only.
5. **Asynchronous API.** The gateway accepts an operation, persists it, and responds immediately with `202 Accepted` and a status URL. Results are delivered through status polling and vendor notifications.

## Microservices

### pan-proxy-service

Thin tokenizing reverse proxy in front of the gateway — the only public component that sees raw cardholder data.

- Fronts the vendor-facing API: requests carrying PAN/CVV pass through it in the same single call (no extra step for vendors).
- Extracts PAN/CVV from the payload, exchanges them for a token via `card-vault-service`, and forwards the request to the gateway carrying only token + masked data (BIN, last4, expiry).
- Deliberately dumb: parses, substitutes fields, forwards. No business logic and a minimal change rate, keeping the audited surface small and stable. Requests without card data pass through untouched.

### card-vault-service

Sole custodian of raw card data.

- Stores the PAN encrypted with HSM/KMS-managed keys and issues the token used everywhere else in the platform.
- Holds the CVV only transiently (encrypted, short TTL, pre-authorization) and deletes it irrecoverably once the authorization completes. The CVV is never persisted anywhere else.
- Decrypts Google Pay encrypted payloads — it holds the Google wallet payment-processing certificates — and tokenizes whatever it decrypts, whether that is a device network token (DPAN, `CRYPTOGRAM_3DS` auth method) or the real PAN (`PAN_ONLY` auth method): `googlepay-service` always forwards the still-encrypted envelope and never decrypts it itself, so the same code path is used regardless of the vendor's configured auth method — removing any risk of a real PAN bypassing the vault due to a configuration bug. Apple Pay payloads never carry a real PAN (Apple's wallet specification has no `PAN_ONLY` equivalent) and are decrypted directly by `applepay-service`, which holds its own Apple merchant identity certificate; the vault is not involved in that flow.
- Exposes detokenization exclusively to the `<acquirer-bank>-tx-service`, `visa-token-service`, `mastercard-token-service` and `threeds-service` services (mTLS + service allowlist, audited).

### gateway-service

Public entry point of the platform (behind `pan-proxy-service`).

- Exposes the vendor-facing REST API supporting the card operation types: `authenticate`, `authorise`, `payment` (sale), `void`, `refund`. (`cancel` and `deferred` capture are descoped for v1 — see Planned Extensions.)
- Processes **tokenized requests only** — raw PAN/CVV never reach the gateway, which keeps it and everything behind it out of PCI scope.
- Authenticates every request with **JWT**, scoped per vendor — a vendor can only operate on its own transactions.
- Performs **structural validation** (required fields, formats, amounts, currency, supported method, vendor active) and rejects invalid requests synchronously with `400`.
- Persists the transaction and its domain event atomically (outbox + Debezium) and responds `202 Accepted` with a `Location` header pointing to the status endpoint (`GET /api/v1/payments/{id}`).
- Owns the **canonical payment record and state machine**, updated by consuming result events from the processing services.
- Exposes a public, **unauthenticated** 3DS challenge-callback endpoint (`POST /api/v1/payments/{id}/3ds-challenge-result`) — hit by the payer's browser redirected back from the issuer's ACS, not by the vendor — which relays the result to `threeds-service` and then redirects the browser to the vendor's `returnUrl`. Passes through `pan-proxy-service` untouched, like any request without card data.

### vendor-service

Manages the vendors (merchants/providers) registered on the platform: onboarding, credentials, configuration (enabled payment methods, acquirer bank, TID pool size, and — for vendors enabling `GOOGLE_PAY` — the allowed card auth method(s): `CRYPTOGRAM_3DS`, `PAN_ONLY`, or both). Publishes vendor lifecycle events so other services (gateway, tid-assignment-service) can keep local replicas of the configuration they need.

### card-transactions-service

Card processing entry point: direct card payments, card-rail operations handed off by the wallet services (Apple Pay / Google Pay), and payments with stored network tokens.

- Consumes payment events from the gateway (`authorise`, `payment`, `void`, `refund` — not `authenticate`, which `threeds-service` consumes directly).
- Performs **card-specific business validation** (card data rules, operation allowed for the current transaction lifecycle, duplicates).
- For `CARD` `authorise`: after its own validation passes, publishes the operation for `threeds-service` to check the referenced `authenticate` transaction against its remaining cumulative ceiling (same vendor, `AUTHENTICATED`, unexpired, sum of amounts not exceeded); consumes the resulting event and either proceeds to TID assignment or publishes a rejection (`REJECTED_INVALID_STATE`). See `threeds-service`.
- For `CARD` `payment`: after its own validation passes, publishes the operation for `threeds-service` to run the embedded 3DS step; consumes the resulting `THREEDS_AUTHENTICATED` (proceeds to TID assignment) or terminal (`DECLINED`/`FAILED`) event. Not applicable to `APPLE_PAY`/`GOOGLE_PAY` `payment`, which proceed directly to TID assignment.
- For `refund` (`CARD` and wallets): validates the original `payment` is `SETTLED`, belongs to the vendor, is within its **refund eligibility window** (own configuration property, default 180 days — a platform-wide operational parameter, not per-vendor; owned here because refund eligibility is a method-specific business rule, per Validation Strategy), and that this `refund`'s amount plus its `refundedAmount` so far does not exceed the original amount (partial and multiple `refund`s allowed, unlike `void`). Owns and increments `refundedAmount` on the original `payment` itself, analogous to how `threeds-service` owns the `authenticate` ceiling.
- On success, publishes the validated operation for TID assignment.
- On failure, publishes a rejection event consumed by `notification-service` and by `gateway-service` (state update).

### applepay-service

Apple Pay payments logic.

- Consumes Apple Pay operations published by the gateway. The Apple payment token travels **encrypted end-to-end** — `pan-proxy-service` passes it through untouched (no raw PAN in the payload).
- Decrypts the payload itself, holding its own Apple merchant identity certificate. Apple's wallet specification guarantees the decrypted content is never the real PAN — always a device-specific network token (DPAN) plus a single-use cryptogram — so `card-vault-service` is **not involved** in this flow, and the service stays **out of PCI scope** by protocol guarantee rather than by delegation.
- Validates the wallet operation (cryptogram, expiry, amount consistency) and publishes it as a card-rail operation consumed by `card-transactions-service`, carrying the DPAN as the transaction's `cardToken`; from there the standard card flow applies (validation, TID, acquirer).

`applepay-simulator-service` (development only) simulates Apple's payloads and responses in the development environment.

### googlepay-service

Consumes Google Pay operations published by the gateway; the wallet payload passes through `pan-proxy-service` untouched, same as Apple Pay. Unlike `applepay-service`, it **never decrypts the payload itself**: depending on the vendor's configured card auth method (`vendor-service`), the decrypted content may be a device network token (`CRYPTOGRAM_3DS`) or the **real PAN** (`PAN_ONLY`), so `googlepay-service` always forwards the still-encrypted envelope to `card-vault-service`, which decrypts and tokenizes it regardless of which case it turns out to be, returning only a token. This single code path — never branching on vendor configuration — keeps `googlepay-service` **out of PCI scope** and removes any risk of a configuration bug routing a real PAN outside the vault. From there: wallet validation, then hand-off to the card rail carrying the vault token as `cardToken`. `googlepay-simulator-service` (development only) simulates Google's responses.

### threeds-service

EMV 3-D Secure (3DS) cardholder authentication, `CARD` only (`APPLE_PAY`/`GOOGLE_PAY` are exempt — the wallet's own device-level cardholder verification already covers this). Serves three distinct calls, all against the scheme's Directory Server:

- **Standalone `authenticate`**: consumed directly from the gateway (never reaches `card-transactions-service`, `tid-assignment-service` or `<acquirer-bank>-tx-service` — 3DS is resolved at the card scheme/issuer level, not the acquirer bank). Detokenizes the card via `card-vault-service` immediately before sending the authentication request (AReq) — the same last-hop pattern as `<acquirer-bank>-tx-service` and the network token services — which places it **in PCI scope**. On a **frictionless** issuer decision (ARes), publishes the terminal outcome directly (`AUTHENTICATED` / `DECLINED` / `FAILED`). On a **challenge** decision, transitions the transaction to `CHALLENGE_REQUIRED` and publishes a challenge URL for the gateway to expose via polling; consumes the challenge-completion callback (received by the gateway's public, unauthenticated endpoint and relayed here) and publishes the final outcome, or `FAILED` if the challenge window elapses. On success, owns a **running cumulative-ceiling total** for the transaction's `amount` (see next point) — analogous to how `tid-assignment-service` owns a TID pool.
- **`authorise` reference check**: consumes `authorise` operations published by `card-transactions-service` after its own validation. Atomically checks the referenced `authenticate` transaction — same vendor, `AUTHENTICATED`, unexpired, and this `authorise`'s amount plus the running cumulative total already consumed against it does not exceed its own amount (same currency required) — increments the running total on success, and publishes the result for `card-transactions-service` to continue or reject. Also **consumes `void` events** for `authorise` transactions it previously admitted: a successful `void` decrements the running total, freeing that amount for a further `authorise` within the `authenticate`'s validity window (the window is `threeds-service`'s own configuration property — regulation-driven, platform-wide, default 7 calendar days; same ownership pattern as the refund window in `card-transactions-service`). Does not itself talk to the Directory Server for this step (3DS already happened in the standalone `authenticate`).
- **`payment`'s embedded 3DS step**: consumes `CARD` `payment` operations published by `card-transactions-service` after its own validation, before TID assignment. Runs the same AReq/ARes flow as standalone `authenticate` (detokenize, frictionless or challenge), but publishes `THREEDS_AUTHENTICATED` to continue the *same* transaction toward TID assignment, rather than a terminal outcome. No cumulative-ceiling tracking here — `payment` doesn't reference an external `authenticate`.

**One service across all card schemes** (unlike `visa-token-service` / `mastercard-token-service`, which are split because VTS and MDES are genuinely different provisioning APIs): EMV 3DS 2.x defines a common message specification across schemes — only the Directory Server endpoint and per-scheme certificates differ, handled as internal adapters within the service.

`bank-simulator-service` (development only) also simulates the scheme Directory Server / issuer ACS behavior — see below.

### visa-token-service / mastercard-token-service

Network tokenization services (Visa VTS and Mastercard MDES).

- Consume `payment.authorised` events: after a successful first authorization, they provision a **network token** for the card against the network API and store the vault-token ↔ network-token mapping in their own database.
- Publish `network-token.provisioned` events; `card-transactions-service` keeps a local replica and routes subsequent stored-credential operations through the network token (higher approval rates, resilience to card reissuance).
- Provisioning requires the real PAN, so they detokenize against `card-vault-service` at the moment of the network call — the same last-hop pattern as `<acquirer-bank>-tx-service` — which places **both services in PCI scope**.

### tid-assignment-service

Manages the **TID (Terminal ID) pool** for card payments. The number of TIDs per vendor and acquirer bank is limited and determines how many operations that vendor can run in parallel against that bank.

- Consumes validated card operations **before** they reach the acquirer service.
- Allocates a free TID from the vendor/acquirer pool and attaches it to the operation; operations wait when the pool is exhausted (natural throttling).
- Releases the TID when the operation result event is published by the acquirer service.

### \<acquirer-bank\>-tx-service (one service per acquirer bank)

Bank integration adapter (e.g. `redsys-tx-service`, `stripe-tx-service`).

- Consumes TID-assigned operations for its bank.
- Detokenizes the card against `card-vault-service` immediately before calling the bank — the only point in the flow where the real PAN is used.
- Calls the acquirer bank API and handles its protocol specifics (retries, timeouts, certificates).
- Publishes the operation result (`authorised`, `declined`, `failed`), consumed by `notification-service`, `gateway-service`, and `tid-assignment-service`.

Having **one service per acquirer bank** (instead of a single bank-integration service) is deliberate:

- **Runtime isolation**: a degraded bank must not backpressure the others — separate consumer groups, thread pools and timeouts per bank.
- **Independent releases and certifications**: acquirer integrations go through certification cycles; changing bank A's code must not redeploy bank B's certified artifact.
- **Credential isolation**: certificates and secrets are scoped per bank (also a plus in PCI audits).
- **Independent scaling**: each bank has its own volume and TID pool.

Code reuse across adapters comes from the shared `bank-adapter-starter` library (see Shared Libraries), never from merging deployables — each service implements only its bank's protocol adapter.

### notification-service

Notifies the vendor of the final result of each operation (webhook to the vendor's configured callback URL), consuming rejection and result events.

### settlement-service

End-of-day settlement: generates, per acquirer bank, the file with all money-moving operations of the day and delivers it over **SFTP**. A background batch process — no request/response path.

- A **single service for all banks** (the opposite strategy to `<acquirer-bank>-tx-service`, deliberately): the forces that demand per-bank isolation don't apply to a nightly batch — no latency to protect, no noisy neighbor, failures are retried within the settlement window. Per-bank variability (file format, SFTP credentials/endpoint) is handled by **per-bank adapters** behind hexagonal ports (`SettlementFileFormat`, file delivery), so extracting a bank into its own service later stays cheap.
- Builds its **own projection**: consumes the lifecycle events of money-moving operations (sales, refunds, voids) into its own PostgreSQL, modeled for settlement (operations grouped by bank and cutoff window, with batch-inclusion state). It never queries the gateway's active database. Short retention: rows are kept only a few days after batch confirmation — the canonical archive lives in reporting.
- Each bank has its own **cutoff time and timezone**; file generation runs per bank at its cutoff.
- **Pre-generation reconciliation**: before generating a file, aggregate counts and amounts of the projection are compared against the reporting archive (Aurora) for that bank and window. A mismatch **blocks generation and raises an alert** — an incomplete settlement file must never be sent. (Same pattern as the active-database purger: reconcile projections before an irreversible step.)
- **Idempotent regeneration**: the file for a given bank/day is a versioned, traceable artifact that can be regenerated (SFTP failure, bank rejection) without double-settling.
- Publishes `settlement.batch.completed` events, consumed by reporting/backoffice.
- **Post-settlement reconciliation** against the bank's response files (what the bank confirms it settled) closes the loop.
- **PCI note**: the service joins the CDE **only if** some bank's file format requires the full PAN — in that case it detokenizes in batch against `card-vault-service` at generation time (same last-hop pattern as `<acquirer-bank>-tx-service`). With token/masked-PAN formats it stays out of scope. Verify per bank.

### reporting-service

Builds the historical/reporting store by consuming **all** transaction lifecycle events:

- Writes the canonical archive to **Aurora PostgreSQL**: exact aggregates (reconciliation, settlement), transaction detail, exports, and the source for rebuilding OpenSearch indices when mappings change.
- Indexes transactions into **OpenSearch** for search, filtering and listings, using time-based indices managed with ILM (rollover and deletion per retention policy).

### backoffice (vendor UI)

Web application where vendors consult their transactions. Uses OpenSearch for searches, listings and filters, and the Aurora PostgreSQL reporting database for transaction detail, exact reports and exports. Every query is scoped to the authenticated vendor (JWT), same identity model as the gateway API.

### bank-simulator-service (development only)

A **single** simulator service emulating every acquirer bank API for the development environment, with one path prefix per bank (e.g. `/redsys/...`, `/stripe/...`) and per-bank behavior modules. Grouping the simulators — the opposite strategy to the per-bank `<acquirer-bank>-tx-service` services — is deliberate: none of the forces that demand isolation apply here (no PCI scope, no noisy-neighbor impact, no certifications), and a single container keeps the development environment light.

- Supports scenario triggering through **magic data**: specific amounts or test cards force `approved`, `declined`, timeouts or slow responses — simulating failure paths is its main value.
- Also simulates the card scheme **Directory Server / issuer ACS** behavior consumed by `threeds-service`'s dev integration: magic data forces frictionless-approved, frictionless-declined, or challenge-required outcomes, and a simulated challenge page completes the redirect-and-callback flow end to end.
- Shares the bank protocol contracts with the real adapters (see Shared Libraries) so it cannot drift out of sync.
- Never deployed to production.

## Shared Libraries

Cross-cutting code is reused through internal libraries under `libs/`, consumed by services as Gradle dependencies — **reuse lives in libraries, isolation lives in deployment**:

- `bank-adapter-starter` — common machinery for the `<acquirer-bank>-tx-service` services: idempotent Kafka consumption, `card-vault-service` client, transactional outbox publishing, retry/timeout handling and observability.
- `bank-protocol-contracts` — per-bank protocol models shared between each `<acquirer-bank>-tx-service` service and `bank-simulator-service`.

## Payment Flow (cards)

1. The vendor calls the public REST API (JWT) with an operation request; `pan-proxy-service` swaps PAN/CVV for a `card-vault-service` token and forwards the tokenized request to the gateway — one single call from the vendor's perspective.
2. The gateway validates structurally, persists the transaction + outbox event, and responds `202 Accepted` with the status URL.
3. Debezium relays the event to Kafka; `card-transactions-service` consumes it and runs card business validation.
   - **Invalid** → rejection event → `notification-service` notifies the vendor, `gateway-service` marks the transaction as rejected.
4. `tid-assignment-service` allocates a TID for the vendor/acquirer pair.
5. `<acquirer-bank>-tx-service` detokenizes the card via `card-vault-service`, sends the request to the bank and publishes the result.
6. The result event is consumed by `gateway-service` (state update, visible via polling), `notification-service` (vendor webhook), and `tid-assignment-service` (TID release).

**Wallet variant**: Apple Pay / Google Pay operations are first consumed by `applepay-service` / `googlepay-service` (payload decryption — inside the service itself for Apple Pay, delegated to `card-vault-service` for Google Pay — plus wallet validation) and join the card rail at `card-transactions-service` from step 3 onwards, with no 3DS step at all (see 3DS variant below). In step 5, `<acquirer-bank>-tx-service` detokenizes via `card-vault-service` for `CARD` and `GOOGLE_PAY` transactions; for `APPLE_PAY`, the `cardToken` is already the network-issued DPAN and is sent to the bank as-is, no vault call needed.

**Stored-credential variant**: when a network token exists for the card (provisioned by `visa-token-service` / `mastercard-token-service` after the first authorization), `card-transactions-service` routes the operation through the network token instead of the vault token.

**3DS variant (`CARD` only — `APPLE_PAY`/`GOOGLE_PAY` never involve `threeds-service`)**:

- `authenticate` is consumed directly by `threeds-service` from the gateway — it never reaches `card-transactions-service`, `tid-assignment-service` or `<acquirer-bank>-tx-service`, since 3DS is resolved at the card scheme/issuer level, not the card rail. `threeds-service` detokenizes via `card-vault-service` and sends the authentication request to the card scheme's Directory Server. A frictionless result resolves immediately; a challenge result exposes a `challengeUrl` via polling, the vendor redirects the payer's browser there, and the issuer's ACS redirects it back to the gateway's public callback endpoint, which resumes processing. Its `amount` is a cumulative ceiling, not a reservation — see next point.
- `authorise` **always** references a prior successful `authenticate` transaction, and **several `authorise`s may share one `authenticate`**: at step 3, once `card-transactions-service`'s own validation passes, it publishes the operation for `threeds-service` to check the reference against the `authenticate`'s remaining ceiling (cumulative amount, not a one-time reference) before continuing to TID assignment (step 4) — `authorise` never runs 3DS itself. A later `void` on that `authorise` (see below) is also consumed by `threeds-service`, freeing its amount back to the ceiling.
- `payment` performs 3DS **as part of the same transaction**: at step 3, `card-transactions-service` publishes it for `threeds-service` to run the embedded 3DS step (frictionless or challenge, same mechanics as standalone `authenticate`); once `threeds-service` reports success, the same transaction continues to TID assignment (step 4) and onward, unlike `authorise`, which references a separately submitted `authenticate` rather than running 3DS inline.

**Refund variant (`refund`)**: reverses an already-`SETTLED` `payment`, **partially or fully, possibly across multiple `refund`s**. Runs through the same pipeline as any other operation (steps 3–6): `card-transactions-service` validates the original `payment` is `SETTLED`, belongs to the vendor, and that this `refund`'s amount plus its `refundedAmount` so far does not exceed the `payment`'s original amount; `tid-assignment-service` allocates a TID; `<acquirer-bank>-tx-service` sends the reversal to the bank — a genuinely new, separately-settling money movement, unlike `void` (which prevents money from moving in the first place, is full-amount only, and at most once per transaction — no cumulative tracking needed there). On success, `card-transactions-service` increments the original `payment`'s `refundedAmount`; once it reaches the full amount the transaction transitions to `REFUNDED`, otherwise it remains `SETTLED` (confirmed 2026-07-17: no distinct partially-refunded state — `refundedAmount` distinguishes the case).

The active-database purge policy (Data Lifecycle & Scalability) keeps a `SETTLED` `payment` un-purged for as long as it remains within its refund eligibility window, so `refund` always finds the original transaction in the active database — no cross-store read path needed. That window's length (default 180 days) directly trades off against the active database's size; see the open trade-off noted in that section.

### Transaction state machine (owned by gateway-service)

```
authenticate (CARD only, terminal — referenced by authorise, never continues):
REGISTERED → VALIDATED → SENT_TO_THREEDS → AUTHENTICATED | DECLINED | FAILED   (frictionless)
                                │
                                └─► CHALLENGE_REQUIRED → AUTHENTICATED | DECLINED | FAILED (challenge, or timeout)

authorise (CARD: requires a prior authenticate — no 3DS of its own):
REGISTERED → VALIDATED → TID_ASSIGNED → SENT_TO_ACQUIRER → AUTHORISED | DECLINED | FAILED
     └── REJECTED                                               │
                                                                  ├─► VOIDED   (via void)
                                                                  └─► EXPIRED  (window elapsed, not voided)

payment (CARD: 3DS embedded in the same transaction):
REGISTERED → VALIDATED → SENT_TO_THREEDS → THREEDS_AUTHENTICATED → TID_ASSIGNED → SENT_TO_ACQUIRER → AUTHORISED
                                │                                                                        │
                                └─► CHALLENGE_REQUIRED → THREEDS_AUTHENTICATED (continues) | DECLINED | FAILED
AUTHORISED ──► VOIDED    (via void, before settlement cutoff)
AUTHORISED ──► SETTLED   (settlement batch confirmed)
SETTLED  ──► REFUNDED   (via refund, once fully refunded)
```

`void` and `refund` are the follow-up operations (reference the original transaction, only accepted when its state allows it), but differ in cardinality: `void` is full-amount and **at most once** per transaction (`cancel`'s old role, now folded in — see Planned Extensions for the descoped `cancel`/`deferred`); `refund` allows **partial amounts and multiple occurrences** against the same `payment`, capped at its original amount — same asymmetry applies to `authenticate`, which can back **multiple** `authorise`s up to its own amount (voiding one frees its share back). `APPLE_PAY`/`GOOGLE_PAY` `authorise`/`payment` follow the same diagrams with the 3DS-related states simply absent (no `authenticate` reference, no embedding). See the application requirements spec (`doc/specs/application-requirements.md`) for the full state table.

## Validation Strategy

Validation is layered so that fast feedback does not require synchronous coupling:

1. **Structural** — gateway, synchronous `400` (fields, formats, vendor, method).
2. **Method-specific business rules** — owning method service (e.g. `card-transactions-service`), asynchronous; failure is a business outcome (rejection event), not an HTTP error.
3. Every service **re-validates state on consumption** — an earlier validation result may be stale by processing time.

## Communication

### Synchronous (REST)

- Vendor-facing only: submit operations, poll transaction status. JSON over HTTP, versioned under `/api/v1/...`, JWT-authenticated per vendor.

### Asynchronous (Kafka)

- The default mechanism for service-to-service communication.
- Events are past-tense facts (`payment.authorised`), published through the transactional outbox.
- Topic naming: `<service-name>.<entity>.<event>` (e.g. `card-transactions-service.card-payment.rejected`).
- Events are the public contract of a service: versioned, backward compatible.
- Consumers are idempotent — the same event may be delivered more than once.

## Event Payload Strategy

Events carry the **full transaction data** (event-carried state transfer), not just an ID:

- Consumers never call back to the gateway to fetch transaction data — no synchronous coupling, no query storm at 5M transactions/day, and the stream is self-contained (replays, new consumers, rebuilding projections).
- Event schemas are managed in a **schema registry** (Avro or Protobuf) with mandatory backward compatibility — with full payloads, every consumer depends on the event contract.
- An event is an **immutable snapshot** at publication time, never current state — which is why every service re-validates state on consumption.
- Card data in events is restricted to **token + masked fields** (BIN, last4, expiry). Raw PAN and CVV are never published to Kafka (see PCI DSS Compliance).

## PCI DSS Compliance

The cardholder data environment (CDE) is limited to: **`pan-proxy-service`, `card-vault-service`, the `<acquirer-bank>-tx-service` services, `visa-token-service`, `mastercard-token-service` and `threeds-service`**. Everything else — gateway, Kafka, processing services, wallet services, reporting, backoffice — handles tokens and masked data only, keeping it out of PCI scope.

- Raw card data enters the platform only through `pan-proxy-service` and is tokenized before reaching the gateway, in the same API call (no extra steps for vendors).
- PAN at rest exists only in `card-vault-service`, encrypted with HSM/KMS-managed keys.
- CVV is never stored beyond authorization: transient in the vault with a short TTL and irrecoverably deleted once the authorization completes. It is never written to Kafka — topic retention counts as storage under PCI DSS.
- Detokenization happens only at the last hop before an external network — bank adapters, the network token services, and `threeds-service` (provisioning/authentication all require the real PAN) — restricted by mTLS and a service allowlist, and audited.
- **Google Pay** payloads are decrypted **inside** `card-vault-service`, which holds the Google wallet payment-processing certificates; `googlepay-service` never decrypts the payload itself (regardless of the vendor's configured card auth method), which is what keeps it out of scope — see `googlepay-service`.
- **Apple Pay** payloads are decrypted by `applepay-service` itself, **outside** the vault: Apple's wallet specification guarantees the decrypted content is never the real PAN (always a DPAN + cryptogram), so no cardholder data ever reaches the service — it stays out of scope by protocol guarantee, not by delegation. See `applepay-service`.
- `settlement-service` joins the CDE only if a bank's settlement file format requires the full PAN (batch detokenization at file generation); with token/masked formats it stays out of scope.
- Baseline hygiene everywhere: TLS/mTLS in transit, encryption at rest for brokers and databases, and no card data in logs.

## Data Consistency

- Consistency across services is **eventual**, achieved through events rather than distributed transactions.
- Multi-service flows are **choreography-based sagas** with the gateway as the state holder; failure paths publish compensating/rejection events (including TID release).

## Data Lifecycle & Scalability

Target volume: **~5 million transactions per day**. The design separates the hot processing path from historical consultation with two stores:

- **Active database (gateway)** — holds only in-flight transactions and those still within their operational window (eligible for `void`, pending failure analysis, or — for a `SETTLED` `payment` — still within its **refund eligibility window**, `card-transactions-service`'s own configuration property, default 180 days, application requirements spec FR-O8/NFR-7). The transactions table is **partitioned by day**.
- **Historical/reporting store (Aurora PostgreSQL + OpenSearch)** — fed by `reporting-service` from the lifecycle events. This is where the backoffice reads; the active database never serves backoffice queries.

Purge policy for the active database:

1. A background process removes transactions once they are in a **terminal state**, their **operational window has expired** (for a `SETTLED` `payment`, this includes its refund eligibility window — a `refund` must be able to validate against, and increment `refundedAmount` on, the original transaction, so it is never purged before that window closes), and their presence in the historical store has been **verified** (batch reconciliation by ID). If the reporting projection lags, purging stalls automatically instead of losing data.
   - **Cross-service consistency requirement:** the refund eligibility window is `card-transactions-service`'s own configuration; the purge threshold above is `gateway-service`'s own configuration (database-per-service — no shared config). The platform does not automatically keep them in sync: `gateway-service` must always be configured with a purge threshold at least as long as `card-transactions-service`'s refund eligibility window, or `refund` will fail to find rows it should still be able to reverse.
2. Purging is implemented by **dropping day partitions**, not row `DELETE`s — instant and free of vacuum bloat at this volume. The few rows in an old partition still inside their operational window are relocated before the drop.
3. Transactions are therefore **duplicated in both stores during the overlap period** — this is expected; readers are unambiguous (processing reads active, backoffice reads historical).

Additional scaling measures:

- Status polling (`GET /payments/{id}`) scales with **read replicas** of the active database.
- Intermediate services (`card-transactions-service`, `tid-assignment-service`, `<acquirer-bank>-tx-service`) keep only short-retention local data; the gateway and the historical store hold the canonical records.
- Historical retention is policy-driven: card schemes require ~13 months for disputes; regulatory requirements may extend it. OpenSearch indices are rolled over and deleted by ILM; the Aurora archive is partitioned by month.
- **Accepted trade-off (raised 2026-07-13, accepted 2026-07-17):** the refund eligibility window (default 180 days) directly sets how long every settled `payment` stays in the active database, working against the reason the active/historical split exists (keeping the hot, day-partitioned table lean). This cost is accepted at the 180-day default; it should be reconsidered explicitly only if the window ever changes materially (e.g. extended toward the ~13-month scheme dispute window).

## Repository Layout

Each microservice would ideally live in its own repository. For simplicity this project is a monorepo where each top-level directory is an independent, separately buildable and deployable unit (`libs/` is the exception: shared internal libraries consumed as Gradle dependencies, never deployed):

```
payments-app/
├── CLAUDE.md                    # Development practices
├── README.md
├── doc/
│   ├── ARCHITECTURE.md          # This document
│   └── specs/                   # Feature specifications (Spec-Driven Development)
├── pan-proxy-service/                   # PCI scope
├── card-vault-service/                  # PCI scope
├── gateway-service/
├── vendor-service/
├── card-transactions-service/
├── applepay-service/
├── googlepay-service/
├── visa-token-service/          # PCI scope
├── mastercard-token-service/    # PCI scope
├── threeds-service/             # PCI scope
├── tid-assignment-service/
├── <acquirer-bank>-tx-service/          # One per acquirer bank — PCI scope
├── notification-service/
├── settlement-service/          # PCI scope only if a bank file format requires full PAN
├── reporting-service/
├── backoffice/                  # Vendor-facing web application
├── bank-simulator-service/              # Development only — all banks in one service
├── applepay-simulator-service/          # Development only
├── googlepay-simulator-service/         # Development only
└── libs/                        # Shared internal libraries (bank-adapter-starter, bank-protocol-contracts)
```

## Microservice Internal Structure

Every service follows the same hexagonal layout:

```
<service-name>/
└── src/main/java/com/kaleido/software/<service-name>/
    ├── domain/          # Entities, value objects, domain services, domain events
    ├── application/     # Use cases and ports (interfaces)
    └── infrastructure/  # Adapters: REST controllers, Kafka, JPA, Debezium outbox, configuration
```

- **domain** — pure Java, no Spring or persistence annotations. Contains all business rules.
- **application** — orchestrates use cases; defines inbound ports (use case interfaces) and outbound ports (repository, event publisher interfaces).
- **infrastructure** — implements the ports: REST controllers (inbound), Kafka producers/consumers, JPA repositories (outbound), and Spring configuration.

## Testing Strategy

- **Unit tests** (JUnit 6 + Mockito) cover the domain and application layers — written first, following TDD.
- **Integration tests** use Testcontainers to spin up real PostgreSQL and Kafka instances; no dependency on locally installed infrastructure.
- End-to-end flows in development run against `bank-simulator-service`, using its magic-data scenarios to exercise failure paths (declines, timeouts, slow banks).
- CI must pass (build + all tests) before any merge into `main`.

## Planned Extensions

Discussed and intentionally deferred:

- **APM services** (`paypal-payments`, etc.) for redirect-based methods (PayPal, AmazonPay, AliPay) with a payment-only lifecycle.
- **Hosted checkout**: register a transaction first (`REGISTERED` state, no method), then the payer completes it in a platform-hosted UI — keeps vendors out of PCI DSS scope. Requires session expiry and one-time UI tokens.
- **`cancel` and `deferred` (capture)**: removed from v1 (2026-07-13) — to be reintroduced with different behavior than previously specified in this document. `void` now covers releasing an unsettled `authorise` (`cancel`'s old role, 2026-07-13); `deferred` capture has no replacement yet.
- **fraud-screening** service; its position in the chain (blocking pre-auth vs. post-auth analysis) is still to be decided.
- **3DS / redirect challenge flows for APMs**: cards are covered in v1 via `authenticate` + `threeds-service` (see that section); extending the same challenge-redirect mechanism to APMs remains deferred.
- **Internal BI/analytics tier** (e.g. Amazon Redshift) fed from the same Kafka events for long-range trend analysis — never used to serve backoffice queries.
