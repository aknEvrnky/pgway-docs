# Entrypoint

Listen address where clients connect. Points at a **Flow**.

**Kind:** `Entrypoint` · **Version:** `v1`

## Spec fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `title` | string | no | Human-readable title |
| `protocol` | string | **yes** | Listener / proxy protocol (MVP: `http`) |
| `host` | string | **yes** | Bind host (e.g. `0.0.0.0`) |
| `port` | number | **yes** | Bind port (must be ≠ 0) |
| `flow_id` | string | **yes** | Flow `metadata.name` |

## Validation & notes

- Schema requires `protocol`, `host`, non-zero `port`, and `flow_id`.
- Domain protocols include `http`, `https`, `socks5`; the implemented client path today is **HTTP proxy** (CONNECT + plain HTTP).
- Multiple entrypoints can run in one Data Plane process on different ports.
- After apply, the Data Plane hot-reloads and binds the listener (all-in-one or `pgway-dp`).

## Example

```yaml
kind: Entrypoint
version: v1
metadata:
  name: main-ep
spec:
  title: Main Gateway
  protocol: http
  host: 0.0.0.0
  port: 8080
  flow_id: main-flow
```

Clients:

```bash
curl -x http://localhost:8080 https://example.com
```
