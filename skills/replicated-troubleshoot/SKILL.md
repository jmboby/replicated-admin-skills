---
name: replicated-troubleshoot
description: Use when adding Preflight checks or Support Bundle specs to a Helm chart for Replicated distribution — covers dual-mode delivery (KOTS CRDs + Helm Secrets), collector patterns, file path conventions, and common pitfalls
---

# Replicated Troubleshoot Patterns

## Overview

Patterns for adding Preflight checks and Support Bundle specs to Helm charts distributed via Replicated. Covers the dual-mode delivery pattern (KOTS/EC vs Helm CLI), collector types, analyzer patterns, and file path conventions.

## When to Use

- Adding preflight checks to a Replicated Helm chart
- Adding support bundle collectors and analyzers
- Debugging "No matching files" errors in textAnalyze
- Troubleshooting CRD-not-found errors during helm install

## Dual-Mode Delivery

Troubleshoot CRDs (`kind: Preflight`, `kind: SupportBundle`) only exist in KOTS/EC environments. Plain Helm installs don't have them. You need both delivery modes:

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

### 2. Manifest Files (Dual Rendering)

Each manifest renders both the standalone CRD (for KOTS/EC) and a Secret wrapper (for Helm CLI):

```yaml
{{- /* Standalone resource for KOTS/EC (CRDs exist) */}}
{{- if .Capabilities.APIVersions.Has "troubleshoot.sh/v1beta2" }}
{{ include "myapp.preflight" . }}
{{- end }}
---
{{- /* Secret-wrapped spec for Helm CLI installs (no CRDs) */}}
apiVersion: v1
kind: Secret
metadata:
  name: {{ include "myapp.fullname" . }}-preflight
  labels:
    troubleshoot.sh/kind: preflight
stringData:
  preflight.yaml: |
    {{ include "myapp.preflight" . | nindent 4 }}
```

For support bundles, use label `troubleshoot.sh/kind: support-bundle`.

### 3. Running Preflights

**KOTS/EC:** Automatic — KOTS discovers the CRD resource and runs preflights before deploy.

**Helm CLI:** Manual via `kubectl preflight`:
```bash
# Extract spec from rendered chart and pipe to preflight tool
helm template myapp ./chart \
  --show-only templates/preflight.yaml \
  | yq '.stringData."preflight.yaml"' \
  | kubectl preflight -
```

## Collector File Path Conventions

**Critical:** Preflight and support bundle `run` collectors write output to DIFFERENT paths.

| Context | Output Path |
|---------|------------|
| Preflight `run` collector | `<collectorName>.log` |
| Support bundle `run` collector | `<collectorName>/stdout.txt` |
| Support bundle `logs` collector | `<name>/<pod>/<container>.log` |
| Support bundle `http` collector | `http/<collectorName>.json` |

**Common mistake:** Using the support bundle path format (`<collectorName>/stdout.txt`) in a preflight textAnalyze. This produces "No matching files".

**How to verify:** Extract the preflight bundle and inspect actual file paths:
```bash
tar -xzf preflightbundle-*.tar.gz
find preflightbundle-*/ -name "*.log" -o -name "stdout.txt"
```

## Empty Collectors Section

An empty `collectors:` key (or `collectors: []`) causes the troubleshoot tool to run **default collectors** — gathering cluster info, pod logs, node info, etc. This makes preflights very slow.

**Fix:** Only render the `collectors:` section when at least one collector is active:

```gotemplate
spec:
  {{- if or (not .Values.postgresql.enabled) .Values.ingress.tls.cloudflare.enabled }}
  collectors:
    {{- if not .Values.postgresql.enabled }}
    - run:
        collectorName: myapp-db-check
        # ...
    {{- end }}
    {{- if .Values.ingress.tls.cloudflare.enabled }}
    - run:
        collectorName: myapp-cloudflare-check
        # ...
    {{- end }}
  {{- end }}
  analyzers:
    # analyzers always render
```

## Conditional Preflight Checks

Use Helm conditionals to gate collectors AND their matching analyzers:

```gotemplate
{{- if not .Values.postgresql.enabled }}
# Collector: runs nc check against external DB
- run:
    collectorName: myapp-db-check
    image: busybox:1.36
    command: ["sh", "-c"]
    args:
      - |
        nc -zv {{ .Values.externalDatabase.host }} {{ .Values.externalDatabase.port }} 2>&1 && echo "connected" || echo "connection_failed"
{{- end }}

# ... in analyzers section:
{{- if not .Values.postgresql.enabled }}
- textAnalyze:
    checkName: External Database Connectivity
    fileName: myapp-db-check.log
    regex: "connected"
    outcomes:
      - fail:
          when: "false"
          message: |
            Cannot connect to external database at {{ .Values.externalDatabase.host }}:{{ .Values.externalDatabase.port }}.
      - pass:
          when: "true"
          message: External database is reachable.
{{- end }}
```

## Required Preflight Checks (Replicated Bootcamp)

For Replicated distribution, these 5 checks are recommended:

1. **External DB connectivity** — conditional `run` collector with `nc -zv`
2. **External endpoint connectivity** — conditional `run` collector for auth/API endpoints
3. **Cluster resources** — `nodeResources` analyzer for CPU and memory
4. **Kubernetes version** — `clusterVersion` analyzer with minimum version
5. **Distribution check** — `distribution` analyzer, fail on unsupported (docker-desktop, microk8s)

## Support Bundle Checklist

1. **Per-component log collectors** — separate `logs` collector per component with `maxLines`/`maxAge` limits
2. **Health endpoint** — `http` collector hitting in-cluster service DNS + `textAnalyze`
3. **Workload status** — `deploymentStatus`/`statefulsetStatus` per workload
4. **Known failure patterns** — `textAnalyze` with regex matching real app log messages
5. **Cluster health** — `storageClass` and `nodeResources` (node readiness) analyzers

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Rendering `kind: Preflight` on Helm CLI install | Gate with `.Capabilities.APIVersions.Has "troubleshoot.sh/v1beta2"` |
| Using `<collectorName>/stdout.txt` in preflight | Use `<collectorName>.log` for preflight run collectors |
| Empty `collectors:` section | Omit the section entirely when no collectors are active |
| Faking `--api-versions` for local testing | Use Secret-based delivery + extract spec with `yq` |
| Hardcoding namespace in support bundle specs | Use `{{ .Release.Namespace }}` — resolved at Helm template time |
| Missing `imagePullSecrets` on run collector pods | Use proxied images or ensure pull access on cluster |
