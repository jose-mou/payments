# Event Catalog Specification

- **Status:** Draft — pending review
- **Date:** 2026-07-17
- **Scope:** The concrete Kafka event contracts of the platform: serialization, envelope, canonical payload, topic catalog (producer, trigger, key, consumers), and schema-evolution rules. Derived from `doc/specs/application-requirements.md` (behavior — FR-*/NFR-* referenced throughout) and `doc/ARCHITECTURE.md` (structure). Where this document names a field or state, the requirements spec is the source of truth for its meaning.
- **References:** `doc/specs/application-requirements.md`, `doc/ARCHITECTURE.md`.

## 1. Serialization and schema registry

- **EV-S1** — Events are serialized with **Apache Avro** (decided 2026-07-17, closing the "Avro or Protobuf" question left open in `ARCHITECTURE.md`). Schemas live in a **Confluent-compatible schema registry**; producers and consumers resolve schemas by id from the registry, never from bundled copies.
- **EV-S2** — Subject naming: **TopicNameStrategy** (`<topic>-value`; keys are plain strings, no key schema). One value schema per topic.
- **EV-S3** — Compatibility mode: **`BACKWARD_TRANSITIVE`**, enforced by the registry on every topic (NFR-4). Practical consequences:
  - New fields must be optional with a default (`["null", T]`, default `null`).
  - Fields are never removed or renamed within a topic's lifetime.
  - A genuinely breaking change requires a **new topic** with a `.v2` suffix (e.g. `…card-operation.registered.v2`); old and new run in parallel until all consumers migrate. Version suffixes are omitted for v1 topics (implicit `v1`).
- **EV-S4** — **Enumerated values are Avro `string`, not Avro `enum`**: adding a symbol to an Avro `enum` breaks backward compatibility, while the platform must be able to add operation types, states, and result codes without breaking consumers. The allowed values are documented per field in the requirements spec (FR-D1, FR-R2); consumers must tolerate unknown values (route to a dead-letter/ignore path, never crash).
- **EV-S5** — Logical types: money as `long` **minor units** (FR-O10), currency as ISO 4217 `string`, instants as `timestamp-millis`, ids as `string` with `uuid` logical type.
- **EV-S6** — Events are published exclusively through the **transactional outbox** (Debezium relay, `ARCHITECTURE.md` §Architectural Principles); the outbox row stores the Avro-serialized value and the destination topic. Application code never produces to Kafka directly.

## 2. Envelope

- **EV-E1** — Every event value embeds the same leading envelope fields (NFR-4), followed by the event payload:

| Field | Avro type | Notes |
|---|---|---|
| `eventId` | `string` (uuid) | Unique per event; consumer dedup key (NFR-3) |
| `eventType` | `string` | `<entity>.<event>` (matches the topic suffix) |
| `occurredAt` | `timestamp-millis` | When the fact became true (not publish time) |
| `schemaVersion` | `int` | Producer-declared payload version, monotonic per topic |
| `causationId` | `["null","string"]`, default `null` | `eventId` of the event that caused this one — makes every state transition traceable (FR-L2, NFR-9) |

- **EV-E2** — The envelope is a shared named Avro record (`com.kaleido.software.events.EventEnvelope` fields inlined at the top of each topic schema) maintained in `libs/` so every service compiles against the same definition.

## 3. Canonical transaction snapshot

- **EV-D1** — Transaction lifecycle events carry the **full canonical snapshot** (event-carried state transfer, FR-D1 — never just an id). Avro record `TransactionSnapshot`:

