# Troubleshooting Orchestrator Tests

## Deployment Issues

### RHDH pod stuck at 0/1 Running

**Symptoms:** Pod shows Running but 0/1 ready. Readiness probe fails.

**Diagnosis:**
```bash
oc logs deployment/redhat-developer-hub -n rhdh -c redhat-developer-hub --tail=100
```

**Common causes:**
- Plugin initialization failure (missing config, incompatible versions)
- Stale database schemas from previous version

**Fix:**
```bash
# Check for stale schemas
oc exec $(oc get pod -l app.kubernetes.io/name=backstage -n rhdh -o name) -n rhdh -- \
  psql -U postgres -c "\dt"

# Nuclear option: delete PVC and redeploy
oc delete pvc -l app.kubernetes.io/instance=redhat-developer-hub -n rhdh
cd rhdh-test-instance && ./setup-orchestrator.sh <version>
```

### Helm upgrade "Job is immutable"

**Cause:** SonataFlow database init Job from previous deploy blocks upgrade.

**Fix:**
```bash
oc delete job -l app.kubernetes.io/instance=redhat-developer-hub -n rhdh
# Then retry deploy
```

### OSL operator ImagePullBackOff

**Cause:** Images from internal registry not accessible from cluster.

**Fix:**
```bash
# 1. Connect to VPN
# 2. Mirror images
skopeo copy --all \
  docker://brew.registry.redhat.io/rh-osbs/openshift-serverless-1-logic-rhel8-operator:<tag> \
  docker://quay.io/rh-ee-<user>/logic-rhel8-operator:<tag>

# 3. Update config/image-mirrors.conf
# 4. Redeploy with mirrors
./setup-orchestrator.sh <version> --setup-mirrors
```

### Namespace stuck in Terminating

**Cause:** Knative Custom Resource finalizers prevent namespace deletion.

**Fix:**
```bash
# Delete Knative CRs first
oc delete knativeserving --all -n knative-serving
oc delete knativeeventing --all -n knative-eventing

# If still stuck, patch finalizers
oc patch namespace <ns> -p '{"metadata":{"finalizers":null}}' --type=merge
```

### CRC DiskPressure

**Cause:** Container images filling CRC disk.

**Fix:**
```bash
# SSH into CRC node and prune
ssh -i ~/.crc/machines/crc/id_ecdsa core@$(crc ip)
sudo crictl rmi --prune
```

### "database does not exist" for SonataFlow

**Cause:** SonataFlow database init Job failed or was interrupted.

**Fix:**
```bash
PG_POD=$(oc get pod -l app=backstage-psql -n <ns> -o name)
oc exec $PG_POD -n <ns> -- psql -U postgres -c "CREATE DATABASE sonataflow;"
```

## E2E Test Failures

### Data index unhealthy — tests skip

**Symptom:** Tests skip with message about data index health check.

**Cause:** `ensureDataIndexOrSkip` in `beforeEach` checks `sonataflow-platform-data-index-service` health. If unhealthy, entire describe block skips.

**Diagnosis:**
```bash
NS=$(oc get project -l app.kubernetes.io/part-of=sonataflow -o name | head -1 | cut -d/ -f2)
oc get pods -n $NS | grep data-index
oc logs deployment/sonataflow-platform-data-index-service -n $NS --tail=50
```

**Fix:**
```bash
oc rollout restart deployment/sonataflow-platform-data-index-service -n $NS
# Wait for rollout
oc rollout status deployment/sonataflow-platform-data-index-service -n $NS --timeout=120s
```

### Keycloak login stuck in browser

**Cause:** Self-signed certificates not accepted in test browser.

**Fix:**
1. Get Keycloak URL: `oc get route keycloak -n rhdh-keycloak -o jsonpath='{.spec.host}'`
2. Visit URL in real browser, accept certificate
3. Ensure popups are allowed for the RHDH URL

### RBAC tests fail with permission errors

**Cause:** Baseline role setup/cleanup timing. `orchestrator.spec.ts` adds a baseline role; `orchestrator-rbac.spec.ts` removes it. Running both in unexpected order causes state conflicts.

**Fix:** Run RBAC tests in isolation:
```bash
./run-e2e.sh -w orchestrator --grep "RBAC"
```

### Tests fail with "entity not found" for catalog

**Cause:** Catalog locations in `tests/config/app-config-rhdh.yaml` point to GitHub repos that must be accessible.

**Diagnosis:**
```bash
# Check if template repos are accessible
curl -s -o /dev/null -w "%{http_code}" \
  https://github.com/testetson22/greeting_54mjks/blob/main/templates/greeting/greeting.yaml
```

### Stale browser context from previous run

**Cause:** `SKIP_KEYCLOAK_DEPLOYMENT` reuses Keycloak but sessions may have expired.

**Fix:** Either remove `SKIP_KEYCLOAK_DEPLOYMENT` or restart Keycloak:
```bash
oc rollout restart deployment/keycloak -n rhdh-keycloak
```

## Smoke Test Failures

### skopeo copy fails

**Cause:** Missing authentication for OCI registry.

**Fix:**
```bash
# Login to registry
skopeo login quay.io
# Or for Red Hat registry
skopeo login registry.redhat.io
```

### Backend fails to start

**Cause:** Missing plugin configuration. The smoke harness merges `pluginConfig` from plugin metadata but may lack runtime dependencies (data index URL, etc.).

**Fix:** Ensure `test.env` provides all required URLs:
```bash
# workspaces/orchestrator/smoke-tests/test.env
LOKI_BASE_URL=https://example_url
SONATAFLOW_DATA_INDEX_URL=http://localhost:9999
```

### Plugin ID not found (404 on /api/pluginId)

**Cause:** Backend plugin loaded but route not registered. May indicate missing config or dependency.

**Note:** The smoke test treats 404 as a warning, not a failure. Check `results.json` for details.

## Dependency Resolution Issues

### yarn install fails with resolution conflicts

**Cause:** Conflicting versions of `@playwright/test` or `@red-hat-developer-hub/e2e-test-utils`.

**Fix:**
```bash
# Clean everything
rm -rf node_modules yarn.lock workspaces/orchestrator/e2e-tests/node_modules

# Use run-e2e.sh which handles resolutions
./run-e2e.sh -w orchestrator
```

### e2e-test-utils git ref not found

**Cause:** `package.json` points to a branch that was deleted or rebased.

**Fix:** Update the dependency to a stable version:
```json
"@red-hat-developer-hub/e2e-test-utils": "^1.1.29"
```

Or use the override:
```bash
E2E_TEST_UTILS_VERSION=1.1.29 ./run-e2e.sh -w orchestrator
```

## Diagnostic Commands

```bash
# Cluster overview
oc get pods -n rhdh
oc get pods -n orchestrator
oc get sonataflow --all-namespaces
oc get csv --all-namespaces | grep -E "logic|serverless"

# RHDH health
RHDH_URL=$(oc get route redhat-developer-hub -n rhdh -o jsonpath='{.spec.host}')
curl -sk "https://$RHDH_URL/.backstage/health/v1/readiness"

# Data index health
oc port-forward svc/sonataflow-platform-data-index-service 8080:80 -n orchestrator &
curl -s http://localhost:8080/q/health/ready

# SonataFlow status
oc get sonataflow -n orchestrator -o wide
oc get sonataflowplatform -n orchestrator -o yaml

# Operator status
oc get csv -n openshift-operators | grep -E "logic|serverless"
```
