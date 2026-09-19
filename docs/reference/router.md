# Router

Optional rule engine on a flow. Rules are evaluated **in order**; the **first match** wins. Each rule’s `target` is a LoadBalancer `metadata.name`.

**Kind:** `Router` · **Version:** `v1`

Behavior guide: [Routing](../guides/routing.md).

## Spec fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `title` | string | no | Human-readable title |
| `description` | string | no | Free-form description |
| `rules` | list | **yes** | Non-empty ordered rule list |

### `rules[]`

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | **yes** | Rule identity (unique within the router in practice) |
| `match` | object | **yes** | Condition tree — see below |
| `target` | string | **yes** | LoadBalancer name to use when this rule matches |

### `match`

A match must define **at least one** of: `type`, `all`, or `any`. Optional `not` negates a single condition (cannot stand alone).

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | one mode | Single-condition match type |
| `value` | string | if `type` set and not `catch_all` | Match operand |
| `all` | list of conditions | one mode | AND — every condition must match |
| `any` | list of conditions | one mode | OR — any condition may match |
| `not` | condition | no | Negate this one condition |

### Condition (`all` / `any` / `not` entries)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | **yes** | Match type |
| `value` | string | **yes*** | Operand (*not required for `catch_all`) |

## Match types

Validated on apply (domain):

| `type` | `value` | Behavior |
|--------|---------|----------|
| `host` | glob pattern | Match hostname (`filepath.Match`); **port stripped** (same as `host_suffix`) |
| `host_suffix` | suffix | Host suffix (port stripped; leading `.` normalized) |
| `path_prefix` | prefix | Prefix of request path |
| `path_regex` | regex | Regex against request path |
| `method` | method | HTTP method (case-insensitive) |
| `header` | `Name:Value` | Header equality (split on first `:`) |
| `catch_all` | *(omit)* | Always matches |

## Examples

### Ordered rules with catch-all

```yaml
kind: Router
version: v1
metadata:
  name: main-router
spec:
  title: Main Router
  description: Route by host; default to RR
  rules:
    - id: api
      match:
        type: host_suffix
        value: .api.example.com
      target: api-rr
    - id: catch-all
      match:
        type: catch_all
      target: main-rr
```

### Composite `all`

```yaml
kind: Router
version: v1
metadata:
  name: composite-router
spec:
  rules:
    - id: get-de
      match:
        all:
          - type: method
            value: GET
          - type: header
            value: "X-Region:de"
      target: de-rr
    - id: fallback
      match:
        type: catch_all
      target: main-rr
```

### `any` + `not`

```yaml
match:
  any:
    - type: host
      value: "*.cdn.example.com"
    - type: path_prefix
      value: /static/
  not:
    type: method
    value: OPTIONS
```
