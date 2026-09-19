# CLI (`pgctl`)

`pgctl` is the operator client for the Control Plane over **gRPC**. It does not proxy client HTTP traffic.

Auth and tokens: [Authentication](authentication.md). Config dial address: [Configuration](../getting-started/configuration.md).

## Global flags

| Flag | Description |
|------|-------------|
| `--config <path>` | Config file (same search rules as other binaries if omitted) |
| `--token <bearer>` | Override stored credentials for this invocation |

Token resolution: `--token` → `PGWAY_TOKEN` / config `token` → `~/.pgctl/credentials`.

Typical dial:

```bash
PGWAY_GRPC_LISTEN_ADDR=localhost:9090 ./build/pgctl …
# or put grpc.listen_addr in the config file
```

## Auth

| Command | Description |
|---------|-------------|
| `pgctl init --bootstrap-token <tok>` | Create first admin (`PGWAY_BOOTSTRAP_TOKEN` also accepted). Optional `--username` / `--password` (prompted if omitted) |
| `pgctl login --username <u>` | Issue session token. `--ttl`, `--no-expiry`, `--password` |
| `pgctl logout` | Revoke current token and clear `~/.pgctl/credentials` |

## Resources

### Apply

```bash
pgctl apply -f stack.yaml
pgctl apply --file proxy.yaml
```

Multi-document YAML (`---` separators) is supported. Kind/version must match the [resource reference](../reference/index.md).

`pgctl apply` sorts documents by dependency order before applying
(`proxy` → `pool` → `balancer` → `router` → `flow` → `entrypoint`), so reverse file order still works.
Single-resource gRPC apply (and dashboard/REST later) still requires bottom-up creation: referenced targets must already exist.

Missing forward references return gRPC `FailedPrecondition` (same family as in-use deletes).

### Get

```bash
pgctl get proxy
pgctl get proxy <name>
pgctl get pool
pgctl get balancer
pgctl get router
pgctl get flow
pgctl get entrypoint
```

List (no name) vs get-one (with name), per kind.

### Delete

```bash
pgctl delete proxy <name>
pgctl delete pool <name>
pgctl delete balancer <name>
pgctl delete router <name>
pgctl delete flow <name>
pgctl delete entrypoint <name>
```

Deletes are **reject-only**: if another resource still references the target, the API returns
gRPC `FailedPrecondition` and names the dependents. Safe teardown order:

`entrypoint` → `flow` → `router` → `balancer` → `pool` → `proxy`

A proxy listed in a **static** pool cannot be deleted until removed from (or the pool deleted).
A proxy matched only by a **dynamic** pool selector may be deleted.

## Users (admin)

| Command | Description |
|---------|-------------|
| `pgctl user create <name>` | Create user; prints one-time temporary password. `--role admin` optional |
| `pgctl user list` | List users |
| `pgctl user change-password` | Change own password |
| `pgctl user change-password <name>` | Admin reset |
| `pgctl user delete <name>` | Delete user (last admin cannot be deleted) |

## Agents (admin)

Distributed Data Planes only — not required for all-in-one `pgway`.

| Command | Description |
|---------|-------------|
| `pgctl agent token create` | Single-use registration token for `pgway-dp` first start |
| `pgctl agent list` | List agents (`active` / `passive` / `disconnected`) |
| `pgctl agent delete <name>` | Revoke credentials and remove registry row |

```bash
REG=$(pgctl agent token create)
PGWAY_AGENT_REGISTRATION_TOKEN="$REG" ./build/pgway-dp --config ./dp.toml
pgctl agent list
```

## Quick recipes

```bash
# Fresh all-in-one box
./build/pgway --config ./config.toml
# … copy bootstrap_token from logs …
./build/pgctl init --bootstrap-token 'pgw_…'
./build/pgctl apply -f stack.yaml
./build/pgctl get entrypoint

# Re-auth later
./build/pgctl login --username admin
```

## Related

- [First run](../getting-started/first-run.md)
- [Authentication](authentication.md)
- [Resource reference](../reference/index.md)
