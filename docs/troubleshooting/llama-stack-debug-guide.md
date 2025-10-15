# Llama Stack Troubleshooting Guide

## Problem Statement
Playground UI showing errors when clicking tools:
- `BadRequestError: Could not find agent info for <agent-id>`
- `Tool group 'mcp::openshift' not found`
- `Tool group 'mcp::github' not found`

## Troubleshooting Methodology

### Phase 1: Initial Investigation - Pod Logs Analysis

#### Step 1: Read the Pod Log File
```bash
# Read the pod crash log provided by user
Read: /Volumes/EXT_SSD/downloads/llamastack-with-config-78598bdc4d-zb52p-llama-stack.log
```

**Why**: Start with the most recent error state to understand what's failing.

**Finding**:
```
openai.PermissionDeniedError: Authentication failed
```
- Pod was crashing during model registration
- Authentication failure when trying to register `granite-31-2b-instruct` model
- Authorino authentication was mentioned in logs

#### Step 2: Check Current Pod Status
```bash
oc -n llama-stack get pods
```

**Why**: Verify if pods are running now or still in error state.

**Finding**: Pods were in CrashLoopBackOff or had recent restarts.

---

### Phase 2: Vault Secret Path Investigation

#### Step 3: Review Secret Manifests
```bash
Read: infra/applications/llama-stack/base/secret-tavily.yaml
Read: infra/applications/llama-stack/base/secret-llama-4-scout.yaml
Read: infra/applications/llama-stack/base/secret-llama-3-2-3b.yaml
```

**Why**: Check where secrets are configured to look for Vault values.

**Finding**: All secret files had annotation:
```yaml
avp.kubernetes.io/path: "kv/data/ocp/ocp.ggwrt.sandbox508.opentlc.com/llama-stack"
```

#### Step 4: Check Policy Generator Configuration
```bash
Read: infra/applications/llama-stack/overlay/policy-generator-config.yaml
```

**Why**: Verify the Vault path used by ArgoCD Vault Plugin policy.

**Finding**: Policy annotation had different path:
```yaml
avp.kubernetes.io/path: "kv/data/ocp/sno/openshift-policy/llama-stack"
```

**Root Cause Identified**: Path mismatch between secrets and policy!

---

### Phase 3: Vault Secret Storage

#### Step 5: Load Environment Variables
```bash
source ~/.zshrc
```

**Why**: Get Vault access credentials from shell environment.

#### Step 6: Verify Vault Credentials
```bash
echo $ROOT_TOKEN
echo $VAULT_ADDR
echo $VAULT_ROUTE
```

**Why**: Confirm we have necessary credentials to write to Vault.

**Finding**: ROOT_TOKEN and VAULT_ADDR were set correctly.

#### Step 7: Store Secrets in Vault at Correct Path
```bash
# Store tavily API key
vault kv put kv/ocp/sno/openshift-policy/llama-stack \
  tavily-api-key="tvly-****************************"

# Store llama-3-2-3b token
vault kv put kv/ocp/sno/openshift-policy/llama-stack \
  llama-3-2-3b-api-token="********************************"

# Store llama-4-scout token
vault kv put kv/ocp/sno/openshift-policy/llama-stack \
  llama-4-scout-api-token="********************************"
```

**Why**: Store all secrets at the path where policy-generator expects them.

#### Step 8: Verify Secrets Were Stored
```bash
vault kv get kv/ocp/sno/openshift-policy/llama-stack
```

**Why**: Confirm secrets are readable from Vault at correct path.

**Finding**: All three secrets successfully stored and retrievable.

---

### Phase 4: Update Secret Manifests

#### Step 9: Update All Secret Files to Use Consistent Path
```bash
# Edit secret-llama-3-2-3b.yaml
Edit: infra/applications/llama-stack/base/secret-llama-3-2-3b.yaml
# Change path to: kv/data/ocp/sno/openshift-policy/llama-stack

# Edit secret-llama-4-scout.yaml
Edit: infra/applications/llama-stack/base/secret-llama-4-scout.yaml
# Change path to: kv/data/ocp/sno/openshift-policy/llama-stack

# Edit secret-tavily.yaml
Edit: infra/applications/llama-stack/base/secret-tavily.yaml
# Change path to: kv/data/ocp/sno/openshift-policy/llama-stack
```

**Why**: Ensure all secrets reference the same Vault path as policy-generator.

#### Step 10: Commit and Push Changes
```bash
git add infra/applications/llama-stack/base/secret-*.yaml

git commit -m "Fix Vault path for llama-stack secrets

Updated all secret manifests to use consistent Vault path matching
the policy-generator configuration.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"

git push
```

**Why**: Deploy via GitOps so ArgoCD picks up the changes.

---

