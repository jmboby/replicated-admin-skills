---
name: replicated-release-management
description: Use when setting up release versioning with release-please, Replicated release promotion workflows, semver strategy, or coordinating GitHub Releases with Replicated Vendor Portal releases
---

# Replicated Release Management

## Overview

Coordinate automated semver versioning (via release-please) with Replicated release creation and promotion. One version flows through GitHub tags, Docker image tags, Helm chart versions, and Replicated releases.

## When to Use

- Setting up version management for a Replicated-distributed app
- Configuring release-please with Helm charts
- Building promote workflows for Unstable → Beta → Stable

## Release Flow

```
merge PRs to main → release-please updates Release PR (version bump + CHANGELOG)
                   ↓
merge Release PR → tag v1.2.0 created → Replicated Release workflow triggers
                   ↓
                   build images (tagged 1.2.0 + latest)
                   → replicated release create --promote Unstable
                   → test on CMX cluster
                   → GitHub Release with Helm chart artifact
                   ↓
manual promote → replicated release promote → Beta or Stable
```

## release-please Setup

### Workflow

```yaml
name: Release Please
on:
  push:
    branches: [main]
permissions:
  contents: write
  pull-requests: write
jobs:
  release-please:
    runs-on: ubuntu-latest
    steps:
      - uses: googleapis/release-please-action@v4
        with:
          release-type: simple
          token: ${{ secrets.GITHUB_TOKEN }}
```

**Requires:** GitHub Actions must be allowed to create PRs (repo Settings → Actions → Workflow permissions).

### Config files

`release-please-config.json`:
```json
{
  "packages": {
    ".": {
      "release-type": "simple",
      "bump-minor-pre-major": true,
      "bump-patch-for-minor-pre-major": true,
      "extra-files": [
        "chart/Chart.yaml",
        "chart/values.yaml"
      ]
    }
  }
}
```

`.release-please-manifest.json`:
```json
{
  ".": "1.0.0"
}
```

### Version annotations

Add `# x-release-please-version` comments to lines release-please should update:

**chart/Chart.yaml:**
```yaml
version: 1.0.0 # x-release-please-version
appVersion: "1.0.0" # x-release-please-version
```

**chart/values.yaml:**
```yaml
api:
  image:
    tag: "1.0.0" # x-release-please-version
frontend:
  image:
    tag: "1.0.0" # x-release-please-version
```

### Version bumping rules

| Commit prefix | Version bump | Example |
|--------------|-------------|---------|
| `feat:` | Minor | 1.0.0 → 1.1.0 |
| `fix:` | Patch | 1.0.0 → 1.0.1 |
| `feat!:` or `BREAKING CHANGE` | Major | 1.0.0 → 2.0.0 |
| `chore:`, `docs:`, `ci:` | No bump | — |

## Replicated Release Workflow

Triggers on tags created by release-please:

```yaml
name: Replicated Release
on:
  push:
    tags: ["v*.*.*"]
```

### Extract version from tag

```yaml
- name: Extract version
  run: |
    VERSION="${GITHUB_REF_NAME#v}"
    echo "version=${VERSION}" >> $GITHUB_OUTPUT
```

### Image tagging

Tag images with semver AND latest:
```bash
docker build -t $IMAGE:${VERSION} -t $IMAGE:latest .
docker push $IMAGE:${VERSION}
docker push $IMAGE:latest
```

### Create Replicated release

```bash
replicated release create \
  --version ${VERSION} \
  --promote Unstable \
  --ensure-channel
```

### Attach chart to GitHub Release

```yaml
- uses: softprops/action-gh-release@v2
  with:
    files: my-chart-${VERSION}.tgz
```

## Promote Workflow

Manual dispatch for promoting to Beta/Stable:

```yaml
name: Promote
on:
  workflow_dispatch:
    inputs:
      release-sequence:
        required: true
      channel:
        type: choice
        options: [beta, stable]
      version:
        required: true
```

**Channel slugs must be lowercase** (`stable` not `Stable`).

### Email notifications

Configure in Vendor Portal → Channels → Stable → Notifications for promotion alerts.

## HelmChart CR Version

The `replicated/` directory contains KOTS manifests. Use `$VERSION` placeholder in the HelmChart CR:

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
        tag: $VERSION
```

The release workflow substitutes `$VERSION` via `sed` before packaging.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Release workflow triggers on push to main | Trigger on tags (`v*.*.*`) — let release-please control when |
| Missing `x-release-please-version` annotations | Without annotations, release-please can't find version lines |
| Uppercase channel slugs in promote | Use lowercase (`stable`, `beta`, `unstable`) |
| Chart.yaml version out of sync with images | Add both to release-please `extra-files` with annotations |
| `$VERSION` not substituted in HelmChart CR | `sed` before packaging the release directory |
