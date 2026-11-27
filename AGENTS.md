# AGENTS.md

Booking API: a backend service for reserving resources (rooms, slots) with idempotency, rate limiting and concurrency safety.

## Stack
- Java 21, Spring Boot 3.x, Maven (use the wrapper `./mvnw`)
- PostgreSQL 16, Flyway for migrations
- JUnit 5, Testcontainers (PostgreSQL), AssertJ
- Bucket4j for rate limiting, Micrometer for metrics

## Commands
- Build and run all checks: `./mvnw verify`
- Unit tests only: `./mvnw test`
- Run locally: `docker compose up -d db && ./mvnw spring-boot:run`

`./mvnw verify` must pass before you consider any task done.

## Architecture
- Package by feature: `com.example.booking.<feature>` (e.g. `booking`, `idempotency`, `ratelimit`)
- Layers inside a feature: `web` (controllers, DTOs) -> `service` -> `repository`
- Controllers never access repositories directly.
- Specs live in `specs/`. Read the relevant spec before changing a feature.

## Non-negotiable rules
- Never concatenate SQL strings. Use parameters or Spring Data / JPA queries.
- Every schema change is a new Flyway migration. Never edit an existing migration.
- Never store or log secrets, tokens or full request bodies.
- Rate limiting applies per authenticated user id, never per IP.
- Overlapping bookings must be prevented by the database (exclusion constraint), not only by application code.
- Every new endpoint needs at least one integration test using Testcontainers.
- Do not add a new dependency without asking. Use the libraries listed above.
- Do not change files under `.github/`, `src/main/resources/db/migration/` (existing files) or `pom.xml` unless the task says so.

## Workflow
1. Read the spec in `specs/`. If something is ambiguous, ask instead of guessing.
2. Propose a short plan (list of tasks) and wait for approval.
3. Implement task by task, with tests.
4. Run `./mvnw verify` and report the result honestly, including failures.

## Lessons learned (update this section when the agent makes a mistake)
- (empty for now)