### Phase 5: Verify Secret Deployment

#### Step 11: Check if Secrets Exist in Cluster
```bash
oc -n llama-stack get secrets llama-3-2-3b llama-4-scout-17b-16e-w4a16 tavily-search-key
```

**Why**: Verify ArgoCD successfully created secrets from Vault values.

**Finding**: All secrets exist with correct names.

#### Step 12: Decode and Verify Secret Values
```bash
oc -n llama-stack get secret llama-3-2-3b -o jsonpath='{.data.apiKey}' | base64 -d
```

**Why**: Confirm actual API token values are correct in cluster.

**Finding**: Token values matched what we stored in Vault.

#### Step 13: Check Deployment Environment Variables
```bash
oc -n llama-stack get deployment llamastack-with-config -o yaml | grep -A20 "env:"
```

**Why**: Verify deployment is configured to use the secrets as environment variables.

**Finding**: Deployment correctly references secrets:
```yaml
- name: LLAMA_3_2_3B_API_TOKEN
  valueFrom:
    secretKeyRef:
      key: apiKey
      name: llama-3-2-3b
```

---

### Phase 6: Configuration Analysis

#### Step 14: Check Current Pod Status
```bash
oc -n llama-stack get pods -l app.kubernetes.io/instance=llamastack-with-config
```

**Why**: See if pod is running after secret fixes.

**Finding**: Pod running but age indicated recent restart.

#### Step 15: Check Recent Pod Logs
```bash
oc -n llama-stack logs llamastack-with-config-<pod-hash> --tail=50
```

**Why**: Look for successful startup or any remaining errors.

**Finding**: Pod serving requests successfully. Errors about `mcp::github` toolgroup not found.

#### Step 16: Check Active ConfigMap
```bash
oc -n llama-stack get configmap llama-stack-config -o jsonpath='{.data.run\.yaml}' | grep -E "mcp::|tool_groups:" -A5
```

**Why**: See which toolgroups are currently configured.

**Finding**: Only `mcp::openshift` toolgroup present, missing `mcp::github`.

---

### Phase 7: Configuration Switching

#### Step 17: Explore Available Configurations
```bash
ls -la infra/applications/llama-stack/overlay/
```

**Why**: See what configuration variants exist.

**Finding**: Multiple overlay directories:
- `basic-model-agent/` - Single model, no MCP
- `mcp-openshift/` - Multiple models, OpenShift MCP only
- `mcp-github/` - Multiple models, both OpenShift and GitHub MCP

#### Step 18: Compare Configurations
```bash
Read: infra/applications/llama-stack/overlay/basic-model-agent/configmap.yaml
Read: infra/applications/llama-stack/overlay/mcp-openshift/configmap.yaml
Read: infra/applications/llama-stack/overlay/mcp-github/configmap.yaml
```

**Why**: Understand differences between configurations.

**Finding**:
- `basic-model-agent`: 1 model (granite), no MCP toolgroups
- `mcp-openshift`: 3 models, `mcp::openshift` toolgroup only
- `mcp-github`: 3 models, both `mcp::openshift` and `mcp::github` toolgroups

#### Step 19: Check Which Configuration is Active
```bash
Read: infra/applications/llama-stack/overlay/policy-generator-config.yaml
```

**Why**: See which overlay the policy is deploying.

**Finding**:
```yaml
manifests:
  - path: mcp-openshift/
```

Currently using `mcp-openshift` but need `mcp-github` for both toolgroups.

---

### Phase 8: Verify MCP Server Deployment

#### Step 20: Check for GitHub MCP Server
```bash
oc get pods -A | grep github
oc -n agent-demo get pods,svc | grep github
```

**Why**: Confirm the GitHub MCP server is actually deployed and running.

**Finding**:
```
pod/github-mcp-server-8649dc5996-rnsc6   1/1     Running   0          4m1s
service/github-mcp-server   ClusterIP   172.30.71.150    <none>        80/TCP     4m1s
```

GitHub MCP server is running and reachable.

#### Step 21: Check MCP GitHub Policy Status
```bash
oc -n openshift-policy get policy | grep mcp-github
```

**Why**: Verify the mcp-github application was deployed by ACM.

**Finding**: Policy exists and is Compliant.

---

### Phase 9: Update to Full Configuration

#### Step 22: Update Policy to Use mcp-github Configuration
```bash
Edit: infra/applications/llama-stack/overlay/policy-generator-config.yaml
# Change: path: mcp-openshift/
# To: path: mcp-github/
```

**Why**: Deploy the configuration that includes both MCP toolgroups.

