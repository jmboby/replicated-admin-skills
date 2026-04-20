---
name: replicated-kots-config
description: Use when authoring or debugging the KOTS config screen — license-gated features, validation, generated secrets, install-type availability, helm-cli compatibility
---

# KOTS Config Screen Patterns

## Overview

Patterns and gotchas for writing `replicated/kots-config.yaml` (the Config CRD that powers the admin console's config screen). Covers rubric items 5.1-5.4 and the subtle ways config choices break helm-cli install availability.

## When to Use

- Adding or editing items in `replicated/kots-config.yaml`
- Gating features on license entitlements
- Generating secrets that must survive upgrades
- Adding validation rules to text / password fields
- Debugging "helm-cli install unavailable" on the vendor portal

## License-gated features (rubric 5.1 — three layers)

**Layer 1 — license = entitlement, config = intent.** Separate concerns: the license encodes what the customer is entitled to; the config item lets the operator decide whether to turn it on.

**Layer 2 — default from license, don't templatize `readonly`.**

```yaml
- name: live_tracking_enabled
  title: Live Drone Tracking
  type: bool
  default: 'repl{{ ternary "1" "0" (LicenseFieldValue "live_tracking_enabled" | ParseBool) }}'
```

**DO NOT** use `readonly: 'repl{{ not (LicenseFieldValue ...) }}'`. The KOTS config schema requires `readonly` to be a literal boolean and will fail with:

```
config-is-invalid: failed to decode config content: json: cannot unmarshal
string into Go struct field ConfigItem.spec.groups.items.readonly of type bool
```

That error invalidates the release for helm-cli / existing-cluster installs (EC tolerates it via a different validation path).

**Layer 3 — `and`-guard in the HelmChart CR.** Enforce entitlement at render time regardless of what the operator does with the config toggle:

```yaml
# replicated/my-app-chart.yaml
spec:
  values:
    api:
      liveTrackingEnabled: 'repl{{ and (ConfigOptionEquals "live_tracking_enabled" "1") (LicenseFieldValue "live_tracking_enabled" | ParseBool) }}'
```

Operators without the license entitlement can toggle the config to `1` but the rendered value is still `false`. Defense in depth: tampering with the kotsadm DB doesn't bypass the feature gate.

## Generated secrets that survive upgrades (rubric 5.2)

Use `default:` with `RandomString`, not `value:`. KOTS evaluates `default:` once on first render and caches; `value:` re-evaluates every render (password churns).

```yaml
- name: db_password
  title: Database Password
  type: password
  default: 'repl{{ RandomString 32 }}'
  when: 'repl{{ ConfigOptionEquals "database_type" "embedded" }}'
```

## Validation on text/password fields (rubric 5.3)

Both text and password items support `validation.regex`:

```yaml
- name: tls_email
  title: Contact Email
  type: text
  validation:
    regex:
      pattern: '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$'
      message: Enter a valid email address (e.g. ops@example.com).

- name: webhook_url
  title: Notification Webhook URL
  type: text
  default: ""
  validation:
    regex:
      pattern: '^$|^https?://[^\s]+$'
      message: Must be blank or a valid http(s) URL.
```

Use `^$|...` for "allow blank OR match" so operators can leave optional fields empty.

## `help_text` on every item (rubric 5.4)

Backfill `help_text` on every item. The config screen UX is the operator's only documentation at install time — write it as if explaining to an SRE who hasn't read your README.

## `when`-gated fields render as empty string (NOT the default)

This is the sharpest edge in KOTS config. A field with:

```yaml
- name: postgres_instances
  type: text
  default: "1"
  when: 'repl{{ ConfigOptionEquals "database_type" "embedded" }}'
```

...returns `""` (empty string) from `ConfigOption "postgres_instances"` when `database_type=external`, **not** `"1"`. Any downstream consumer that doesn't handle empty string will break.

**In the HelmChart CR** — defend with sprig `default`:

```yaml
postgresql:
  instances: repl{{ ConfigOption "postgres_instances" | default "1" | ParseInt }}
```

This matters because Helm schema validation runs before any template conditional. Even if `postgresql.enabled=false` means the Cluster CR won't render, the schema still rejects `instances: 0`.

## YAML type inference on rendered templates

Single-quoting a KOTS-template result changes the rendered value's YAML type:

```yaml
# BAD — rendered '1' is a YAML STRING, fails schema type=integer
instances: 'repl{{ ConfigOption "postgres_instances" | default "1" | ParseInt }}'

# GOOD — rendered 1 is a YAML INTEGER
instances: repl{{ ConfigOption "postgres_instances" | default "1" | ParseInt }}
```

Rule of thumb: only quote values that are **supposed** to be strings (`"1Gi"`, URLs, IDs). Leave numerics and booleans unquoted.

## Top-level KOTS-only resources break helm-cli availability

The vendor portal classifies a release as helm-cli-installable only if every resource can be rendered by plain Helm. A top-level K8s resource that uses KOTS templating disqualifies the release.

**Broken pattern — standalone Secret at `replicated/` level:**

```yaml
# replicated/my-secret.yaml — DISABLES helm-cli availability for the whole release
apiVersion: v1
kind: Secret
metadata:
  annotations:
    kots.io/when: 'repl{{ ConfigOptionEquals "x" "1" }}'
stringData:
  token: 'repl{{ ConfigOption "api_token" }}'
```

**Fix — move the Secret into `chart/templates/` as a Helm-conditional template, plumb values:**

```yaml
# chart/templates/my-secret.yaml
{{- if .Values.feature.enabled }}
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
stringData:
  token: {{ .Values.feature.apiToken | quote }}
{{- end }}
```

```yaml
# replicated/my-app-chart.yaml
spec:
  values:
    feature:
      enabled: repl{{ ConfigOptionEquals "x" "1" }}
      apiToken: repl{{ ConfigOption "api_token" }}  # fed from ConfigOption → HelmChart values → chart render
```

Helm-CLI operators pass `--set feature.apiToken=xxx` at install.

## Debugging install-type availability regressions

When helm / EC / KOTS availability flips from on to off for new releases:

```bash
APP_ID=$(replicated app ls --output json | jq -r '.[] | select(.slug=="<slug>") | .id')
replicated api get /v3/app/$APP_ID/channel/<channel-id>/releases \
  | python3 -c "import json, sys; d = json.load(sys.stdin); [print(r['sequence'], r['semver'], list(r.get('installationTypes', {}).keys())) for r in reversed(d['releases'])]"
```

Find the first release where the install type disappears. `git log <last-good-tag>..<first-bad-tag>` surfaces the commits between. Often the culprit is a single commit introducing a KOTS-only resource or a schema-failing field — not whatever lint error happens to be current.

Lint errors don't always correlate with availability transitions. Don't assume the newest lint error is the root cause without metadata-diff confirmation.

## Common pitfalls

| Pitfall | Fix |
|---|---|
| `readonly` as templated string | KOTS schema requires literal bool; drop it and use Layer 3 and-guard instead |
| `RandomString` password churning on re-render | Use `default:` not `value:` |
| Empty-string from `when`-gated ConfigOption | `\| default "..."` in downstream template |
| `got string, want integer` schema error | Unquote the template; YAML infers int from unquoted rendered number |
| Helm-CLI unavailable after adding a config-driven feature | Check for new top-level KOTS-templated resources; move into chart/templates/ |
| `config-option-password-type` warn on a "secret name" text field | False positive on the keyword; leave warning or rename field |

## Verification checklist

- [ ] `replicated release lint` — no `config-is-invalid` errors
- [ ] Release metadata `installationTypes` includes every install mode expected (helm, kots, embeddedCluster)
- [ ] Every bool item wrapped in a Layer 3 `and`-guard in the HelmChart CR if license-gated
- [ ] Every `when`-gated field consumed via `\| default "..."` where emptiness would break schema or app logic
- [ ] `help_text` present on every item
- [ ] Generated-secret items use `default:` + `RandomString`, not `value:`
