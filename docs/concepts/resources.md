# Concepts — Resources & flow model

pgway configuration is a small set of **resources** wired into a pipeline. Traffic always enters at an **Entrypoint**, follows a **Flow**, optionally hits a **Router**, then a **LoadBalancer** that picks a member of a **Pool**, which resolves to one or more **Proxy** upstreams.

This page is the mental model. Per-resource YAML schemas and `pgctl` examples come later.

## Pipeline

```mermaid
flowchart LR
  EP[Entrypoint] --> FL[Flow]
  FL -.optional.-> RT[Router]
  FL --> LB[LoadBalancer]
  RT --> LB
  LB --> PL[Pool]
  PL --> PX[Proxy]
```

| Resource | Required? | Role |
|----------|-----------|------|
| **Entrypoint** | Yes | Listen address (`host:port`) and which flow to run |
| **Flow** | Yes | Binds the entrypoint to a balancer; optionally attaches a router |
| **Router** | No | Condition-based choice of which balancer to use |
| **LoadBalancer** | Yes | Strategy in front of exactly one pool |
| **Pool** | Yes* | Group of proxies (static members or dynamic labels) |
| **Proxy** | Yes* | One upstream (`http://…` or `socks5://…`) |

\*A pool without resolvable proxies cannot serve traffic. Proxies can also sit behind a balancer indirectly via the pool.

**Minimal valid path:** `Entrypoint → Flow → LoadBalancer → Pool → Proxy` (no router).

**Rules of thumb**

- The router does not know about pool internals — it only chooses a **balancer** (or equivalent target id).
- The load balancer does not invent membership — it selects among proxies the **pool** resolved.
- One process can run **multiple entrypoints** on different ports, each with its own flow.

## Proxy

A single upstream endpoint. Identity is `metadata.name`. Optional **labels** (e.g. `provider`, `region`) are how **dynamic pools** discover members.

Credentials can live in a URL shorthand (`http://user:pass@host:port` or `socks5://user:pass@host:port`) or as explicit fields. Upstream protocols: **HTTP** and **SOCKS5**. Clients still connect to pgway with the HTTP proxy protocol (CONNECT + plain HTTP).

## Pool

Two kinds:

| Type | How members are chosen | Notes |
|------|------------------------|--------|
| **static** | Explicit `members: [{proxy_id, weight?}]` | Weight defaults to `1`. Required for **weighted** balancers. |
| **dynamic** | Label `selector.allow` (all listed labels must match) | Membership changes as proxies are applied/updated. No per-member weights. |

Static and dynamic are mutually exclusive on one pool: static has members and no selector; dynamic has a selector and no members.

## LoadBalancer

Always references **one** `pool_id`. Strategy (`spec.type`):

| Type | Behavior | Pool constraint |
|------|----------|-----------------|
| `round-robin` | Rotate evenly across resolved proxies | Static or dynamic |
| `weighted` | Smooth weighted round-robin (nginx-style) using static member weights | **Static only** |
| `least-bytes` | Prefer the proxy with the fewest accumulated **downstream** bytes (response / tunnel→client), updated on connection `Release` | Static or dynamic |

For `least-bytes`, optional `reset_interval` (Go duration, default `1m`) clears counters lazily on the next selection after the interval elapses. In-flight byte estimates are not used yet.

## Router

Optional rule engine on the flow. Rules run **in order**; first match wins. Each rule points at a **target** balancer id.

Useful match types include `host`, `host_suffix`, `path_prefix`, `path_regex`, `method`, `header`, `catch_all`, plus composites `all` / `any` / `not`.

Without a router, the flow’s default `balancer_id` handles all traffic.

## Flow & Entrypoint

- **Flow** — `balancer_id` (default / required path) and optional `router_id`.
- **Entrypoint** — `protocol` (HTTP for MVP), `host`, `port`, `flow_id`.

Client → entrypoint → flow (router?) → load balancer → pool → proxy → target.

## Control Plane vs Data Plane (resources in motion)

Resources are written to the **Control Plane** (BadgerDB) via gRPC / `pgctl` (and experimental REST). The **Data Plane** loads them into memory and serves traffic. Config changes can hot-reload (in-process or via agent Watch in distributed mode) without rewriting this mental model — only *where* the config is applied changes.

## What’s next in these docs

- [Binaries & planes](binaries.md) — Control Plane vs Data Plane and the four binaries
- [Installation](../getting-started/installation.md) — build from source (no releases yet)
- [First run](../getting-started/first-run.md) — bootstrap and first apply
- [Resource reference](../reference/index.md) — full YAML fields per kind
