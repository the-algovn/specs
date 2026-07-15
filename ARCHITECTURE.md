# Platform architecture

The current shape of the algovn platform — what runs, where it lives, and the
conventions a new product is expected to follow. This describes what exists
today, not what is planned; [`ROADMAP.md`](ROADMAP.md) covers what is next.

## The shape

```
Browser
  │  TLS terminates at the Cloudflare edge; SPA assets are edge-cached
  ▼
Cloudflare edge ── tunnel (outbound only, no open ports) ──▶ cloudflared
                                                                 │
                                                                 ▼
                                                          Kong (single gateway)
                                                                 │
                    ┌────────────────────────────────────────────┼──────────────┐
                    ▼                                            ▼              ▼
             id.algovn.com                              api.algovn.com    algovn.com
                Zitadel                                api-control-plane    landing /
             (OIDC issuer)                                     │            showcase
                                                               │ gRPC h2c :9090
                                                               ▼
                                                       internal services
                                                    (pure gRPC, never public)
                                                               │
                        ┌──────────────────────┬───────────────┴──────┐
                        ▼                      ▼                      ▼
                   Postgres (CNPG)          Redis                 RabbitMQ
                                                                (topic exchange
                                                                   "events")
```

Everything runs on a k3s cluster managed entirely by GitOps. Nothing is applied
by hand: a merge to `main` in [`iac`](https://github.com/the-algovn/iac) is the
only way state reaches the cluster.

## Repositories

| Repo | Role |
| ---- | ---- |
| [`iac`](https://github.com/the-algovn/iac) | The cluster. Ansible (nodes) → bootstrap (Argo CD) → everything else via GitOps. Source of truth for all deployed state, including every product's manifests and API registrations. |
| [`api-control-plane`](https://github.com/the-algovn/api-control-plane) | The single public API host. JWT auth, JSON⇄gRPC transcoding, SSE fan-out. Every product's HTTP surface. |
| [`protos`](https://github.com/the-algovn/protos) | gRPC contracts, buf-managed. Generated Go published as a Go module. |
| [`web`](https://github.com/the-algovn/web) | Frontend monorepo: `@algovn/ui` design system, `landing`, `showcase`. Product SPAs live here as `apps/<product>`. |
| [`specs`](https://github.com/the-algovn/specs) | Product catalog and this document. No code. |
| `<product>-service` | One repo per product. Pure gRPC, cluster-internal. Currently: `the-button-service`. |

## Cluster

Two nodes, k3s, Argo CD app-of-apps with automated sync, self-heal and prune.
Drift is reverted; sync history is the audit log. Since 2026-07-15 both nodes are
VMs on a single Proxmox VE host (192.168.102.100) — no real HA; the split exists
for blast-radius isolation and node-rebuild practice. The host also runs home-lab
LXCs (OpenBao, AdGuard, NAS) outside the cluster.

- **`algovn`** — 4 vCPU / 8GB VM (.111). Runs the platform's control plane. The
  lightweight stack choices below (VictoriaMetrics over Prometheus, Argo CD slim,
  no tracing backend) date from the Pi-era 4GB budget and are kept on merit.
- **`algovn-w1`** — 8 vCPU / 16GB VM (.112). Runs Postgres, Redis, RabbitMQ,
  VictoriaMetrics, Loki, and the workloads that need headroom.

**Exposure** is Cloudflare Tunnel only — no open ports, home IP hidden, works
behind CGNAT. `external-dns` watches Ingresses and manages the CNAMEs, so
publishing a service is declarative. `cert-manager` issues a wildcard
`*.algovn.com` via DNS-01 so LAN/fallback access keeps real TLS even when the
tunnel is down.

**Secrets** live in OpenBao (Vault-compatible, in an LXC on the Proxmox host —
outside the cluster, so the root of trust survives cluster rebuilds) and sync in
via External Secrets Operator; the public repo holds only ExternalSecret
references. A rebuild needs exactly one manual secret step: creating ESO's
AppRole bootstrap Secret from the host's keyfiles (`iac/docs/runbooks/secrets.md`).

> **There are no backups.** Persistent volume data — identities, click counters,
> Loki history — is lost if `algovn-w1`'s disk dies. This is an accepted,
> deliberate risk, not an oversight. The barman-cloud plugin is deployed and the
> CNPG backup path is a known, executable change when the risk stops being
> acceptable.

**Observability**: VictoriaMetrics + vmagent + vmalert + Alertmanager, Grafana
dashboards provisioned from git, Loki + Alloy for logs. Alerts go to Telegram.
There is no tracing backend — it does not fit the RAM budget.

## Gateway — Kong

One gateway for every public hostname. Kong does TLS, routing, and rate limiting;
it does **not** validate product API tokens (see below). Plugins bind by
`konghq.com/plugins` annotation on the Ingress they guard, so routing and
protection live in the same file. Rate limiting is per-client-IP, resolved from
`CF-Connecting-IP` with `trusted_ips` scoped to the pod CIDR and a NetworkPolicy
admitting only cloudflared to the proxy port — without that, every request buckets
as the cloudflared pod IP and the limit becomes a global cap.

`api.algovn.com` is split into three ingresses with different rate-limit tiers:
`/events` (SSE, sized for NAT cohorts), `/the-button` (clicks), and everything
else on a tighter default.

## Identity — Zitadel

The only OIDC issuer, at `id.algovn.com`. One shared user pool: sign up once, SSO
into every product. Passwordless — Google, GitHub, and passkeys; no password
authenticator is enabled anywhere.

Each product is a Zitadel *project*; each client (SPA, CLI, machine) an
*application*. SPAs use authorization-code + PKCE, no client secrets in browsers.
Customer teams map to Zitadel *organizations* with project grants — the native B2B
model, no custom schema.

**OpenFGA** is deployed for relationship-based, per-resource permissions, and is
cluster-internal only. The boundary is load-bearing: *Zitadel owns who you are*
(users, orgs, org-level roles — delivered in the token); *OpenFGA owns what you
can touch* (per-resource relations, written and queried by the owning app). Roles
stay coarse and live in the token; anything per-resource belongs in OpenFGA.

Contract details apps code against: `iac/docs/authnz-conventions.md`.

## The public API — api-control-plane

One host, `api.algovn.com`, one path prefix per product. A product does not get
its own hostname, its own TLS, or its own auth code. The control plane centralizes
what every product would otherwise rebuild:

- **Registration** — declarative YAML per product, in
  `iac/apps/api-control-plane/registrations/`, ConfigMap-mounted with hot reload.
  PR-reviewed like any other GitOps change. There is no runtime registration API.
- **AuthN/Z** — Bearer JWT verified here (JWKS from Zitadel, cached), *not* by
  Kong's jwt plugin: one host mixes anonymous and protected routes, and auth
  config belongs next to auth enforcement. Per-route rules are `anonymous`,
  `authenticated`, or `role:<r>`.
- **Transcoding** — each gRPC method is exposed at an explicit HTTP verb + path
  declared in the registration (e.g. `GET /the-button/counter`), with a JSON body
  mapped to a unary gRPC call via `dynamicpb`. Method descriptors come from gRPC
  server reflection on the upstream; the registration's `routes` list is the
  authoritative allowlist of exposed endpoints. Internal services stay pure gRPC
  and never learn about HTTP.
- **Realtime** — one shared SSE fan-out. A service publishes JSON to the RabbitMQ
  topic exchange `events` with a routing key of `<product>.<topic>`; browsers
  subscribe at `GET /events/<channel>`. No replay or history: channels carry
  snapshots or fire-and-forget events, and a reconnecting client gets the next
  message.
- **CORS** — handled centrally; product SPAs live on other origins.

Verified `Authorization` headers are forwarded upstream as gRPC metadata on every
rule. Services do a read-only decode of the claims and **never re-verify** — the
gateway is the sole verified ingress.

Registration schema and channel naming: `iac/docs/api-conventions.md`.

## Internal services — gRPC conventions

Never public. Port **9090** named `grpc`, plaintext h2c in-cluster, headless
Service with the grpc-go DNS resolver and `round_robin`. Implement
`grpc_health_v1` and use native gRPC probes. Every call carries a deadline;
retries only on idempotent methods. Prometheus metrics on **9091** named `metrics`,
scraped by a `VMServiceScrape`.

Contracts live in `protos`, buf-managed, with `buf lint` + `buf breaking` gating
CI. Generated Go is committed and consumed as a Go module. A released `vN` package
is never mutated.

Full conventions: `iac/docs/grpc-conventions.md`. Manifest skeleton:
`iac/templates/grpc-service/`.

## Data

- **Postgres** — one shared CNPG cluster on `algovn-w1`. Each product gets its own
  database and owner role, declaratively onboarded, credentials in OpenBao. Currently
  hosts `zitadel` and `openfga`.
- **Redis** — single instance, AOF `everysec` + RDB. Hot control state only:
  counters, throttles, short-lived tokens. Written cluster-mode-ready by
  construction (single-key ops only), so it can shard later without an app rewrite.
  **`maxmemory-policy` must be `noeviction`**: products such as
  `the-button-service` depend on specific keys (e.g. `counter:global`,
  `applied:*` idempotency markers) never being evicted under memory pressure —
  an eviction there silently corrupts exactly-once accounting the same way a
  data loss would.
- **RabbitMQ** — single node, topic exchange `events`. The transport behind SSE.
  The request path never depends on it: if RabbitMQ is down, SSE fails and clients
  fall back to polling, but reads and writes keep working.

## Web

pnpm monorepo, Tailwind v4, shared `@algovn/ui` design system.

- **`apps/landing`** — Next.js, `algovn.com`. Terminal-themed portfolio, with a
  hidden interactive easter egg.
- **`apps/showcase`** — the design-system showcase.
- **Product SPAs** — React + Vite, served under a path (`base: '/<product>/'`),
  built into an nginx static image with immutable hashed assets. Auth via
  `oidc-client-ts` against Zitadel with PKCE; the access token is held in memory
  only.

A Cloudflare cache rule on a product's `assets/*` is **mandatory before launch**,
not an optimization: at the platform's 10k-concurrent target, uncached bundles
exceed the home uplink on their own.

## Conventions a new product inherits

Everything below is already built. A new product does not redesign any of it:

1. Contract in `protos`, `buf breaking`-gated.
2. A pure-gRPC service repo, cluster-internal, following the 9090/9091 conventions.
3. A registration YAML in `iac` — that is the entire public API surface.
4. Login for free via Zitadel; permissions in OpenFGA if they are per-resource.
5. Realtime for free by publishing to `events`.
6. An SPA in `web/apps/<product>`, served under its path.
7. Its manifests in `iac/apps/<product>/` plus an Argo Application, both merged to
   `main`. Nothing is deployed any other way.

## Products

**The Button** — see [`products/the-button.md`](products/the-button.md).

Backend complete and platform prerequisites live: the `algovn.button.v1` contract
is published, `the-button-service` implements proof-of-work gating, click batching,
achievements and the tick leader, Redis is deployed, and the gateway's per-route
rate limits and error mappings are in place.

Not yet shipped: the service is **not deployed** (no `iac/apps/the-button/`, no
Argo Application, no `the_button` database), it is **not registered** with the
control plane, and the **SPA does not exist**. `api.algovn.com/the-button` routes
to the control plane, which has no registration for that prefix. See
[`ROADMAP.md`](ROADMAP.md).

---

*Design records from the build phase are archived outside these repos. This
document is the living description; when it disagrees with an old plan, this
document is right.*
