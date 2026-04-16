---
name: orchestrator-release-testing
description: Test orchestrator plugins across RHDH and OSL versions. Covers e2e tests, smoke tests, pre-release OSL validation, and version matrix management. Use when testing a new orchestrator release, running orchestrator e2e or smoke tests, validating plugin compatibility across versions, deploying pre-release OSL builds, or debugging orchestrator test failures.
---

# Orchestrator Release Testing

Comprehensive testing of orchestrator plugins (backend, frontend, scaffolder actions) against RHDH and OpenShift Serverless Logic (OSL) version combinations. Covers automated e2e, smoke tests, and manual pre-release validation.

## Quick Reference

| Task | Command |
|------|---------|
| Deploy + test (CRC) | `cd rhdh-test-instance && ./setup-orchestrator.sh <version>` |
| E2E from repo root | `./run-e2e.sh -w orchestrator` |
| E2E from workspace | `cd workspaces/orchestrator/e2e-tests && yarn install && yarn test` |
| Smoke test | `cd smoke-tests && node smoke-test.ts --plugins-yaml <path>` |
| Teardown | `cd rhdh-test-instance && ./cleanup.sh --include-operators --delete-namespace` |
| CI trigger | Comment `/test e2e-ocp-helm` on PR |

## Workflow 1: Full Release Validation

Use for GA releases or release candidates. Validates orchestrator plugins against a target RHDH + OSL combination.

### Step 1: Determine version matrix

Identify the versions under test:

- **RHDH version**: Helm chart version (e.g. `1.10`, `next`, or CI tag like `1.10-200-CI`)
- **OSL version**: OpenShift Serverless Logic operator version (e.g. `1.37`)
- **e2e-test-utils version**: The test library version (check `workspaces/orchestrator/e2e-tests/package.json`)
- **Plugin versions**: Resolved from catalog index at `quay.io/rhdh/plugin-catalog-index:<tag>`

For version compatibility details, see [references/version-matrix.md](references/version-matrix.md).

### Step 2: Deploy RHDH with orchestrator

**Option A — Full stack via setup script (recommended for manual testing):**

```bash
cd rhdh-test-instance
# Edit .env for OSL version pins if needed (see Step 2b)
./setup-orchestrator.sh <rhdh-version>
# Example: ./setup-orchestrator.sh 1.10
```

**Option B — Make targets:**

```bash
cd rhdh-test-instance
make setup VERSION=1.10
# With pre-release OSL mirrors:
make setup VERSION=1.10 MIRRORS=true
```

**Option C — E2E tests handle deployment automatically** (preferred for CI):
The orchestrator specs' `beforeAll` calls `deploySonataflow()` and `rhdh.deploy()`. No manual deploy needed when running e2e.

### Step 2b: Pre-release OSL versions

For testing unreleased OSL builds (requires VPN for internal registry):

1. Mirror images from internal registry:
   ```bash
   skopeo copy --all \
     docker://internal-registry/image \
     docker://quay.io/rh-ee-<user>/image
   ```

2. Update `rhdh-test-instance/config/image-mirrors.conf` with source/mirror pairs.

3. Set OSL overrides in `rhdh-test-instance/.env`:
   ```bash
   export OSL_IIB_IMAGE="quay.io/rh-ee-<user>/iib:<tag>"
   export OSL_VERSION="1.38"
   export OSL_LOGIC_PACKAGE="logic-operator"
   export OSL_LOGIC_CHANNEL="stable"
   export OSL_LOGIC_CSV="logic-operator.v1.38.0"
   export OSL_SERVERLESS_CHANNEL="stable-1.38"
   ```

4. Deploy with mirrors: `./setup-orchestrator.sh <version> --setup-mirrors`

For detailed OSL pinning procedures, see [references/version-matrix.md](references/version-matrix.md).

### Step 3: Run e2e tests

**From repo root (recommended — single Playwright process, correct dependency resolution):**

```bash
./run-e2e.sh -w orchestrator
```

Key environment variables:

| Variable | Default | Purpose |
|----------|---------|---------|
| `RHDH_VERSION` | `1.10` | RHDH Helm chart version |
| `INSTALLATION_METHOD` | `helm` | `helm` or `operator` |
| `E2E_TEST_UTILS_PATH` | (empty) | Local path to e2e-test-utils for development |
| `E2E_TEST_UTILS_VERSION` | (empty) | Override e2e-test-utils npm/git version |
| `SKIP_KEYCLOAK_DEPLOYMENT` | (empty) | Set to reuse existing Keycloak |
| `GIT_PR_NUMBER` | (empty) | PR number for plugin resolution |

**Filter specific tests:**

```bash
./run-e2e.sh -w orchestrator --grep "Greeting"
./run-e2e.sh -w orchestrator --grep "RBAC"
./run-e2e.sh -w orchestrator --project=orchestrator --workers=2
```

**List tests without running:**

```bash
./run-e2e.sh -w orchestrator --list
```

### Step 4: Run smoke tests

Smoke tests validate plugin loading without a cluster. Run after publishing plugins:

```bash
cd smoke-tests
npm install
 node smoke-test.ts \
  --plugins-yaml ../workspaces/orchestrator/smoke-tests/dynamic-plugins.test.yaml \
  --config app-config.yaml \
  --env-file ../workspaces/orchestrator/smoke-tests/test.env
```

Or trigger via PR comment: `/smoketest`

### Step 5: Evaluate results

