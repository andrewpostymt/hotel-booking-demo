# Hotel Booking Demo — API Platform

A reference implementation of a Postman-native API platform across three hotel booking services. Demonstrates spec-first development, automated governance, multi-environment testing, and CI/CD using the Postman CLI and GitHub Actions.

---

## Services

| Service | Spec File | Description |
|---------|-----------|-------------|
| Room Booking Core Services | `room-booking-core-services.openapi.yaml` | Availability, pricing, reservations, and customer credit across all hotel properties |
| Recommendation Service | `recommendation-service.openapi.yaml` | Cross-sell and upsell engine — personalised offers post-booking and at key funnel steps |
| Entertainment & Nightlife Reservations | `entertainment-nightlife-reservations.openapi.yaml` | Event sync with Opera PMS, rate plan lookups, and resync for failed event records |

---

## Repository Structure

```
.
├── room-booking-core-services.openapi.yaml
├── recommendation-service.openapi.yaml
├── entertainment-nightlife-reservations.openapi.yaml
├── postman/
│   ├── collections/
│   │   ├── [Baseline] room-booking-core-services/
│   │   ├── [Smoke] room-booking-core-services/
│   │   ├── [Contract] room-booking-core-services/
│   │   ├── [Regression] room-booking-core-services/
│   │   ├── [Baseline] recommendation-service/
│   │   ├── [Smoke] recommendation-service/
│   │   ├── [Contract] recommendation-service/
│   │   └── [Regression] recommendation-service/
│   └── environments/
│       ├── room-booking-core-services - dev.environment.yaml
│       ├── room-booking-core-services - qa2.environment.yaml
│       ├── room-booking-core-services - qa4.environment.yaml
│       ├── room-booking-core-services - preprod.environment.yaml
│       ├── room-booking-core-services - nonprod-dev.environment.yaml
│       ├── room-booking-core-services - nonprod-qa.environment.yaml
│       ├── room-booking-core-services - nonprod-stage.environment.yaml
│       └── room-booking-core-services - nonprod-preprod.environment.yaml
├── .postman/
│   ├── resources.yaml        # Maps local files to Postman Cloud IDs
│   └── workflows.yaml        # Spec-to-collection sync relationships
└── .github/
    └── workflows/
        ├── ci.yml            # Main CI/CD pipeline (governance + tests)
        └── postman-bootstrap.yml  # One-time workspace bootstrap
```

---

## Collections

Each service has four collection types. All collections are stored as Postman v3 YAML and synced to Postman Cloud via `postman workspace push`.

| Type | Purpose |
|------|---------|
| **[Baseline]** | All endpoints with representative happy-path requests and saved example responses. Powers the mock server. |
| **[Smoke]** | Lightweight sanity checks — one request per critical path. Runs on every push. |
| **[Contract]** | Schema validation and response contract assertions against live or mock endpoints. Runs on every push. |
| **[Regression]** | Full suite of happy-path and error-path scenarios with test assertions. Used for pre-release validation. |

---

## Authentication

All services use **OAuth 2.0 Client Credentials**.

Credentials are resolved from **AWS Secrets Manager** (`api-credentials-{env}`) at runtime. Each environment includes the corresponding secrets ARN. In CI, a pre-resolved token is injected via environment variables — no secrets fetch is required during test runs.

```
grant_type:    client_credentials
scope:         room-booking:read room-booking:write
tokenUrl:      /auth/token
```

---

## Environments

Eight environments are maintained, progressing from inner-loop development through to pre-production validation:

| Environment | Purpose |
|-------------|---------|
| `dev` | Local and inner-loop development. Points to the mock server by default. |
| `qa2` | Shared QA environment — integration testing with dependent services. |
| `qa4` | Secondary QA environment for parallel feature validation. |
| `preprod` | Pre-production environment matching production configuration. |
| `nonprod-dev` | Non-production replica of dev for CI pipeline runs. |
| `nonprod-qa` | Non-production replica of QA for automated testing. |
| `nonprod-stage` | Staging environment for release candidate validation. |
| `nonprod-preprod` | Non-production pre-production for final sign-off before release. |

---

## CI/CD Pipeline

The pipeline is defined in `.github/workflows/ci.yml` and runs on:
- Every push to `main` that touches a spec file, collection, or the workflow itself
- Every pull request to `main`
- A scheduled run every 6 hours (`0 */6 * * *`)
- Manual trigger via `workflow_dispatch`

