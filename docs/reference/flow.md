# Flow

Binds an entrypoint to routing: optional **router**, otherwise a default **load balancer**.

**Kind:** `Flow` · **Version:** `v1`

## Spec fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `router_id` | string | at least one* | Router `metadata.name` |
| `balancer_id` | string | at least one* | LoadBalancer `metadata.name` |

\*Schema requires **at least one** of `router_id` or `balancer_id` to be non-empty.

## Runtime behavior

| Config | What runs |
|--------|-----------|
| Only `balancer_id` | All traffic → that balancer |
| `router_id` set | Router decides the balancer; **`balancer_id` is ignored** at request time |
| Both set | Valid YAML; `balancer_id` is a common “default” for humans / docs, but unused while the router is attached |

## Examples

### Balancer only (minimal)

```yaml
kind: Flow
version: v1
metadata:
  name: simple-flow
spec:
  balancer_id: main-rr
```

### Router + documented default balancer

```yaml
kind: Flow
version: v1
metadata:
  name: main-flow
spec:
  router_id: main-router
  balancer_id: main-rr
```
