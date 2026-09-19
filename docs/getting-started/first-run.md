# First run

Two ways to bring pgway up:

| Mode | Binaries | Agent registration? |
|------|----------|---------------------|
| **All-in-one** | `pgway` | **No** — CP and DP share the process; no agent bootstrap |
| **Distributed** | `pgway-cp` + `pgway-dp` (+ `pgctl`) | **Yes** — after admin init, create an agent registration token with `pgctl agent token create` |

Both modes need the **user** bootstrap token from CP **stderr** on first start (`pgctl init`). Only the split CP/DP layout adds the **agent** registration step.

Assumes you already [built binaries](installation.md) and have a [config file](configuration.md).

---

## Path A — All-in-one (recommended first)

### 1. Start `pgway`

```bash
./build/pgway --config ./config.toml
# or, if config.toml is on a search path:
./build/pgway
```

On a **first** start with an empty user store, the Control Plane prints a one-time bootstrap token to **stderr** (it is **not** written to structured logs) and refuses normal APIs until you run `pgctl init`. Example:

```text
pgway: no users found — initialize with:
  pgctl init --bootstrap-token pgw_…
```

Structured logs still show a warn without the secret:

```json
{"level":"warn","msg":"no users found — initialize with pgctl init (bootstrap token written to stderr only)", ...}
{"level":"info","msg":"grpc started","addr":":9090"}
{"level":"info","msg":"gateway started"}
```

What matters:

| Field / line | Meaning |
|--------------|---------|
| stderr `pgctl init --bootstrap-token …` | One-time secret for `pgctl init` (use **your** value) |
| `grpc started` / `addr` | CP gRPC listen (default `:9090`) — `pgctl` dials this |
| `gateway started` | Local Data Plane ready (no separate agent) |
| `restapi started` | REST / dashboard port (default `:8081`) — **experimental** |

!!! warning
    If you restart the server **before** `pgctl init`, a **new** bootstrap token is generated. The previous one is invalid.

    Do not commit bootstrap tokens or paste them into shared channels.

### 2. Initialize the admin user

```bash
./build/pgctl init --bootstrap-token 'pgw_…'   # paste the token from stderr
```

This creates the first admin and stores a session under `~/.pgctl/credentials`. Later:

```bash
./build/pgctl login --username admin
```

Point `pgctl` at the CP if needed:

```bash
PGWAY_GRPC_LISTEN_ADDR=localhost:9090 ./build/pgctl init --bootstrap-token 'pgw_…'
```

!!! note "No agent token here"
    All-in-one does **not** use `pgctl agent token create`. The gateway is already inside the same process as the CP.

### 3. Apply a minimal stack

Save as `stack.yaml` (replace the proxy URL with a real upstream you control):

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
---
kind: Pool
version: v1
metadata:
  name: main-pool
spec:
  title: Main Pool
  type: static
  members:
    - proxy_id: proxy-1
---
kind: LoadBalancer
version: v1
metadata:
  name: main-rr
spec:
  title: Main Round Robin
  type: round-robin
  pool_id: main-pool
---
kind: Flow
version: v1
metadata:
  name: main-flow
spec:
  balancer_id: main-rr
---
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

```bash
./build/pgctl apply -f stack.yaml
./build/pgctl get proxy
./build/pgctl get pool
./build/pgctl get balancer
./build/pgctl get flow
./build/pgctl get entrypoint
```

The in-process Data Plane hot-reloads; the entrypoint should listen shortly after apply.

### 4. Send traffic

```bash
curl -x http://localhost:8080 https://example.com
```

---

## Path B — Distributed (`pgway-cp` + `pgway-dp`)

Same admin bootstrap as above, plus **agent registration** so the standalone Data Plane can authenticate to the CP.

### 1. Start the Control Plane

```bash
./build/pgway-cp --config ./cp.toml
```

Look for the stderr bootstrap-token line and `grpc started` (there is no local `gateway started` on CP-only).

### 2. Initialize the admin

```bash
./build/pgctl init --bootstrap-token 'pgw_…'
```

### 3. Create an agent registration token

This step exists **only** for separate Data Plane processes:

```bash
./build/pgctl agent token create
# prints a single-use registration token — save it for the next command
```

!!! info "Two different tokens"
    - **Bootstrap token** (CP log) → `pgctl init` (first **user** / admin)
    - **Registration token** (`pgctl agent token create`) → first start of **`pgway-dp`** (agent)

    All-in-one never needs the registration token. Distributed mode always needs it once per new agent (or again after revoke / expired credentials).

### 4. Start the Data Plane

DP config must dial the CP (`grpc.listen_addr`) and set agent identity / state path — see [Configuration](configuration.md).

```bash
PGWAY_AGENT_REGISTRATION_TOKEN='<token-from-agent-token-create>' \
  ./build/pgway-dp --config ./dp.toml
```

On success the DP writes `{agent_id, agent_token}` to `agent.state_path`. Later restarts reuse that file — **no** registration token required until you delete the agent or credentials are gone.

```bash
./build/pgctl agent list    # active / passive / disconnected
```

### 5. Apply stack and send traffic

Same `stack.yaml` / `pgctl apply` / `curl -x …` as Path A. Entrypoints are served by **`pgway-dp`**, not by `pgway-cp`.

```bash
./build/pgctl apply -f stack.yaml
curl -x http://<dp-host>:8080 https://example.com
```

---

## Dashboard?

`restapi started` on `:8081` does **not** mean the UI is ready for production. Prefer `pgctl`. See the [Welcome](../index.md) experimental notice.

## What’s next

- [Resources & flow model](../concepts/resources.md) — extend the stack (router, weighted, least-bytes)
- [Resource reference](../reference/index.md) — full YAML fields per kind
- [Binaries & planes](../concepts/binaries.md) — CP vs DP responsibilities
- Authentication / agent deep-dives: [Authentication](../guides/authentication.md)