| Field | Avro type | FR-D1 notes |
|---|---|---|
| `transactionId` | `string` (uuid) | Partition key of all lifecycle events (FR-D3) |
| `operationType` | `string` | `AUTHENTICATE` \| `AUTHORISE` \| `PAYMENT` \| `VOID` \| `REFUND` |
| `originalTransactionId` | `["null","string"]` | Follow-ups only |
| `vendorId` | `string` | |
| `vendorReference` | `string` | ≤ 64 chars, opaque |
| `paymentMethod` | `string` | `CARD` \| `APPLE_PAY` \| `GOOGLE_PAY` |
| `amount` | `long` | Minor units |
| `currency` | `string` | ISO 4217 alphabetic |
| `cardToken` | `["null","string"]` | Vault token, or DPAN for `APPLE_PAY` (FR-D2) |
| `maskedCard` | `["null", MaskedCard]` | Record: `bin`, `last4`, `expiryMonth` (int), `expiryYear` (int), `scheme` |
| `networkTokenUsed` | `boolean` | |
| `acquirerId` | `["null","string"]` | Absent for `authenticate` |
| `tid` | `["null","string"]` | Present from `TID_ASSIGNED` onward |
| `refundedAmount` | `long`, default `0` | `payment` (and placeholder on `authorise`) |
| `threeDsAuthenticationId` | `["null","string"]` | `CARD` `authorise` only |
| `challengeUrl` | `["null","string"]` | Only while `CHALLENGE_REQUIRED` |
| `returnUrl` | `["null","string"]` | `authenticate` / `CARD` `payment` |
| `hmacKeyVersion` | `["null","string"]` | Master-key version for the derived `notificationSecret` (NFR-12); the secret itself never appears in any event |
| `state` | `string` | Lifecycle state (§5 of requirements) |
| `result` | `["null", Result]` | Record: `platformCode` (`string`, FR-R2), `acquirerCode` (`["null","string"]`) |
| `createdAt`, `updatedAt` | `timestamp-millis` | UTC |

- **EV-D2** — An event is an **immutable snapshot at publication time**, never current state; every consumer re-validates on consumption (FR-L4).
- **EV-D3** — **PCI constraints on every schema** (FR-D2, NFR-6): no raw PAN, no CVV, no decrypted wallet payloads beyond DPAN + cryptogram (`APPLE_PAY` only). Wallet hand-off events may carry a `walletData` record (`eci`, single-use `cryptogram`) — permitted by FR-D2. No secret material ever appears in any event: webhook/redirect signing uses per-transaction **derived** secrets (NFR-12), so there is nothing to distribute — only the non-secret `hmacKeyVersion` travels in the snapshot.

## 4. Topic catalog

Topic naming `<service-name>.<entity>.<event>`, past-tense facts. Key = `transactionId` and payload = envelope + `TransactionSnapshot` (+ listed extras) unless stated otherwise. `notification-service` additionally consumes every terminal-outcome and `AUTHORISED` event for the vendor webhook (FR-N1); `reporting-service` consumes **all** topics below. Only routing-relevant consumers are listed.

### gateway-service

| Topic | Trigger | Consumers |
|---|---|---|
| `gateway-service.card-operation.registered` | `CARD` `authorise`/`payment`, and **all** `void`/`refund` (any method — follow-ups carry no wallet payload, FR-O11) accepted (`202`) | `card-transactions-service` |
| `gateway-service.threeds-authentication.registered` | `authenticate` accepted — never enters the card rail (FR-O1) | `threeds-service` |
| `gateway-service.applepay-operation.registered` | `APPLE_PAY` `authorise`/`payment` accepted. Extra: encrypted Apple payload | `applepay-service` |
| `gateway-service.googlepay-operation.registered` | `GOOGLE_PAY` `authorise`/`payment` accepted. Extra: still-encrypted Google envelope (FR-M3) | `googlepay-service` |
| `gateway-service.threeds-challenge.callback-received` | Public challenge callback hit (FR-A9). Payload: envelope + `transactionId`, correlation token, CRes data, `receivedAt` — no snapshot | `threeds-service` |
| `gateway-service.transaction.expired` | Operational-window scheduler: `AUTHORISED` `authorise` neither voided nor resolved in window (FR-O2) | `notification-service` |

### applepay-service / googlepay-service

| Topic | Trigger | Consumers |
|---|---|---|
| `<wallet>-service.wallet-operation.validated` | Wallet payload decrypted (Apple: in-service; Google: via vault) and wallet validation passed; joins the card rail. `cardToken` = DPAN (`APPLE_PAY`) or vault token (`GOOGLE_PAY`); extra `walletData` (EV-D3) | `card-transactions-service` |
| `<wallet>-service.wallet-operation.rejected` | Wallet validation failed (`REJECTED_*`) | `gateway-service`, `notification-service` |