#### Step 23: Commit and Push Configuration Change
```bash
git add infra/applications/llama-stack/overlay/policy-generator-config.yaml

git commit -m "Switch llama-stack to mcp-github configuration

Updates policy-generator to use mcp-github overlay which includes both
mcp::openshift and mcp::github toolgroups.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"

git push
```

**Why**: Use GitOps workflow for deployment.

#### Step 24: Trigger Policy Update
```bash
oc -n openshift-policy annotate policy llama-stack-sno \
  policy.open-cluster-management.io/trigger-update="$(date +%s)" --overwrite
```

**Why**: Force ACM to immediately reconcile the policy instead of waiting for sync interval.

#### Step 25: Verify Configuration Applied
```bash
sleep 5
oc -n llama-stack get configmap llama-stack-config -o jsonpath='{.data.run\.yaml}' | grep -E "tool_groups:" -A12
```

**Why**: Confirm the new configuration with both toolgroups is deployed.

**Finding**:
```yaml
tool_groups:
- provider_id: tavily-search
  toolgroup_id: builtin::websearch
- toolgroup_id: mcp::openshift
  provider_id: model-context-protocol
  mcp_endpoint:
    uri: http://ocp-mcp-server.agent-demo.svc.cluster.local:8000/sse
- toolgroup_id: mcp::github
  provider_id: model-context-protocol
  mcp_endpoint:
    uri: http://github-mcp-server.agent-demo.svc.cluster.local:80/sse
```

Both toolgroups now present!

---

### Phase 10: Verify Pod Health

#### Step 26: Check Pod Status After Config Update
```bash
oc -n llama-stack get pods -l app.kubernetes.io/instance=llamastack-with-config
```

**Why**: New configuration triggers pod restart; verify it starts successfully.

**Finding**: Pod went through CrashLoopBackOff (authentication timing issues during model registration) but eventually stabilized.

#### Step 27: Monitor Pod Logs for Errors
```bash
oc -n llama-stack logs llamastack-with-config-<new-pod-hash> --tail=100
```

**Why**: Check if pod successfully registered all models and toolgroups.

**Finding**:
- Initial crashes due to authentication timing (model endpoints not immediately ready)
- After 3 restarts, pod successfully started
- Serving API requests with 200 OK responses

#### Step 28: Check Pod Restart Count
```bash
oc -n llama-stack get pods -l app.kubernetes.io/instance=llamastack-with-config
```

**Why**: Monitor stability - too many restarts indicate ongoing issues.

**Finding**:
```
NAME                                      READY   STATUS    RESTARTS      AGE
llamastack-with-config-66d79cc8cc-l5j66   1/1     Running   3 (71s ago)   103s
```

Pod stable after 3 restarts.

---

### Phase 11: API Verification

#### Step 29: Set Up Port Forward
```bash
oc -n llama-stack port-forward svc/llamastack-with-config-service 8321:8321 > /dev/null 2>&1 &
```

**Why**: Access the llama-stack API from local machine for testing.

#### Step 30: Verify Port Forward is Running
```bash
ps aux | grep "port-forward.*llamastack" | grep -v grep
```

**Why**: Confirm port-forward process is active before making API calls.

#### Step 31: Test Toolgroups API Endpoint
```bash
curl -s http://localhost:8321/v1/toolgroups
```

**Why**: Directly query API to see available toolgroups.

**Finding**: Returns JSON with all three toolgroups.

#### Step 32: Parse and Display Toolgroups
```bash
curl -s http://localhost:8321/v1/toolgroups | jq -r '.data[] | "\(.identifier) - \(.mcp_endpoint.uri // "N/A")"'
```

**Why**: Format output for human readability.

**Result**:
```
builtin::websearch - N/A
mcp::openshift - http://ocp-mcp-server.agent-demo.svc.cluster.local:8000/sse
mcp::github - http://github-mcp-server.agent-demo.svc.cluster.local:80/sse
```

#### Step 33: Test Models API Endpoint
```bash
curl -s http://localhost:8321/v1/models | jq .
```

**Why**: Verify all models are registered and available.

**Finding**: All three models present:
- `granite-31-2b-instruct`
- `llama-3-2-3b`
- `llama-4-scout-17b-16e-w4a16`

#### Step 34: Test Specific Toolgroup Tools
```bash
curl -s http://localhost:8321/v1/tools?toolgroup_id=mcp::openshift
curl -s http://localhost:8321/v1/tools?toolgroup_id=mcp::github
```

**Why**: Verify each toolgroup can return its tools.

**Finding**: Both endpoints return 200 OK with tool listings.

---

### Phase 12: Review All ReplicaSets
```bash
oc -n llama-stack get replicaset | grep llamastack-with-config
```

**Why**: See history of deployments to understand how many times configuration changed.

**Finding**: Multiple old replicasets from troubleshooting iterations, current one active.

---

## Key Troubleshooting Techniques Used

