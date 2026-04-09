---
name: replicated-image-proxy
description: Use when configuring image proxying through a custom Replicated domain — setting up custom domains, configuring external registries, updating Helm chart image references for app and subchart images
---

# Replicated Image Proxy Setup

## Overview

Route all container images through a custom proxy domain (CNAME to `proxy.replicated.com`) so customers pull images using license-based auth without direct registry access. Satisfies Tier 2.2 of the bootcamp rubric.

## When to Use

- Setting up a custom domain for image proxying
- Configuring image references in Helm charts for Replicated distribution
- Troubleshooting image pull errors through the proxy
- Adding new subcharts that need image proxying

## Setup Steps

### 1. Create custom domain

Create a CNAME record pointing to `proxy.replicated.com`:
```
images.yourdomain.com → proxy.replicated.com
```

### 2. Set as default in Vendor Portal

**Vendor Portal → Images → Custom Domains** → add domain and set as **default**.

Only releases promoted **after** setting the default use the new domain.

### 3. Add external registries

**Vendor Portal → Images → External Registries** — add each registry your images come from:

| Registry | Endpoint | Auth needed |
|----------|----------|-------------|
| GHCR (private) | `ghcr.io` | GitHub PAT with `read:packages` |
| Docker Hub | `docker.io` | Docker Hub token or username/password |

**All registries must be added** — including Docker Hub for public images like `busybox`, `nats`, etc. The proxy needs credentials configured to pull from each source.

### 4. Test in Vendor Portal

**Vendor Portal → Images → External Registries → Test Connection**
```
repo/image:tag
jmboby/my-image:1.0.0
```

## Image Reference Format

All images use the same format through the custom domain:

```
<custom-domain>/proxy/<app-slug>/<registry>/<org>/<image>:<tag>
```

### Examples

| Image type | Reference |
|-----------|-----------|
| Private GHCR | `images.example.com/proxy/my-app/ghcr.io/myorg/my-api:1.0.0` |
| Docker Hub official | `images.example.com/proxy/my-app/library/busybox:1.36` |
| Docker Hub org | `images.example.com/proxy/my-app/natsio/nats-box:0.19.3` |
| Public GHCR | `images.example.com/proxy/my-app/cloudnative-pg/cloudnative-pg:1.29.0` |

## Helm Chart Configuration

### App images (values.yaml)

```yaml
api:
  image:
    repository: images.example.com/proxy/my-app/ghcr.io/myorg/my-api
    tag: "1.0.0"
```

### Subchart images — per-image override

Each subchart needs individual image overrides. Do NOT rely on `global.image.registry` — it may not apply consistently to all images in the subchart.

**NATS example:**
```yaml
nats:
  nats:
    image:
      registry: images.example.com/proxy/my-app
      repository: library/nats
  reloader:
    image:
      registry: images.example.com/proxy/my-app
      repository: natsio/nats-server-config-reloader
  natsBox:
    image:
      registry: images.example.com/proxy/my-app
      repository: natsio/nats-box
```

**CloudNativePG operator:**
```yaml
cloudnative-pg:
  image:
    repository: images.example.com/proxy/my-app/cloudnative-pg/cloudnative-pg
```

**CloudNativePG database instance** (Cluster CR):
```yaml
spec:
  imageName: images.example.com/proxy/my-app/cloudnative-pg/postgresql:18.3-system-trixie
```

**Replicated SDK:**
```yaml
replicated:
  image:
    registry: images.example.com/proxy/my-app
    repository: library/replicated-sdk-image
```

### Utility images in templates

```yaml
image: images.example.com/proxy/my-app/library/busybox:1.36
image: images.example.com/proxy/my-app/library/alpine:3.19
```

### imagePullSecrets

The Replicated SDK creates `enterprise-pull-secret` automatically from the customer license. Reference it in deployments and hook jobs:

```yaml
# _helpers.tpl
{{- define "myapp.imagePullSecrets" -}}
{{- if .Values.global }}
{{- if .Values.global.replicated }}
{{- if .Values.global.replicated.dockerconfigjson }}
- name: enterprise-pull-secret
{{- end }}
{{- end }}
{{- end }}
{{- range .Values.imagePullSecrets }}
- name: {{ .name }}
{{- end }}
{{- end }}
```

Use in all pod specs (deployments AND hook jobs):
```yaml
spec:
  imagePullSecrets:
    {{- include "myapp.imagePullSecrets" . | nindent 8 }}
```

## Verification

Check all pods use the custom domain:
```bash
kubectl get pods -o custom-columns='NAME:.metadata.name,IMAGE:.spec.containers[*].image'
```

Every image should start with `images.example.com/proxy/my-app/`.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Missing external registry for Docker Hub | Add `docker.io` in Vendor Portal with auth |
| Using `global.image.registry` for subcharts | Use per-image `registry` overrides — global is inconsistent |
| Forgetting imagePullSecrets on hook Jobs | Add the helper to all pod specs including pre/post-install hooks |
| Not setting custom domain as default | Vendor Portal → Images → Custom Domains → set default |
| CNPG database image not proxied | Set `spec.imageName` on the Cluster CR |
