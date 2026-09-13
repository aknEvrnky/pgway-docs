# LoadBalancer

Sits in front of **exactly one** pool and chooses which resolved proxy handles each request.

**Kind:** `LoadBalancer` · **Version:** `v1`

Behavior and strategy details: [Load balancing](../guides/load-balancing.md).

## Spec fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `title` | string | no | Human-readable title |
| `type` | string | **yes** | Strategy — see below |
| `pool_id` | string | **yes** | Pool `metadata.name` |
| `reset_interval` | string | no | Go duration (e.g. `30s`, `1m`). **Only** for `least-bytes` |

## `type` values

| Value | Behavior | Pool constraint | `reset_interval` |
|-------|----------|-----------------|------------------|
| `round-robin` | Even rotation | Static or dynamic | Must **not** be set |
| `weighted` | Smooth weighted RR (nginx-style) using static member weights | **Static only** | Must **not** be set |
| `least-bytes` | Prefer proxy with fewest accumulated **downstream** bytes (via `Release`) | Static or dynamic | Optional; default **`1m`** |

## Validation & notes

- Schema rejects unknown `type`, empty `pool_id`, `reset_interval` on non–least-bytes, or non-positive / unparsable durations.
- For `least-bytes`, omitted `reset_interval` resolves to `1m`. Counters reset **lazily** on the next selection after the interval elapses.
- Downstream = response / tunnel→client bytes (not request body / upstream-only).
- Weighted apply fails if the referenced pool is not static.

## Examples

### Round-robin

```yaml
kind: LoadBalancer
version: v1
metadata:
  name: main-rr
spec:
  title: Round Robin
  type: round-robin
  pool_id: dynamic-pool
```

### Weighted

```yaml
kind: LoadBalancer
version: v1
metadata:
  name: main-weighted
spec:
  type: weighted
  pool_id: static-pool
```

### Least-bytes

```yaml
kind: LoadBalancer
version: v1
metadata:
  name: main-least-bytes
spec:
  type: least-bytes
  pool_id: dynamic-pool
  reset_interval: 30s   # optional; omit → 1m
```
