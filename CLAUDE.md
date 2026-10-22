# CLAUDE.md — NestaCare

Guidance for Claude Code (and human contributors) working in this repository.

## Project

NestaCare is a household services marketplace. Customers discover, compare and book local cleaners, cooks and other household service providers.

**Current phase:** Repository initialization. No application code exists yet.

## Technology Stack

| Layer | Choice |
|---|---|
| Frontend | React, TypeScript, Vite |
| Backend | Java 21, Spring Boot |
| Database | PostgreSQL |
| Architecture | Modular monolith |
| Local development | Docker Compose |
| Cloud (later) | AWS: ECS Fargate, RDS (PostgreSQL), S3, CloudFront, ECR, CloudWatch |
| Infrastructure as code | AWS CDK |
| CI/CD | GitHub Actions |

## Repository Layout

- `docs/` — all design documentation (see workflow below)
- `backend/` — Spring Boot modular monolith
- `frontend/` — React + Vite single-page app
- `infrastructure/` — AWS CDK app
- `.github/workflows/` — CI/CD pipelines
- `docker-compose.yml` — local development services

## Module Boundaries (Backend)

The backend is one deployable unit split into four business modules:

| Module | Responsibility |
|---|---|
| `provider` | Provider profiles, services offered, pricing, availability, onboarding |
| `search` | Discovery, filtering, ranking and comparison of providers |
| `booking` | Booking requests, confirmation, cancellation, booking lifecycle |
| `customer` | Customer accounts, profiles, addresses, booking history views |

Rules:

1. Each module lives in its own top-level package (e.g. `com.nestacare.booking`).
2. A module exposes only a small public API (interfaces, DTOs, events). Everything else is internal.
3. Modules never import another module's internal classes, repositories or entities.
4. Each module owns its own database tables. No module reads or writes another module's tables directly.
5. Cross-module communication happens through public APIs or domain events, never shared mutable state.
6. A shared kernel (common types, error handling, config) is allowed but kept minimal. Business logic does not go there.
7. Module boundaries are enforced by automated tests (see Testing).

## Coding Standards

### Java / Spring Boot
- Java 21 features where they improve clarity (records, sealed types, pattern matching).
- Package by module, then by layer inside the module (`api`, `domain`, `internal`/`infrastructure`).
- Constructor injection only; no field injection.
- DTOs at the API boundary; JPA entities never leave their module.
- Validate input at the boundary (Bean Validation).
- Database schema changes through versioned migrations (e.g. Flyway), never manual edits.
- Consistent error responses through a central exception handler.
- Configuration through environment variables / Spring profiles; no secrets in code.

### TypeScript / React
- TypeScript `strict` mode on; avoid `any`.
- Organize by feature (`features/search`, `features/booking`, ...) with shared UI in `components/`.
- Function components and hooks only.
- Typed API client layer; components do not call `fetch` directly.
- Linting and formatting via ESLint + Prettier.

### General
- Small, focused changes and pull requests.
- Conventional commit messages (`feat:`, `fix:`, `docs:`, `chore:`, ...).
- Never commit secrets, `.env` files or credentials. Use `.env.example` for templates.
- Keep code comments short and useful; explain *why*, not *what*.

## Documentation-First Workflow

Design is written down and approved before code is written.

| Step | Output | Location |
|---|---|---|
| 1. Requirements analysis | Functional and non-functional requirements, user stories, MVP scope | `docs/requirements/` |
| 2. Architecture | System context, module map, deployment view | `docs/architecture/` |
| 3. High-level design | Module responsibilities, APIs, data ownership, key flows | `docs/hld/` |
| 4. Low-level design | Class/package design, DB schema, API contracts, sequence diagrams | `docs/lld/` |
| 5. Architecture decisions | One ADR per significant decision (`NNNN-title.md`) | `docs/adr/` |
| 6. Implementation | Code + tests that follow the approved design | `backend/`, `frontend/`, `infrastructure/` |
| 7. Review | Design and code review notes, findings, follow-ups | `docs/reviews/` |

If implementation needs to diverge from an approved design, update the design document (or add an ADR) first.

## Testing Expectations

- **Backend unit tests:** JUnit 5 + Mockito for domain and service logic in every module.
- **Module boundary tests:** Spring Modulith verification and/or ArchUnit to enforce the rules above.
- **Backend integration tests:** Spring Boot tests against real PostgreSQL via Testcontainers.
- **API tests:** controller/contract tests for each public endpoint.
- **Frontend tests:** Vitest + React Testing Library for components and hooks.
- **CI:** all tests, linting and builds must pass in GitHub Actions before merging.
- New features and bug fixes ship with tests.

## Phase Gates — Approval Required

Work proceeds one phase at a time:

1. Initialization (current)
2. Requirements analysis
3. Architecture and HLD
4. LLD and ADRs
5. Implementation (backend, frontend, local Docker Compose)
6. CI/CD and infrastructure (CDK)
7. AWS deployment

**At the end of each phase, summarize what was done and stop. Ask the user for explicit approval before starting the next phase.**

Without explicit approval, do not:
- install dependencies,
- generate application code ahead of an approved design,
- run `cdk init`, `cdk deploy` or any AWS command,
- create or modify cloud resources,
- push to remote repositories.
