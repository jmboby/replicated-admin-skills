---
name: replicated-troubleshoot
description: Use when adding Preflight checks or Support Bundle specs to a Helm chart for Replicated distribution — covers Secret-based delivery, in-app bundle generation via SDK, collector patterns, file path conventions, and common pitfalls
---

# Replicated Troubleshoot Patterns

## Overview

Patterns for adding Preflight checks and Support Bundle specs to Helm charts distributed via Replicated. Covers Secret-based delivery (recommended for both Helm CLI and KOTS v1.94.2+), in-app support bundle generation and upload, collector types, and file path conventions.

## When to Use

- Adding preflight checks to a Replicated Helm chart
- Adding support bundle collectors and analyzers
- Building in-app support bundle generation (e.g., admin page)
- Debugging "No matching files" errors in textAnalyze
- Troubleshooting CRD-not-found errors during helm install

## Spec Delivery: Secrets (Recommended)

Per Replicated docs, the recommended approach for **both Helm CLI and KOTS v1.94.2+** is to deliver specs as Kubernetes Secrets. Do NOT use standalone CRD resources or `.Capabilities.APIVersions` gates.

### 1. Named Templates (Shared Spec)

Define specs as named templates in `_preflight.tpl` and `_supportbundle.tpl`:

```gotemplate
{{- define "myapp.preflight" -}}
apiVersion: troubleshoot.sh/v1beta2
kind: Preflight
metadata:
  name: {{ include "myapp.fullname" . }}-preflight
spec:
  # collectors and analyzers here
{{- end }}
```

### 2. Secret Manifests

Render the spec inside a Secret with the correct label and data key:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: {{ include "myapp.fullname" . }}-preflight
  labels:
    troubleshoot.sh/kind: preflight
stringData:
  preflight-spec: |
    {{ include "myapp.preflight" . | nindent 4 }}
```

For support bundles:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: {{ include "myapp.fullname" . }}-supportbundle
  labels:
    troubleshoot.sh/kind: support-bundle
stringData:
  support-bundle-spec: |
    {{ include "myapp.supportbundle" . | nindent 4 }}
```

**Critical:** The data key MUST be `preflight-spec` or `support-bundle-spec`. Arbitrary names like `preflight.yaml` will not be discovered. Verify by checking the SDK's own secret: `kubectl get secret <sdk>-supportbundle -o jsonpath='{.data}' | base64 -d`.

### 3. Running from CLI

```bash
# Preflights
helm template myapp ./chart \
  --show-only templates/preflight.yaml \
  | yq '.stringData."preflight-spec"' \
  | kubectl preflight -

# Support bundles
helm template myapp ./chart \
  --show-only templates/supportbundle.yaml \
  | yq '.stringData."support-bundle-spec"' \
  | kubectl support-bundle -
```

## In-App Support Bundle Generation (SDK Upload)

**Note:** This is an advanced pattern for Helm CLI installs where the KOTS admin console is not available. In standard Replicated deployments (EC v3 / KOTS), customers generate support bundles from the Embedded Cluster admin console — no in-app integration is needed. This pattern is primarily useful for bootcamp exercises or Helm-only distributions where you want bundle generation from within the app UI.

### Architecture

1. Embed `support-bundle` CLI binary in the app container image
2. Go handler execs a shell script that collects and uploads
3. The CLI discovers specs from Secrets via `--load-cluster-specs`
4. Upload the tarball to the local SDK endpoint with `curl`

### Upload to SDK

The SDK's `POST /api/v1/supportbundle` endpoint accepts a gzipped tarball. **You must use `curl`** — busybox `wget --post-file` returns 400.

```bash
curl -sf -X POST \
  -H "Content-Type: application/gzip" \
  -H "Content-Length: $(wc -c < /tmp/bundle.tar.gz | tr -d ' ')" \
  --data-binary @/tmp/bundle.tar.gz \
  http://myapp-sdk:3000/api/v1/supportbundle
```

**Do NOT use `--auto-upload`** — it targets `replicated.app` (cloud) and requires a license ID. The local SDK endpoint handles forwarding to the Vendor Portal.

### Handler Script Pattern

```go
script := fmt.Sprintf(`rm -f /tmp/support-bundle.tar.gz
support-bundle --load-cluster-specs -n %s -o /tmp/support-bundle.tar.gz >/dev/null 2>&1
curl -sf -X POST \
  -H "Content-Type: application/gzip" \
  -H "Content-Length: $(wc -c < /tmp/support-bundle.tar.gz | tr -d ' ')" \
  --data-binary @/tmp/support-bundle.tar.gz \
  %s/api/v1/supportbundle
rm -f /tmp/support-bundle.tar.gz`, namespace, sdkURL)
```

