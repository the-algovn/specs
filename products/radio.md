# Tần Số 42 (radio)

An AI-powered Vietnamese radio station, broadcasting 24/7. Request a song,
dedicate it to someone, and hear the AI DJ read your message on air — the
old-school đêm-khuya dedication show, rebuilt as software.

Anyone can listen at algovn.com/radio without an account. Logging in (Zitadel
SSO) unlocks the interactive half: requesting songs and sending on-air
messages. The host is **Tiểu Dương Dương** — a warm, soft-spoken **chatty
companion**: she talks every song or two, tells small stories into the night,
and the music is her backdrop, not the other way around.

**Why:** the dedication-show format is the warmest thing radio ever did ·
live audio is a new platform capability worth proving · an always-on AI
product with a personality is a better showcase than another CRUD app.

**Personal-use music, honestly stated:** tracks are resolved and downloaded
from YouTube via the owner's own account (yt-dlp), cached in a local library.
This is a personal project streaming to a personal audience; the owner accepts
the takedown risk that comes with a public stream of sourced music. No
licensing story is claimed.

This file is the product spec. The subsystem designs live next to it:

- [`radio/lab.md`](radio/lab.md) — Phase 0: the shared admin console and
  component lab that de-risk voice, scripts, ingest and rendering before
  the broadcast is built.
- [`radio/architecture.md`](radio/architecture.md) — the detailed system
  architecture: modules, flows, contract, schema, deployment inventory,
  config surface, capacity.
- [`radio/audio-path.md`](radio/audio-path.md) — rendering, HLS publishing,
  presence, the format clock.
- [`radio/ingest.md`](radio/ingest.md) — how music gets in: search, ranking,
  playlist import, yt-dlp operations, failure classes.
- [`radio/dj-brain.md`](radio/dj-brain.md) — how the DJ thinks and speaks:
  briefs, persona bible, script generation, TTS, fallbacks.

## Features

- Live 24/7 audio stream (HLS), public, no login to listen.
- **The call-in box**: one free-text field, like a real phone-in —
  "Cho mình xin bài Em Của Ngày Hôm Qua, gửi tặng Ngọc — chúc ngủ ngon nha".
  An LLM parses it, YouTube search resolves the track, you confirm the match
  (parsed fields editable) before it queues.
- On-air messages: dedications attached to a request, or standalone
  shoutouts. Recipients are free-text names — whoever isn't listening at that
  moment misses it, and that ephemerality is the point. The DJ **retells**
  your message; raw text never airs verbatim. She judges each message's
  weight: a casual "chào cả nhà" gets a warm line, a birthday confession
  gets the full đêm-khuya treatment.
- Now-playing, queue peek ("sắp phát") and history in the SPA — synced to
  what you *hear*, not to the server clock. The queue never reveals a
  dedication's recipient before it airs — no spoiled surprises.
- The chatty format clock: a talk slot every 1–2 songs (intros, backsells,
  musings, dedications), the hourly station ID, weather color for Hà Nội and
  TP.HCM.
- Cost-aware broadcasting: with zero listeners the station plays music-only
  and spends nothing on LLM/TTS; Tiểu Dương Dương "comes back" (jingle +
  proper greeting) when someone tunes in. She never greets individual joins.
- Station seeding: admin imports the owner's **YouTube playlists**; the
  library keeps growing from listener requests.

**Scale target:** modest and honest — tens of concurrent listeners, designed
so public spikes are absorbed by Cloudflare's cache rather than the uplink.

## Architecture

One new repo, `radio-service` (Go): **one container image, two
entrypoints**, sharing one codebase, the `radio` database in the shared
CNPG cluster, and Redis (full detail:
[`radio/architecture.md`](radio/architecture.md)):

- **`radio-api`** — a standard platform citizen: pure gRPC :9090
  (`algovn.radio.v1` in `protos`), metrics :9091. Call-in parsing +
  moderation (one LLM call), track resolution previews, request/shoutout
  intake, queue and history reads, admin ops (`role:admin` — coarse, per
  platform authnz conventions).
- **`radio-studio`** — the broadcast engine. Single-replica Deployment on
  `algovn-w1`: program-director loop, DJ brain (LLM), TTS, ffmpeg rendering,
  HLS publishing, and the ingest workers (it owns the media PVC). An nginx
  sidecar serves the HLS output from a shared volume.

```
listeners ─▶ Cloudflare (segments edge-cached) ─▶ Kong
                                                   ├─ algovn.com/radio/        → radio SPA (static)
                                                   ├─ algovn.com/radio/stream/ → studio nginx (live.m3u8 + segments)
                                                   └─ api.algovn.com/radio/*   → api-control-plane ─gRPC→ radio-api
radio-studio ─renders─▶ HLS volume (◀─ nginx)      api.algovn.com/events/<channel> (SSE)
     │  └─now-playing─▶ RabbitMQ "events" ────────────┘
     └─jobs/state─▶ Postgres `radio` + Redis ◀── radio-api
```

