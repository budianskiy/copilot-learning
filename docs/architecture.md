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

Record your plan here before editing code.

- Presentation tier decision: Keep route-specific rendering in the existing React components and route all HTTP calls through the shared API client. Each view handles loading, empty, and API error states accessibly; it does not calculate business results.
- Logic tier decision: Keep HTTP status codes and request validation at the Express route boundary, and put reusable business behavior in the backend service. Invalid input and missing references return deliberate 4xx responses rather than database cast errors or 500 responses.
- Data tier decision: Use Mongoose schemas to require the fields in the API contract, model user/team relationships with ObjectId references, and provide an empty members array by default. Keep seeding idempotent and preserve records that are not owned by the seed data.
- Activity tracking approach: Validate the submitted user and team IDs and confirm both records exist before creating an activity. Store the references, activity type, minutes, timestamp, and a deterministic points value; use one point per minute as the initial scoring rule.
- Team management approach: Require a non-empty name, preserve the submitted name and mascot, and store member user IDs in the team's members array. Listing and creation responses expose those same fields.
- Leaderboard approach: Sum activity points by team, sort totals descending, and assign ranks starting at 1. Return the team name and total with each rank.
- Workout suggestions approach: Find the user by email, prefer workouts matching the user's fitness goal and recent activity pattern, and include a concise reason for the top suggestion. Return a deliberate 4xx response when no user matches.

## Test plan

Run the checks in dependency order: data, API, presentation, then builds. A failure at an earlier tier should be fixed before interpreting failures in dependent tiers.

- Data-tier command: `npm run test:data --prefix octofit-tracker/backend` checks required schema fields, references/defaults, and representative idempotent seeding. Failures point to model constraints or seed behavior.
- API command: `npm run test:api --prefix octofit-tracker/backend` checks validation and persistence for activities and teams, leaderboard aggregation/ranking, and user-specific workout recommendations. Failures point to route/service behavior or its use of the models.
- Frontend command: `npm test --prefix octofit-tracker/frontend` checks API requests, rendered data, and accessible loading, empty, and error states for each route. Failures point to the API client or presentation components.
- Backend build command: `npm run build --prefix octofit-tracker/backend` runs TypeScript type checking; failures indicate backend compile-time errors.
- Frontend build command: `npm run build --prefix octofit-tracker/frontend` runs TypeScript type checking and the Vite production build; failures indicate frontend type or bundling errors.

Initial starter baseline: data tests have 4 failures and 1 pass (the idempotent seed test passes); API tests have 4 failures; frontend tests have 15 failures. These are contract failures from intentionally incomplete starter implementations, not expected passing results.
