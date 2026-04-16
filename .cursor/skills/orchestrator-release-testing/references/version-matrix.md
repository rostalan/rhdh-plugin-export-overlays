# Version Matrix and Compatibility

## Component Relationships

Orchestrator testing involves four independently versioned components:

```
RHDH Helm Chart (e.g. 1.10)
  └─ Catalog Index Image (quay.io/rhdh/plugin-catalog-index:<tag>)
       └─ Orchestrator plugin OCI images ({{inherit}} resolution)

OSL Operators (e.g. 1.37)
  ├─ OpenShift Serverless (knative-serving, knative-eventing)
  └─ Serverless Logic (logic-operator)
       └─ SonataFlow platform + data index

e2e-test-utils (e.g. 1.1.29)
  └─ install-orchestrator.sh (pins specific OSL CSV)
  └─ OrchestratorPage, helpers, fixtures

Playwright (e.g. 1.57.0)
  └─ Must match across workspace package.json and run-e2e.sh
```

## RHDH Version Selection

The `rhdh-test-instance/helm/deploy.sh` resolves chart versions from Quay:

| Input | Resolution |
|-------|-----------|
| `next` | Latest tag from `quay.io/rhdh/chart` API |
| `1.10` | Newest tag matching `^1.10-` |
| `1.10-200-CI` | Used as-is (CI build) |

The catalog index tag follows: `next` for version=next, else `<major>.<minor>` extracted from chart version.

## OSL Version Pinning

### Current defaults in e2e-test-utils

`install-orchestrator.sh` (v1.1.29) hard-pins:
- **Package**: `logic-operator`
- **Channel**: `stable`
- **Starting CSV**: `logic-operator.v1.37.2`
- **Serverless channel**: `stable`

### Manual OSL pinning via .env

Set in `rhdh-test-instance/.env`:

```bash
# Pin to specific GA version
export OSL_VERSION="1.37"
export OSL_LOGIC_PACKAGE="logic-operator"
export OSL_LOGIC_CHANNEL="stable"
export OSL_LOGIC_CSV="logic-operator.v1.37.2"
export OSL_SERVERLESS_CHANNEL="stable-1.37"

# Use latest GA (leave all empty)
export OSL_VERSION=""
```

### Pre-release OSL via IIB

```bash
export OSL_IIB_IMAGE="quay.io/rh-ee-<user>/iib:<tag>"
export OSL_VERSION="1.38"
export OSL_LOGIC_PACKAGE="logic-operator"
export OSL_LOGIC_CHANNEL="stable"
export OSL_LOGIC_CSV="logic-operator.v1.38.0"
export OSL_SERVERLESS_CHANNEL="stable-1.38"
```

When `OSL_IIB_IMAGE` is set, `setup-orchestrator.sh` creates a custom `CatalogSource` named `osl-custom-catalog`.

### Legacy vs current OSL package names

Since e2e-test-utils 1.1.26, the package and channel names changed:

| Field | Legacy (pre-1.1.26) | Current |
|-------|---------------------|---------|
| Package | `logic-operator-rhel8` | `logic-operator` |
| Channel | `alpha` | `stable` |

Ensure `.env` pins match the expected format for your e2e-test-utils version.

## e2e-test-utils Version Management

### How the workspace pins it

`workspaces/orchestrator/e2e-tests/package.json` has:
```json
"dependencies": {
  "@red-hat-developer-hub/e2e-test-utils": "<version-or-git-ref>"
}
```

This can be an npm version, a git branch ref (`user/repo#branch`), or a file path.

### Override mechanisms in run-e2e.sh

| Variable | Effect |
|----------|--------|
| `E2E_TEST_UTILS_PATH=/abs/path` | Builds local copy, adds `file:` resolution |
| `E2E_TEST_UTILS_VERSION=1.1.29` | Adds version resolution override |

Both inject yarn `resolutions` in the generated root `package.json`, overriding whatever the workspace pins.

### Recent orchestrator-relevant changes

| Version | Change |
|---------|--------|
| 1.1.29 | OLM wait uses label selectors; CSV pin → `logic-operator.v1.37.2`; git dep build support |
| 1.1.26 | OSL package/channel alignment to `stable` / `logic-operator` |
| 1.1.20 | `OrchestratorPage` page object added |
| 1.1.19 | `installOrchestrator()` function added |

## Plugin Version Resolution

Orchestrator plugin versions are not pinned explicitly. They use `{{inherit}}` in dynamic plugin configs, which resolves from the catalog index image at deploy time.

The catalog index image tag is derived from the RHDH version:
- `CATALOG_INDEX_TAG=next` for RHDH `next`
- `CATALOG_INDEX_TAG=1.10` for RHDH `1.10`

Override with: `export CATALOG_INDEX_TAG=<custom-tag>`

### PR-based resolution

When `GIT_PR_NUMBER` is set, the e2e deploy merges plugin metadata from the PR's published OCI artifacts, allowing testing of unreleased plugin changes.

### Nightly resolution

When `E2E_NIGHTLY_MODE=true` or `JOB_NAME` starts with `periodic-`, plugin resolution uses nightly builds.

## Compatibility Constraints

- Orchestrator plugins require matching data-index URL configuration
- RBAC tests require `permission.enabled: true` in RHDH config
- SonataFlow workflow images must align with the installed OSL version (the e2e setup patches these automatically)
- Switching RHDH versions without full cleanup causes database migration conflicts
- Playwright version must match between workspace `package.json` and `run-e2e.sh` resolution