**Delivery is the load-bearing trick.** HLS segments get unique names, so a
Cloudflare cache rule caches them `immutable`: N listeners cost roughly one
origin fetch per 4-second segment; only the tiny cache-bypassed manifest poll
rides the tunnel. Details in [`radio/audio-path.md`](radio/audio-path.md).

**Coordination** between api and studio is Postgres (queue + job tables —
durable, no new infra) plus Redis for hot state. Now-playing and queue
changes publish to the `events` exchange (`radio.nowplaying`, `radio.queue`)
→ the platform's existing SSE. There is deliberately **no per-user SSE
channel**: the my-requests view updates from the public queue channel plus
polling, keeping SSE anonymous-safe. LLM/TTS/yt-dlp egress uses the home
residential IP; optional account cookies come from OpenBao if gated videos
ever need them.

## Data

- **Postgres `radio`:** `tracks` (yt_id unique, canonical title/artist,
  duration, file path, loudnorm params, mood/energy/era, status,
  last_aired_at), `requests` (user, track, raw_text, parsed fields,
  recipient, message, sanitized_digest, weight, status: pending/approved/
  ready/queued/aired/failed/rejected), `shoutouts`, `play_history` (kind:
  track/dj/jingle), `dj_segments` (brief, script, summary, model, tokens,
  tts_chars, cost_usd — the bill's audit log), `ingest_jobs` (kind, payload,
  attempts, locked_by).
- **Redis:** `radio:presence` (ZSET scored by last heartbeat),
  `radio:nowplaying` snapshot, `radio:quota:<user>:<day>`,
  `radio:spend:<date>` (the budget counter). Presence and quotas tolerate
  loss; nothing here is a source of truth (`noeviction` is already platform
  policy).
- **Media PVC:** local-path on `algovn-w1`, 20 GB to start (~8 MB/track ≈
  2,500 tracks). Layout and eviction policy in
  [`radio/ingest.md`](radio/ingest.md).

## API surface (registration sketch)

| Route | Rule | Purpose |
| ----- | ---- | ------- |
| `GET /radio/now-playing`, `/radio/queue`, `/radio/history` | anonymous | reads |
| `POST /radio/heartbeat` | anonymous | presence (session UUID) |
| `POST /radio/resolve` | authenticated | parse call-in text → parsed fields + top-5 candidates |
| `POST /radio/requests` | authenticated | confirm a candidate (+ dedication) → queue |
| `GET /radio/requests/mine` | authenticated | my requests + statuses |
| `POST /radio/shoutouts` | authenticated | standalone on-air message |
| `POST /radio/admin/import` | role:admin | YouTube playlist URL → seed ingest |
| `POST /radio/admin/skip`, `POST /radio/admin/requests/remove` | role:admin | on-air control (ids in body — registration paths allow no parameters) |

The registration also declares the SSE channels — `radio.nowplaying` and
`radio.queue`, both `anonymous` (native EventSource cannot send an
Authorization header; the SPA opens one EventSource per channel). Exact
verb/path/field mapping is fixed at implementation time against
`iac/docs/api-conventions.md`.

## Queue policy

Round-robin across requesters, FIFO within one user. Quotas: 3 pending and
10 per day per user (days — like every station clock — are
Asia/Ho_Chi_Minh). Duplicates handled warmly: if the track is already queued,
a second dedication **merges** onto the same play; if it aired < 2 h ago,
friendly reject — "vừa phát xong, để khuya nhé". Requesting while the
station is in music-only mode just works: the requester's own presence
heartbeat wakes the DJ.

**Request lifecycle:** submit → parse+moderation → `approved` → ingest (if
not cached) → `ready` → queued → rendered → `aired`; failures become
`failed` with the user notified — the DJ never announces a failure.
Shoutouts ride the same parse+moderation call (they simply lack a song
query) and the same states minus the ingest steps: pending → approved →
aired (or rejected).

## Abuse & safety

- Requests and shoutouts require login, pass the moderation verdict, and air
  only as the DJ's paraphrase of a sanitized digest.
- Quotas + round-robin stop one person monopolizing the station.
- The anonymous heartbeat grants nothing but a presence bit, is Kong
  rate-limited per-IP, and its worst-case abuse (fake listeners keeping the
  DJ awake) is bounded by the budget cap, not the wallet.
- **Budget cap:** a Redis daily spend counter (TTS chars + LLM tokens priced
  at write time, audited in `dj_segments`) enforces a configurable hard
  ceiling, default **$1/day**. At the cap: music-only mode + canned
  segments, Telegram alert, resets at midnight Asia/Ho_Chi_Minh.
