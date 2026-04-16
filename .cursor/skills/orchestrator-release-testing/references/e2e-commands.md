# E2E Test Commands Reference

## run-e2e.sh

### Synopsis

```bash
./run-e2e.sh [-w <workspace>]... [-- <playwright-args>...]
```

All arguments not matching `-w`/`--workspace` are passed to `npx playwright test`.

### Examples

```bash
# Run all orchestrator tests
./run-e2e.sh -w orchestrator

# Run only greeting workflow tests
./run-e2e.sh -w orchestrator --grep "Greeting"

# Run only RBAC tests
./run-e2e.sh -w orchestrator --grep "RBAC"

# Run with 2 workers
./run-e2e.sh -w orchestrator --workers=2

# List all test names without running
./run-e2e.sh -w orchestrator --list

# Run specific project
./run-e2e.sh -w orchestrator --project=orchestrator

# Run multiple workspaces
./run-e2e.sh -w orchestrator -w rbac

# Run with local e2e-test-utils
E2E_TEST_UTILS_PATH=/home/user/rhdh-e2e-test-utils ./run-e2e.sh -w orchestrator

# Run with specific e2e-test-utils version
E2E_TEST_UTILS_VERSION=1.1.29 ./run-e2e.sh -w orchestrator

# Run with specific RHDH version
RHDH_VERSION=1.9 ./run-e2e.sh -w orchestrator

# Run in nightly mode
E2E_NIGHTLY_MODE=true ./run-e2e.sh -w orchestrator

# Skip Keycloak deploy (reuse existing)
SKIP_KEYCLOAK_DEPLOYMENT=true ./run-e2e.sh -w orchestrator

# PR-based plugin resolution
GIT_PR_NUMBER=123 ./run-e2e.sh -w orchestrator
```

### What run-e2e.sh does internally

1. Discovers workspaces with both `e2e-tests/package.json` and `playwright.config.ts`
2. Generates a root `package.json` with yarn workspaces and resolutions
3. Writes `.yarnrc.yml` with `nodeLinker: node-modules`
4. Cleans all `node_modules` and `yarn.lock`
5. Runs `yarn install`
6. Generates a root `playwright.config.ts` merging all workspace configs
7. Installs Chromium via Playwright
8. Runs `npx playwright test` with forwarded arguments

### Prerequisites

- `node` >= 22, `yarn` >= 3
- `jq` installed
- `oc` CLI logged in (unless using `--list`)
- `corepack enable` (handled by script)

## Environment Variables

### Deployment configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `RHDH_VERSION` | `1.10` | RHDH Helm chart version |
| `INSTALLATION_METHOD` | `helm` | Deployment method: `helm` or `operator` |
| `CATALOG_INDEX_IMAGE` | (empty) | Override full catalog index image |
| `SKIP_KEYCLOAK_DEPLOYMENT` | (empty) | Skip Keycloak deployment |

### Plugin resolution

| Variable | Default | Description |
|----------|---------|-------------|
| `GIT_PR_NUMBER` | (empty) | Resolve plugins from PR artifacts |
| `E2E_NIGHTLY_MODE` | (empty) | Use nightly plugin builds |
| `JOB_NAME` | (empty) | CI job name; `periodic-*` triggers nightly mode |

### Test library

| Variable | Default | Description |
|----------|---------|-------------|
| `E2E_TEST_UTILS_PATH` | (empty) | Local path to e2e-test-utils (builds before use) |
| `E2E_TEST_UTILS_VERSION` | (empty) | npm/git version override |

### Runtime

| Variable | Default | Description |
|----------|---------|-------------|
| `CI` | `true` | Enables namespace teardown via reporter |
| `K8S_CLUSTER_ROUTER_BASE` | (from .env) | Cluster router domain (e.g. `apps-crc.testing`) |

### Orchestrator-specific (set by test setup, not user)

| Variable | Set by | Description |
|----------|--------|-------------|
| `SONATAFLOW_DATA_INDEX_URL` | `deploySonataflow()` | Data index service URL |
| `KEYCLOAK_AUTH_BASE_URL` | Keycloak deploy | Keycloak base URL |
| `KEYCLOAK_AUTH_REALM` | Keycloak deploy | Auth realm name |
| `NAME_SPACE` | RHDH deploy | Target namespace |

## Direct Workspace Execution

For development iteration on test code (faster than run-e2e.sh):

```bash
cd workspaces/orchestrator/e2e-tests
yarn install
yarn test                  # All tests
yarn test:headed           # With visible browser
yarn test:ui               # Playwright UI mode (interactive)
yarn report                # Show HTML report from last run

# With specific test filter
npx playwright test --grep "Greeting"
npx playwright test --grep "RBAC" --headed
```

**Gotcha:** Direct execution uses the workspace's `playwright.config.ts` and `.env`, not the generated root config from `run-e2e.sh`. Dependency resolution may differ.

## CI Triggers

| Command | Where | Effect |
|---------|-------|--------|
| `/publish` | PR comment | Publish test OCI plugin images |
| `/smoketest` | PR comment | Run smoke tests against published images |
| `/test e2e-ocp-helm` | PR comment | Run e2e on OCP cluster (Helm deploy) |
| `/test e2e-ocp-helm-nightly` | PR comment | Run e2e with nightly plugins |

## Test Coverage Map

### orchestrator.spec.ts

| Describe block | Tests |
|----------------|-------|
| Greeting workflow | Run workflow, verify tabs, check run details, re-run |
| Failswitch workflow | Completed/failed/running states, abort, all-runs table, rerun from failure |
| Cross-workflow link | `test.fixme` (needs OSL 1.37+) |
| Token propagation API | `test.skip` (full body present but skipped) |
| Entity-workflow integration | Template picker, `orchestrator.io/workflows` annotation, Workflows tab, breadcrumbs, template run visibility |

### orchestrator-rbac.spec.ts

| Describe block | Tests |
|----------------|-------|
| Global workflow permissions | Read/write, read-only, denied |
| Per-workflow permissions | Greeting denied, read-write, read-only |
| Instance visibility | Primary vs secondary user |
| Instance admin view | `orchestrator.instanceAdminView` override |
| Template + orchestrator perms | RHIDP-11839/11840: template runs with/without orchestrator permissions |

All RBAC tests run in `test.describe.serial` mode.

## Report and Results

After `run-e2e.sh` completes:
- HTML report: `playwright-report/` at repo root
- JSON results: `playwright-report/results.json`
- View report: `npx playwright show-report`