**Key details:**
- `rm -f` before collecting — the CLI appends `(1)` if file exists
- `>/dev/null 2>&1` — suppress JSON analysis dump to stdout
- No `set -e` — the CLI exits non-zero when analyzers have warnings, which would kill the upload
- Add `curl` to the Docker image: `apk add --no-cache curl`

### Required RBAC

The service account needs broad read permissions plus exec/create for collectors:

```yaml
# Role (namespace-scoped)
rules:
  - apiGroups: [""]
    resources: ["secrets", "configmaps", "pods", "pods/log", "pods/exec",
                "services", "events", "persistentvolumeclaims"]
    verbs: ["get", "list", "create", "delete"]
  - apiGroups: ["apps"]
    resources: ["deployments", "statefulsets", "replicasets"]
    verbs: ["get", "list"]
  - apiGroups: ["batch"]
    resources: ["jobs"]
    verbs: ["get", "list"]
  - apiGroups: ["networking.k8s.io"]
    resources: ["ingresses"]
    verbs: ["get", "list"]
  - apiGroups: ["rbac.authorization.k8s.io"]
    resources: ["roles", "rolebindings"]
    verbs: ["get", "list"]

# ClusterRole (cluster-scoped)
rules:
  - apiGroups: [""]
    resources: ["nodes", "namespaces", "persistentvolumes"]
    verbs: ["get", "list"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list"]
  - apiGroups: ["rbac.authorization.k8s.io"]
    resources: ["clusterroles", "clusterrolebindings"]
    verbs: ["get", "list"]
```

**`pods/exec` + `create`** are critical — the SDK's own support bundle spec uses exec collectors to curl the SDK API for `app-info.json`. Without these permissions, the Vendor Portal shows "No app-info file found".

## Collector File Path Conventions

**Critical:** Different collector types and contexts use different output paths.

| Collector | Context | Output Path |
|-----------|---------|------------|
| `run` | Preflight | `<collectorName>.log` |
| `run` | Support bundle | `<collectorName>/<collectorName>.log` |
| `logs` | Support bundle | `<name>/<pod>/<container>.log` |
| `http` | Support bundle | `<collectorName>.json` |
| `exec` | Support bundle | `<collectorName>/<namespace>/<pod>/<collectorName>-stdout.txt` |

**Always verify paths empirically** — extract the actual bundle and check:
```bash
tar -xzf support-bundle-*.tar.gz
find support-bundle-*/ -type f | grep <collectorName>
```

## Collector Pitfalls

### exec collector can't target its own pod

When `support-bundle` runs inside a pod (e.g., from an admin handler), the `exec` collector silently fails if it targets the same pod. No error, no output.

**Fix:** Use `http` collector instead. The http collector runs from inside the cluster and can reach the in-cluster service FQDN:

```yaml
- http:
    collectorName: myapp-health
    get:
      url: http://myapp-api.myns.svc.cluster.local:8080/healthz
```

### http collector runs in-cluster

The `http` collector makes requests **from inside the cluster**, not from the client machine. In-cluster DNS resolves correctly. It only fails when running `kubectl support-bundle` from a local machine outside the cluster.

### Empty collectors section triggers defaults

An empty `collectors:` or `collectors: []` causes the tool to run all default collectors (cluster info, pod logs, etc.), making collection very slow.

**Fix:** Omit `collectors:` entirely when no conditional collectors are active.

### textAnalyze glob matches multiple files

A glob like `myapp/api-logs/*/*` matches every file under that path. If a pod has multiple containers (e.g., `api.log` + `wait-for-db.log`), the analyzer runs once per file, producing duplicate results.

**Fix:** Be specific: `myapp/api-logs/*/api.log`

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Using standalone CRD resources | Use Secrets with `troubleshoot.sh/kind` label — works everywhere |
| Secret data key `support-bundle.yaml` | Must be `support-bundle-spec` or `preflight-spec` |
| Using `--auto-upload` for SDK | Targets replicated.app. Use `curl --data-binary` to local SDK |
| Using busybox `wget --post-file` for SDK upload | Returns 400. Use `curl --data-binary` |
| Using `set -e` in support-bundle wrapper | CLI exits non-zero on warnings. Kills upload step |
| `exec` collector targeting own pod | Silently fails. Use `http` collector instead |
| Missing `pods/exec` RBAC | SDK exec collectors fail silently. No app-info.json in bundle |
| Using `<collectorName>/stdout.txt` in preflight | Use `<collectorName>.log` for preflight run collectors |
| Empty `collectors:` section | Omit entirely when no collectors are active |
| Broad log glob producing duplicates | Narrow to specific container: `*/api.log` not `*/*` |
