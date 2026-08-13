# KyiPasses

KyiPasses is a Singapore-focused climbing-pass exchange app. Climbers choose a gym and branch, browse short-lived posts, and chat with the person offering or requesting passes.

## Product shape

The first release is intentionally list-first:

1. Sign in with a username (development only).
2. Choose a gym, then a branch.
3. Browse the branch’s `Selling` or `Buying` tab.
4. Create a detailed post with day-pass type, quantity, and price per pass. The post automatically expires after 24 hours.
5. Search across all gyms or filter by gym, branch, price, quantity, pass type, and expiry.
6. Manage your own posts from My Listings, including editing, closing, reposting, and duplicating.
7. Open a chat to coordinate the exchange. Chats remain available across listing expiry and sessions.

The future gym discovery view can be a separate tab with an interactive Singapore map and gym pins. Gym branches already carry coordinates in the planned data model so this can be added without changing the marketplace concept.

## Recommended stack

### Frontend

- Next.js App Router and TypeScript.
- Tailwind CSS.
- TanStack Query (or a similarly small query/cache layer).
- HTTP polling for chat in the first slice; WebSockets/STOMP can be added once message volume and realtime expectations justify it.

### Backend

- Java 21.
- Spring Boot 4.x / Spring Framework 7.x.
- Maven.
- PostgreSQL.
- Flyway migrations.
- Spring Web, Validation, Security, and Data JDBC or JPA.