### card-transactions-service

| Topic | Trigger | Consumers |
|---|---|---|
| `card-transactions-service.threeds-check.requested` | Own validation passed for `CARD` `authorise` (reference check, FR-O12) or `CARD` `payment` (embedded 3DS step, FR-O3) | `threeds-service` |
| `card-transactions-service.card-operation.validated` | Operation cleared card-rail validation (and 3DS steps where applicable); ready for TID | `tid-assignment-service` |
| `card-transactions-service.card-operation.rejected` | Business validation failed (`REJECTED_*`, includes `REJECTED_INVALID_STATE` relayed from a failed reference check) | `gateway-service`, `notification-service` |
| `card-transactions-service.card-payment.refund-recorded` | Successful (`AUTHORISED`) `refund` consumed from acquirer results: `refundedAmount` incremented on the original `payment` (FR-O8, NFR-11). **Key: `originalTransactionId`** — serializes increments per payment. Snapshot = the original `payment` with updated `refundedAmount` | `gateway-service` (drives `SETTLED` → `REFUNDED` when full) |

### threeds-service

| Topic | Trigger | Consumers |
|---|---|---|
| `threeds-service.authentication.authenticated` | Standalone `authenticate` reached `AUTHENTICATED` (frictionless or post-challenge) | `gateway-service`, `notification-service` |
| `threeds-service.authentication.declined` / `.failed` | Terminal negative outcome (`.failed` includes `FAILED_CHALLENGE_TIMEOUT`) | `gateway-service`, `notification-service` |
| `threeds-service.authentication.challenge-required` | Issuer requires challenge (`authenticate` or `CARD` `payment`); snapshot carries `challengeUrl` (FR-A7) | `gateway-service` |
| `threeds-service.payment-authentication.authenticated` | Embedded 3DS step succeeded — `THREEDS_AUTHENTICATED`, same transaction continues (FR-O3) | `card-transactions-service`, `gateway-service` |
| `threeds-service.payment-authentication.declined` / `.failed` | Embedded 3DS step terminal failure | `gateway-service`, `notification-service` |
| `threeds-service.authorise-reference.approved` | Ceiling check passed atomically; running total incremented (FR-O12, NFR-11) | `card-transactions-service` |
| `threeds-service.authorise-reference.rejected` | Ceiling check failed (wrong vendor / not `AUTHENTICATED` / window elapsed / ceiling exceeded); `card-transactions-service` publishes the resulting `card-operation.rejected` | `card-transactions-service` |

### tid-assignment-service

| Topic | Trigger | Consumers |
|---|---|---|
| `tid-assignment-service.tid.assigned` | Free TID allocated from the vendor/acquirer pool; snapshot carries `tid`. Single topic for all banks; each `<acquirer-bank>-tx-service` consumes with its own group filtering on `acquirerId` (confirmed 2026-07-17 — revisit only if bank count or volume grows) | `<acquirer-bank>-tx-service`s, `gateway-service` |

### \<acquirer-bank\>-tx-service (one set of topics per bank)

Schemas shared via `bank-adapter-starter` — identical `acquirer-operation` record across banks; cross-bank consumers subscribe by pattern (`^[a-z0-9-]+-tx-service\.acquirer-operation\.authorised$`, etc.).

| Topic | Trigger | Consumers |
|---|---|---|
| `<bank>-tx-service.acquirer-operation.authorised` | Bank approved the operation (any type) | `gateway-service`, `notification-service`, `tid-assignment-service` (TID release); `visa-token-service`/`mastercard-token-service` (first authorization → provisioning); `card-transactions-service` (`refund` → refund-recorded); `threeds-service` (`void` of an `authorise` → ceiling release, FR-O2) |
| `<bank>-tx-service.acquirer-operation.declined` / `.failed` | Bank declined / technical failure (`.failed` outcomes are flagged for reconciliation, never auto-retried — FR-R4) | `gateway-service`, `notification-service`, `tid-assignment-service` |

### visa-token-service / mastercard-token-service

