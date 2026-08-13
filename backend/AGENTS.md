# Backend agent guide

## Stack

- Java 21.
- Spring Boot 4.x / Spring Framework 7.x.
- Maven unless the project is explicitly changed to Gradle.
- Spring Web, Spring Validation, Spring Data JDBC or JPA, Spring Security, and Actuator as needed.
- PostgreSQL for local and production-like development.
- Flyway for every schema change and seed migration.

Pin a compatible Spring Boot 4.x patch release in the build file. Do not silently downgrade to Spring Boot 3.x: Java 21 and the requested Spring 4.x generation are project constraints.

## Package boundaries

Organize by feature rather than by technical layer alone:

```text
com.kyipasses
├── auth
├── gym
├── listing
└── chat
```

Each feature may contain its controller, application/service logic, repository, persistence model, and API DTOs. Keep controllers thin and do not expose persistence entities directly.

## Persistence model

The initial schema should cover:

- `users`: id, username (unique), created_at.
- `gyms`: id, name, slug, created_at.
- `gym_branches`: id, gym_id, name, address, latitude, longitude, created_at.
- `user_passes`: id, owner_id, branch_id, pass_type, pass_expires_at nullable, pass_expiry_type (`FIXED_DATE`/`ROLLING_VALIDITY`/`UNKNOWN`), created_at, updated_at. This is an optional private My Passes feature, not required for listing creation.
- `listings`: id, branch_id (the preferred branch for a BUY post), owner_id, type (`SELL`/`BUY`), pass_type, pass_description, quantity_total, quantity_remaining, price_per_pass_cents, currency, pass_expires_at nullable, pass_expiry_type (`FIXED_DATE`/`ROLLING_VALIDITY`/`UNKNOWN`), post_expires_at, status, created_at, updated_at, closed_at, previous_listing_id nullable, repost_count.
- `conversations`: id, listing_id, owner_id, interested_user_id, created_at, updated_at, last_message_at.
- `conversation_participants`: conversation_id, user_id, archived_at nullable, last_read_at nullable. Archive state is per participant, not shared conversation status.
- `messages`: id, conversation_id, sender_id, body, created_at.

Add foreign keys, indexes for branch/type/status/post expiry, pass type, price, and conversation participants, and a uniqueness rule for one conversation per listing/interested user. Use UTC timestamps. Use `BIGINT`/`Long` IDs initially unless there is a strong reason to introduce UUIDs.

For now `pass_type` is a controlled value with only `DAY_PASS`. Keep the schema/API extensible for future pass types rather than accepting arbitrary free text.

## Flyway rules

- Put migrations in `src/main/resources/db/migration`.
- Use names such as `V1__create_core_tables.sql` and `V2__seed_singapore_gyms.sql`.
- Never edit an applied migration; add a new migration.
- Keep seed data deterministic and use stable slugs.
- Prefer database constraints for invariants, with application validation for clear errors.

## Expiration behavior

On creation, set `post_expires_at = created_at + 24 hours` on the server. Active-listing queries must include `status = 'ACTIVE' AND post_expires_at > current time`. A scheduled job should mark overdue posts as `EXPIRED`; this is cleanup and observability, not the only protection. A create-conversation or other future exchange endpoint must re-check the listing in a transaction, while existing conversation access remains allowed after expiry.

## Listing lifecycle and quantity rules

- The owner may edit an active listing. Preserve an audit-friendly `updated_at`; do not silently rewrite its creation or post expiry time.
- `quantity_total` means the amount currently offered/requested by this listing, not the user's total pass inventory. For `SELL`, `quantity_remaining` is available; for `BUY`, it is still requested. Both are scoped to the selected branch/listing and cannot be negative; remaining cannot exceed total.
- Example: a seller owns 10 passes and wants to keep 2, so this listing's total quantity is 8. If 3 are no longer available, the owner may update remaining quantity to 5. There is no separate inventory total across gyms in the MVP.
- A partial sale or agreement may reduce remaining quantity. Until an exchange/reservation workflow is implemented, this is an owner-controlled update and is not proof of settlement. Only the listing owner may update either quantity field.
- A BUY listing's branch is the buyer's preferred branch in the MVP. A future multi-branch request can add multiple acceptable branches.
- Closing is an explicit owner action and sets the listing to `CLOSED`, records `closed_at`, and removes it from active marketplace results. It does not archive chats, block new messages, or delete pass/chat data.
- Post expiry sets the listing to `EXPIRED` for marketplace purposes. It does not block existing participants from opening or messaging the preserved conversation.
- Reposting an expired/closed listing creates a new ID, new `created_at`, and new `post_expires_at`; it must link `previous_listing_id` and be subject to a repeated-repost limit. Never silently extend the old post.
- Do not implement automatic `SOLD_OUT` closure in the MVP. Keep the status/model extensible for it later.

Temporary reservations should use a separate table or reservation record with `quantity`, `expires_at`, and a unique active reservation rule. Reserve and release operations must lock/re-check the listing in a transaction. Do not decrement permanent remaining quantity merely because someone is chatting.

## Search and filtering

Provide a paginated all-gym listing query in addition to branch-scoped queries. Filter by gym, branch, `SELL`/`BUY`, pass type, price-per-pass range, remaining quantity, and post expiry only as an internal active-result constraint. Support newest, cheapest, and soonest-post-expiring sort orders. A future pass-expiry filter may use coarse validity ranges, but exact `pass_expires_at` and `pass_expiry_type` must remain private unless product/privacy requirements change. Use parameterized queries and stable tie-breakers so pagination does not reorder equal results.

Interpret “24 hours” as the lifetime of each post, not a limit on how many posts a user may create. If product testing shows a need for one active post per user/branch/type, add that as a separately documented rule.

## Development authentication

Implement a clearly isolated development authentication mechanism that creates or finds a user by username and establishes a local session. Never present username-only auth as production-ready. Before launch, replace it with a real identity flow, rate limits, account abuse controls, and privacy controls.

## API and error handling

- Validate request bodies with Bean Validation.
- Return consistent error responses with a machine-readable code and human-readable message.
- Use `404` for resources the current user should not be able to discover when appropriate; do not leak private conversation existence.
- Use `409` for invalid state transitions and duplicate conversation creation races.
- Centralize exception handling and date/time serialization.
- Apply pagination to listings, conversations, and messages before the dataset grows.
- A quick-interest action should carry structured quantity data alongside its rendered message; do not attempt to parse quantity from arbitrary chat text.
- Conversation history is retained. Listing close/expiry never archives or blocks related conversations. Archive/restore updates only the current user's row in `conversation_participants`; it never changes the other participant's chat list.
- Opening or sending in an archived conversation clears the current user's `archived_at`. The idempotent listing chat endpoint may return an existing conversation for a closed/expired listing and must not create duplicate threads.
- Public listing DTOs must omit `pass_expires_at` and `pass_expiry_type`. Owner activity/pass DTOs may include them after authorization.

## Backend checks

Run formatting, compilation, unit tests, integration tests against PostgreSQL/Testcontainers when available, and Flyway validation. Important tests cover separate post/pass expiry, exact 24-hour post boundaries, owner edit/close/repost authorization, quantity bounds, branch/type/search filtering, private pass-expiry DTOs, participant-local archive/restore transitions, conversation access after listing close/expiry, duplicate conversation races, and message sender authorization.
