---
name: replicated-sdk-integration
description: Use when adding the Replicated SDK to a Helm chart — subchart setup, deployment naming, license entitlement checks, custom metrics, update detection, and license validity enforcement
---

# Replicated SDK Integration

## Overview

The Replicated SDK is a subchart that exposes an in-cluster HTTP API for license enforcement, custom metrics, update detection, and support bundle upload — without requiring the KOTS Admin Console.

## When to Use

- Adding SDK to a new Helm chart
- Implementing license-gated features
- Sending custom app metrics to Vendor Portal
- Adding update-available banners
- Enforcing license validity/expiry

## Subchart Setup

### Chart.yaml

```yaml
dependencies:
  - name: replicated
    repository: oci://registry.replicated.com/library
    version: "1.19.1"
```

No `condition` — SDK should always be present.

### values.yaml

```yaml
replicated:
  nameOverride: "my-app-sdk"
  image:
    registry: images.example.com/proxy/my-app
    repository: library/replicated-sdk-image
```

**Naming:** Use `nameOverride` (not `fullnameOverride`) to brand the SDK deployment. Unlike most Helm charts, the SDK chart uses `nameOverride` as the **full deployment name** — it does NOT prepend the Helm release name. So `nameOverride: "sdk"` gives deployment `sdk`, not `<release>-sdk`. Set the full desired name including your app prefix (e.g., `nameOverride: "my-app-sdk"`).

**Image proxy:** Override the SDK image registry to use your custom proxy domain.

### Update dependencies

```bash
helm dependency update chart/
```

## SDK API Reference

