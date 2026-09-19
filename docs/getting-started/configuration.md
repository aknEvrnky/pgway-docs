# Configuration

Every binary loads settings the same way: a config **file** is required, then optional **environment variables** and **flags** override it.

## Precedence

Highest wins:

1. **Flags** — `--config <path>` (all binaries); `--token` (`pgctl`)
2. **Environment** — `PGWAY_` + key in uppercase (e.g. `PGWAY_TOKEN`, `PGWAY_LOG_LEVEL`, `PGWAY_GRPC_LISTEN_ADDR`)
3. **Config file**
4. **Built-in defaults** (see table below)

A config file must exist on a search path (or via `--config`) even if you override every value with env vars.

## Where the file is loaded from

With `--config /path/to/file.yml`, that file is used exclusively.

Otherwise the process looks for `config.{yml,yaml,json,toml}` in order (first match wins):

1. `/etc/pgway/`
2. `$HOME/.pgway/`
3. `.` (current working directory)

Starting point in the repo: copy the checked-in defaults, then tweak paths for your machine:

```bash
cp config/default.yml ./config.yml
# or:
# mkdir -p ~/.pgway && cp config/default.yml ~/.pgway/config.yml
```

!!! tip "Local development"
    Prefer a project-local path for Badger and agent state (for example under `./var/…`) so you do not need root under `/var/pgway`.

## Default config file

Canonical source in the main repository:

