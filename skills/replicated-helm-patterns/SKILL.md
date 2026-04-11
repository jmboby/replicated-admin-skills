---
name: replicated-helm-patterns
description: Use when working with Helm charts for Replicated distribution — operator CRD ordering, post-install hooks, webhook timing, subchart dependencies, and KOTS manifest structure
---

# Replicated Helm Patterns

## Overview

Patterns and solutions for common Helm chart challenges when distributing via Replicated — particularly around CRD ordering, operator webhooks, and KOTS manifest structure.

## When to Use

- Adding an operator (like CloudNativePG) as a subchart
- Dealing with CRD chicken-and-egg problems
- Structuring KOTS manifests for Replicated releases
- Troubleshooting webhook timing issues during Helm install

## CRD Chicken-and-Egg Problem

**Problem:** When an operator subchart installs CRDs and your chart creates a CR (Custom Resource) in the same release, Helm validates ALL manifests against the API server before applying any. The CRDs don't exist yet → validation fails.

**Error:**
```
resource mapping not found for name: "my-db" namespace: "" from "":
no matches for kind "Cluster" in version "postgresql.cnpg.io/v1"
ensure CRDs are installed first
```

**Solution:** Make the CR a `post-install`-only hook so it's applied after the main release (including CRDs):

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: {{ include "myapp.fullname" . }}-db
  annotations:
    "helm.sh/hook": post-install
    "helm.sh/hook-weight": "10"
spec:
  instances: {{ .Values.postgresql.instances }}
  # ...
```

**Critical warnings:**
- **Never use `post-upgrade`** on Cluster CRs. Adding `post-upgrade` causes Helm to re-create the resource on every upgrade, which can wipe stored data. Use `post-install` only.
- **Never use `--wait` with post-install hooks.** Helm `--wait` waits for all pods to be Ready before running hooks. If any pod depends on the hook-created resource (e.g., an app pod waiting for the database CR), this deadlocks: `--wait` blocks hooks → hooks never create the DB → pods never ready → `--wait` never returns.

## Operator Webhook Timing

**Problem:** Even with the CR as a post-install hook, the operator's webhook may not be ready when the hook fires. The operator deployment needs time to start and register its webhook endpoints.

**Error:**
```
failed calling webhook "mcluster.cnpg.io": no endpoints available for service "cnpg-webhook-service"
```

**Solution:** Add a wait job as a post-install hook with lower weight (runs before the CR hook):

```yaml
{{- if and .Values.postgresql.enabled .Values.cloudnativepg.enabled }}
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "myapp.fullname" . }}-wait-operator
  annotations:
    "helm.sh/hook": post-install,post-upgrade
    "helm.sh/hook-weight": "1"
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  backoffLimit: 3
  template:
    spec:
      restartPolicy: OnFailure
      imagePullSecrets:
        {{- include "myapp.imagePullSecrets" . | nindent 8 }}
      containers:
        - name: wait
          image: busybox:1.36
          command:
            - sh
            - -c
            - |
              echo "Waiting for operator webhook..."
              for i in $(seq 1 60); do
                if nc -z cnpg-webhook-service.{{ .Release.Namespace }}.svc 443 2>/dev/null; then
                  echo "Webhook ready after $i attempts"
                  sleep 5
                  exit 0
                fi
                echo "Attempt $i/60 — not ready..."
                sleep 5
              done
              echo "Timed out"
              exit 1
{{- end }}
```

**Hook ordering:**
1. Weight 0: RBAC resources (if needed)
2. Weight 1: Wait job (polls webhook endpoint)
3. Weight 10: CR creation (operator processes it)

**Key details:**
- Use `busybox` with `nc -z` for TCP check — no RBAC or kubectl needed
- Add `imagePullSecrets` if images go through a proxy
- Extra `sleep 5` after webhook is reachable for initialization
- `hook-delete-policy: before-hook-creation,hook-succeeded` cleans up old jobs

## KOTS Manifest Structure

The `replicated/` directory contains manifests for KOTS/EC installs alongside the packaged Helm chart:

```
replicated/
  my-chart.yaml         # HelmChart CR — maps values for KOTS Config Screen
  kots-app.yaml          # Application CR — icon, status informers, ports
  drone-rx-1.0.0.tgz    # Packaged chart (generated at release time)
```

### HelmChart CR

```yaml
apiVersion: kots.io/v1beta2
kind: HelmChart
metadata:
  name: my-app
spec:
  chart:
    name: my-app
    chartVersion: $VERSION
  values:
    api:
      image:
        repository: images.example.com/proxy/my-app/ghcr.io/myorg/my-api
        tag: $VERSION
```

### Application CR

```yaml
apiVersion: kots.io/v1beta1
kind: Application
metadata:
  name: my-app
spec:
  title: My App
  icon: https://example.com/icon.svg
  statusInformers:
    - deployment/my-app-api
    - deployment/my-app-frontend
  ports:
    - serviceName: my-app-frontend
      servicePort: 3000
      localPort: 3000
```

## The .replicated Config File

Place at repo root — the `replicated` CLI reads this automatically:

```yaml
appSlug: "my-app"
charts:
  - path: ./chart
manifests:
  - ./replicated/*.yaml
```

## BYO (Bring Your Own) Pattern

For stateful components with embedded default + external opt-in:

```yaml
# values.yaml
postgresql:
  enabled: true    # Deploy embedded DB

externalDatabase:
  host: ""
  port: 5432
  name: "myapp"
  user: "myapp"
  password: ""
```

```yaml
# Cluster CR — conditional on embedded mode
{{- if .Values.postgresql.enabled }}
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
# ...
{{- end }}
```

```yaml
# Database URL helper — switches based on mode
{{- define "myapp.databaseURL" -}}
{{- if .Values.postgresql.enabled }}
postgres://myapp:$(DB_PASSWORD)@{{ include "myapp.fullname" . }}-db-rw:5432/myapp
{{- else }}
postgres://{{ .Values.externalDatabase.user }}:$(DB_PASSWORD)@{{ .Values.externalDatabase.host }}:{{ .Values.externalDatabase.port }}/{{ .Values.externalDatabase.name }}
{{- end }}
{{- end }}
```

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| CR in same release as operator CRDs | Make CR a post-install hook |
| Webhook timeout on hook | Add wait job with `nc -z` at lower hook weight |
| Using `--auto` with `.replicated` config | Don't — `--auto` ignores `.replicated` and defaults to `./manifests` |
| Hook jobs missing imagePullSecrets | Add the helper — hook pods need pull secrets too |
| `helm.sh/hook-weight` doesn't wait | Correct — it only orders, doesn't wait. Use a Job to actually wait |
| `post-install,post-upgrade` on Cluster CRs | Use `post-install` only — `post-upgrade` re-creates the resource, potentially wiping data |
| `helm install --wait` with post-install hooks | Remove `--wait` — deadlocks when pods depend on hook-created resources |
