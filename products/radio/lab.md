# Tần Số 42 — Phase 0: the component lab

The admin console and test bench that de-risks the radio before Phase 1
builds the broadcast. Subsystem design under [`../radio.md`](../radio.md).

The station's scariest unknowns are not architectural, they are
*qualitative*: does a premium Vietnamese TTS voice actually sound like a
đêm-khuya host? Do Flash-class scripts feel like radio or like a chatbot?
Does yt-dlp behave on this network? What duck/fade values make a voice-over
sound broadcast-real? Phase 0 answers these with ears and eyes, cheaply,
and leaves behind three durable assets: the platform's shared admin
console, the radio's real component packages, and a recorded set of
decisions and fixtures that Phase 1 consumes.

Phase 0 runs **local-only** (the dev Tilt environment). Nothing deploys;
nothing touches `iac`.

## Deliverable 1 — the platform console shell

**`web/apps/console`** — a new React + Vite app in the web monorepo, built
on `@algovn/ui`, run under the dev Tiltfile beside the other apps (port
`:5174`, pointed at the **local** gateway via `VITE_*` env like the other
SPAs — never bare `pnpm dev`). Not deployed in Phase 0, but written
deploy-ready (static nginx image pattern, `base: '/console/'`) so
promotion later is a manifest, not a rewrite.

The shell is platform infrastructure; benches are plug-ins:

- **Module descriptor** `{id, title, icon, group, routes, requiredRole}` +
  a static registry the sidebar renders from. Radio's benches are the
  first module group; other products add theirs without touching the
  shell.
- Shell owns: navigation, OIDC session, an environment badge (LOCAL, loud),
  error/toast surfaces.
- **Shared bench primitives** every module reuses: `RunPanel` (form →
  invoke → result pane with JSON / audio-player / cost tabs) and the
  **Ledger drawer** (session spend from the lab backend).

## Auth — a real OIDC SPA app from day one

Phase 0 deliberately exercises the production auth path. The Button
already proved the SPA sign-in pattern (project `the-thing`,
`oidc-client-ts`, in-memory tokens); the console copies that pattern and
adds its own Zitadel application:

- Zitadel project **`console`**, coarse role **`admin`**, granted to the
  owner.
- Application: Web + PKCE, redirect `http://localhost:5174`, with both
  documented gotchas applied — **Auth Token Type: JWT** (else the gateway
  401s) and **`accessTokenRoleAssertion`** (else no roles claim).
- The console signs in via `oidc-client-ts` (pattern copied from
  The Button's `lib/auth.ts`), token held in memory only.
- Every lab route registers `rule: role:admin`; the local gateway verifies
  against real Zitadel JWKS — and its `AUDIENCE` allowlist gains the
  console project + client ids. Logging into your own local lab with your
  real account IS the test.

## Deliverable 2 — the lab backend

**`radio-lab` is the first binary of the new `radio-service` repo**: pure
gRPC on `:9290` (reflection, health, the standard shape), implementing
**`algovn.radiolab.v1.LabService`**. A separate proto package on purpose —
the lab surface is tooling; `algovn.radio.v1` (Phase 1's forever contract)
never carries bench methods, and `radiolab.v1` can die without a breaking
change. Local proto development rides `go.work` (no release tagging needed
until something deploys).

Methods are thin wrappers over the real internal packages — `voice/`,
`brain/`, `callin/`, `ingest/`, `render/`, `spend/` are born in Phase 0
and consumed verbatim by Phase 1:

| Method | Wraps |
| ------ | ----- |
| `SynthesizeVoice` | voice: text + voice id/params → take |
| `GenerateScript` | brain: brief in → script + validation + summary |
| `ParseCallIn` | callin: raw text → parsed, verdict, digest, weight |
| `SearchTracks` | ingest: query → ranked candidates (scores visible) |
| `DownloadTrack` | ingest: candidate → download + ffprobe + loudnorm |
| `RenderPreview` | render: track + take + duck/fade knobs → preview |
| `GetLedger` | spend: session cost lines |
| `SaveFixture` | records a bench case into the Phase-1 test corpus |

**Binary artifacts bypass the gateway.** Unary JSON transcoding caps
bodies at 1 MiB, so audio never travels through it: responses carry an
`artifact_id`; a tiny HTTP file endpoint on `:9291` serves the bytes
(Vite-proxied, dev-only). Same split as production will use for the
stream: control data through the gateway, bytes outside it.

Storage: gitignored `lab-data/` (takes, downloads, renders) + an
append-only `ledger.jsonl`. Personas live in the repo (`persona/*.md`) —
they are product artifacts, committed. Structured state lives in
Postgres — the `ledger_line` table plus, as of the tracks-library
bench, a `track` table. Still no Redis, no RabbitMQ.

## Registration — a deliberate dev-only exception

`dev/registrations/radio-lab.yaml`: `prefix: /radio-lab`,
`upstream: 127.0.0.1:9290`, every route `role:admin`. This registration
**never gets an `iac` twin** — the lab does not deploy. That is a stated
exception to the update-both-registrations convention, recorded in the
file's header comment.

## The bench modules (built in this order, each a shippable slice)

1. **Voice audition** — one text, N Vietnamese voices side by side
   (Chirp3-HD variants, Neural2, optionally FPT), char count + cost per
   take, save named takes. *Decides: the station voice.*
2. **Brain playground** — persona editor (loads/saves `persona/*.md`),
   brief builder (segment type, clock, tracks, dedications + weights,
   memory), model picker, generate → script + digit-lint/length validation
   — and a **"Speak it"** button chaining into TTS. The core
   brief→script→voice→ears loop. *Decides: LLM provider/model, persona v0.*
3. **Call-in parse** — raw call-in text → parsed fields, verdict, weight;
   one-click save-as-fixture. *Produces: Phase 1's moderation test corpus.*
4. **Ingest** — search → ranked candidate table, download one → ffprobe +
   loudness readout. *Validates: yt-dlp + ranking on this network.*
5. **Mini-render** — a voice take ducked over a downloaded track's intro,
   duck-depth/offset/fade knobs tuned by ear. *Decides: Phase 1's render
   defaults.*

## Exit criteria

Phase 0 is done when: the TTS voice is chosen · the LLM provider/model is
chosen · persona v0 is committed and passes the owner's ear test · yt-dlp
is validated on ≥10 searches + 3 downloads with sane ranking · one saved
render of "she intros a real song" genuinely sounds like radio · ≥10 parse
fixtures are recorded · ledger totals match the provider dashboards. The
decisions are written back into [`dj-brain.md`](dj-brain.md) and
[`architecture.md`](architecture.md) as the new defaults.

## Non-goals

No HLS, no director/scheduler, no persistence beyond files, no deployment,
no SSE, no `algovn.radio.v1` product contract yet, and no UI polish beyond
the design system's defaults. It is a lab bench, not the product.

## Costs and testing

Bench spend is pennies (a TTS audition ≈ $0.01; a script ≈ $0.0002) and
every call lands in the ledger — which itself validates the price-at-write
arithmetic Phase 1's budget cap depends on. The internal packages carry
unit tests from day 0 (fake providers); recorded fixtures become Phase 1's
regression corpus; the shell gets a module-registry type test and a smoke
test, no more.

## What Phase 0 hands to Phase 1

The `voice/ brain/ callin/ ingest/ render/ spend/` packages, tested and
tuned · persona v0 · the parse fixture corpus · render defaults · the
proven OIDC SPA pattern · and a permanent platform console whose next
modules (radio's real admin: playlist import, skip, queue ops) arrive with
Phase 1.

**Status:** spec.