Base URL: `http://<nameOverride>:3000` (in-cluster, e.g., `http://my-app-sdk:3000`)

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/v1/license/fields/<name>` | GET | Get license entitlement field |
| `/api/v1/license/info` | GET | Get license validity/expiry |
| `/api/v1/app/updates` | GET | Check for available updates |
| `/api/v1/app/custom-metrics` | POST | Send custom app metrics |

## License Entitlement

Gate features by querying the SDK at runtime. **Do NOT pass license fields through Helm values or env vars** — query the SDK directly so changes take effect without redeployment.

### Create license field

Vendor Portal → License Fields → Create:
- Field name: `my_feature_enabled`
- Type: Boolean
- Default: `false`

### Go implementation

```go
func (c *Client) IsFeatureEnabled(fieldName string) bool {
    resp, err := c.httpClient.Get(
        fmt.Sprintf("%s/api/v1/license/fields/%s", c.baseURL, fieldName))
    if err != nil {
        return false // fail closed
    }
    defer resp.Body.Close()
    var field struct{ Value interface{} `json:"value"` }
    json.NewDecoder(resp.Body).Decode(&field)
    switch v := field.Value.(type) {
    case bool:
        return v
    case string:
        return v == "true" || v == "1"
    }
    return false
}
```

### Gate a feature

```go
if !sdkClient.IsFeatureEnabled("live_tracking_enabled") {
    http.Error(w, "premium feature — upgrade your license", http.StatusForbidden)
    return
}
```

### How updates work

1. Install with entitlement disabled → feature returns 403
2. Update license in Vendor Portal (no app redeploy needed)
3. Feature becomes available — SDK picks up change within seconds

## Custom Metrics

Send app-specific metrics to the Vendor Portal Instance Details page.

```go
func (c *Client) SendMetrics(data map[string]interface{}) {
    body := map[string]interface{}{"data": data}
    jsonData, _ := json.Marshal(body)
    resp, err := c.httpClient.Post(
        fmt.Sprintf("%s/api/v1/app/custom-metrics", c.baseURL),
        "application/json", bytes.NewReader(jsonData))
    if err != nil {
        log.Printf("sdk: metrics send failed: %v", err)
        return
    }
    defer resp.Body.Close()
    if resp.StatusCode >= 300 {
        log.Printf("sdk: metrics send returned %d", resp.StatusCode)
    }
}
```

**Always log errors** — don't silently swallow failures.

### Background sender pattern

```go
func StartMetricsSender(ctx context.Context, client *Client, source MetricsSource, interval time.Duration) {
    sendMetrics(ctx, client, source) // send immediately on startup
    ticker := time.NewTicker(interval)
    defer ticker.Stop()
    for {
        select {
        case <-ctx.Done(): return
        case <-ticker.C: sendMetrics(ctx, client, source)
        }
    }
}
```

Metrics should reflect real app activity. Send every 30s–5min depending on use case.

## Update Detection

```go
func (c *Client) GetUpdates() ([]UpdateInfo, error) {
    resp, err := c.httpClient.Get(fmt.Sprintf("%s/api/v1/app/updates", c.baseURL))
    // ... decode []UpdateInfo{VersionLabel, CreatedAt, ReleaseNotes}
}
```

Empty array = on latest version. Non-empty = update available. Show a dismissible banner in the UI.

## License Validity

```go
func (c *Client) GetLicenseInfo() (*LicenseInfo, error) {
    resp, err := c.httpClient.Get(fmt.Sprintf("%s/api/v1/license/info", c.baseURL))
    // ... decode — see SDK Response Format Quirks below for correct struct tags
}
```

Required behavior:
- **Expired** → show warning banner or block access
- **Invalid/revoked** → show warning or block
- **Valid** → normal operation

Check on startup + periodic recheck. License can change without app redeployment.

## SDK Response Format Quirks

The SDK API has several undocumented field name and type quirks. **Always verify against the real endpoint** rather than trusting documentation:

```bash
kubectl exec -n <ns> <api-pod> -- wget -qO- http://<sdk-svc>:3000/api/v1/license/info
kubectl exec -n <ns> <api-pod> -- wget -qO- http://<sdk-svc>:3000/api/v1/license/fields/<name>
```

### Boolean fields return JSON booleans, not strings

`/license/fields/<name>` returns `"value": true` (boolean) for Boolean type fields, not `"value": "true"` (string). Use `interface{}` with a type switch:

```go
var field struct{ Value interface{} `json:"value"` }
json.NewDecoder(resp.Body).Decode(&field)
switch v := field.Value.(type) {
case bool:
    return v
case string:
    return v == "true" || v == "1"
}
```

### `licenseID` uses capital D

The `/license/info` endpoint returns `"licenseID"` (capital D), not `"licenseId"`. Go JSON struct tags are case-sensitive:

```go
// Correct
type LicenseInfo struct {
    LicenseID string `json:"licenseID"`
    // ...
}
```

### No `isExpired` field — parse `expires_at` instead

The `/license/info` response has no top-level `isExpired` field. Expiry is nested in `entitlements.expires_at.value` as an RFC3339 date string. You must parse and compare:

```go
type LicenseInfo struct {
    LicenseID   string `json:"licenseID"`
    LicenseType string `json:"licenseType"`
    Entitlements struct {
        ExpiresAt struct {
            Value string `json:"value"`
        } `json:"expires_at"`
    } `json:"entitlements"`
}

func isExpired(info *LicenseInfo) bool {
    if info.Entitlements.ExpiresAt.Value == "" {
        return false // no expiry set
    }
    t, err := time.Parse(time.RFC3339, info.Entitlements.ExpiresAt.Value)
    if err != nil {
        return false
    }
    return time.Now().After(t)
}
```

## SDK Unavailability (Local Dev)

When the SDK is unreachable (local dev, standalone testing):
- License checks: fail closed (feature disabled) or fail open (valid) — choose per feature
- Metrics: fail silently (best-effort)
- Updates: return empty (no banner)

Configure SDK URL via env var: `REPLICATED_SDK_URL` (default: `http://<app>-sdk:3000`)

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Passing license fields via Helm values | Query SDK at runtime — enables license updates without redeploy |
| Silent metric failures | Log errors — you need visibility when metrics aren't arriving |
| Only checking license on startup | Check periodically — license can be updated at any time |
| Using `fullnameOverride` for SDK naming | SDK chart only supports `nameOverride` |
| Forgetting to proxy SDK image | Override `replicated.image.registry` in values |
| `LicenseField.Value` typed as `string` | Use `interface{}` with type switch — Boolean fields return JSON booleans |
| `json:"licenseId"` (lowercase d) | Use `json:"licenseID"` (capital D) — SDK returns capital D |
| Checking `info.IsExpired` | No such field — parse `entitlements.expires_at.value` as RFC3339 |
