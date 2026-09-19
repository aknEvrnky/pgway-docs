# Proxy

A single upstream proxy endpoint. Referenced by static pool `members[].proxy_id` or discovered via **labels** for dynamic pools.

**Kind:** `Proxy` · **Version:** `v1`

## Spec fields

You must use **either** `url` **or** the explicit `protocol` + `host` + `port` form (not a mix of incomplete fields).

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `url` | string | one mode | Shorthand URL, e.g. `http://user:pass@host:port` or `socks5://host:port` |
| `protocol` | string | if no `url` | Upstream protocol (`http`, `https`, `socks5`) |
| `host` | string | if no `url` | Upstream host / IP |
| `port` | number | if no `url` | Upstream port (must be ≠ 0) |
| `auth` | object | no | Basic credentials when not embedded in `url` |

### `auth` (optional)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `user` | string | no | Username |
| `pass` | string | no | Password |

## Validation & notes

- Schema: if `url` is set → accepted; otherwise `protocol`, `host`, and non-zero `port` are required.
- Domain accepts protocols `http`, `https`, and `socks5`. Upstream SOCKS5 is supported on the dial / CONNECT path (and configured like any other Proxy). The **client-facing entrypoint** remains an HTTP proxy listener.
- Scheme-less URLs are treated as `http` when parsed.
- Labels belong under `metadata.labels`, not `spec`.

## Examples

### URL shorthand

```yaml
kind: Proxy
version: v1
metadata:
  name: proxy-1
  labels:
    provider: example
    region: us-east
spec:
  url: http://user:pass@1.2.3.4:8080
```

### SOCKS5 upstream

```yaml
kind: Proxy
version: v1
metadata:
  name: proxy-socks
spec:
  url: socks5://user:pass@1.2.3.4:1080
```

### Explicit fields

```yaml
kind: Proxy
version: v1
metadata:
  name: proxy-2
spec:
  protocol: http
  host: 1.2.3.4
  port: 8080
  auth:
    user: user
    pass: pass
```
