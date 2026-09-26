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
| REST Proxy CRUD (+ conflict status mapping) | Done (dashboard Proxies UI) |
| REST Pool CRUD (+ conflict status mapping) | Done (dashboard Pools UI) |
| REST resource CRUD + agents + users API | Planned ([#102](https://github.com/aknEvrnky/pgway/issues/102), [#103](https://github.com/aknEvrnky/pgway/issues/103)) — Proxy/Pool routes already exist |
| Dashboard login page / route guards | Done ([#22](https://github.com/aknEvrnky/pgway/issues/22)) |
| Dashboard Proxies list / create / edit / delete | Done (slice of [#105](https://github.com/aknEvrnky/pgway/issues/105)) |
| Dashboard Pools list / create / edit / delete | Done (slice of [#105](https://github.com/aknEvrnky/pgway/issues/105)) |
| Dashboard UI (other resources, YAML, Vue Flow, agents home, users admin) | Planned ([#105](https://github.com/aknEvrnky/pgway/issues/105)–[#109](https://github.com/aknEvrnky/pgway/issues/109)) |
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
4. Open **Proxies** (`/proxies`) to list, create (URL or manual fields), edit, and delete upstreams. The list supports search, protocol filter, and cursor pagination via REST (`search`, `protocol`, `page_size`, `page_token`). Name is required. Edit omits password to keep existing credentials. Delete fails with a clear conflict when a pool still references the proxy.
5. Open **Pools** (`/pools`) to list, create/edit static members (`proxy_id` + weight) or dynamic label selectors (`selector.allow`), and delete. The list supports search, type filter (static/dynamic), and cursor pagination via REST (`search`, `type`, `page_size`, `page_token`). Delete fails with a conflict when a load balancer still references the pool. Weighted balancers require a static pool.

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
