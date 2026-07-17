# vendor-service Specification

- **Status:** Approved — reviewed 2026-07-17
- **Date:** 2026-07-17
- **Scope:** Per-service specification of `vendor-service`: vendor onboarding, configuration, credentials, and the configuration-replication events other services consume. Behavior constraints come from the application requirements spec (FR-M5–FR-M7, FR-A5); event contracts from the event catalog spec.
- **References:** `doc/ARCHITECTURE.md`, `doc/specs/application-requirements.md`, `doc/specs/event-catalog.md`.

## 1. Responsibility (bounded context)

Owns the **vendor** aggregate: identity, status, credentials, and processing configuration. It is the single writer of vendor data; every other service that needs it (gateway, TID assignment, notification, card rail) keeps a local replica built **exclusively** from this service's events — never by querying it.

Out of scope: transactions (gateway), TID pools themselves (`tid-assignment-service` owns allocation; this service only stores the configured pool size).

## 2. Domain model

Aggregate `Vendor`:

| Field | Type | Rules |
|---|---|---|
| `vendorId` | UUID | Assigned here at onboarding; immutable |
| `name` | string ≤ 128 | Unique, non-empty |
| `status` | `ACTIVE` \| `DISABLED` | New vendors start `ACTIVE`; `DISABLED` vendors fail structural validation platform-wide (FR-A5) |
| `enabledPaymentMethods` | set of `CARD` \| `APPLE_PAY` \| `GOOGLE_PAY` | Non-empty (FR-M5) |
| `acquirerId` | string | Must reference a deployed `<acquirer-bank>-tx-service`; one acquirer per vendor in v1. Changing it is rejected (`409`) while the vendor has non-terminal transactions (decided 2026-07-17, §3); follow-ups always route to the **original transaction's** acquirer (the snapshot carries its `acquirerId`), so settled `payment`s never block the change |
| `tidPoolSize` | int | ≥ 1; consumed by `tid-assignment-service` |
| `googlePayAuthMethods` | set of `CRYPTOGRAM_3DS` \| `PAN_ONLY` | Mandatory and non-empty iff `GOOGLE_PAY` enabled (FR-M6) |
| `threeDsRule` | `{ thresholdMinorUnits: long > 0, operator: LESS_THAN \| GREATER_THAN }` | Optional; only if `CARD` enabled; at most one (FR-M7, FR-O15) |
| `webhookUrl` | URL | `https` only; webhook target for `notification-service` (FR-N1). **No webhook secrets** — signing is per-transaction (NFR-12) |
| `apiCredentials` | see §5 | For vendor JWT authentication (FR-A5) |
| `createdAt`, `updatedAt` | UTC instants | |

## 3. API (operator-facing, `/api/v1/vendors`)

Administrative API used by the platform operator (not by vendors; not exposed through `pan-proxy-service`). Synchronous REST — configuration management is low-volume and needs read-your-writes, so the asynchronous pipeline does not apply.

| Endpoint | Behavior |
|---|---|
| `POST /api/v1/vendors` | Onboard: validates §2 rules, persists vendor + `vendor.created` event (outbox), returns `201` with the vendor (credentials returned **once**, §5) |
| `PUT /api/v1/vendors/{vendorId}` | Full configuration update; persists + `vendor.updated`. `409` on concurrent modification (optimistic locking). An `acquirerId` change additionally requires **zero non-terminal transactions** for the vendor, checked at that moment via a synchronous internal query to `gateway-service` — an explicitly justified exception to event-first (low-volume admin operation needing current truth, which an eventual replica cannot guarantee); otherwise `409` (decided 2026-07-17) |
| `POST /api/v1/vendors/{vendorId}/disable` / `.../enable` | Status change; publishes `vendor.disabled` / `vendor.updated` |
| `GET /api/v1/vendors/{vendorId}`, `GET /api/v1/vendors` | Read; list is paginated |

Validation failures return `400` with the same machine-readable error-list format as the gateway (FR-A2). Operator authentication (decided 2026-07-17): **corporate SSO via OIDC** — staff authenticate against the corporate IdP and this API validates those JWTs, requiring an operator role claim; per-person audit trail on every mutation. The same identity model applies to the operator side of backoffice.

## 4. Events

Producer only — this service consumes nothing. Topics, key (`vendorId`), compaction, and the no-secrets rule are defined in the event catalog (§vendor-service). Every event carries the **full configuration snapshot**, so consumers can rebuild replicas from the compacted topic alone; consumers must treat unknown fields/values tolerantly (EV-S4).

Replica consistency is eventual: a configuration change (e.g. disabling a vendor) takes effect in consumers when the event is processed. In-flight transactions are **not** retroactively affected; each service re-validates against its replica at its own processing time (FR-L4).

## 5. Credentials (vendor JWT, FR-A5)

- Onboarding generates a **client credential** (client id + secret) returned exactly once in the `201` response and stored hashed (bcrypt/argon2) — unrecoverable afterwards; a rotation endpoint (`POST .../credentials/rotate`) issues a new pair and invalidates the old one after a grace period (default 24 h, configurable).
- Vendors exchange the client credential for a short-lived **JWT** (OAuth2 `client_credentials` grant) at a token endpoint exposed by this service through the public edge; the JWT carries `vendorId` as subject and is validated by `gateway-service` against this service's published signing keys (JWKS endpoint, keys rotated).
- Issuance ownership confirmed 2026-07-17: **`vendor-service` issues the vendor JWTs itself** in v1 — self-contained, no external IdP dependency. (The corporate OIDC IdP of §3 authenticates *staff*, not vendors; the two identity planes stay separate.)

## 6. Configuration

| Property | Default | Notes |
|---|---|---|
| `vendor.credential-rotation-grace` | 24 h | Old credential validity after rotation |
| `vendor.jwt-ttl` | 15 min | Issued vendor JWT lifetime |

## 7. Persistence

Own PostgreSQL: `vendor` table (aggregate, JSONB for method-specific config), `outbox` (Debezium-relayed), credential hash storage. Low volume — no partitioning.

## 8. Testing notes

- Domain: validation rules of §2 (TDD, full coverage).
- Integration (Testcontainers): outbox → event publication per mutation; credential lifecycle (issue, authenticate, rotate, grace expiry); replica-rebuild scenario consuming own compacted topic.

## 9. Open questions for review

None currently — all previously listed questions (1–3) were resolved on 2026-07-17; see `Resolved` below.

## Resolved

- **Open questions 1–3 resolved (2026-07-17):**
  1. **Operator authentication:** corporate **SSO via OIDC** for the admin API (and the operator side of backoffice): staff JWTs from the corporate IdP, operator role claim required, per-person auditability (§3).
  2. **Token issuance:** `vendor-service` issues the vendor JWTs itself in v1 (`client_credentials` + JWKS, §5); no external IdP for the vendor identity plane.
  3. **Acquirer migration:** v1 **rejects** an `acquirerId` change (`409`) while the vendor has non-terminal transactions, verified at change time via a justified synchronous internal query to `gateway-service` (§3); follow-ups always route to the original transaction's acquirer, so settled `payment`s never block. Assisted migration (TID-pool drain, dual-acquirer window) is post-v1.
