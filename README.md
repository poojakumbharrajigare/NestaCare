# NestaCare

NestaCare is a household services marketplace. Customers discover, compare and book local cleaners, cooks and other household service providers.

> **Status:** Repository initialized. Requirements analysis is the next phase. No application code yet.

## MVP Features

- **Provider profiles** — providers list their services, pricing, service area and availability.
- **Search and discovery** — customers find providers by service type, location, price and rating.
- **Compare providers** — side-by-side view of services, prices and ratings.
- **Booking** — customers request a booking; providers confirm or decline; either side can cancel.
- **Customer accounts** — profile, saved addresses and booking history.

## Technical Stack

| Layer | Technology |
|---|---|
| Frontend | React, TypeScript, Vite |
| Backend | Java 21, Spring Boot |
| Database | PostgreSQL |
| Local development | Docker Compose |
| Cloud hosting (planned) | AWS ECS Fargate, RDS, S3, CloudFront, ECR, CloudWatch |
| Infrastructure as code | AWS CDK |
| CI/CD | GitHub Actions |

## Planned Architecture

The backend is a **modular monolith**: one Spring Boot application split into modules with strict boundaries. Each module owns its data and exposes a small public API.

```
                    ┌──────────────────────────┐
  Browser ──────────▶  React SPA (Vite build)  │
                    └────────────┬─────────────┘
                                 │ REST / JSON
                    ┌────────────▼─────────────────────────────┐
                    │        Spring Boot modular monolith       │
                    │  ┌──────────┐ ┌────────┐ ┌─────────┐ ┌──────────┐
                    │  │ provider │ │ search │ │ booking │ │ customer │
                    │  └──────────┘ └────────┘ └─────────┘ └──────────┘
                    └────────────┬─────────────────────────────┘
                                 │
                         ┌───────▼───────┐
                         │  PostgreSQL   │
                         └───────────────┘
```

| Module | Responsibility |
|---|---|
| `provider` | Provider profiles, services, pricing, availability |
| `search` | Discovery, filtering, ranking, comparison |
| `booking` | Booking requests and lifecycle |
| `customer` | Customer accounts, addresses, booking history |

**Planned AWS deployment:**

- Frontend static build on **S3**, served through **CloudFront**
- Backend container image in **ECR**, running on **ECS Fargate**
- **RDS for PostgreSQL** as the database
- Logs and metrics in **CloudWatch**
- All infrastructure defined with **AWS CDK**, deployed by **GitHub Actions**

## Repository Structure

```
nestacare/
├── CLAUDE.md               # Guidance for Claude Code and contributors
├── README.md
├── docs/
│   ├── requirements/       # Requirements, user stories, MVP scope
│   ├── architecture/       # System context and architecture views
│   ├── hld/                # High-level design
│   ├── lld/                # Low-level design
│   ├── reviews/            # Design and code review notes
│   └── adr/                # Architecture decision records
├── backend/                # Spring Boot modular monolith (Java 21)
├── frontend/               # React + TypeScript + Vite app
├── infrastructure/         # AWS CDK app
├── .github/
│   └── workflows/          # GitHub Actions pipelines
└── docker-compose.yml      # Local development services
```

## Development Workflow

NestaCare follows a documentation-first process: requirements → architecture → HLD → LLD/ADRs → implementation → review. See [CLAUDE.md](CLAUDE.md) for details, coding standards and testing expectations.
