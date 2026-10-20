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

- Presentation tier decision: Keep route selection in `App.tsx` and data access in the shared API client. Each capability component owns its request, accessible loading/error/empty states, and rendering; use relative `/api` URLs through the Vite proxy.
- Logic tier decision: Keep Express routes thin and put validation and capability behavior in `octofitService.ts`. Return deliberate 4xx responses for invalid input, malformed or missing references, and unknown users; do not expose cast failures as 500 responses.
- Data tier decision: Enforce required fields and sensible constraints in the Mongoose schemas, model user/team references explicitly, and keep seed operations idempotent without deleting unrelated records.
- Activity tracking approach: Accept `userId`, `teamId`, `type`, and positive `minutes`; validate both IDs and confirm both records exist before writing. Award one point per minute and store the references, activity details, computed points, and `occurredAt`.
- Team management approach: Require a non-empty name, preserve the submitted name and mascot, and default members to an empty array. Store member references using the existing Mongoose relationship.
- Leaderboard approach: Aggregate activity points by team, sort by total descending, and return `teamName`, `totalPoints`, and a 1-based rank for each result.
- Workout suggestions approach: Resolve the user by the decoded email path parameter; use the user's goal and recent activities to prioritize matching workouts and include a reason. Return a deliberate 4xx when no user matches.

Implement in dependency order: schema constraints first, then service/API behavior and its contracts, then presentation components and their request states. Preserve the passing seed contract while changing data behavior.

## Test plan

Run the focused contract for a tier while implementing it, then run both builds after the contracts pass. A failed assertion identifies a mismatch with the stated data, API, or presentation contract; a build failure identifies a TypeScript or bundling problem.

- Data tier: `npm run test:data --prefix octofit-tracker/backend` checks schema validation and idempotent seed behavior. Failures indicate missing model constraints/defaults or seed logic that does not preserve existing records.
- API tier: `npm run test:api --prefix octofit-tracker/backend` checks activity/team writes, input and reference rejection, leaderboard aggregation/ranking, and personalized workout suggestions. Failures indicate route/service behavior or response data/status does not match the API contract.
- Presentation tier: `npm test --prefix octofit-tracker/frontend` checks API-backed rendering, workout request URL, and accessible loading, empty, and error states. Failures indicate missing UI integration or user-visible request states.
- Backend build: `npm run build --prefix octofit-tracker/backend` checks the Express/TypeScript application compiles.
- Frontend build: `npm run build --prefix octofit-tracker/frontend` checks the Vite/TypeScript application bundles.

Baseline observed before implementation: data tests had 4 failures and 1 pass (the seed contract passed); all 4 API tests failed; all 15 frontend tests failed. The build commands have not yet been run.
