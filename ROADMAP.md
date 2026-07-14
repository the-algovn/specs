# Roadmap

Build order only — no dates. Products are referenced by the name used in
the README portfolio table.

## Now

**The Button — ship it.** The backend is done and the platform is ready; nothing
runs yet. Remaining, in dependency order:

1. `the_button` database in the shared CNPG cluster (sealed credentials).
2. Deploy the service — `iac/apps/the-button/` (namespace, Deployment, headless
   Service, VMServiceScrape, sealed PoW secret) plus its Argo Application.
3. Register it — `the-button.yaml` in
   `iac/apps/api-control-plane/registrations/`. Until this lands,
   `api.algovn.com/the-button` has no route.
4. Build the SPA — `web/apps/the-button` (Vite, PKCE login, WASM proof-of-work
   solver in a Web Worker, SSE counter with polling fallback).
5. Cloudflare cache rule on `/the-button/assets/*` — a launch blocker, not an
   optimization: uncached bundles at the concurrency target exceed the uplink.
6. Pre-launch load evidence — calibrate the proof-of-work factor against the real
   solver on real devices, then k6 for the SSE ramp and the submit soak.

## Next

## Later
