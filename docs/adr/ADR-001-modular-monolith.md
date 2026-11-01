# ADR-001: Use a modular monolith for the NestaCare MVP backend

| Field      | Value                                                                                                                                                                    |
| ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Status     | **Accepted** (2026-10-31)                                                                                                                                                |
| Date       | 2026-10-31                                                                                                                                                               |
| Deciders   | Pooja Kumbhar (approved 2026-10-31)                                                                                                                                      |
| Related    | [Architecture options](../architecture/architecture-options.md), [Requirements](../requirements/requirements.md) (NFR-06, NFR-07, NFR-16, NFR-17, NFR-18, Q-6, R-4, R-7) |
| Supersedes | —                                                                                                                                                                        |

## Context

NestaCare's MVP is a household services marketplace: provider discovery, comparison, and a request-to-book flow with slot holds, expiry and cancellation. The architecture has to meet these constraints:

- **Booking correctness is the top risk (R-4).** A provider must never have two overlapping PENDING/CONFIRMED bookings (BR-05), and neither may a customer (BR-06). This must be enforced by PostgreSQL (NFR-07) and hold under concurrency (EC-23 … EC-29). Booking creation must check provider availability, service area, status and rate at commit time (BR-03, BR-08, BR-09, EC-06, EC-26).
- **Load is modest.** The ceiling is 10k providers, 100k customers, 5k bookings/day and 50 search req/s (NFR-01, NFR-06). Uptime target is 99.5 % (NFR-04).
- **Boundaries must be real and tested** (NFR-16, R-7), and the design must be ready for later microservice extraction (CLAUDE.md).
- **Delivery constraints:** CI ≤ 15 min (NFR-17), local stack in ≤ 30 min (NFR-18), prefer simple solutions, ask before adding AWS cost (CLAUDE.md rules 8, 9).
- Assumed team: 1–5 engineers on one backend.

## Decision

Build the backend as **one Spring Boot (Java 21) application, deployed as one Docker image on one ECS Fargate service, using one PostgreSQL (RDS) instance**, and structure it as a **modular monolith**:

1. **Modules.** Four core modules — `provider`, `search`, `booking`, `customer` — and two supporting modules — `identity` (accounts, credentials, roles, verification, reset) and `notification` (event-driven email). A `shared` kernel holds only value types (IDs, Money, TimeRange, error format, event envelope). This also proposes an answer to Q-6: reviews belong to `booking`, and admin features belong to the module that owns the data.
2. **Public API + internals.** Each module exposes a small public API (Java interfaces and DTOs). Everything else is internal. Other modules use only the public API.
3. **Schema per module.** Each module owns one PostgreSQL schema. No cross-schema foreign keys, joins or views. Modules refer to each other by ID and copy snapshots when history must stay fixed.
4. **No dependency cycles.** `search → provider, booking`; `booking → provider, customer, identity`; `customer, provider → identity`. When `provider` or `customer` need booking data, they define a lookup interface that `booking` implements. Nothing calls `search` or `notification`.
5. **Communication.** Synchronous in-process calls when the caller needs an answer. Domain events through a transactional outbox (Spring Modulith event publication registry) for everything that can happen later: notifications, search read model, auto-decline on suspension.
6. **Transactions.** A transaction writes to one module's schema only. Booking creation may *read* provider and customer data through their APIs inside the booking transaction, and the booking overlap guarantees are PostgreSQL exclusion constraints on the `booking` table alone.
7. **Enforcement.** Spring Modulith module verification (and ArchUnit where needed) plus a migration check for cross-schema references run in CI. Changing a boundary needs an ADR update.

