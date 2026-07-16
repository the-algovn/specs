# Tần Số 42 (radio)

An AI-powered Vietnamese radio station, broadcasting 24/7. Request a song,
dedicate it to someone, and hear the AI DJ read your message on air — the
old-school đêm-khuya dedication show, rebuilt as software.

Anyone can listen at algovn.com/radio without an account. Logging in (Zitadel
SSO) unlocks the interactive half: requesting songs and sending on-air
messages. The host is **Tiểu Dương Dương** — a warm, soft-spoken late-night
persona (an LLM writing her scripts, a Vietnamese neural TTS voice reading
them), brighter in the morning, intimate after midnight.

**Why:** the dedication-show format is the warmest thing radio ever did ·
live audio is a new platform capability worth proving · an always-on AI
product with a personality is a better showcase than another CRUD app.

**Personal-use music, honestly stated:** tracks are resolved and downloaded
from YouTube via the owner's own account (yt-dlp), cached in a local library.
This is a personal project streaming to a personal audience; the owner accepts
the takedown risk that comes with a public stream of sourced music. No
licensing story is claimed.

## Features

- Live 24/7 audio stream (HLS), public, no login to listen.
- **The call-in box**: one free-text field, like a real phone-in —
  "Cho mình xin bài Em Của Ngày Hôm Qua, gửi tặng Ngọc — chúc ngủ ngon nha".
  An LLM parses it, YouTube search resolves the track, you confirm the match
  (parsed fields editable) before it queues.
- On-air messages: dedications attached to a request, or standalone
  shoutouts. Recipients are free-text names — whoever isn't listening at that
  moment misses it, and that ephemerality is the point. The DJ **retells**
  your message; raw text never airs verbatim.
- Now-playing, queue peek ("sắp phát") and history in the SPA — synced to
  what you *hear*, not to the server clock.
- The format clock: 2–4 songs per music block → DJ segment (intros, time
  checks, dedications, occasional weather color) → hourly station-ID jingle.
- Cost-aware broadcasting: with zero listeners the station plays music-only
  and spends nothing on LLM/TTS; Tiểu Dương Dương "comes back" (jingle +
  proper greeting) when someone tunes in. She never greets individual joins.

**Scale target:** modest and honest — tens of concurrent listeners, designed
so public spikes are absorbed by Cloudflare's cache rather than the uplink.

## Architecture

One new repo, `radio-service` (Go), two binaries sharing one codebase, the
`radio` database in the shared CNPG cluster, and Redis:

- **`radio-api`** — a standard platform citizen: pure gRPC :9090
  (`algovn.radio.v1` in `protos`), metrics :9091. Call-in parsing +
  moderation (one LLM call), track resolution previews, request/shoutout
  intake, queue and history reads, admin ops (`role:radio-admin`).
- **`radio-studio`** — the broadcast engine. Single-replica Deployment on
  `algovn-w1`: program-director loop, DJ brain (LLM), TTS, ffmpeg rendering,
  HLS publishing, and the ingest workers (it owns the media PVC). An nginx
  sidecar serves the HLS output from a shared volume.

```
listeners ─▶ Cloudflare (segments edge-cached) ─▶ Kong
                                                   ├─ algovn.com/radio/        → radio SPA (static)
                                                   ├─ algovn.com/radio/stream/ → studio nginx (live.m3u8 + segments)
                                                   └─ api.algovn.com/radio/*   → api-control-plane ─gRPC→ radio-api
radio-studio ─renders─▶ HLS volume (◀─ nginx)      api.algovn.com/events/radio.* (SSE)
     │  └─now-playing─▶ RabbitMQ "events" ────────────┘
     └─jobs/state─▶ Postgres `radio` + Redis ◀── radio-api
```

**Delivery is the load-bearing trick.** HLS segments get unique names
(`seg-<epoch>-<n>.ts`), so a Cloudflare cache rule caches them
`immutable, max-age=86400`: N listeners cost roughly one origin fetch per
4-second segment. The manifest is cache-bypassed; its ~1 KB per listener per
4–6 s is all that rides the tunnel. Same mandatory-before-launch cache rule
every SPA already needs.

**Coordination** between api and studio is Postgres (queue + job tables —
durable, no new infra) plus Redis for hot state. Now-playing and queue
changes publish to the `events` exchange (`radio.nowplaying`, `radio.queue`)
→ the platform's existing SSE. There is deliberately **no per-user SSE
channel**: the my-requests view updates from the public queue channel plus
polling, keeping SSE anonymous-safe. LLM/TTS/yt-dlp egress uses the home
residential IP; optional account cookies come from OpenBao if gated videos
ever need them.

## The audio path

**The packager is a Go scheduler, not a process.** Each program item is
rendered ahead of air time by one ffmpeg invocation into final form:
loudness-normalized (−14 LUFS integrated, measured parameters from ingest),
AAC-LC 128k, **already cut into ~4 s `.ts` segments**. A Go goroutine then
publishes segments into `live.m3u8` on the wall clock — when a segment's air
time arrives, it appears in the manifest (window ~60 s, accurate `EXTINF`,
`EXT-X-PROGRAM-DATE-TIME` on every segment). There is no always-on encoder:
steady-state CPU is nginx serving small files. A studio restart resumes at
"next unpublished segment" with one `EXT-X-DISCONTINUITY`.

