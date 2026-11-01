
# NestaCare — Engineering Instructions

## Product
NestaCare is a household services marketplace.

## MVP scope
- Provider discovery by city, category, price
- Provider comparison
- Create and cancel bookings
- View customer bookings and statuses

## Technology
- Java 21, Spring Boot, Spring Modulith
- React, TypeScript, Vite
- PostgreSQL
- Docker, AWS ECS Fargate
- S3, CloudFront, RDS, ECR, CloudWatch
- AWS CDK (TypeScript)
- GitHub Actions (CI/CD)

## Architecture
- Modular monolith for MVP
- Separate provider, search, booking, customer modules
- REST APIs
- Clear module boundaries
- Database transactions for booking consistency
- Prepare for future microservice extraction

## Development rules
1. Requirements before architecture.
2. HLD before LLD.
3. Review designs before implementation.
4. Do not write application code until approval.
5. Document assumptions and tradeoffs.
6. Write unit and integration tests.
7. Never commit secrets.
8. Ask before introducing AWS resources with costs.
9. Prefer simple solutions over premature optimization.
10. Explain changes and affected files.

## AI workflow
For every phase:
- Read existing project documentation.
- Identify missing requirements.
- Propose a plan.
- Create or update documentation.
- Identify risks and open decisions.
- Wait for approval before the next phase.
