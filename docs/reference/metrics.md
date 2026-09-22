# Metrics reference

pgway exports **OpenTelemetry metrics** over **OTLP/gRPC** (push) when `[otel] enabled = true`. There is no Prometheus `/metrics` scrape endpoint.

Point `otel.endpoint` at your collector; scrape Prometheus / build Grafana dashboards from your own stack — or run the bundled [Observability stack](../guides/observability.md). See [Configuration](../getting-started/configuration.md) for keys.

Export uses **TLS by default**. Set `otel.insecure = true` only when the collector listens on plaintext OTLP/gRPC (e.g. a local dev collector).

## Resource attributes

| Attribute | Value |
|-----------|--------|
| `service.name` | `otel.service_name`, or binary default (`pgway`, `pgway-cp`, `pgway-dp`) |
| `service.version` | Build version when set; otherwise `dev` |
| `process.runtime.name` | `go` (via OTel resource defaults / contrib) |

## Data plane (`pgway`, `pgway-dp`)

| Name | Type | Unit | Attributes |
|------|------|------|------------|
| `pgway.proxy.requests` | counter | `{request}` | `entrypoint`, `result`, `protocol` |
| `pgway.proxy.duration` | histogram | `s` | `entrypoint`, `result`, `protocol` |
| `pgway.proxy.bytes` | counter | `By` | `entrypoint`, `direction`, `protocol` |
| `pgway.proxy.active` | up-down counter | `{connection}` | `protocol` |

`pgway.proxy.duration` uses explicit bucket boundaries (seconds): `0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10, 30, 60, 300`.

### Attribute values

| Attribute | Values |
|-----------|--------|
| `protocol` | `http`, `connect` |
| `result` | `ok`, `error`, `timeout`, `rejected` |
| `direction` | `in`, `out` |
| `entrypoint` | Entrypoint resource id |

**`result` mapping (approx.):**

- `ok` — successful HTTP forward or CONNECT tunnel completion
- `timeout` — upstream/gateway timeout (HTTP 504)
- `rejected` — fail_closed 503, body too large (413), missing entrypoint, no matching flow/pool/rule (no upstream selected)
- `error` — other failures (502, dial/DNS, etc.)

Labels such as `proxy_id` / `pool_id` are **not** exported (cardinality).

## Control plane (`pgway-cp`, `pgway`)

| Name | Type | Unit | Attributes |
|------|------|------|------------|
| `pgway.cp.rpc.requests` | counter | `{request}` | `method`, `result` |
| `pgway.cp.rpc.duration` | histogram | `s` | `method`, `result` |

Both duration histograms share the same explicit bucket boundaries (seconds): `0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10, 30, 60, 300`.

| Attribute | Values |
|-----------|--------|
| `method` | Short RPC name (e.g. `ApplyProxy`, `ListAgents`) — last path segment of the gRPC full method |
| `result` | `ok`, `rejected`, `timeout`, `error` |

**`result` mapping:** `rejected` — client-side rejections (unauthenticated, permission denied, rate limited, invalid argument); `timeout` — deadline exceeded; `error` — any other handler failure.

Stream RPCs (e.g. Watch) are not counted in this release.

## Go runtime

When OTel is enabled, pgway starts [`go.opentelemetry.io/contrib/instrumentation/runtime`](https://pkg.go.dev/go.opentelemetry.io/contrib/instrumentation/runtime) so GC, memory, and related Go runtime metrics are exported alongside app metrics. Exact metric names follow upstream contrib; see that package’s docs for the current set.
