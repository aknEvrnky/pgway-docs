# Configuration

Every binary loads settings the same way: a **TOML** config **file** is required, then optional **environment variables** and **flags** override it.

Process/runtime config is TOML. Control-plane **resource** specs (`pgctl apply`, `stack.yaml`) remain YAML.

## Precedence

Highest wins:

1. **Flags** — `--config <path>` (all binaries); `--token` (`pgctl`)
2. **Environment** — `PGWAY_` + key in uppercase; nested keys use `_` for `.` (e.g. `PGWAY_TOKEN`, `PGWAY_LOG_LEVEL`, `PGWAY_GRPC_LISTEN_ADDR`, `PGWAY_AGENT_REGISTRATION_TOKEN`)
3. **Config file**
4. **Built-in defaults** (see table below)

A config file must exist on a search path (or via `--config`) even if you override every value with env vars.

## Where the file is loaded from

With `--config /path/to/file.toml`, that file is used exclusively.

Otherwise the process looks for `config.toml` in:

1. `/etc/pgway/`
2. `$HOME/.pgway/`
3. `.` (current working directory)

Starting point in the repo: copy the checked-in defaults, then tweak paths for your machine:

```bash
cp config/default.toml ./config.toml
# or:
# mkdir -p ~/.pgway && cp config/default.toml ~/.pgway/config.toml
```

!!! tip "Local development"
    Prefer a project-local path for Badger and agent state (for example under `./var/…`) so you do not need root under `/var/pgway`.

## Default config file

Canonical source in the main repository:

