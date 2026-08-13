# KyiPasses agent guide

KyiPasses is a Singapore climbing-pass exchange app. The product requirements, MVP scope, API outline, roadmap, and domain decisions live in [README.md](README.md). Treat the README as the product/spec reference and update it when product behavior changes.

## Repository layout

- `frontend/` — Next.js web application. Read [frontend/AGENTS.md](frontend/AGENTS.md) before frontend work.
- `backend/` — Java 21/Spring Boot API and Flyway migrations. Read [backend/AGENTS.md](backend/AGENTS.md) before backend work.

## Agent workflow

- Keep frontend and backend independently runnable.
- Read the relevant directory guide before making changes.
- Keep API DTOs, routes, validation, and behavior aligned with the README and with each other.
- Backend owns authorization, validation, timestamps, expiry, and state transitions; frontend treats those values as authoritative.
- Keep changes small and reviewable. Update README/API documentation when behavior changes.
- Add tests for new behavior and run the relevant lint, typecheck, build, compilation, migration, unit, and integration checks before handoff.

## Safety and privacy

- Username-only auth is development-only. Do not present it as production security.
- Do not commit credentials, secrets, personal data, or real user information.
- Keep personal contact information private by default.
- Do not add payment processing, pass verification claims, or exchange guarantees without an explicit product decision.

## Shared engineering rules

- Use `SELL` and `BUY` for API/database values; use “Selling” and “Buying” in the UI.
- Store money as integer cents with an explicit currency such as `SGD`; never use floating point for prices.
- Use UTC timestamps and ISO-8601 API values.
- Use explicit API DTOs rather than exposing persistence entities.
- Validate ownership and state on every mutating endpoint.
- Use optimistic locking or an equivalent transaction strategy for quantity changes and future reservations.
- Never allow listing quantities to become negative.
