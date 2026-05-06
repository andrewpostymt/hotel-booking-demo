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

### `postman-cs/postman-repo-sync-action`

Runs after bootstrap to materialise the workspace into the repository and keep it continuously in sync.

What it handles:

**Workspace linking** — Registers the GitHub repository URL against the Postman workspace via the Bifrost integration backend. This surfaces the repo link in the workspace UI so consumers can navigate directly from a collection to its source.

**Environment provisioning** — Creates or updates Postman environments for each deployment tier defined in `environments-json`. Maps each environment slug to a system environment ID and runtime base URL, so environment variables are resolved correctly per tier without manual configuration.

**Mock server** — Creates a mock server linked to the Baseline collection and writes the resulting URL back into the appropriate environment files. Subsequent runs reuse the existing mock if a valid `mock-url` is supplied.

**Monitor creation** — Creates a cloud monitor targeting the Smoke collection on the configured cron schedule (e.g. `0 */6 * * *`). The monitor runs independently of the CI pipeline and surfaces results in the Postman workspace dashboard. Can be set to `cli` mode to skip cloud monitor creation and rely solely on the CI pipeline instead.

**Artifact export and repo commit** — Exports the Baseline and Contract collections and all environments as YAML into the `postman/` directory, then commits and pushes the result back to the branch. This keeps the repository in sync with Postman Cloud without manual `workspace push` operations.

**Collection and spec lifecycle** — Supports `refresh` mode (overwrite in place) or `version` mode (create a new named version) for both collections and specs. Used to control how breaking changes are propagated.

```yaml
- uses: postman-cs/postman-repo-sync-action@main
  with:
    project-name: room-booking-core-services
    workspace-id: <workspace-id>
    baseline-collection-id: <baseline-collection-id>
    smoke-collection-id: <smoke-collection-id>
    contract-collection-id: <contract-collection-id>
    spec-id: <spec-id>
    environments-json: '["dev","qa2","qa4","preprod","nonprod-dev","nonprod-qa","nonprod-stage","nonprod-preprod"]'
    monitor-cron: '0 */6 * * *'
    generate-ci-workflow: 'false'
    postman-api-key: ${{ secrets.POSTMAN_API_KEY }}
    postman-access-token: ${{ secrets.POSTMAN_ACCESS_TOKEN }}
    github-token: ${{ secrets.GITHUB_TOKEN }}
```

**Note**: Set `generate-ci-workflow: 'false'` to prevent the action from overwriting the custom `ci.yml`. The action defaults to generating its own CI workflow file.

---

## Secrets Required

| Secret | Usage |
|--------|-------|
| `POSTMAN_API_KEY` | Authenticates the Postman CLI for all `postman` commands |
| `POSTMAN_ACCESS_TOKEN` | Required by the bootstrap action for workspace and governance operations |

---

## Example Developer Workflow

The following describes how a developer working on a hotel platform API would interact with this system from a feature branch through to production readiness.

**1. Local development**

The developer opens the Postman desktop app in Local Mode. The workspace is connected to this repository, so all collections, environments, and specs are available locally as v3 YAML. They select the `dev` environment, which points to the mock server, and explore the existing requests before making changes.

**2. Spec-first change**

The developer updates `room-booking-core-services.openapi.yaml` to add a new field or endpoint. In the desktop app, the spec-to-collection link triggers a prompt to regenerate the linked Baseline, Smoke, and Contract collections. They review the generated requests, add representative example responses to the Baseline for mock coverage, and write test assertions in the Contract collection.

**3. Push to feature branch**

The developer runs `postman workspace push` to sync local changes to Postman Cloud, then commits the updated YAML files and opens a pull request against `main`.

**4. CI runs automatically**

On the pull request, the CI pipeline executes:
- **Governance** — `postman spec lint` validates the spec against the team's ruleset in Postman Cloud. Any violations block the merge.
- **Smoke tests** — a fast sanity check across critical paths, running against the `dev` environment (mock server).
- **Contract tests** — schema and response contract assertions validate that the implementation matches the spec.

The developer can see pass/fail status directly in the GitHub pull request checks, and full run history in the Postman workspace.

**5. Merge and promotion**

Once the pull request is approved and CI passes, it merges to `main`. The repo-sync action updates the Postman workspace, refreshes environment configurations for the next tier (`qa2`), and the cloud monitor picks up the new smoke collection on its next scheduled run.

The same CI pipeline runs on merge, this time targeting `nonprod-qa`, giving the team an automated gate before any change reaches the shared QA environment.

**6. Pre-release validation**

Before a release to `preprod`, the Regression collection is run manually or as a scheduled job against `nonprod-stage`. This full suite covers happy-path and error-path scenarios across all endpoints. Credentials are resolved from AWS Secrets Manager at runtime — no tokens are stored in the repo.

**7. Ongoing monitoring**

The cloud monitor runs the Smoke collection against `nonprod-qa` every 6 hours. Failures surface in the Postman workspace dashboard and can be configured to alert via webhook. The scheduled CI run provides a parallel signal from within the GitHub Actions environment.

---

## Local Development

This repo is connected to a Postman workspace in Local Mode. Collections, environments, and specs are stored as v3 YAML and synced to Postman Cloud using the Postman desktop app or CLI.

**To sync local changes to Postman Cloud:**
```sh
postman workspace push
```

**Resources file** (`.postman/resources.yaml`): maps each local file path to its Postman Cloud ID. The `workspace.id` field must be set — the desktop app will blank this on connect, requiring manual restoration before any push.

**Spec-to-collection relationships** are defined in `.postman/workflows.yaml`. Changes to a spec trigger collection regeneration for all linked collections when synced via the app.
