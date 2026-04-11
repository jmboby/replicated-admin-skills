---
name: replicated-ci-setup
description: Use when setting up CI/CD pipelines with Replicated — GitHub Actions workflows, RBAC policies, CLI commands for release creation, customer management, cluster provisioning, and Helm installs on CMX
---

# Replicated CI/CD Setup

## Overview

Set up GitHub Actions CI/CD using the `replicated` CLI (not `replicated-actions`) to create releases, provision CMX clusters, and test Helm installs. The CLI supports Embedded Cluster workflows that the GitHub Actions don't.

## When to Use

- Setting up CI/CD for a Replicated-distributed app
- Creating PR or release workflows with CMX testing
- Troubleshooting Replicated CI failures

## CLI vs Actions

**Always use the `replicated` CLI**, not `replicatedhq/replicated-actions`. The CLI:
- Supports Embedded Cluster (actions don't)
- Reads the `.replicated` config file for release packaging
- Gives better error output for debugging
- Single dependency (binary) instead of multiple action versions

## Required Secrets

| Secret | Description |
|--------|------------|
| `REPLICATED_APP` | App slug (e.g., `my-app`) |
| `REPLICATED_API_TOKEN` | Scoped CI service account token |
| `GITHUB_TOKEN` | Auto-provided, needs `packages: write` for GHCR |

## RBAC Policy for CI Service Account

Create a custom RBAC policy in Vendor Portal → Team → RBAC Policies. Resource names are **lowercase**:

```json
{
  "v1": {
    "name": "CI/CD",
    "resources": {
      "allowed": [
        "kots/app/*/read",
        "kots/app/*/release/create",
        "kots/app/*/release/*/read",
        "kots/app/*/release/*/update",
        "kots/app/*/channel/create",
        "kots/app/*/channel/*/read",
        "kots/app/*/channel/*/promote",
        "kots/app/*/channel/*/update",
        "kots/app/*/channel/*/archive",
        "kots/app/*/channel/*/releases/read",
        "kots/app/*/license/create",
        "kots/app/*/license/*/read",
        "kots/app/*/license/*/update",
        "kots/app/*/license/*/delete",
        "kots/app/*/license/*/archive",
        "kots/license/*/archive",
        "kots/license/*/delete",
        "kots/cluster/create",
        "kots/cluster/list",
        "kots/cluster/*",
        "kots/cluster/*/kubeconfig",
        "kots/cluster/*/delete",
        "registry/namespace/*/pull"
      ]
    }
  }
}
```

Key gotchas:
- Resource names are **lowercase** (`kots/` not `KOTS/`)
- `kots/cluster/*/kubeconfig` is a separate permission from `kots/cluster/*`
- `kots/app/*/license/*/archive` needed for customer cleanup

## The .replicated Config File

Place at repo root. The CLI reads this automatically (do NOT use `--auto` flag):

```yaml
appSlug: "my-app"
charts:
  - path: ./chart
manifests:
  - ./replicated/*.yaml
```

The `replicated/` directory contains KOTS manifests (HelmChart CR, Application CR).

## CLI Commands for CI

### Install CLI in GitHub Actions

```yaml
- name: Install Replicated CLI
  run: |
    curl -s https://api.github.com/repos/replicatedhq/replicated/releases/latest \
      | grep "browser_download_url.*linux_amd64.tar.gz" \
      | cut -d '"' -f 4 \
      | xargs curl -sL | tar xz -C /usr/local/bin replicated
```

### Create Release

```bash
# Reads .replicated config, packages chart, collects manifests
replicated release create \
  --version "${VERSION}" \
  --promote "${CHANNEL}" \
  --ensure-channel
```

**Do NOT use `--auto -y`** — this ignores the `.replicated` config and defaults to `./manifests`.

### Create Customer

```bash
replicated customer create \
  --name "test-customer" \
  --email "test@example.com" \
  --channel "${CHANNEL}" \
  --expires-in 24h \
  --kots-install=false \
  --helm-install \
  --ensure-channel \
  --output json
```

**Critical:**
- `--kots-install=false` for Helm-only releases, otherwise: `Cannot assign customer with KOTS install enabled to a channel with a helm-cli-only release`
- `--email` is **required** when `--helm-install` is set, otherwise: `email is required for customers with helm install enabled`

### Create Cluster

```bash
replicated cluster create \
  --distribution k3s \
  --version 1.34 \
  --name "test-cluster" \
  --ttl 30m \
  --wait 10m \
  --output json
```

### Get Kubeconfig

```bash
replicated cluster kubeconfig --id "${CLUSTER_ID}" --output-path kubeconfig.yaml
```

**Use `--output-path`**, not stdout redirect (`>`). The CLI prints warnings to stdout that corrupt the YAML if you redirect.

### Helm Install from Replicated Registry

```bash
helm registry login registry.replicated.com \
  --username "${EMAIL}" \
  --password "${LICENSE_ID}"

helm install my-app \
  oci://registry.replicated.com/${APP}/${CHANNEL}/my-chart \
  --version "${CHART_VERSION}" \
  --namespace default
```

**`--version` must match the chart version in `Chart.yaml`** (e.g., `1.2.3`), NOT the Replicated release label. If you push a chart with `version: 1.2.3` in `Chart.yaml`, use `--version 1.2.3` even if the Replicated release is labelled `v1.2.3` or `my-release`.

**Never use `--wait` with post-install hooks.** Helm `--wait` waits for all pods to be Ready before running hooks. If any pod depends on a hook-created resource (e.g., a database CR), this deadlocks: hooks never run → pods never ready → `--wait` never returns.

### Cleanup

```bash
replicated cluster rm --id "${CLUSTER_ID}"
replicated customer archive --id "${CUSTOMER_ID}"
replicated channel rm "${CHANNEL_ID}"  # requires channel ID, not name
```

**Note:** `channel rm` requires the channel **ID**, not the name/slug.

## PR Workflow Pattern

```
lint-and-test → build-and-push → test-on-cmx
                                  ├── release create (temp channel)
                                  ├── customer create
                                  ├── cluster create
                                  ├── helm install
                                  ├── smoke test
                                  └── cleanup (always)
```

## workflow_call Pattern for Chaining Workflows

**Problem:** Tags created by `GITHUB_TOKEN` don't trigger other workflows (GitHub security restriction). So release-please's tag won't fire your Replicated Release workflow if both use `push: tags`.

**Solution:** Use `workflow_call` to chain release-please → Replicated Release directly:

```yaml
# release-please.yaml
jobs:
  release-please:
    runs-on: ubuntu-latest
    outputs:
      release_created: ${{ steps.rp.outputs.release_created }}
      tag_name: ${{ steps.rp.outputs.tag_name }}
    steps:
      - uses: googleapis/release-please-action@v4
        id: rp
        with:
          release-type: simple

  replicated-release:
    needs: release-please
    if: ${{ needs.release-please.outputs.release_created }}
    uses: ./.github/workflows/replicated-release.yaml
    with:
      tag: ${{ needs.release-please.outputs.tag_name }}
    secrets: inherit
```

```yaml
# replicated-release.yaml
on:
  workflow_call:
    inputs:
      tag:
        required: true
        type: string
  push:
    tags: ["v*.*.*"]   # keep for manual re-runs
```

## Release Workflow Pattern

Trigger on tags (`v*.*.*`), not on push to main:

```
lint-and-test → build-and-push → create-release (Unstable) → test-on-cmx → GitHub Release
```

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Using `--auto -y` with `.replicated` config | Remove `--auto -y` — CLI reads `.replicated` automatically |
| Using `replicated-actions` | Switch to `replicated` CLI for EC support |
| Channel slugs with uppercase | Use lowercase (`stable` not `Stable`) |
| Customer with KOTS enabled on Helm-only channel | Add `--kots-install=false --helm-install` |
| Missing `--email` with `--helm-install` | Add `--email` — required for Helm-install customers |
| `channel delete` with channel name | Use `channel rm` with channel **ID** |
| Missing `kots/cluster/*/kubeconfig` RBAC | Add it — separate from `kots/cluster/*` |
| `--wait` with post-install hooks | Remove `--wait` — deadlocks if pods depend on hook-created resources |
| Kubeconfig stdout redirect (`> kubeconfig.yaml`) | Use `--output-path kubeconfig.yaml` — CLI warnings corrupt redirected output |
| `helm install --version` uses release label | Use chart version from `Chart.yaml`, not the Replicated release label |
| release-please tag doesn't trigger release workflow | Use `workflow_call` to chain — `GITHUB_TOKEN` tags don't fire other workflows |
