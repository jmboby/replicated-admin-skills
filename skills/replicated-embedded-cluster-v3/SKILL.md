---
name: replicated-embedded-cluster-v3
description: Use when packaging or troubleshooting Embedded Cluster v3 releases — custom proxy domains, noProxy=true image templating, per-entrypoint Traefik NodePorts, v1beta3 preflight limitations, airgap bundle image discovery, kube-apiserver flag overrides
---

# Embedded Cluster v3 Packaging

## Overview

Embedded Cluster v3 is Replicated's k0s-based distribution for customer-managed single-node / small-cluster installs. This skill captures patterns that are subtly different from KOTS-on-existing-cluster.

## When to Use

- Publishing a Replicated release that supports EC v3
- Troubleshooting EC v3 install failures (image pulls, Helm schema validation, preflight rendering)
- Configuring custom proxy domains on EC v3
- Extending kube-apiserver settings on EC v3 (NodePorts, etc.)

## Custom proxy domain on EC v3

Set `proxyRegistryDomain` in the EC Config CRD so EC containerd is configured with license credentials for the custom domain:

```yaml
# replicated/embedded-cluster.yaml
apiVersion: embeddedcluster.replicated.com/v1beta1
kind: Config
metadata:
  name: my-app
spec:
  version: "3.0.0-alpha-31+k8s-1.34"
  domains:
    proxyRegistryDomain: images.example.com
```

Also configure the same domain in Vendor Portal → Images → Custom Domains as the default for proxy.replicated.com.

## `ReplicatedImage*` + `noProxy=true` pattern

Chart defaults in custom-domain proxy form (e.g. `images.example.com/proxy/my-app/ghcr.io/org/image:tag`) work great for helm-cli installs but will DOUBLE-PREFIX on EC alpha-31 if wrapped with `ReplicatedImageName`/`ReplicatedImageRegistry`/`ReplicatedImageRepository` without the `noProxy=true` flag.

**Why:** EC alpha-31's `pkg/template/image_context.go` doesn't check if the input already matches the configured ProxyDomain — it unconditionally prepends. The "return unchanged if matches" shortcut was only added in later versions.

**Fix:** Pass `true` as the 2nd positional arg to every `ReplicatedImage*` call where the input already contains the proxy path prefix:

```yaml
# replicated/my-app-chart.yaml (HelmChart CR)
spec:
  values:
    api:
      image:
        registry: 'repl{{ ReplicatedImageRegistry (HelmValue ".Values.api.image.registry") true }}'
        repository: 'repl{{ ReplicatedImageRepository (HelmValue ".Values.api.image.repository") true }}'
    busybox:
      image: 'repl{{ ReplicatedImageName (HelmValue ".Values.busybox.image") true }}'
```

Per the docs preview at `deploy-preview-3968--replicated-docs-upgrade.netlify.app/vendor/replicated-onboarding-air-gap`: *"The `true` parameter sets `noProxy` to `true`, indicating 'the image reference value in values.yaml already contains the proxy path prefix.'"*

**Behavior with `true`:**
- **Online (EC, KOTS, helm-CLI)** — value returned unchanged (custom domain preserved everywhere)
- **Airgap (EC, KOTS)** — BYO rewriting still happens because the `isAirgap` branch runs before the `noProxy` check