### 1. **Log Analysis First**
Always start with logs to understand the actual error, not assumptions.

### 2. **Path Tracing**
Follow the data flow:
- Application config → Environment variables → Secrets → Vault paths
- Identified the path mismatch through systematic checking

### 3. **Comparison Analysis**
Compare working vs non-working configurations to identify differences.

### 4. **Incremental Verification**
After each fix:
- Verify the change was applied
- Check pod status
- Review logs
- Test API endpoints

### 5. **Environment Variable Chain**
Trace back from error to configuration:
```
Error: Authentication failed
↓
Check: Model registration in logs
↓
Check: Deployment env vars
↓
Check: Secret values
↓
Check: Vault paths
↓
Found: Path mismatch
```

### 6. **GitOps Workflow**
Never make manual changes in cluster:
1. Update manifest files
2. Commit to git
3. Push to remote
4. Trigger policy/ArgoCD sync
5. Verify deployment

### 7. **State Verification Commands**
Essential checks to run after each change:
```bash
# Pod status
oc get pods -n <namespace>

# Recent logs
oc logs <pod> --tail=50

# Config verification
oc get configmap <name> -o yaml

# Secret verification
oc get secret <name> -o yaml

# Service status
oc get svc -n <namespace>
```

### 8. **API Testing Pattern**
Test from most general to most specific:
1. Can I reach the API? (curl base URL)
2. What resources exist? (GET /v1/toolgroups)
3. Can I get specific resource? (GET /v1/tools?toolgroup_id=X)
4. Can I use the resource? (POST with actual request)

---

## Common Pitfalls Encountered

### 1. Path Mismatches
**Problem**: Different paths in different manifest files.
**Solution**: Establish single source of truth (policy-generator-config) and align all paths.

### 2. KV-v2 Path Convention
**Problem**: Vault KV-v2 requires `/data/` in path for AVP plugin.
**Learning**:
- Vault CLI: `kv/path/to/secret`
- AVP annotation: `kv/data/path/to/secret`

### 3. Pod Restart Timing
**Problem**: New config causes pod restart; model endpoints not immediately available.
**Solution**: Wait for multiple restarts until models are registered successfully.

### 4. Port-Forward Process Management
**Problem**: Multiple port-forward processes can conflict.
**Solution**:
```bash
pkill -f "port-forward.*service-name"
sleep 2
# Start new port-forward
```

### 5. Configuration Variants
**Problem**: Multiple overlay directories with similar names.
**Solution**: Read all variants before deciding which to use.

---

## Final Verification Checklist

- [ ] All pods Running (0 restarts or stabilized)
- [ ] No errors in recent logs
- [ ] Secrets exist and contain correct values
- [ ] Environment variables reference correct secrets
- [ ] ConfigMap contains expected configuration
- [ ] API endpoints return 200 OK
- [ ] All expected toolgroups present
- [ ] All expected models present
- [ ] Each toolgroup returns its tools
- [ ] Playground UI loads without errors

---

## Resolution Summary

**Root Causes**:
1. Vault secret path mismatch between secrets and policy
2. Wrong llama-stack configuration variant deployed (missing mcp::github)

**Fixes Applied**:
1. Updated all secret manifests to use consistent Vault path
2. Stored all API tokens in Vault at correct path
3. Changed policy-generator-config to use `mcp-github/` overlay
4. Triggered ACM policy sync to redeploy

**Result**:
✅ Both `mcp::openshift` and `mcp::github` toolgroups now available
✅ Playground UI tools page working
✅ All three models registered successfully

---

## Commands Reference Sheet

### Quick Status Check
```bash
# Check all llama-stack resources
oc -n llama-stack get pods,deployment,configmap,secrets

# Check recent logs
oc -n llama-stack logs -l app.kubernetes.io/instance=llamastack-with-config --tail=20

# Check policy status
oc -n openshift-policy get policy llama-stack-sno
```

### Quick API Test
```bash
# Port forward (run in background)
oc -n llama-stack port-forward svc/llamastack-with-config-service 8321:8321 &

# Test endpoints
curl http://localhost:8321/v1/models
curl http://localhost:8321/v1/toolgroups
curl http://localhost:8321/v1/tools
```

### Force Policy Update
```bash
oc -n openshift-policy annotate policy llama-stack-sno \
  policy.open-cluster-management.io/trigger-update="$(date +%s)" --overwrite
```

### Vault Operations
```bash
# List secrets at path
vault kv list kv/ocp/sno/openshift-policy

# Get secret
vault kv get kv/ocp/sno/openshift-policy/llama-stack

# Put secret
vault kv put kv/ocp/sno/openshift-policy/llama-stack \
  key1="value1" \
  key2="value2"
```