Full module definitions, dependency rules and the extraction path are in [architecture-options.md §7–8](../architecture/architecture-options.md#7-module-boundaries).

### Technology stack

The modular monolith is built and run with the stack below. Each choice was compared with its alternatives in [architecture-options.md §9](../architecture/architecture-options.md#9-technology-choices).

| Area                   | Chosen                                             | Why                                                                                                                                                              | Main alternative rejected, and why                                                                                          |
| ---------------------- | -------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| Frontend               | React, TypeScript, Vite                            | Static SPA on S3 + CloudFront needs no server. Largest ecosystem and hiring pool. Vite keeps the bundle small (NFR-03). Strict TypeScript is required by the DoD | Next.js: server rendering needs a Node.js runtime, which adds cost and a second backend runtime                             |
| Backend                | Java 21, Spring Boot                               | Spring Modulith gives boundary checks and the transactional outbox this ADR relies on. Spring Security covers FR-AUTH and NFR-08. Java 21 is LTS                 | Node.js (NestJS): no modular-monolith tooling, so boundaries and outbox would be hand-built                                 |
| Database               | PostgreSQL on RDS                                  | Exclusion constraints on time ranges enforce no overlapping bookings in the database (NFR-07). Schemas per module. Managed backups and PITR (NFR-05)             | MySQL: no exclusion constraints, so overlap checks would rely on application locking                                        |
| Local development      | Docker Compose                                     | One command for Postgres, Mailpit and the app, using the same Postgres version as RDS (NFR-18)                                                                   | Native installs: version drift and slow onboarding                                                                          |
| Cloud hosting          | ECS Fargate, RDS, S3 + CloudFront, ECR, CloudWatch | Same container as local with no servers to patch. Managed database. Cheap static hosting with a CDN. IAM-based image pulls. Monitoring built in                  | EKS: control plane cost and Kubernetes upkeep for one service. Lambda: JVM cold starts and poor fit for jobs and the outbox |
| Infrastructure as code | AWS CDK in TypeScript (approved, OD-7)             | Typed code with high-level ECS/RDS constructs. CloudFormation manages state                                                                                      | Terraform: a separate language plus its own state storage, and multi-cloud isn't needed                                     |
| CI/CD                  | GitHub Actions                                     | The repo is on GitHub. Checks run on every pull request. OIDC to AWS means no stored AWS keys (CLAUDE.md rule 7)                                                 | AWS CodePipeline: separate from pull requests, with slower feedback                                                         |

## Rationale

| Criterion                 | Why the modular monolith wins for this MVP                                                                                                                                 |
| ------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Reliability / consistency | Overlap constraints, compare-and-set transitions and commit-time validation stay inside one local ACID transaction. That is the simplest design that provably meets NFR-07 |
| Development complexity    | One codebase, one build, IDE-wide refactoring. Module boundaries keep the code understandable as it grows                                                                  |
| Scalability               | Stateless instances scale horizontally behind the ALB. That is well beyond what NFR-06 needs. Search can later use a read replica or be extracted first                    |
| AWS cost                  | ≈ $195–205/month per environment (indicative), the same as a traditional monolith and about half of microservices (≈ $430–520)                                             |
| Testability               | Module-scoped integration tests (`@ApplicationModuleTest`), event scenario tests, Testcontainers concurrency tests, and boundary checks built in                           |
| Deployment complexity     | One image, one pipeline, one Flyway run, simple rollback. Supports "every `main` commit deployable" (NFR-17)                                                               |
| Future migration          | Boundaries, APIs, events and schemas already exist. Extraction swaps an adapter and moves one schema. Wrong boundaries are cheap to fix while still in one repo            |

## Alternatives considered and rejected

### Traditional (layered) monolith — rejected

Same runtime and cost as the chosen option, and slightly faster in the first weeks.

Rejected because:
- It can't meet NFR-16: there are no module boundaries to test.
- Coupling grows unchecked (R-7): shared entities, cross-feature joins, long transactions.
- Extraction later would mean untangling data and transactions first, often a partial rewrite. That contradicts "prepare for future microservice extraction" in CLAUDE.md.
- The modular monolith gives up very little speed and costs nothing extra to run, so there is no tradeoff that favours this option.

### Microservices — rejected for the MVP

Would give independent scaling, deployment and failure isolation per service.

Rejected because:
- **Consistency gets harder exactly where the risk is.** Booking validation would depend on network calls to provider and customer services. Cross-service flows (suspend → auto-decline → notify) would become sagas that need a broker, dead-letter handling and monitoring.
- **It costs more** — roughly 2–2.5× the infrastructure per environment (multiple Fargate services, a database per service, broker, more observability), plus much more engineering and on-call time. That conflicts with CLAUDE.md rules 8 and 9.
- **Delivery targets are at risk:** N pipelines and an end-to-end suite threaten CI ≤ 15 min (NFR-17), and N services threaten 30-minute onboarding (NFR-18).
- **The benefits aren't needed.** The load is small and there is one team.
- **The boundaries aren't settled yet** (Q-6 was open until this ADR). Moving a boundary across services means moving data between databases and versioning APIs.

This option is **deferred, not ruled out**: modules can be extracted one at a time when a trigger in [architecture-options.md §8.1](../architecture/architecture-options.md#81-when-to-extract) applies.

### Variants considered inside the modular monolith

| Variant                                       | Decision         | Reason                                                                                                                          |
| --------------------------------------------- | ---------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| One shared schema, boundaries by package only | Rejected         | Nothing stops cross-module joins, and extraction needs the data split anyway                                                    |
| Separate database per module (in one app)     | Rejected         | Loses the single transaction for booking-creation reads, and adds RDS cost with no MVP benefit                                  |
| Message broker (SNS/SQS) from day one         | Rejected         | The Postgres outbox gives reliable async delivery without new AWS resources. Externalize events only when a module is extracted |
| Separate `admin` and `review` modules         | Rejected for now | Too little logic to justify. Placed in the modules that own the data. Can be split later at low cost                            |
| ArchUnit only, hand-written outbox            | Fallback         | Spring Modulith approved (OD-3); use this only if it turns out unsuitable                                                       |

Rejected technology alternatives (Angular, Vue, Kotlin, .NET, Go, MySQL, DynamoDB, EKS, Lambda, Terraform, Jenkins and others) are listed with reasons in [architecture-options.md §9](../architecture/architecture-options.md#9-technology-choices).

## Consequences

### Positive
- The highest-risk rules (no double booking, one winner for concurrent transitions) are enforced by one database in one transaction.
- Lowest running cost and the simplest operations for the MVP.
- Boundaries are explicit, tested, and cheap to correct while the domain is still being learned.
- There is a documented, step-by-step path to extract services (notification → search → others).

### Negative / tradeoffs accepted
- **Shared failure domain.** A defect or resource spike in one module can affect all of them. Mitigation: ≥ 2 tasks across AZs, separate thread pools for scheduled jobs and event handlers, DB statement timeouts, search load tests.
- **Scale together.** The whole app scales for the busiest path (search). At NFR-06 load this waste is small.
- **One deploy for everything.** A change in one module redeploys all of them. Acceptable for one team with a fast CI.
- **Discipline needed.** Boundary rules add some upfront design to every cross-module feature, and CI checks must be kept strict.
- **Search staleness.** The search list reads an event-fed projection (≤ 60 s stale, as AC-SRCH-06 allows). The profile calendar and booking always check live data.
- **Dependency on Spring Modulith** for boundary checks and the outbox. ArchUnit and a hand-written outbox are the fallback.
- **JVM memory.** Each Fargate task needs about 1 GB, more than Go or Node would. At MVP load this is a small cost.
- **SPA and SEO.** Public provider pages get less SEO benefit than server rendering would give. SEO isn't an MVP goal, and prerendering can be added later.
- **AWS lock-in.** Only `infrastructure/` is AWS-specific. The app is a standard container on standard PostgreSQL.

### Follow-ups
1. ~~Update `requirements.md` Q-6 with the module ownership decided here.~~ Done in requirements v0.2 (2026-10-31).
2. HLD per module in `docs/hld/`, including API operations, event payloads and the auth token style (OD-5).
3. Separate ADRs for: booking concurrency and constraints (EC-23 … EC-29), and deployment topology and cost (Multi-AZ RDS, NAT vs VPC endpoints, staging) — **for cost approval before any AWS resources are created**.

## Review triggers

Revisit this ADR if any of the following happens:
- More than one team works on the backend and release coordination becomes a bottleneck.
- Sustained load passes about 50 % of the NFR-06 ceiling, or search and booking need very different scaling.
- Incidents show one module repeatedly harming booking availability.
- A module needs a different technology (e.g. a search engine for geo search).