**[`config/default.yml`](https://github.com/aknEvrnky/pgway/blob/main/config/default.yml)**

Full contents (keep this in sync with `main` when the file changes):

```yaml
# pgway default configuration.
# Every key can also be set via a PGWAY_-prefixed env var (e.g. PGWAY_TOKEN)
# or, for the file location, the --config flag. Precedence:
#   flags > env vars > this file > built-in defaults.

# BadgerDB data directory (pgway, pgway-cp).
badger_path: /var/pgway/lib

# How often to run Badger value log GC (pgway, pgway-cp). 0 disables.
badger_gc_interval: 5m

# gRPC listen address for the control plane. pgway-dp / pgctl dial this address.
grpc_listen_addr: ":9090"

# gRPC keepalive ping interval (server + client). 0 disables.
grpc_keepalive_interval: 1m

# How long to wait for a keepalive ping ACK. Required when interval > 0.
grpc_keepalive_timeout: 20s

# REST API listen address (pgway, pgway-cp).
rest_listen_addr: ":8081"

# Outgoing control-plane auth token — pgctl in distributed mode.
# Prefer the PGWAY_TOKEN env var for secrets; leave empty for all-in-one.
# pgway-dp uses an agent token from agent_state_path after registration.
token: ""

# Default lifetime of login-issued tokens (pgway, pgway-cp).
token_ttl: 720h

# Single-use agent registration token default TTL (pgway, pgway-cp).
registration_token_ttl: 24h

# Sliding TTL for per-agent bearer tokens; extended on every heartbeat.
agent_token_ttl: 168h

# Derived agent status boundary: within → active, beyond → disconnected.
agent_heartbeat_threshold: 30s

# --- Data plane (pgway-dp) agent identity ---

# Unique agent name registered with the CP. Empty → hostname.
agent_name: ""

# Operator-declared labels advertised at Register (seed for placement).
agent_labels: {}

# Persisted agent credentials ({agent_id, agent_token}). Dir 0700, file 0600.
agent_state_path: /var/lib/pgway/agent.json

# How often pgway-dp sends Heartbeat RPCs.
heartbeat_interval: 10s

# First-time Register secret. Prefer PGWAY_REGISTRATION_TOKEN; leave empty
# when agent_state_path already holds credentials.
registration_token: ""

# Max request body for non-CONNECT proxy HTTP. 0 = unlimited.
# Accepts integers or human sizes (10MiB, 512KiB, 100MB). Also PGWAY_MAX_REQUEST_BODY_BYTES.
max_request_body_bytes: 10MiB

# Global log level: debug | info | warn | error (also PGWAY_LOG_LEVEL).
log_level: info
```

## Keys

| Key | Default | Used by | Description |
|-----|---------|---------|-------------|
| `badger_path` | `/var/pgway/lib` | `pgway`, `pgway-cp` | BadgerDB directory |
| `badger_gc_interval` | `5m` | `pgway`, `pgway-cp` | Badger value log GC period; `0` disables |
| `grpc_listen_addr` | `:9090` | all | CP listen address; DP/`pgctl` dial this host:port |
| `grpc_keepalive_interval` | `1m` | all | gRPC keepalive ping period (server + client); `0` disables |
| `grpc_keepalive_timeout` | `20s` | all | Keepalive ping ACK wait; must be `> 0` when interval is enabled |
| `rest_listen_addr` | `:8081` | `pgway`, `pgway-cp` | REST API (dashboard; **experimental**, auth incomplete) |
| `token` | *(empty)* | `pgctl` | Bearer for CP calls; prefer `PGWAY_TOKEN` or `~/.pgctl/credentials` |
| `token_ttl` | `720h` | `pgway`, `pgway-cp` | Default login token lifetime |
| `registration_token_ttl` | `24h` | `pgway`, `pgway-cp` | Default TTL for single-use agent registration tokens |
| `agent_token_ttl` | `168h` | `pgway`, `pgway-cp` | Sliding TTL for per-agent tokens |
| `agent_heartbeat_threshold` | `30s` | `pgway`, `pgway-cp` | Active vs disconnected boundary |
| `agent_name` | *(hostname)* | `pgway-dp` | Unique agent identity |
| `agent_labels` | `{}` | `pgway-dp` | Labels advertised at Register |
| `agent_state_path` | `/var/lib/pgway/agent.json` | `pgway-dp` | Persisted `{agent_id, agent_token}` (dir `0700`, file `0600`) |
| `heartbeat_interval` | `10s` | `pgway-dp` | Heartbeat period |
| `registration_token` | *(empty)* | `pgway-dp` | First Register secret; prefer `PGWAY_REGISTRATION_TOKEN` |
| `max_request_body_bytes` | `10MiB` | `pgway`, `pgway-dp` | Cap for non-CONNECT proxy request bodies; `0` = unlimited; accepts `10MiB` / `512KiB` / bare bytes |
| `log_level` | `info` | all | `debug` \| `info` \| `warn` \| `error` |

`pgctl` credentials after `init` / `login` live under **`~/.pgctl/credentials`**, separate from the shared `~/.pgway/` config search path.

## Examples

### All-in-one (`pgway`) — local

```yaml
# config.yml
badger_path: ./var/lib
grpc_listen_addr: ":9090"
rest_listen_addr: ":8081"
log_level: info
token_ttl: 720h
```

```bash
./build/pgway --config ./config.yml
# or from the directory that contains config.yml:
./build/pgway
```

### Control Plane only (`pgway-cp`)

Same shape as all-in-one for storage and listen addresses; no entrypoint traffic.

```yaml
badger_path: /var/pgway/lib
grpc_listen_addr: ":9090"
rest_listen_addr: ":8081"
log_level: info
```

### Data Plane agent (`pgway-dp`)

`grpc_listen_addr` is the **CP address to dial**, not a local listen for gRPC admin APIs.

```yaml
# dp.yml
grpc_listen_addr: "cp-host:9090"
agent_name: edge-1
agent_labels:
  zone: edge
agent_state_path: ./var/agent.json
heartbeat_interval: 10s
log_level: info
```

First start needs a one-time registration token (created on the CP with `pgctl agent token create`); later starts reuse `agent_state_path`. Details belong in the agent / first-run guides.

### CLI (`pgctl`)

Often only needs how to reach the CP and a token:

```yaml
# optional pgctl-oriented snippet
grpc_listen_addr: "localhost:9090"
token: ""   # or set PGWAY_TOKEN / use ~/.pgctl/credentials
```

```bash
./build/pgctl --config ./config.yml get proxy
PGWAY_GRPC_LISTEN_ADDR=localhost:9090 PGWAY_TOKEN=… ./build/pgctl get proxy
```

## REST and the dashboard

`rest_listen_addr` enables the Control Plane REST surface used by the Nuxt dashboard. That path is **experimental**: expect breaking changes and incomplete authentication. Prefer **gRPC + `pgctl`** for real configuration until the dashboard matures.

## What’s next

[First run](first-run.md) — bootstrap admin, apply a minimal stack, send traffic. See also [Binaries & planes](../concepts/binaries.md) and [Installation](installation.md).
