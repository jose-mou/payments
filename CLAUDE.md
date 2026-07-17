# CLAUDE.md

Guidance for Claude Code (and any contributor) working in this repository.

## Language

- **All documentation, code, comments, commit messages, and identifiers must be written in English.** No exceptions.

## Project Overview

Payments API built as a set of microservices. Each microservice lives in its own top-level directory under the repository root (e.g. `gateway-service/`, `card-vault-service/`). See `doc/ARCHITECTURE.md` for the full architecture, including the service map, payment flows, and the transaction state machine.

**Current state: design phase — no code exists yet.** The repository contains only documentation (`doc/ARCHITECTURE.md`, `doc/specs/`). Work happens in the specs until they are agreed; service directories, Gradle projects, and the commands below come into existence only when implementation starts.

### Document precedence

- `doc/ARCHITECTURE.md` governs **structure** (services, boundaries, communication).
- `doc/specs/application-requirements.md` governs **behavior** (operations, lifecycle, data model). Where they overlap, architecture wins on structure, requirements win on behavior.
- Requirements use stable IDs (`FR-*`, `NFR-*`). Retired IDs are never reused for unrelated requirements. Significant decisions and scope changes are recorded inline with their date (e.g. "2026-07-13").

## Technology Stack

- **Language:** Java 25
- **Framework:** Spring Boot 4.x
- **Build tool:** Gradle (one Gradle project per microservice directory)
- **Database:** PostgreSQL — one database per microservice (no shared schemas)
- **Messaging:** Apache Kafka for asynchronous, event-driven communication
- **API style:** REST (JSON) for synchronous, client-facing endpoints
- **Testing:** JUnit 6, Mockito, Testcontainers (PostgreSQL and Kafka), Spring Boot Test

## Architecture Rules

- **DDD + Hexagonal architecture (ports & adapters)** inside every microservice:
  - `domain/` — entities, value objects, domain services, domain events. No framework dependencies.
  - `application/` — use cases / application services, ports (interfaces).
  - `infrastructure/` — adapters: REST controllers, Kafka producers/consumers, JPA repositories, configuration.
  - Dependencies point inward only: `infrastructure` → `application` → `domain`.
- **Event-driven first:** communication between microservices must go through Kafka events whenever possible. Synchronous service-to-service REST calls are the exception and must be justified.
- **Transactional outbox:** state changes and their events are persisted in the same database transaction; a Debezium relay publishes the outbox to Kafka. Never publish events directly from application code.
- **Event-carried state transfer:** events carry the full transaction snapshot, not just an ID — consumers never call back to fetch data, and every consumer re-validates state on consumption (an event is an immutable snapshot, not current state). Consumers are idempotent.
- Each microservice owns its data. Never access another service's database directly.
- Domain events are the public contract of a service; version them and keep them backward compatible.
- **Asynchronous API:** the gateway responds `202 Accepted` + status `Location`; results arrive via polling and webhooks. `gateway-service` owns the canonical payment record and state machine.
- **PCI boundary:** raw PAN/CVV exist only inside the CDE services (`pan-proxy-service`, `card-vault-service`, `<acquirer-bank>-tx-service`, `visa-token-service`, `mastercard-token-service`, `threeds-service`). Everywhere else — including Kafka payloads and logs — card data is restricted to token + masked fields (BIN, last4, expiry). Never widen this boundary casually; detokenization happens only at the last hop before an external network.
- Shared code lives in `libs/` (internal Gradle libraries, never deployed) — **reuse lives in libraries, isolation lives in deployment**; never merge deployables to share code.

## Development Methodology

- **Spec-Driven Development is mandatory:** every feature starts with a written specification (in `doc/specs/`, in English) describing the expected behavior, API contracts, and events involved. The spec is reviewed and agreed upon before any code is written, and implementation must not deviate from it — if the spec needs to change, update the spec first.
- **TDD is mandatory:** write a failing test first, then the minimal code to make it pass, then refactor. Business logic in the domain layer must be fully covered by unit tests.
- **Trunk-based development:** short-lived branches off `main`, merged via small pull requests. `main` must always be releasable.
- All PRs require passing CI (build + tests) before merge.
- Integration tests use Testcontainers; never depend on locally installed databases or brokers.

## Conventions

- Package root: `com.kaleido.software.<service-name>`
- REST endpoints are versioned: `/api/v1/...`
- Kafka topics: `<service-name>.<entity>.<event>` (e.g. `payment-service.payment.completed`)
- One microservice = one directory = one deployable Spring Boot application.

## Common Commands

Run from inside a microservice directory (once services exist — see Current State above):

```bash
./gradlew build         # build + all tests
./gradlew bootRun       # run the service locally
./gradlew test          # unit tests only
./gradlew test --tests "com.kaleido.software.<service>.SomeTest"   # single test class
```