**E2E pass criteria:**
- All specs in `orchestrator.spec.ts` pass (greeting workflow, failswitch, entity-workflow integration)
- All specs in `orchestrator-rbac.spec.ts` pass (global/per-workflow RBAC, instance visibility, admin override)
- Data index health checks recover without manual intervention

**Smoke pass criteria:**
- All backend plugins load and respond on `/api/<pluginId>`
- No startup crashes in Backstage backend

**Known skipped tests:**
- Token propagation API tests (`test.skip` in `orchestrator.spec.ts`)
- Cross-workflow link tests (`test.fixme`, needs OSL 1.37+)

## Workflow 2: Version Bump Testing

Use when bumping e2e-test-utils, Playwright, or plugin dependencies.

### Updating e2e-test-utils

The orchestrator workspace pins `@red-hat-developer-hub/e2e-test-utils` in `workspaces/orchestrator/e2e-tests/package.json`. To test a new version:

**Against a local build:**

```bash
E2E_TEST_UTILS_PATH=/path/to/rhdh-e2e-test-utils ./run-e2e.sh -w orchestrator
```

**Against a specific version:**

```bash
E2E_TEST_UTILS_VERSION=1.1.29 ./run-e2e.sh -w orchestrator
```

The `run-e2e.sh` script injects yarn resolutions to override the workspace's pinned version.

### Updating Playwright

The Playwright version is pinned in both `e2e-tests/package.json` (`@playwright/test`) and `run-e2e.sh` (resolution override). Both must match. Current pin: `1.57.0`.

## Workflow 3: Pre-release / Nightly Testing

### PR-based plugin testing

When orchestrator plugin changes are in a PR on this repo:

1. Comment `/publish` on the PR to publish test OCI images
2. Comment `/smoketest` to run smoke tests against published images
3. Comment `/test e2e-ocp-helm` to trigger cluster e2e

### Nightly mode

E2E runs with nightly plugin resolution when `JOB_NAME` starts with `periodic-` or `E2E_NIGHTLY_MODE=true`:

```bash
E2E_NIGHTLY_MODE=true ./run-e2e.sh -w orchestrator
```

## Test Architecture

### What the e2e tests deploy

The orchestrator `beforeAll` (`test.runOnce("orchestrator-setup")`) does:

1. Configure RHDH with Keycloak auth
2. Clone `rhdhorchestrator/serverless-workflows` repo
3. Apply greeting + failswitch SonataFlow manifests
4. Patch SonataFlowPlatform for PostgreSQL persistence
5. Align workflow container images to detected OSL version
6. Set `SONATAFLOW_DATA_INDEX_URL` env
7. Deploy RHDH via `rhdh.deploy()`

Both `orchestrator.spec.ts` and `orchestrator-rbac.spec.ts` share the same `runOnce` key, so deployment happens once per Playwright worker.

### Required cluster state

- OpenShift cluster (CRC or full) with `oc` logged in
- OpenShift Serverless and Serverless Logic operators installed
- Keycloak deployed (auto-deployed unless `SKIP_KEYCLOAK_DEPLOYMENT` set)

### Config files consumed by e2e

| File | Purpose |
|------|---------|
| `tests/config/app-config-rhdh.yaml` | RHDH app config (RBAC, catalog locations, orchestrator dataIndex URL) |
| `tests/config/rhdh-secrets.yaml` | K8s Secret (envsubst applied for `SONATAFLOW_DATA_INDEX_URL`) |
| `.env` | Local env loaded by Playwright config (e.g. `K8S_CLUSTER_ROUTER_BASE`) |

### Credentials

| Service | User | Password |
|---------|------|----------|
| Keycloak Admin | admin | admin123 |
| Primary test user | test1 | test1@123 |
| Secondary test user | test2 | test2@123 |
| OpenShift | kubeadmin | `crc console --credentials` |

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| Data index unhealthy mid-test | SonataFlow platform restart needed | `oc rollout restart deployment/sonataflow-platform-data-index-service -n <ns>` |
| RHDH pod stuck 0/1 | Plugin init failure or stale DB | Check pod logs; delete stale DB schemas |
| Helm upgrade "Job immutable" | Pre-existing SonataFlow DB init Job | `oc delete job -l app.kubernetes.io/instance=redhat-developer-hub -n rhdh` |
| `yarn install` resolution conflict | Mismatched e2e-test-utils version | Use `E2E_TEST_UTILS_PATH` or `E2E_TEST_UTILS_VERSION` |
| OSL ImagePullBackOff | Internal registry unreachable | Mirror with skopeo, deploy with `--setup-mirrors` |
| Namespace stuck Terminating | Knative CR finalizers | Delete KnativeServing/KnativeEventing CRs before their namespaces |
| Tests skip with "data index" | `ensureDataIndexOrSkip` health check failed | Verify SonataFlow platform pods are running |
| Browser login stuck | Self-signed cert not accepted | Visit Keycloak URL directly in browser, accept cert |
| Smoke test OCI pull fails | skopeo not installed or auth missing | `dnf install skopeo` and configure registry auth |

For detailed diagnostics, see [references/troubleshooting.md](references/troubleshooting.md).

## Additional Resources

- E2E test utilities docs: https://github.com/redhat-developer/rhdh-e2e-test-utils/tree/main/docs
- Serverless workflows repo: https://github.com/rhdhorchestrator/serverless-workflows
- CLAUDE.md in this repo for architecture and CI details