### Stage 1 — API Governance

```yaml
- name: Install Postman CLI
  run: curl -o- "https://dl-cli.pstmn.io/install/unix.sh" | sh

- name: Login to Postman CLI
  run: postman login --with-api-key ${{ secrets.POSTMAN_API_KEY }}

- name: Run API Governance
  run: postman spec lint <spec-id> --workspace-id <workspace-id> --report-events
```

**`postman spec lint`** lints the OpenAPI spec against the governance ruleset configured in the Postman workspace. Rules are maintained in Postman Cloud — no local ruleset file is needed. The `--report-events` flag sends results to Postman's reporting API so violations appear in the workspace dashboard.

This job must pass before tests are allowed to run (`needs: governance`).

### Stage 2 — Smoke & Contract Tests

```yaml
- name: Resolve Postman Resource IDs
  run: |
    ruby <<'RUBY'
    require 'yaml'
    config = YAML.load_file('.postman/resources.yaml') || {}
    cloud = config.fetch('cloudResources', {})
    collections = cloud.fetch('collections', {})
    environments = cloud.fetch('environments', {})
    smoke = collections.find { |path, _| path.include?('[Smoke]') }&.last
    contract = collections.find { |path, _| path.include?('[Contract]') }&.last
    environment = environments.find { |path, _| path.include?('- dev.environment') }&.last || environments.values.first
    # writes POSTMAN_SMOKE_COLLECTION_UID, POSTMAN_CONTRACT_COLLECTION_UID,
    # POSTMAN_ENVIRONMENT_UID to $GITHUB_ENV
    RUBY

- name: Run Smoke Tests
  run: postman collection run "$POSTMAN_SMOKE_COLLECTION_UID" -e "$POSTMAN_ENVIRONMENT_UID" --report-events

- name: Run Contract Tests
  run: postman collection run "$POSTMAN_CONTRACT_COLLECTION_UID" -e "$POSTMAN_ENVIRONMENT_UID" --report-events
```

**Resolve step**: reads `.postman/resources.yaml` to map local collection paths to their Postman Cloud UIDs, then exports them as environment variables for the subsequent steps. This keeps the workflow file decoupled from hardcoded IDs — the mapping is the source of truth.

**`postman collection run`** executes the collection against the specified environment. The `--report-events` flag streams test results to Postman Cloud, where they appear as run history in the workspace.

---

## Postman Actions

### `postman-cs/postman-bootstrap-action@main`

Used in `.github/workflows/postman-bootstrap.yml` for one-time workspace setup.

```yaml
- uses: postman-cs/postman-bootstrap-action@main
  with:
    project-name: room-booking-core-services
    spec-url: https://raw.githubusercontent.com/andrewpostymt/hotel-booking-demo/main/room-booking-core-services.openapi.yaml
    workspace-id: <workspace-id>
    domain: booking-services
    domain-code: RBS
    governance-mapping-json: '{"booking-services":"Booking Services"}'
    postman-api-key: ${{ secrets.POSTMAN_API_KEY }}
    postman-access-token: ${{ secrets.POSTMAN_ACCESS_TOKEN }}
```

This action:
1. Uploads the spec to Postman Spec Hub and links it to the workspace
2. Generates Baseline, Smoke, and Contract collections from the spec
3. Assigns the spec to the correct governance group
4. Returns `spec-id`, `baseline-collection-id`, `smoke-collection-id`, `contract-collection-id` as step outputs for use in downstream steps or to populate `resources.yaml`

**Note**: Use `@main`, not `@v0`. The `v0` tag does not support async collection generation.

---

## Secrets Required

| Secret | Usage |
|--------|-------|
| `POSTMAN_API_KEY` | Authenticates the Postman CLI for all `postman` commands |
| `POSTMAN_ACCESS_TOKEN` | Required by the bootstrap action for workspace and governance operations |

---

## Local Development

This repo is connected to a Postman workspace in Local Mode. Collections, environments, and specs are stored as v3 YAML and synced to Postman Cloud using the Postman desktop app or CLI.

**To sync local changes to Postman Cloud:**
```sh
postman workspace push
```

**Resources file** (`.postman/resources.yaml`): maps each local file path to its Postman Cloud ID. The `workspace.id` field must be set — the desktop app will blank this on connect, requiring manual restoration before any push.

**Spec-to-collection relationships** are defined in `.postman/workflows.yaml`. Changes to a spec trigger collection regeneration for all linked collections when synced via the app.
