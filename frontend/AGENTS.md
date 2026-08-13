# Frontend agent guide

## Stack

- Next.js App Router with TypeScript.
- Tailwind CSS for layout and styling.
- A small client-side query/cache layer (TanStack Query is acceptable) for API data and polling.
- Zod or equivalent for parsing API responses at the boundary.

Use the current stable versions when scaffolding. Keep the frontend independently runnable from `frontend/` and configure the API base URL through an environment variable such as `NEXT_PUBLIC_API_BASE_URL`.

## MVP routes

- `/login` — username-only development login.
- `/` — gym picker and branch picker.
- `/gyms/[gymId]/branches/[branchId]` — branch marketplace with `Selling` and `Buying` tabs.
- `/search` — all-gym listing search with gym, branch, pass type, price, quantity, expiry, and sort controls.
- `/listings/[listingId]` — listing details and the `Chat` action.
- `/my-listings` — the current user's Active, Closed, and Expired posts with edit, close, repost, and duplicate actions.
- `/my-passes` — optional private pass records, including pass expiry metadata; never expose these fields in public listing cards.
- `/chats` — current user's conversations.
- `/chats/[conversationId]` — messages and send box.

The navigation should make the current gym and branch obvious. Preserve the selected tab in the URL (`?type=SELL` or `?type=BUY`) so links and refreshes are predictable.

## UX requirements

- Show an empty state when a branch has no active listings.
- Show the post expiry on active listing cards as “Post expires in 8 hours” using server-provided `postExpiresAt`; do not show `passExpiresAt` or exact pass expiry on public posts.
- In owner-only My Listings/My Passes views, distinguish “Post expires in 8 hours” from “Passes expire on 30 August”. Pass expiry is optional and can display “Pass expiry unknown”.
- Listing cards and detail views must show pass type, quantity available/requested, remaining quantity, and price per pass.
- Give the owner clear `Edit` and `Close post` actions. Editing must make the effect on remaining quantity obvious and require confirmation before closing.
- Provide search and filter controls for gym, branch, pass type, price range, quantity, expiry, and sort order. Keep filters in the URL so searches are shareable and refresh-safe.
- Disable create/chat actions while requests are pending and show actionable error messages.
- Prevent the listing owner from opening a conversation with themselves.
- Make the exchange direction unmistakable with badges, copy, and color that also work without color alone.
- Add an “I’m interested in X passes” quick action in the chat entry flow, with the quantity editable before sending.
- Separate active and archived chats per user. Preserve message history, show why a chat was archived, and let a user restore an archived conversation without changing the other participant's chat list or implying that the listing is open.
- Opening or sending a message in an archived chat should return it to the current user's active chats. A closed or expired listing must not make the conversation read-only.
- Keep the first version mobile-first: climbers are likely to use it on the way to or from a gym.
- Include loading, error, empty, and expired states for each main screen.
- Use accessible labels, keyboard navigation, visible focus states, and sufficient contrast.

## Data and state rules

- Treat API DTOs as the contract; do not mirror backend entity classes in the UI.
- Do not derive whether a listing is active solely from the browser clock.
- Refresh listing data after creating/cancelling a post and after opening a conversation.
- Refresh listing data after editing, closing, or updating remaining quantity; handle a conflict if another user changed the listing first.
- Poll the active conversation while it is open (for example every 5–10 seconds) until a WebSocket transport is deliberately introduced.
- Treat archive state as participant-local. Archive/restore actions must not mutate the other participant's chat list or the listing.
- Keep public listing DTOs free of pass-expiry fields. Owner DTOs may include them after access checks.
- Keep username-only auth behind an obvious development-only boundary so replacing it with real auth does not require rewriting page components.

## Component direction

Prefer reusable components such as `GymPicker`, `BranchPicker`, `SearchFilters`, `ListingTabs`, `ListingCard`, `CreateListingForm`, `EditListingForm`, `MyListingsTabs`, `MyPassesList`, `ConversationList`, `ArchiveToggle`, `InterestQuickAction`, and `MessageThread`. Keep route components focused on composition and data loading.

## Frontend checks

Before handing off a change, run the package lint, typecheck, unit tests, and production build. Add component or browser tests for login, tab filtering, search/filter URL state, listing quantity and price editing, My Listings tabs/actions, private pass expiry visibility, participant-local archive/restore behavior, expiry states, and starting a chat from a post.
