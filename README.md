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


## Security checks

Each type of check runs in its own workflow, on every pull request and on every push to `main`.
All GitHub Actions are pinned to a commit SHA, not a tag.

| Check | Tool | Workflow | Blocks the PR? | Why this tool |
|---|---|---|---|---|
| Secrets | Gitleaks | `security.yml` | Yes | Finds passwords or keys pushed to git |
| SAST | SonarCloud | `sonar.yml` | Yes (Quality Gate) | Same tool as the backend. Also checks workflows, Dockerfile and nginx config |
| SCA | Trivy + dependency-review | `sca.yml` | Yes (HIGH/CRITICAL) | Trivy checks the app dependencies. dependency-review stops a PR that adds a vulnerable package |
| Container & IaC | Trivy | `container.yml` | Yes (HIGH/CRITICAL with a fix) | Checks the Dockerfile and the built image |
| DAST | OWASP ZAP | `dast.yml` | Yes | Tests the app while it is running in a container |
| Accessibility | pa11y-ci | `a11y.yml` | Yes | Tests the page in a real browser against WCAG 2 AA |

### What I fixed

- **SAST**: 7 issues fixed (rating went from E to A).
  - API key passed with `ARG` in the Dockerfile, now passed as a build secret
  - Actions pinned by SHA instead of tags
  - `npm ci` now runs with `--ignore-scripts`
  - nginx no longer runs as root
  - SonarCloud also caught two issues in my own new workflows: a write permission set too wide, and a `npx` call. Both fixed.
- **SCA**: updated `fast-uri` (3.1.3 to 3.1.7), which closed 6 high alerts. Updated Angular (22.0.5 to 22.2.0) to fix a sanitization bypass.
- **Container & IaC**: added `USER 101` in the Dockerfile. Moved from nginx 1.27 (Alpine 3.21) to `stable-alpine` (Alpine 3.24). Patched `libexpat` for CVE-2026-93990.
- **DAST**: went from 10 alerts to 0. Added security headers in `nginx.conf` (CSP, X-Frame-Options, X-Content-Type-Options, Referrer-Policy, Permissions-Policy, COOP/COEP/CORP) and hid the nginx version. The Angular update also fixed a vulnerable JS library found by ZAP.
- **Accessibility**: no errors found. Note: only the home page is tested in CI, without backend data.

### Accepted exceptions

These are not fixes. They are alerts I decided to ignore, with a reason.

- ZAP (`.zap/rules.tsv`):
  - 10109: only an information message, not a vulnerability
  - 10110: found inside the Angular framework code, not in my code
  - 10049: public static files, nothing sensitive
  - 10055: Angular needs `unsafe-inline` for styles. Scripts are still limited to `'self'`.
- Trivy image scan: only fails on CVEs that already have a fix.
- Trivy SCA: dev dependencies are not scanned because they are not shipped to users. Dependabot still watches them.