**The program director** keeps a render-ahead buffer of ~2–5 minutes of
finished segments in front of the playhead. When it runs low it picks the
next item: pending requests first (round-robin across requesters), else
auto-rotation from the library (no-repeat window; mood/era tags exist from
ingest, daypart-aware selection lands in Phase 3); every 2–4 songs a DJ
segment; hourly, the station ID. A crash never catches the stream mid-word —
the "live" is rendered seconds ahead, like real radio automation.

**Transitions:** MVP joins items with short fade-out/fade-in and lets DJ talk
and jingle stings cover the seams — which is how old-school radio actually
sounded. True crossfades are Phase 3, baked at render time (item N+1 renders
mixed with item N's tail) so the scheduler stays a dumb concatenator forever.

**Now-playing follows the ear.** HLS listeners run ~12–20 s behind the studio
playhead. The SPA maps SSE events onto the player's `PROGRAM-DATE-TIME`
position (hls.js exposes it), so the card flips when *you hear* the change,
not when the server made it.

## Track ingest

A five-stage pipeline on a Postgres job table (`FOR UPDATE SKIP LOCKED`, two
download workers, polite throttling):

1. **resolve** — yt-dlp search, top-5 candidates with thumbnails; the user
   confirms one. Dedupe by `yt_id`: an already-`ready` track queues
   instantly.
2. **download** — bestaudio m4a to the media PVC.
3. **analyze** — two-pass loudnorm measurement; parameters stored, original
   kept untouched (normalization applies at render time).
4. **enrich** — one LLM call turns the messy YouTube title into canonical
   *title/artist* and tags *mood/energy/era*. This is also the injection
   scrub: raw YouTube strings never enter any later prompt.
5. **ready** — eligible for queue and rotation.

**Playlist import** (admin): paste a YouTube/Spotify playlist URL → fans out
into per-track jobs through the same pipeline. This seeds the station: the
default sound of Tần Số 42 is the owner's playlists, and the library keeps
growing from listener requests.

## The DJ brain

**One LLM call parses and moderates each call-in**, returning
`{song_query, recipient, message, verdict, sanitized_digest}`. Rejections
(abuse, doxxing, spam, ads) surface to the user with the reason; the
sanitized digest is the only form of user text that ever reaches an on-air
prompt.

**One LLM call writes each DJ segment** (structured JSON out, length-capped;
20–60 s of speech ≈ 300–800 TTS chars). Context: the persona config (name,
voice, catchphrases, formality — a file in the repo, tweakable without
redeploy of logic), the clock and daypart tone map, weather color
(Open-Meteo, free: Hà Nội + TP.HCM), now/next track metadata, up to 3
dedication digests, and a **rolling show memory** — summaries of her last few
segments, so she doesn't repeat herself and can call back ("như lúc nãy mình
có nhắc…").

**Content policy:** trivia about songs and artists is allowed from model
knowledge, unverified — accepted hallucination risk on a personal station —
with one rail in the persona prompt: neutral/positive stories only, no
scandal or negative claims about real people. The brain has no tools; user
and YouTube content arrive only as delimited, pre-sanitized data blocks.

**Voice:** Vietnamese neural TTS — primary Google `vi-VN` (Chirp3-HD/Neural2
class), FPT.AI as a configured alternative → WAV → the renderer mixes it over
a music bed or ducks it over the next song's intro.

## Queue policy

Round-robin across requesters, FIFO within one user. Quotas: 3 pending and
10/day per user. Duplicates handled warmly: if the track is already queued,
a second dedication **merges** onto the same play (two people dedicating one
song is very radio); if it aired < 2 h ago, friendly reject — "vừa phát
xong, để khuya nhé". Requesting while the station is in music-only mode just
works: the requester's own presence heartbeat wakes the DJ.

**Request lifecycle:** submit → parse+moderation → `approved` → ingest (if
not cached) → `ready` → queued → rendered → `aired`; failures become
`failed` with the user notified — the DJ never announces a failure.

## Data

- **Postgres `radio`:** `tracks` (yt_id unique, title, artist, duration,
  file path, loudnorm params, mood/energy/era, status, last_aired_at),
  `requests` (user, track, raw_text, parsed fields, recipient, message,
  sanitized_digest, status: pending/approved/ready/queued/aired/failed/
  rejected), `shoutouts`, `play_history` (kind: track/dj/jingle),
  `dj_segments` (script, model, tokens, tts_chars, cost_usd — the bill's
  audit log), `ingest_jobs` (kind, payload, attempts, locked_by).
- **Redis:** `radio:presence` (ZSET scored by last heartbeat),
  `radio:nowplaying` snapshot, `radio:quota:<user>:<day>`,
  `radio:spend:<date>` (the budget counter). Presence and quotas tolerate
  loss; nothing here is a source of truth (`noeviction` is already platform
  policy).
- **Media PVC:** local-path on `algovn-w1`, 20 GB to start (~8 MB/track ≈
  2,500 tracks). Disk pressure → stop new downloads, LRU-evict
  never-requested tracks.

## API surface (registration sketch)

| Route | Rule | Purpose |
| ----- | ---- | ------- |
| `GET /radio/now-playing`, `/radio/queue`, `/radio/history` | anonymous | reads |
| `POST /radio/heartbeat` | anonymous | presence (session UUID) |
| `POST /radio/resolve` | authenticated | parse call-in text → parsed fields + top-5 candidates |
| `POST /radio/requests` | authenticated | confirm a candidate (+ dedication) → queue |
| `GET /radio/requests/mine` | authenticated | my requests + statuses |
| `POST /radio/shoutouts` | authenticated | standalone on-air message |
| `POST /radio/admin/import` | role:radio-admin | playlist URL → seed ingest |
| `POST /radio/admin/skip`, `DELETE /radio/admin/requests/{id}` | role:radio-admin | on-air control |

Exact verb/path/field mapping is fixed at implementation time against
`iac/docs/api-conventions.md`.

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
  time-check templates, Telegram alert, resets at midnight.
- Prompt injection is a real input class: no tools on any on-air LLM call,
  delimited data blocks, JSON-schema outputs, YouTube titles canonicalized at
  ingest, scripts length-capped post-generation.

## Failure modes

The station never dies; it gets quieter. LLM down → canned script templates.
TTS down → music-only with jingles. yt-dlp failure → request `failed`,
requester notified. RabbitMQ down → SSE metadata stops, audio keeps playing
(HLS is the source of truth). Studio crash → k8s restart, resume publishing
with a discontinuity tag, ~15 s gap, hls.js recovers. Budget cap → graceful
music-only, never dead air. Alerting via the existing Alertmanager →
Telegram path.

## Observability

Gauges: `listener_count`, `buffer_ahead_seconds`, `daily_spend_usd`.
Counters: TTS chars, LLM tokens, request-state transitions, render failures.
Alerts: manifest staleness (the "stream frozen" canary), render-buffer
underrun, spend ≥ 80% of cap, ingest-job failure spikes. One provisioned
Grafana dashboard: the station console.

## Costs

Target $10–30/month, hard-capped at the configured daily ceiling. TTS
dominates: at a few hours of listened-to airtime daily, roughly 300–600k
chars/month ≈ $5–20 on a premium Vietnamese voice (verify current pricing at
build time); script + parse LLM calls are pennies; yt-dlp and Open-Meteo are
free. `dj_segments` records the actuals; the dashboard shows the burn.

## The SPA

A retro receiver: warm dark palette, glowing frequency dial, an ON AIR lamp
lit by the SSE heartbeat, tactile buttons. One screen — player, now-playing
card (thumbnail + dedication line, synced to the audible track), "sắp phát"
queue peek, history, my-requests with live status, and the call-in modal
with its confirm step. Login gates interaction only. React + Vite at
`web/apps/radio` under `algovn.com/radio/`, hls.js player (Safari plays HLS
natively), final aesthetics via the frontend-design pass at build time.

## Testing

- Unit: program-director selection (fairness, no-repeat, wake/sleep
  transitions), quotas, dedupe/merge rules, budget-cap arithmetic, manifest
  scheduling against a fake clock, moderation + parse against recorded LLM
  fixtures.
- Audio golden tests: render an item → assert duration and integrated LUFS
  within tolerance (ffmpeg measures; no human ear in CI).
- Integration: testcontainers Postgres + Redis.
- Dev loop: Tilt with fake-LLM/fake-TTS modes so request→on-air runs locally
  for $0. The ear test — does it *feel* like radio — stays manual.

## Phases

1. **MVP on air** — playlist-import seeding, library auto-rotation
   (no-repeat shuffle), public HLS stream + Cloudflare cache rule, SPA player
   with ear-synced now-playing, call-in requests end to end (parse +
   moderation verdict enforced from day one), basic DJ intros.
2. **The heart** — dedications and shoutouts retold on air, cost-aware
   presence + wake greeting, queue fairness, quotas, dedication merging.
3. **Polish** — scheduled dedications ("phát lúc 21:00 ngày sinh nhật"),
   render-time crossfades, jingle package, daypart-aware music selection,
   weather color, load-validate the cache path.

**Success criteria:** 7 days on air unattended; request→on-air median under
~10 minutes; a friend sends a dedication and hears it read back; monthly AI
spend inside $10–30 with the cap never hit by surprise; zero manual ops
after deploy.

## Not building (deliberately)

Private user-to-user inbox (messages are on-air dedications only),
registered-recipient tagging or notifications (recipients are free-text
names), multiple channels, vote-to-skip, podcast archives of past shows,
mobile apps, Spotify OAuth sync (YouTube-first; Spotify metadata enrichment
can come later), per-user SSE channels.

**Status:** spec.

**Platform it rides on:** api.algovn.com gateway (api-control-plane), Zitadel
login, RabbitMQ→SSE push, shared Postgres, Redis, GitOps deploy via `iac` —
all described in [`../ARCHITECTURE.md`](../ARCHITECTURE.md).
