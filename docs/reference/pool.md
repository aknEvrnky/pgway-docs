# Pool

A group of proxies in front of a load balancer. Either a **static** membership list or a **dynamic** label selector.

**Kind:** `Pool` · **Version:** `v1`

## Spec fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `title` | string | no | Human-readable title |
| `type` | string | **yes** | `static` or `dynamic` |
| `members` | list | **yes** if static | Explicit proxy membership (forbidden if dynamic) |
| `selector` | object | **yes** if dynamic | Label selector (forbidden if static) |

### `members[]` (static only)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `proxy_id` | string | **yes** | Proxy `metadata.name` |
| `weight` | integer | no | Default **1** if omitted; if set must be **≥ 1** |

### `selector` (dynamic only)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `allow` | map of string → string | **yes** (non-empty) | All listed labels must match the proxy |

## Validation & notes

| Rule | Detail |
|------|--------|
| Static | ≥ 1 member; unique `proxy_id`; no `selector` |
| Dynamic | `selector.allow` non-empty; no `members` |
| Dynamic match | Proxy `metadata.labels` must contain every `allow` key/value |
| Weighted LB | Requires a **static** pool (enforced when applying the balancer) |
| Morph guard | Cannot turn a pool non-static while a weighted balancer still references it |

## Examples

### Static (with optional weights)

```yaml
kind: Pool
version: v1
metadata:
  name: static-pool
spec:
  title: Main Static
  type: static
  members:
    - proxy_id: proxy-1
      weight: 3
    - proxy_id: proxy-2   # weight defaults to 1
```

### Dynamic

```yaml
kind: Pool
version: v1
metadata:
  name: dynamic-pool
spec:
  title: By provider
  type: dynamic
  selector:
    allow:
      provider: example
      region: us-east
```
