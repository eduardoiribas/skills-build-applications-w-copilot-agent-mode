# OctoFit Tracker architecture

OctoFit Tracker is a three-tier application for a school fitness program. The starter code includes API contracts and intentionally incomplete implementation points so you can use GitHub Copilot agent mode in a test-driven loop.

![Three-tier OctoFit architecture](../.github/images/octofit-architecture.svg)

## Tier boundaries

- **Presentation tier**: React 19, Vite, React Router, Bootstrap, and a shared API client in `octofit-tracker/frontend/src`.
- **Logic tier**: Express and TypeScript routes/services in `octofit-tracker/backend/src` expose `/api/*` behavior on port `8000`.
- **Data tier**: Mongoose models and seed data in `octofit-tracker/backend/src/models` and `src/scripts/seed.ts` persist to MongoDB on port `27017`.

## Ports and environment rules

- MongoDB: `27017`
- Express API: `8000`
- Vite frontend: `5173`
- The browser calls relative `/api` paths. Vite proxies them to `http://localhost:8000`, so the unauthenticated API port stays private in Codespaces. `VITE_API_BASE_URL` can override the base URL.

## API contract

| Capability | Method and path | Expected behavior |
| --- | --- | --- |
| Activity tracking | `GET /api/activities`, `POST /api/activities` | List and create valid activity records. POST accepts `userId`, `teamId`, `type`, and `minutes`; `userId` and `teamId` must be well-formed IDs for an existing user and team; otherwise respond with a deliberate 4xx (never a 500 from a cast error). A created activity stores the submitted `user`, `team`, `type`, and `minutes` plus computed `points` and `occurredAt`. |
| Teams | `GET /api/teams`, `POST /api/teams` | List and create teams with members. Created and listed teams keep the submitted `name` and `mascot`. Reject a team without a name with `400`. |
| Leaderboard | `GET /api/leaderboard` | Return teams ordered by the sum of their activity points, highest first. Each entry has `teamName`, `totalPoints`, and a 1-based `rank`. |
| Workout suggestions | `GET /api/workouts/suggestions/:userEmail` | Recommend workouts from recent activity patterns for the matching seeded user's email address. The first suggestion's `focus` matches that user's goal or recent activity, with a `reason` explaining why. An email with no matching user returns a deliberate 4xx (for example, `404`). |

## Implementation decisions

These decisions keep each responsibility in its existing tier and use the API contracts as the source of expected behavior.

- Presentation tier decision: Keep the existing React routes and capability components responsible for rendering. Fetch through the shared API client and expose accessible loading, empty, success, and error states.
- Logic tier decision: Keep HTTP parsing and status codes in Express routes, with capability behavior in explicit service functions. Validate request fields and referenced records before persistence, returning deliberate 4xx responses for invalid input.
- Data tier decision: Put field types, required values, references, and defaults in Mongoose schemas. Keep demo seeding idempotent with upserts; do not clear collections.
- Activity tracking approach: Validate the submitted user and team IDs and confirm both records exist. Persist the submitted activity fields with points and occurrence time computed by the service; list created activity records through the same API contract.
- Team management approach: Require a team name, preserve the submitted name and mascot, and support member references when creating and listing teams.
- Leaderboard approach: Sum activity points per team, sort from highest to lowest, and assign 1-based ranks in the response.
- Workout suggestions approach: Find the user by email, use the fitness goal and recent activity pattern to select and explain suggestions, and return a deliberate 4xx when no user matches.

## Test plan

- `npm run test:data --prefix octofit-tracker/backend`: verifies schema constraints and repeatable seed behavior. Baseline: 4 of 5 tests fail because required validation is missing from the user, team, activity, and workout models; the idempotent seed test passes.
- `npm run test:api --prefix octofit-tracker/backend`: verifies activity and team requests, leaderboard ordering, and workout recommendations. Baseline: all 4 tests fail; invalid requests return `501` instead of `400`, the leaderboard is empty, and the first workout suggestion does not match the user's activity pattern.
- `npm test --prefix octofit-tracker/frontend`: verifies API-backed content and accessible loading, empty, and error states. Baseline: all 15 tests fail; capability views still report `Implementation pending` instead of rendering the expected states and data.
- `npm run build --prefix octofit-tracker/backend`: TypeScript compile check. Baseline: passes; a failure indicates a backend type or compile error.
- `npm run build --prefix octofit-tracker/frontend`: frontend TypeScript and production Vite build check. Baseline: passes; a failure indicates a frontend type, compile, or bundling error.

Run the data tests first, then API tests, then frontend tests as their dependencies are implemented. Run both build checks as final compile and bundle gates. The baseline above records behavior observed before implementation changes.