The current Next.js App Router is the recommended routing model in the official documentation, and Spring’s current project documentation lists Spring Boot 4.x. See [Next.js App Router](https://nextjs.org/docs/app) and [Spring Boot](https://spring.io/projects/spring-boot).

## Repository layout

```text
.
├── AGENTS.md
├── README.md
├── frontend/
│   └── AGENTS.md
└── backend/
    └── AGENTS.md
```

The implementation agents should keep their work independently runnable. Read the directory-specific guide before changing either application.

## MVP domain

- `User` — username-only development identity.
- `Gym` — climbing gym brand or organization.
- `GymBranch` — physical branch, including address and future map coordinates.
- `Listing` — a `SELL` or `BUY` post tied to one branch, with day-pass type, total/remaining quantity, price per pass, optional private pass-expiry metadata, and a 24-hour lifetime.
- `UserPass` — optional private pass information owned by a user, including unknown/fixed/rolling expiry metadata.
- `Conversation` — one private buyer/interested-user thread for one listing.
- `Message` — text inside a conversation.

Store prices as integer cents with an explicit currency such as `SGD`. Listing post expiration is authoritative on the backend: queries must exclude expired posts, and mutations must re-check the listing state. Pass expiry is separate, optional, and private to the owner; it is never displayed on public posts in the MVP.

## Initial API contract

```text
POST /api/auth/dev-login
GET  /api/me
POST /api/auth/logout

GET  /api/gyms
GET  /api/gyms/{gymId}/branches/{branchId}/listings?type=SELL|BUY
POST /api/gyms/{gymId}/branches/{branchId}/listings
GET  /api/listings/{listingId}
PATCH /api/listings/{listingId}
POST /api/listings/{listingId}/close
POST /api/listings/{listingId}/repost
POST /api/listings/{listingId}/duplicate
GET  /api/listings/search?...filters...

GET  /api/me/listings?status=ACTIVE|CLOSED|EXPIRED
GET  /api/me/passes

POST /api/listings/{listingId}/conversations
GET  /api/conversations?status=ACTIVE|ARCHIVED
GET  /api/conversations/{conversationId}/messages
POST /api/conversations/{conversationId}/messages
POST /api/conversations/{conversationId}/archive
POST /api/conversations/{conversationId}/restore
```

Use ISO-8601 UTC timestamps and explicit DTOs. Search supports gym, branch, pass type, price-per-pass, remaining quantity, and post-expiry sorting. Exact pass expiry is intentionally omitted from public listing DTOs. Create/edit requests may include optional `passExpiresAt` and `passExpiryType` values (`FIXED_DATE`, `ROLLING_VALIDITY`, or `UNKNOWN`); the current controlled pass type is `DAY_PASS`. BUY posts contain desired quantity and one preferred branch. The full agent-facing contract and implementation rules are in [AGENTS.md](AGENTS.md), [frontend/AGENTS.md](frontend/AGENTS.md), and [backend/AGENTS.md](backend/AGENTS.md).

## Listing and chat lifecycle

- Owners can edit active posts and update remaining quantity after a partial exchange. `quantity_total` means the amount this listing currently offers or requests, not all passes the user owns.
- For example, a seller with 10 passes who keeps 2 lists `quantity_total = 8`; if 3 are no longer available, the owner can set `quantity_remaining = 5`.
- Owners can close a post when they have enough buyers or no longer need the request. Closing only removes it from marketplace results.
- Post expiry and pass expiry are separate. A post expires after 24 hours; optional pass expiry is private to the owner and may be fixed-date, rolling-validity, or unknown.
- Conversations and messages remain available after a post closes or expires. A participant may archive their own chat without affecting the other participant; opening or messaging restores it for that participant.
- A chat includes an “I’m interested in X passes” quick action and supports agreeing on quantity and price in the thread.
- Existing participants may continue messaging even when the listing is closed or expired. New messages do not create a new listing.
- Temporary quantity reservations while chatting are a planned exchange-control feature. They must be transactional and expire automatically; chat alone must never reduce permanent inventory.
- Automatically closing a listing when all passes are gone is intentionally not part of the MVP.

## Roadmap

### MVP expansion

- Detailed pass fields: pass type, quantity available/requested, remaining quantity, and price per pass.
- Owner edit/close actions and partial quantity updates.
- All-gym search with gym, branch, price, quantity, pass type, expiry, and sort filters.
- My Listings with active/closed/expired tabs and edit, close, repost, and duplicate actions.
- Chat quick actions, participant-local archive/restore behavior, and preserved history.

### Later

- Temporary reservations while a buyer and seller coordinate.
- Automatic `SOLD_OUT` closure when remaining quantity reaches zero.
- Pass-expiry warnings and public search/filtering by pass validity.
- Followed gym/branch alerts, new-message notifications, expiry reminders, and response alerts.
- Interactive Singapore map with gym pins and map-driven listings.
- Reporting listings/users/messages, user blocking, basic post-exchange reputation, and abuse controls.

## Recommendations before public launch

- Keep username-only auth strictly for local development; it is not enough to identify or protect users.
- Add reporting, blocking, moderation, rate limiting, and message abuse controls before real users rely on the app.
- Make the exchange terms clear: whether a post is for a single pass, a bundle, a transfer, or a refund, and whether price is negotiable.
- Do not process payment or claim that a pass is valid until ownership/transfer rules are understood for each gym.
- Keep personal contact information private by default. If contact details are ever added, expose them only through an explicit user action and clear safety guidance.
- Add safety guidance for public meetups, pass transfers, payment scams, and verifying the pass before relying on it.
- Keep the current pass type controlled as `DAY_PASS`; add more types only when the gym/pass-transfer rules are understood.
- Seed only gyms and branches that the team is allowed to represent, and provide a way to report incorrect branch information.

## Suggested build order

1. Scaffold the frontend and backend shells.
2. Add PostgreSQL configuration and Flyway migrations for users, gyms, branches, and listings.
3. Seed a small set of Singapore gyms/branches.
4. Implement development login and the gym → branch → tabs flow.
5. Add detailed listing creation/edit/close, partial quantity updates, My Listings, repost/duplicate, search/filter/sort, expiry cleanup, and tests.
6. Add conversations, quick interest actions, participant-local archive/restore, and messages with HTTP polling.
7. Add private My Passes data and optional pass-expiry metadata.
8. Add reservations and automatic sold-out closure when their transactional rules are ready.
9. Harden authentication, moderation, privacy, safety guidance, notifications, and exchange-completion flows.