**[`config/default.toml`](https://github.com/aknEvrnky/pgway/blob/main/config/default.toml)**

Full contents (keep this in sync with `main` when the file changes):

```toml
# pgway default configuration.
# Every key can also be set via a PGWAY_-prefixed env var (e.g. PGWAY_TOKEN,
# PGWAY_BADGER_PATH, PGWAY_AGENT_REGISTRATION_TOKEN) or, for the file location,
# the --config flag. Nested keys use underscores in env names (badger.path →
# PGWAY_BADGER_PATH). Precedence:
#   flags > env vars > this file > built-in defaults.

log_level = "info"

# Outgoing control-plane auth token — pgctl in distributed mode.
# Prefer the PGWAY_TOKEN env var for secrets; leave empty for all-in-one.
# pgway-dp uses an agent token from agent.state_path after registration.
token = ""

[badger]
# BadgerDB data directory (pgway, pgway-cp).
path = "/var/pgway/lib"
# How often to run Badger value log GC (pgway, pgway-cp). 0 disables.
gc_interval = "5m"

[grpc]
# gRPC listen address for the control plane. pgway-dp / pgctl dial this address.
listen_addr = ":9090"
# gRPC keepalive ping interval (server + client). 0 disables.
keepalive_interval = "1m"
# How long to wait for a keepalive ping ACK. Required when interval > 0.
keepalive_timeout = "20s"
# Per-client unary RPC token-bucket rate (tokens/sec). 0 disables.
rate_limit_rps = 100
# Token-bucket burst capacity. Required when rate_limit_rps > 0.
rate_limit_burst = 200

[rest]
# REST API listen address (pgway, pgway-cp).
listen_addr = ":8081"

[probes]
# Opt-in process liveness/readiness HTTP listener (K8s probes). Off by default.
enabled = false
# Dedicated bind address — prefer loopback or cluster-internal (e.g. 127.0.0.1:8082).
# Do not reuse entrypoint / gRPC / REST ports; avoid binding on public interfaces.
listen_addr = ":8082"

[auth]
# Default lifetime of login-issued tokens (pgway, pgway-cp).
token_ttl = "720h"
# Single-use agent registration token default TTL (pgway, pgway-cp).
registration_token_ttl = "24h"
# Sliding TTL for per-agent bearer tokens; extended on every heartbeat.
agent_token_ttl = "168h"

[agent]
# Unique agent name registered with the CP. Empty → hostname.
name = ""
# Operator-declared labels advertised at Register (seed for placement).
labels = {}
# Persisted agent credentials ({agent_id, agent_token}). Dir 0700, file 0600.
state_path = "/var/lib/pgway/agent.json"
# How often pgway-dp sends Heartbeat RPCs.
heartbeat_interval = "10s"
# Derived agent status boundary: within → active, beyond → disconnected.
heartbeat_threshold = "30s"
# First-time Register secret. Prefer PGWAY_AGENT_REGISTRATION_TOKEN; leave empty
# when agent.state_path already holds credentials.
registration_token = ""

[proxy]
# Max request body for non-CONNECT proxy HTTP. 0 = unlimited.
# Accepts integers or human sizes (10MiB, 512KiB, 100MB).
# Also PGWAY_PROXY_MAX_REQUEST_BODY_BYTES.
max_request_body_bytes = "10MiB"
# Per-proxy http.Transport pool (pgway, pgway-dp). 0 max_idle_conns = unlimited.
max_idle_conns = 1024
max_idle_conns_per_host = 128
idle_conn_timeout = "90s"
dial_timeout = "10s"

[proxy.dns_cache]
# Fixed-TTL cache for upstream proxy hostname resolution (pgway, pgway-dp).
# enabled=false uses the default resolver on every dial.
enabled = true
# How long successful lookups are reused. Must be > 0 when enabled.
ttl = "5m"

[dataplane]
# setTimeout-style change-event coalesce window (pgway, pgway-dp). 0 disables.
event_coalesce_window = "100ms"
# Early flush when this many events land in one key before the window ends.
event_coalesce_max_buffer = 256
# All-in-one missed-event safety net (Bootstrap + listener reconcile). 0 disables.
# Ignored by pgway-dp (Watch reconnect is the primary heal path).
# Default 5m avoids resetting LB cursors too often when config is unchanged.
event_resync_interval = "5m"
# Distributed DP: fail_open keeps serving last-known config when CP is down;
# fail_closed rejects new proxy requests with 503 after unreachable (see threshold note).
cp_disconnect_strategy = "fail_open"
# Both HB and Watch must be stale, then remain so for this duration, before
# unreachable. After CP death this can take up to ~2× this value (HB age-out + window).
cp_disconnect_unreachable_threshold = "30s"
# Hysteresis before leaving unreachable once proofs are fresh again. 0 = immediate.
cp_disconnect_recover_threshold = "0s"

[otel]
# Opt-in OpenTelemetry metrics (OTLP/gRPC push to your collector). Off by default.
enabled = false
# OTLP/gRPC collector host:port (no scheme).
endpoint = "localhost:4317"
# Use plaintext OTLP/gRPC (no TLS). Default false = TLS to the collector.
insecure = false
# Resource service.name; empty → binary default (pgway / pgway-cp / pgway-dp).
service_name = ""
# Periodic metric export interval.
export_interval = "15s"
```

## Keys

| Key | Env | Default | Used by | Description |
|-----|-----|---------|---------|-------------|
| `log_level` | `PGWAY_LOG_LEVEL` | `info` | all | `debug` \| `info` \| `warn` \| `error` |
| `token` | `PGWAY_TOKEN` | *(empty)* | `pgctl` | Bearer for CP calls; prefer env or `~/.pgctl/credentials` |
| `badger.path` | `PGWAY_BADGER_PATH` | `/var/pgway/lib` | `pgway`, `pgway-cp` | BadgerDB directory |
| `badger.gc_interval` | `PGWAY_BADGER_GC_INTERVAL` | `5m` | `pgway`, `pgway-cp` | Badger value log GC period; `0` disables |
| `grpc.listen_addr` | `PGWAY_GRPC_LISTEN_ADDR` | `:9090` | all | CP **listen** address (`pgway` / `pgway-cp`) |
| `grpc.dial_addr` | `PGWAY_GRPC_DIAL_ADDR` | *(empty → listen_addr)* | `pgway-dp`, `pgctl` | CP address to **dial**; leave empty to reuse `listen_addr` locally |
| `grpc.keepalive_interval` | `PGWAY_GRPC_KEEPALIVE_INTERVAL` | `1m` | all | gRPC keepalive ping period; `0` disables |
| `grpc.keepalive_timeout` | `PGWAY_GRPC_KEEPALIVE_TIMEOUT` | `20s` | all | Keepalive ping ACK wait; must be `> 0` when interval is enabled |
| `grpc.rate_limit_rps` | `PGWAY_GRPC_RATE_LIMIT_RPS` | `100` | `pgway`, `pgway-cp` | Per-client unary RPC token-bucket rate; `0` disables |
| `grpc.rate_limit_burst` | `PGWAY_GRPC_RATE_LIMIT_BURST` | `200` | `pgway`, `pgway-cp` | Token-bucket burst; must be `>= 1` when rps is enabled |
| `rest.listen_addr` | `PGWAY_REST_LISTEN_ADDR` | `:8081` | `pgway`, `pgway-cp` | REST API (dashboard; **experimental**, auth incomplete) |
| `probes.enabled` | `PGWAY_PROBES_ENABLED` | `false` | all | Start dedicated `/healthz` + `/readyz` listener |
| `probes.listen_addr` | `PGWAY_PROBES_LISTEN_ADDR` | `:8082` | all | Probe bind address; prefer `127.0.0.1:8082` or cluster-internal; do not expose publicly |
| `auth.token_ttl` | `PGWAY_AUTH_TOKEN_TTL` | `720h` | `pgway`, `pgway-cp` | Default login token lifetime |
| `auth.registration_token_ttl` | `PGWAY_AUTH_REGISTRATION_TOKEN_TTL` | `24h` | `pgway`, `pgway-cp` | Default TTL for single-use agent registration tokens |
| `auth.agent_token_ttl` | `PGWAY_AUTH_AGENT_TOKEN_TTL` | `168h` | `pgway`, `pgway-cp` | Sliding TTL for per-agent tokens |
| `agent.name` | `PGWAY_AGENT_NAME` | *(hostname)* | `pgway-dp` | Unique agent identity |
| `agent.labels` | — | `{}` | `pgway-dp` | Labels advertised at Register |
| `agent.state_path` | `PGWAY_AGENT_STATE_PATH` | `/var/lib/pgway/agent.json` | `pgway-dp` | Persisted `{agent_id, agent_token}` (dir `0700`, file `0600`) |
| `agent.heartbeat_interval` | `PGWAY_AGENT_HEARTBEAT_INTERVAL` | `10s` | `pgway-dp` | Heartbeat period |
| `agent.heartbeat_threshold` | `PGWAY_AGENT_HEARTBEAT_THRESHOLD` | `30s` | `pgway`, `pgway-cp` | Active vs disconnected boundary |
| `agent.registration_token` | `PGWAY_AGENT_REGISTRATION_TOKEN` | *(empty)* | `pgway-dp` | First Register secret |
| `proxy.max_request_body_bytes` | `PGWAY_PROXY_MAX_REQUEST_BODY_BYTES` | `10MiB` | `pgway`, `pgway-dp` | Cap for non-CONNECT bodies; `0` = unlimited |
| `proxy.max_idle_conns` | `PGWAY_PROXY_MAX_IDLE_CONNS` | `1024` | `pgway`, `pgway-dp` | Global idle conn limit; `0` = unlimited |
| `proxy.max_idle_conns_per_host` | `PGWAY_PROXY_MAX_IDLE_CONNS_PER_HOST` | `128` | `pgway`, `pgway-dp` | Idle conns per upstream host; must be `> 0` |
| `proxy.idle_conn_timeout` | `PGWAY_PROXY_IDLE_CONN_TIMEOUT` | `90s` | `pgway`, `pgway-dp` | How long idle pooled connections are kept |
| `proxy.dial_timeout` | `PGWAY_PROXY_DIAL_TIMEOUT` | `10s` | `pgway`, `pgway-dp` | TCP dial timeout to upstream proxies |
| `proxy.dns_cache.enabled` | `PGWAY_PROXY_DNS_CACHE_ENABLED` | `true` | `pgway`, `pgway-dp` | Cache upstream proxy hostname lookups; `false` = default resolver every dial |
| `proxy.dns_cache.ttl` | `PGWAY_PROXY_DNS_CACHE_TTL` | `5m` | `pgway`, `pgway-dp` | How long successful lookups are reused; must be `> 0` when enabled |
| `dataplane.event_coalesce_window` | `PGWAY_DATAPLANE_EVENT_COALESCE_WINDOW` | `100ms` | `pgway`, `pgway-dp` | Change-event coalesce delay; `0` disables (immediate dispatch) |
| `dataplane.event_coalesce_max_buffer` | `PGWAY_DATAPLANE_EVENT_COALESCE_MAX_BUFFER` | `256` | `pgway`, `pgway-dp` | Early flush threshold per coalesce key; must be `>= 1` when window is enabled |
| `dataplane.event_resync_interval` | `PGWAY_DATAPLANE_EVENT_RESYNC_INTERVAL` | `5m` | `pgway` | Periodic full Resync (missed-event heal); `0` disables; ignored by `pgway-dp` |
| `dataplane.cp_disconnect_strategy` | `PGWAY_DATAPLANE_CP_DISCONNECT_STRATEGY` | `fail_open` | `pgway-dp` | `fail_open` \| `fail_closed` when CP unreachable |
| `dataplane.cp_disconnect_unreachable_threshold` | `PGWAY_DATAPLANE_CP_DISCONNECT_UNREACHABLE_THRESHOLD` | `30s` | `pgway-dp` | Both proofs stale, then this long → unreachable (≈ up to 2× after CP death) |
| `dataplane.cp_disconnect_recover_threshold` | `PGWAY_DATAPLANE_CP_DISCONNECT_RECOVER_THRESHOLD` | `0s` | `pgway-dp` | Hysteresis leaving unreachable; `0` = immediate |
| `otel.enabled` | `PGWAY_OTEL_ENABLED` | `false` | all | Enable OTLP/gRPC metrics push |
| `otel.endpoint` | `PGWAY_OTEL_ENDPOINT` | `localhost:4317` | all | OTLP/gRPC collector `host:port` (no scheme); required when enabled |
| `otel.insecure` | `PGWAY_OTEL_INSECURE` | `false` | all | Plaintext OTLP/gRPC (no TLS); default `false` = TLS to the collector |
| `otel.service_name` | `PGWAY_OTEL_SERVICE_NAME` | *(binary name)* | all | Resource `service.name`; empty → `pgway` / `pgway-cp` / `pgway-dp` |
| `otel.export_interval` | `PGWAY_OTEL_EXPORT_INTERVAL` | `15s` | all | Periodic export interval; must be `> 0` when enabled |

`pgctl` credentials after `init` / `login` live under **`~/.pgctl/credentials`**, separate from the shared `~/.pgway/` config search path.

## Examples

### All-in-one (`pgway`) — local

```toml
# config.toml
log_level = "info"

[badger]
path = "./var/lib"

[grpc]
listen_addr = ":9090"

[rest]
listen_addr = ":8081"

[auth]
token_ttl = "720h"
```

```bash
./build/pgway --config ./config.toml
# or from the directory that contains config.toml:
./build/pgway
```

### Control Plane only (`pgway-cp`)

Same shape as all-in-one for storage and listen addresses; no entrypoint traffic.

```toml
log_level = "info"

[badger]
path = "/var/pgway/lib"

[grpc]
listen_addr = ":9090"

[rest]
listen_addr = ":8081"
```

### Data Plane agent (`pgway-dp`)

Prefer `grpc.dial_addr` for the **CP address to dial**. If empty, clients fall back to `grpc.listen_addr` (legacy / local convenience).

```toml
# dp.toml
log_level = "info"

[grpc]
dial_addr = "cp-host:9090"

[agent]
name = "edge-1"
labels = { zone = "edge" }
state_path = "./var/agent.json"
heartbeat_interval = "10s"
```

First start needs a one-time registration token (created on the CP with `pgctl agent token create`); later starts reuse `agent.state_path`. Details belong in the agent / first-run guides.

### CLI (`pgctl`)

Often only needs how to reach the CP and a token:

```toml
# optional pgctl-oriented snippet
token = ""   # or set PGWAY_TOKEN / use ~/.pgctl/credentials

[grpc]
dial_addr = "localhost:9090"
```

```bash
./build/pgctl --config ./config.toml get proxy
PGWAY_GRPC_DIAL_ADDR=localhost:9090 PGWAY_TOKEN=… ./build/pgctl get proxy
```

## REST and the dashboard

`rest.listen_addr` enables the Control Plane REST surface used by the Nuxt dashboard. That path is **experimental**: expect breaking changes and incomplete authentication. Prefer **gRPC + `pgctl`** for real configuration until the dashboard matures.

## What’s next

[First run](first-run.md) — bootstrap admin, apply a minimal stack, send traffic. See also [Binaries & planes](../concepts/binaries.md) and [Installation](installation.md).
