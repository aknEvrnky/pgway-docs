# Dashboard

The web dashboard is **experimental** and **not production-ready**.

It is a Nuxt 4 + Vue 3 + PrimeVue + Vue Flow UI that talks to the Control Plane over **REST**. Expect incomplete flows, breaking API changes, and rough edges.

![Dashboard demo — Kinetic Console overview](../assets/dashboard-demo.jpg)

*Demo / concept mock of the dashboard overview (not a guarantee of shipped UI).*

!!! warning
    Prefer **`pgctl` + gRPC** for real configuration and operations until the dashboard matures.

    REST requires a **user** bearer token. Default listen is **loopback** (`127.0.0.1:8081`). Do not bind non-loopback without a trusted network / reverse proxy; configure `rest.cors_allow_origins` for browser UIs.

## Status

Work is tracked on the [Dashboard](https://github.com/aknEvrnky/pgway/milestones) milestone. Epic: [#21](https://github.com/aknEvrnky/pgway/issues/21).

| Area | State |
|------|--------|
| Core gateway (CP/DP, CLI, HTTP proxy, balancers) | Usable / tested |
| REST auth / secure defaults / rate limit | Done ([#86](https://github.com/aknEvrnky/pgway/issues/86)); coverage ongoing ([#87](https://github.com/aknEvrnky/pgway/issues/87)) |
| REST login / logout / me + cookie session | Done (dashboard [#22](https://github.com/aknEvrnky/pgway/issues/22) backend) |
| REST Proxy / Pool / LoadBalancer / Router / Flow / Entrypoint CRUD | Done (dashboard resource pages + Flow editor) |
| REST agents + users admin API | Planned ([#102](https://github.com/aknEvrnky/pgway/issues/102), [#103](https://github.com/aknEvrnky/pgway/issues/103)) |
| Dashboard login page / route guards | Done ([#22](https://github.com/aknEvrnky/pgway/issues/22)) |
| Dashboard Proxies / Pools / LB / Routers / Flows / Entrypoints CRUD | Done (slice of [#105](https://github.com/aknEvrnky/pgway/issues/105)) |
| Flow visual editor (Vue Flow) + read-only YAML projection | Done (experimental) |
| Agents page / Settings / notifications | Stub / hidden — CRUD later |
| Overview charts / live logs / health panels | Later ([#20](https://github.com/aknEvrnky/pgway/issues/20), [#14](https://github.com/aknEvrnky/pgway/issues/14), [#9](https://github.com/aknEvrnky/pgway/issues/9)) |

Milestone: [Dashboard](https://github.com/aknEvrnky/pgway/milestones) on GitHub.

## Local UI development (optional)

Only if you are hacking on the frontend. Requires Bun or Node 20+.

```bash
cd frontend
bun install   # or: npm install
bun run dev   # http://localhost:3000
```

The all-in-one / CP process must be running with `rest.enabled = true` (default) and `rest.listen_addr` reachable (default `127.0.0.1:8081`).

1. Add `http://localhost:3000` to `rest.cors_allow_origins` (credentials-enabled CORS).
2. Point the UI at `http://localhost:8081` via `NUXT_PUBLIC_API_BASE` (default) — use **`localhost`**, not `127.0.0.1`, so the httpOnly session cookie stays same-site.
3. Open `/login`, sign in with a CP user (after `pgctl init` / `pgctl login` credentials).
4. Resource pages (same REST filters: `search`, type/protocol chips, cursor pagination via `page_size` / `page_token`):
   - **Proxies** (`/proxies`) — create (URL or manual), edit, delete (blocked while a pool references the proxy).
   - **Pools** (`/pools`) — static members or dynamic `selector.allow`; delete blocked while a load balancer references the pool. Weighted balancers require a static pool.
   - **Load Balancers** (`/load-balancers`) — round-robin, weighted, least-bytes.
   - **Routers** (`/routers`) — ordered match rules → balancer targets.
   - **Flows** (`/flows`) — list + **visual editor** (`/flows/{name}`) for the pipeline graph.
   - **Entrypoints** (`/entrypoints`) — listen host/port bound to a Flow. Detach in the editor deletes the entrypoint (stops the socket); the Flow remains.
5. **Agents** (`/agents`) is a placeholder until agent management CRUD ships — use `pgctl` / gRPC for agents today. **Settings** and header notifications are hidden until there is something to configure.

### Flow visual editor

- Canvas shows Entrypoint → Flow → (optional Router) → LoadBalancer → Pool → (static) Proxy.
- Palette: attach existing resources or create new ones; create router/balancer from the editor binds `flow.router_id` / `flow.balancer_id` (Deploy to persist the Flow).
- Sidebar: Node Settings (including unbind router/balancer, detach entrypoint) and a **read-only** multi-doc YAML projection (`kind` / `version` / `metadata` / `spec`, apply-order sorted) compatible with `pgctl apply`. YAML is generated client-side today — not a server export API.
- Dirty tracking + **Deploy Changes** applies staged resources in Proxy → … → Entrypoint order.
- Entrypoints already bound to another Flow cannot be stolen via the attach dialog (UI-only; Control Plane Apply still allows rebind).

See [Authentication](authentication.md) for REST login/logout/me and [Configuration](../getting-started/configuration.md).

## What to use instead

| Task | Tool |
|------|------|
| Apply / inspect resources | [CLI (`pgctl`)](cli.md) |
| First bootstrap | [First run](../getting-started/first-run.md) |
| Auth & agents | [Authentication](authentication.md) |

## Related

- [Welcome](../index.md) — project-wide experimental notice
- [Binaries & planes](../concepts/binaries.md)
- [Entrypoint](../reference/entrypoint.md) · [Flow](../reference/flow.md) · [Load balancing](load-balancing.md)