| Topic | Trigger | Consumers |
|---|---|---|
| `<scheme>-token-service.network-token.provisioned` | Network token provisioned after first authorization (FR-M4). **Key: `cardToken`**. Payload: envelope + `cardToken`, network-token reference, `scheme`, expiry, `status` — no transaction snapshot | `card-transactions-service` (routing replica) |

### vendor-service

Full vendor-configuration snapshot per event (id, status, enabled methods, `acquirerId`, TID pool size, Google Pay auth methods FR-M6, 3DS rule FR-M7, webhook URL — **no secret material** (EV-D3): webhook signing secrets are per-transaction and derived, NFR-12, so vendor configuration holds none). **Key: `vendorId`; topics are log-compacted** so consumers can rebuild replicas from the topic alone.

| Topic | Trigger | Consumers |
|---|---|---|
| `vendor-service.vendor.created` / `.updated` / `.disabled` | Vendor lifecycle / configuration change | `gateway-service`, `tid-assignment-service`, `notification-service`, `card-transactions-service` |

### settlement-service

| Topic | Trigger | Consumers |
|---|---|---|
| `settlement-service.transaction.settled` | Transaction included in a **confirmed** settlement batch (FR-S1); one event per transaction, key `transactionId`. Extra: `batchId` | `gateway-service` (`AUTHORISED` → `SETTLED`) |
| `settlement-service.settlement-batch.completed` | Batch confirmed for a bank/cutoff. **Key: `batchId`.** Payload: envelope + bank, cutoff window, counts, totals, file reference — no per-transaction list (would not scale; per-transaction facts travel on `transaction.settled`) | `reporting-service`, backoffice |

### notification-service

| Topic | Trigger | Consumers |
|---|---|---|
| `notification-service.webhook.delivered` / `.dead-lettered` | Webhook delivery succeeded / retries exhausted (FR-N1). Payload: envelope + `transactionId`, `vendorId`, notified state, attempt count, last HTTP status — no snapshot | `reporting-service` (backoffice dead-letter visibility) |

`reporting-service` and backoffice produce no events.

## 5. Delivery and retention

- **EV-R1** — All lifecycle topics: at-least-once delivery, idempotent consumers deduplicating on `eventId` (NFR-3); partition key `transactionId` guarantees per-transaction ordering (FR-D3) — except where the catalog overrides the key (`refund-recorded` → `originalTransactionId`; token/vendor/batch topics as stated).
- **EV-R2** — Retention: lifecycle topics **7 days** time-based (replay safety margin; long-term archive is `reporting-service`'s job, never Kafka's); `vendor-service.*` topics **compacted** (EV — vendor section); tuning per topic is an operational concern, not a contract change.
- **EV-R3** — Consumer groups are per service (and per bank for the acquirer adapters — runtime isolation, `ARCHITECTURE.md`). A consumer that cannot process an event after bounded retries parks it in `<consumer-service>.<topic>.dlq` and alerts; DLQ events are never silently dropped.

## 6. Open questions for review

None currently — all previously listed questions (1–4) were resolved on 2026-07-17; see `Resolved` below.

## Resolved

- **Open questions 1–4 resolved (2026-07-17):**
  1. **TID routing:** `tid.assigned` stays a **single topic** with per-bank consumer groups filtering on `acquirerId` — simple and sufficient at ~58 tps average; revisit only if bank count or volume grows.
  2. **`refundedAmount` increment timing:** confirmed at refund **`AUTHORISED`** (as this catalog already modeled via `card-payment.refund-recorded`); FR-S1 corrected in the requirements spec — the `refund`'s own settlement is bookkeeping and changes no state, so a fully-refunded `payment` may be `REFUNDED` before its `refund` settles.
  3. **Webhook secret distribution:** the problem disappears — signing secrets are **per-transaction and derived** (requirements FR-N2/FR-A10/NFR-12): generated at registration and returned in the `202` response, re-derived by `notification-service` at signing time from a shared KMS master key. No secret ever travels on Kafka or appears in vendor configuration; the snapshot carries only `hmacKeyVersion`.
  4. **Network-token lifecycle:** deferred to post-v1 — only `.provisioned` in v1; if a network token stops working, `card-transactions-service` falls back to vault-token routing until lifecycle events are specified together with the stored-credential detail.