- Prompt injection is a real input class; the rails live in
  [`radio/dj-brain.md`](radio/dj-brain.md) and
  [`radio/ingest.md`](radio/ingest.md).

## Failure modes

The station never dies; it gets quieter. LLM down → canned script templates.
TTS down → pre-rendered canned audio, then music-only with sweepers. yt-dlp
broken (YouTube moved) → new ingests stall, rotation unaffected. RabbitMQ
down → SSE metadata stops, audio keeps playing (HLS is the source of truth).
Studio crash → k8s restart, resume publishing with a discontinuity tag,
~15 s gap, hls.js recovers. Budget cap → graceful music-only, never dead
air. Alerting via the existing Alertmanager → Telegram path; the "manifest
staleness" alert is the stream-frozen canary.

## Costs

Target $10–30/month, hard-capped at the configured daily ceiling. TTS
dominates. Honest chatty-mode math: ~10–12 talk slots/hour listened ≈ 6.5k
chars/hour → at ~4 h of listened airtime daily ≈ 780k chars/month —
**~$23/mo on Chirp3-HD, ~$12.50 on Neural2** (verify current pricing at
build time). The $1/day default cap sustains ~5 h of chatty listening daily.
Script + parse LLM calls are pennies; yt-dlp and Open-Meteo are free.
`dj_segments` records the actuals; the dashboard shows the burn.

## The SPA

A retro receiver: warm dark palette, glowing frequency dial, an ON AIR lamp
lit by the SSE heartbeat, tactile buttons. One screen — player, now-playing
card (thumbnail + dedication line, synced to the audible track), "sắp phát"
queue peek, history, my-requests with live status, and the call-in modal
with its confirm step. Login gates interaction only. React + Vite at
`web/apps/radio` under `algovn.com/radio/`, hls.js player (Safari plays HLS
natively), final aesthetics via the frontend-design pass at build time.

## Observability

Gauges: `listener_count`, `buffer_ahead_seconds`, `daily_spend_usd`.
Counters: TTS chars, LLM tokens, request-state transitions, render and
ingest failures by class. Alerts: manifest staleness, render-buffer
underrun, spend ≥ 80% of cap, ingest-failure spikes (the "YouTube changed
something" alarm). One provisioned Grafana dashboard: the station console.

## Testing

- Unit: program-director scheduling (slot cadence, fairness, no-repeat,
  wake/sleep) against a fake clock, quotas, dedupe/merge, budget-cap
  arithmetic, candidate-ranking heuristics, brief/output schema validation
  with recorded LLM fixtures.
- Audio golden tests: render an item → assert duration and integrated LUFS
  within tolerance (ffmpeg measures; no human ear in CI).
- Integration: testcontainers Postgres + Redis.
- Dev loop: Tilt with fake-LLM/fake-TTS/fake-yt-dlp modes so request→on-air
  runs locally for $0. The ear test — does it *feel* like radio — stays
  manual.

## Phases

0. **The lab** — the platform console (`web/apps/console`) + `radio-lab`
   bench: audition voices, tune the persona, validate yt-dlp and the
   render recipe, record fixtures — local-only, real auth, real protos
   (`algovn.radiolab.v1`). Exits with the voice, model, persona and render
   defaults chosen. See [`radio/lab.md`](radio/lab.md).
1. **MVP on air** — YouTube playlist-import seeding, library auto-rotation
   (no-repeat shuffle), public HLS stream + Cloudflare cache rule, SPA
   player with ear-synced now-playing, call-in requests end to end (parse +
   moderation verdict enforced from day one), chatty clock with core
   segment types (intro, backsell, station ID).
2. **The heart** — dedications and shoutouts retold on air with
   weight-judged treatment, musings and wake greeting, cost-aware presence,
   queue fairness, quotas, dedication merging, weather color.
3. **Polish** — scheduled dedications ("phát lúc 21:00 ngày sinh nhật"),
   render-time crossfades, jingle package, daypart-aware music selection
   and transitions, load-validate the cache path.

**Success criteria:** 7 days on air unattended; request→on-air median under
~10 minutes; a friend sends a dedication and hears it read back; monthly AI
spend inside $10–30 with the cap never hit by surprise; zero manual ops
after deploy.

## Not building (deliberately)

Private user-to-user inbox (messages are on-air dedications only),
registered-recipient tagging or notifications (recipients are free-text
names), multiple channels, vote-to-skip, podcast archives of past shows,
mobile apps, Spotify integration (seeding and requests are YouTube-only;
Spotify import would only add a matching problem we don't have), per-user
SSE channels.

**Status:** spec.

**Platform it rides on:** api.algovn.com gateway (api-control-plane), Zitadel
login, RabbitMQ→SSE push, shared Postgres, Redis, GitOps deploy via `iac` —
all described in [`../ARCHITECTURE.md`](../ARCHITECTURE.md).
