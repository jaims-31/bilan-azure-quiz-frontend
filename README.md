# azure-quiz-frontend

Angular application to review Microsoft certifications (AZ-900 to start, AZ-104 next): review by
module or mock exam, accessible from a simple link (no account). Consumes the REST API of
[azure-quiz-backend](../azure-quiz-backend).


## Stack

- Angular 22 (standalone components, signals), Angular Material, ngx-translate (fr/en)
- Vitest (Angular CLI 22 native test runner)
- ESLint (`angular-eslint`) + Prettier, husky + lint-staged on pre-commit

## Run locally

Prerequisites: Node 22+, and the backend (`azure-quiz-backend`) running on `http://localhost:8080`.

```bash
npm install
npm start   # http://localhost:4200, targets the API on localhost:8080 (see src/environments/environment.development.ts)
```

## Tests and quality

```bash
npm test           # Vitest
npm run test:coverage
npm run lint
npm run format:check
```

## Production build

```bash
npm run build:prod
```

Static output in `dist/azure-quiz-frontend/browser` — with `skip_app_build: true` (used in this
project's pipeline), that's the folder to point to as `app_location`, with `output_location` left
empty (see `.github/workflows/deploy.yml`).

`src/environments/environment.ts` ships with placeholder tokens (`REPLACE_WITH_PROD_API_URL`,
`__BACKEND_API_KEY__`) and must never be edited by hand with real values: the CI pipeline resolves
the backend App Service and the Key Vault **by Azure tag** and substitutes both tokens via `sed`,
right before this build step, on the ephemeral CI runner only. See the
[infra repo's README](https://github.com/jaims-31/bilan-azure-quiz-infra#cicd) for the full mechanism.


## Structure

- `src/app/core` — models, services (`QuizApiService` for REST calls, `QuizSessionStore` for
  signal-based quiz session state)
- `src/app/features` — pages: `certifications` (home), `modules` (a certification's modules +
  starting a mock exam), `quiz` (question-by-question flow), `results` (final score)

## Out of scope for this repo

- Provisioning the Azure infrastructure (Static Web App, App Service, database).
