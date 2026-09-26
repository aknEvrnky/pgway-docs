# Observability stack

pgway pushes **OTLP/gRPC metrics** (and optionally **traces**) — there is no Prometheus `/metrics` scrape endpoint. To view the metrics locally without assembling your own pipeline, the repo ships a bundled Docker Compose stack under `docker/observability/`:

| Component | Purpose | Image |
|-----------|---------|-------|
| OTel Collector | receives OTLP (gRPC `:4317`, HTTP `:4318`), re-exports metrics as Prometheus; traces → debug log | `otel/opentelemetry-collector:0.161.0` |
| Prometheus | scrapes the collector; file-based TSDB, 15d retention | `prom/prometheus:v3.14.0` |
| Grafana | dashboards; Prometheus datasource auto-provisioned | `grafana/grafana:13.2.2` |

The scrape config sets `honor_labels: true`, so the OTLP `service.name` resource attribute survives as the Prometheus `job` label (`pgway`, `pgway-cp`, `pgway-dp`). All published ports bind to `127.0.0.1` only.

The bundled stack does **not** include Tempo/Jaeger. Traces received by the collector are written to the collector debug exporter so spans are not dropped; point `otel.endpoint` at Tempo or any other OTLP backend when you want a trace UI.

## Run the stack

```bash
docker compose -f docker/observability/compose.yaml up -d
```

| Endpoint | URL |
|----------|-----|
| Grafana | http://localhost:3000 (admin / admin) |
| Prometheus UI | http://localhost:9091 |
| OTLP gRPC / HTTP | `localhost:4317` / `localhost:4318` |
| Collector health check | http://localhost:13133 |

Prometheus is published on **9091** because pgway's default gRPC listen port is `:9090`.

Prometheus TSDB and Grafana state live in named volumes (`prom-data`, `grafana-data`) and survive `docker compose down`; run `down -v` to wipe them.

## Point pgway at it

```toml
[otel]
enabled = true
endpoint = "localhost:4317"
insecure = true   # the bundled collector listens on plaintext OTLP/gRPC
# Optional traces (requires enabled = true):
# traces_enabled = true
# trace_sample_ratio = 0.1
```

The same keys work as `PGWAY_OTEL_*` environment variables. See [Configuration](../getting-started/configuration.md) for all keys and the [metrics reference](../reference/metrics.md) for what gets exported.

### Traces

With `otel.enabled = true` and `otel.traces_enabled = true`, pgway exports spans over the same OTLP/gRPC endpoint:

- **Proxy hot path:** `pgway.proxy.request` (parent) and `pgway.proxy.upstream` (Dial / RoundTrip child)
- **CP unary gRPC:** `pgway.cp.rpc`

Sampling uses ParentBased + TraceIDRatioBased (`otel.trace_sample_ratio`, default `0.1`). Span attributes mirror metrics (`entrypoint` / `result` / `protocol`, or `method` / `result`) — no raw URLs, headers, or tokens.

To inspect traces in a UI, run Tempo (or Jaeger) and either point pgway at it directly or add an OTLP exporter from the collector to that backend.

## Import the dashboard

A ready-made dashboard export ships with the stack:

**[pgway-otel-metrics.json](https://github.com/aknEvrnky/pgway/blob/main/docker/observability/grafana/dashboards/pgway-otel-metrics.json)**

If you have the repo checked out, the file also lives at `docker/observability/grafana/dashboards/pgway-otel-metrics.json`.

In Grafana: **Dashboards → New → Import**, then upload the file or paste its contents. The dashboard's `datasource` variable falls back to your default Prometheus datasource, and the `job` variable filters by `service.name`.

!!! note
    The export uses the Grafana **dashboards v2** schema; import it into Grafana 13 or newer.

## Related

- [Metrics reference](../reference/metrics.md) — exported names, attributes, result mappings
- [Configuration](../getting-started/configuration.md) — `[otel]` keys
