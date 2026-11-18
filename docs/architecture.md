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

- **Presentation tier decision**: React functional components with hooks. Fetch from `/api/*` endpoints; Vite proxies to localhost:8000. Display forms for creation, lists for data, Bootstrap cards for layout. Use React Router for navigation between Activities, Teams, Leaderboard, and Workouts pages.
- **Logic tier decision**: Express routes in `/api/*` validate input, delegate to service functions, catch errors with meaningful 4xx codes (never 500 from validation). Services orchestrate database queries and business logic.
- **Data tier decision**: Mongoose schemas with required-field validation. User and Team documents reference each other; Activity and Workout are independent but reference User/Team. Seed data creates representative records idempotently for all four capabilities.
- **Activity tracking approach**: Accept `userId`, `teamId`, `type`, `minutes`. Validate user and team IDs exist (reject with 400 if not). Compute `points` from activity `type` and `minutes`. Store with `occurredAt` timestamp. Return created activity with all fields.
- **Team management approach**: Accept `name` and `mascot`. Reject with 400 if `name` missing. Members array starts empty, can be populated separately. Teams are queryable independently of members for now.
- **Leaderboard approach**: Aggregate activities by team. Sum points per team. Sort by total points descending. Return with computed 1-based rank and team name.
- **Workout suggestions approach**: Find user by email (return 404 if not found). Analyze recent activities for `type` patterns. Suggest Workout matching most common type as `focus`. Include `reason` explaining why (e.g., "Matches your recent running activity"). Return array with at least one suggestion.

## Test plan

Record the commands you run and what each failure means.

- **Data-tier command**: `npm run test:data --prefix octofit-tracker/backend`
  - Validates User, Team, Activity, Workout schemas have required fields and structure.
  - **Failures mean**: Schema is missing field definitions or validation rules (e.g., User needs `required: true` on `name`, `email`, `fitnessGoal`).
  - **Success**: All five tests pass, confirming schemas match the contract and seed data is idempotent.

- **API command**: `npm run test:api --prefix octofit-tracker/backend`
  - Tests `/api/activities`, `/api/teams`, `/api/leaderboard`, `/api/workouts/suggestions/:userEmail` endpoints.
  - **Failures mean**: Service functions are not implemented (returning 501) or logic is incomplete (e.g., leaderboard returns empty array, suggestions missing `reason`).
  - **Success**: All four tests pass, confirming endpoints validate input, return proper status codes, and compute correct responses.

- **Frontend command**: `npm test --prefix octofit-tracker/frontend`
  - Tests Activities, Teams, Leaderboard, Workouts components render API data correctly.
  - **Failures mean**: Components still show "Implementation pending" or don't fetch/display data from `/api/*`.
  - **Success**: All 15 tests pass, confirming components fetch data, handle loading/error states, and render lists and forms correctly.

- **Backend build command**: `npm run build --prefix octofit-tracker/backend`
  - Compiles TypeScript to JavaScript for production. Validates no type errors in routes, services, models.
  - **Success**: dist/ folder contains compiled code ready to run with `node dist/server.js`.

- **Frontend build command**: `npm run build --prefix octofit-tracker/frontend`
  - Bundles React app with Vite for production. Validates no imports are broken, all routes are reachable.
  - **Success**: dist/ folder contains optimized HTML, CSS, JS ready to serve.