**Exceptions — DON'T pass `true` when:**
- Chart defaults are in upstream form (e.g., Traefik's `docker.io` / `traefik`). The function SHOULD prepend the proxy path there.
- Replicated SDK: chart default is `library/replicated-sdk-image` (not under `/proxy/<slug>/`). The function would incorrectly prepend `proxy/<slug>/` without `true`. Pass `true` to keep the path unmodified.

## Airgap bundle image discovery

The airgap bundler renders the chart using the HelmChart CR `builder:` section values, then scans rendered manifests for image refs. **Every image you want in the bundle must appear in at least one rendered pod spec when the builder values are applied.**

Conditional templates (e.g., pre-install hook jobs gated on config options) won't render with default builder values. Force-render them:

```yaml
spec:
  builder:
    ingress:
      enabled: true
      hostname: "builder.example.com"
      tls:
        mode: "self-signed"  # force cert-job render to include its alpine/openssl image
```

**Verify:** After a release, check `installationTypes[].airgapBundleImages` in the release metadata to confirm expected images are present:

```bash
replicated api get /v3/app/<app-id>/channel/<chan-id>/releases \
  | jq '.releases[0].airgapBundleImages'
```

## NodePorts for ingress on EC v3

EC's k0s already extends `--service-node-port-range` to `80-32767` by default ([pkg/k0s/config.go L167-169](https://github.com/replicatedhq/ec/blob/0ea20cf0eb442b136a223da13343164cbd873d83/pkg/k0s/config.go#L167-L169)), so NodePort values as low as 80 are accepted out of the box. **No `unsupportedOverrides` needed** for this.

### Traefik v3 NodePort config is per-entrypoint

The Traefik v3 Helm chart's NodePort config lives at `ports.<entrypoint>.nodePort`, NOT at `service.nodePorts.{http,https}`. The latter is a common pattern in other charts but is silently ignored by Traefik v3 — the Service comes up with random NodePorts.

```yaml
# replicated/traefik-chart.yaml (HelmChart CR values)
values:
  service:
    type: NodePort
  ports:
    web:
      nodePort: 80
    websecure:
      nodePort: 443
```

Verify via `helm show values traefik/traefik --version <ver>` before assuming any specific values path works.

### Extending kube-apiserver flags generally

If you genuinely need a wider range or other kube-apiserver flags beyond what EC provides, you *can* use `unsupportedOverrides.k0s`:

```yaml
spec:
  unsupportedOverrides:
    k0s: |
      config:
        spec:
          api:
            extraArgs:
              service-node-port-range: "1-65535"
```

**But first check if EC's defaults already cover your need.** Before adding the override, grep the EC repo at the version you're deploying (`replicatedhq/ec @ pkg/k0s/config.go`) for the flag you think you need — it may already be set.

**Caveats of using the override anyway:**
- Outside Replicated's support SLA (the field name literally says "unsupported")
- `spec.api` can't be modified post-install — any value is install-time-locked; changing it requires full reinstall
- Prefer narrow ranges (e.g. `80-32767`) over wide-open (`1-65535`) to avoid conflicts with host daemons on ports like 22/53/25

## Security: avoid `hostNetwork: true` for Traefik

A common temptation for "just bind to :80/:443" is `hostNetwork: true` on the ingress pod. Reject it for production-adjacent releases:

| Concern | Impact |
|---|---|
| NetworkPolicy bypass | NetworkPolicies don't apply to hostNetwork pods |
| Lateral movement | Compromised Traefik gets direct access to kubelet `:10250`, cloud metadata `169.254.169.254`, localhost services |
| Port binding unrestricted | Compromised container can open arbitrary listeners on the node |
| Reduced isolation | Pods share host's network namespace |

Prefer **per-entrypoint NodePort config** (above; EC's default range already accepts :80/:443) or **MetalLB + LoadBalancer** for production.

## v1beta3 preflight limitations (EC v3 today)

**v1beta3 preflight is rendered via Helm templating**, not KOTS template functions. `ConfigOption`, `ConfigOptionEquals`, `IsAirgap`, `LicenseFieldValue`, `repl{{ }}` all produce `function "X" not defined` errors when used in `replicated/preflight-v1beta3.yaml`.

In theory v1beta3 supports `.Values` conditionals. In practice EC v3's current runner **doesn't wire chart values into the v1beta3 render**, so `.Values.x.enabled`-style conditionals also fail.

**Pattern:** Keep `replicated/preflight-v1beta3.yaml` limited to **static cluster-level checks** (K8s version, CPU, memory, distribution, storage class). Put workload-specific conditional checks (external DB reachability, outbound connectivity) in the chart's v1beta2 `_preflight.tpl` where Helm `.Values` work reliably on the Helm-CLI path.

## Common pitfalls

| Pitfall | Fix |
|---|---|
| Double-prefixed image URLs on EC online | Add `true` (noProxy) to `ReplicatedImage*` calls |
| "SDK pull access denied" | SDK repository is `library/replicated-sdk-image` (outside `/proxy/<slug>/`); pass `true` |
| Hardcoded image in a chart template fails on airgap | Move image to a chart value; wrap with `ReplicatedImageName ... true` in HelmChart CR |
| Cert-job uses `apk add` at runtime | Fails on airgap (no egress); use pre-baked images or split into initContainer pattern |
| Conditional-template images missing from airgap bundle | Force-render via HelmChart CR `builder:` overrides |
| Traefik bound to random NodePort despite declaring 443 | Traefik v3 uses `ports.<entrypoint>.nodePort`, NOT `service.nodePorts.{http,https}`; the latter is silently ignored |
| v1beta3 preflight `function not defined` error | Don't use KOTS template functions in v1beta3; keep cluster-level only |

## Verification checklist before releasing

- [ ] `replicated release lint --app <slug>` — no new errors (ignore `unable-to-render` for ReplicatedImage* — local-only)
- [ ] `replicated release download <seq>` and inspect contents — chart tgz present, no orphan top-level KOTS-only manifests that would break helm-cli availability
- [ ] `airgapBundleImages` in release metadata includes every image used by pre-install hooks and conditionally-rendered templates
- [ ] EC install on a fresh CMX VM completes and all pods reach Running
- [ ] `kubectl get pod <any> -o jsonpath='{.spec.containers[0].image}'` shows **single-prefix** custom-domain URLs (no doubles)
