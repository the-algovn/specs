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
- Free-text song requests, like a real call-in: type
  "Cho mình xin bài X, gửi tặng Ngọc" → backend resolves a concrete track
  (YouTube search) → you confirm the match → it's queued.
- On-air messages: dedications attached to a request, or standalone
  shoutouts. The DJ **retells** your message — never reads raw text verbatim.
- Now-playing + queue + history in the SPA, live via the platform's SSE.
- The format clock: 2–4 songs per music block → DJ segment (intros, time
  checks, dedications, occasional weather color) → hourly station-ID jingle.
- Cost-aware broadcasting: with zero listeners the station plays music-only
  and spends nothing on LLM/TTS; the DJ "comes back" when someone tunes in.

**Scale target:** modest and honest — tens of concurrent listeners, designed
so public spikes are absorbed by Cloudflare's cache rather than the uplink.

## Architecture

One new repo, `radio-service` (Go), two binaries sharing one codebase, the
`radio` database in the shared CNPG cluster, and Redis:

- **`radio-api`** — a standard platform citizen: pure gRPC :9090
  (`algovn.radio.v1` in `protos`), metrics :9091. Request/shoutout intake
  (validate → LLM moderation → persist), track resolution previews, queue and
  history reads, admin ops (skip/remove, `role:radio-admin`).
- **`radio-studio`** — the broadcast engine. Single-replica Deployment on
  `algovn-w1`: program-director loop, LLM script generation, TTS, ffmpeg
  rendering + HLS packaging, and the yt-dlp fetcher (it owns the media PVC).
  An nginx sidecar serves the HLS output from a shared volume.

```
listeners ─▶ Cloudflare (segments edge-cached) ─▶ Kong
                                                   ├─ algovn.com/radio/        → radio SPA (static)
                                                   ├─ algovn.com/radio/stream/ → studio nginx (live.m3u8 + segments)
                                                   └─ api.algovn.com/radio/*   → api-control-plane ─gRPC→ radio-api
radio-studio ─renders─▶ HLS volume (◀─ nginx)      api.algovn.com/events/radio.* (SSE)
     │  └─now-playing─▶ RabbitMQ "events" ────────────┘
     └─jobs/state─▶ Postgres `radio` + Redis ◀── radio-api
```

**Delivery is the load-bearing trick.** HLS segments get unique sequence
names, so a Cloudflare cache rule caches them long-TTL: N listeners cost
roughly one origin fetch per 4-second segment. Only the tiny no-cache
manifest poll (~1 KB per listener per 4–6 s) rides the tunnel. This is the
same mandatory-before-launch cache rule every SPA already needs.

**Coordination** between api and studio is Postgres (queue and job tables —
durable, no new infra) plus Redis for hot state: anonymous listener presence
(heartbeat → TTL keys), the now-playing snapshot, quota counters. Now-playing
and queue changes publish to the `events` exchange (`radio.nowplaying`,
`radio.queue`) → the platform's existing SSE. LLM/TTS/yt-dlp egress uses the
home residential IP (friendly to yt-dlp); optional account cookies come from
OpenBao if gated videos ever need them.

## The studio pipeline

The **program director** keeps a render-ahead buffer of ~2–5 minutes of
finished audio in front of the playhead. When it runs low, it picks the next
item: pending requests first (round-robin across requesters), else
auto-rotation (daypart-aware shuffle, no-repeat window); every 2–4 songs a DJ
segment; hourly, the station ID. A crash never catches the stream mid-word —
the "live" is rendered seconds ahead, like real radio automation.

**DJ segments, two steps.** *Brain:* one cheap-LLM call (Gemini Flash /
Claude Haiku class) with the persona config (name, style, catchphrases),
time of day, now/next track metadata, and the pending-dedication digest →
a structured, length-capped script that never quotes user text verbatim.
*Voice:* Vietnamese neural TTS — primary Google `vi-VN` (Chirp3-HD/Neural2
class), FPT.AI as a configured alternative → WAV.

**Rendering:** each item becomes a finished chunk via one ffmpeg filtergraph —
songs loudness-normalized to −14 LUFS with silence-trimmed edges and
crossfade handles; DJ voice over a music bed, or sidechain-ducked over the
next song's instrumental intro; jingles are pre-rendered assets. A single
long-lived packager ffmpeg concatenates chunks into AAC-LC 128k HLS:
4-second segments, unique names, ~60 s sliding window,
`EXT-X-DISCONTINUITY` after restarts.

**Request lifecycle:** submit → LLM moderation → `approved` → yt-dlp download
+ loudness analysis (Postgres job) → `ready` → queued → rendered → `aired`.
Every transition notifies the requester over SSE. A failed download becomes
`failed` — the user is told; the DJ never announces it.

**Cost-aware mode:** SPA players send an anonymous heartbeat every 30 s
(session UUID, Kong rate-limited) → Redis TTL keys → listener count. Zero
listeners for 10 minutes → music-only, no LLM/TTS calls; the first heartbeat
brings Tiểu Dương Dương back at the next segment slot.

## Data

- **Postgres `radio`:** `tracks` (yt_id, title, artist, duration, file path,
  lufs, status), `requests` (user, track, dedication-to, message, status:
  pending/approved/ready/queued/aired/failed/rejected), `shoutouts`,
  `play_history`, `dj_segments` (script, model, tokens, cost — the bill's
  audit log).
- **Redis:** `radio:presence:<session>` (TTL), `radio:nowplaying`,
  `radio:quota:<user>`. Presence and quotas tolerate loss; nothing here is
  a source of truth (`noeviction` is already platform policy).
- **Media PVC:** local-path on `algovn-w1`, 20 GB to start (~8 MB/track).
  Disk pressure → stop new downloads, LRU-evict never-requested tracks.

## API surface (registration sketch)

| Route | Rule | gRPC |
| ----- | ---- | ---- |
| `GET /radio/now-playing`, `/radio/queue`, `/radio/history` | anonymous | reads |
| `POST /radio/heartbeat` | anonymous | presence |
| `POST /radio/resolve` | authenticated | ResolveTrack (search preview) |
| `POST /radio/requests` | authenticated | CreateRequest (track + optional dedication) |
| `POST /radio/shoutouts` | authenticated | SendShoutout |
| `POST /radio/admin/skip` etc. | role:radio-admin | admin ops |

Exact verb/path/field mapping is fixed at implementation time against
`iac/docs/api-conventions.md`.

## Abuse & safety

Public stream, so: requests and shoutouts require login and pass an LLM
moderation gate before persisting as `approved`; the DJ paraphrase is the
second layer — raw user text never reaches the air. Per-user quotas (pending
cap, daily cap) and round-robin queueing stop one person monopolizing the
station. The anonymous heartbeat is rate-limited at Kong and grants nothing
but a presence bit.

## Failure modes

The station never dies; it gets quieter. LLM down → canned script templates.
TTS down → music-only with jingles. yt-dlp failure → request `failed`,
requester notified. RabbitMQ down → SSE metadata stops, audio keeps playing
(HLS is the source of truth). Studio crash → k8s restart, discontinuity tag,
~15 s gap, hls.js recovers; Alertmanager → Telegram as usual.

## Costs

Target $10–30/month. TTS dominates: at a few hours of listened-to airtime
daily, roughly 300–600k chars/month ≈ $5–20 on a premium Vietnamese voice
(verify current pricing at build time); script LLM is pennies; yt-dlp and
Open-Meteo weather are free. Cost-aware mode makes spend proportional to
actual listening; `dj_segments` records the actuals.

## Testing

- Unit: program-director selection (fairness, no-repeat, dayparts), quotas,
  moderation gate against recorded LLM fixtures.
- Audio golden tests: render an item → assert duration and integrated LUFS
  within tolerance (ffmpeg measures; no human ear in CI).
- Integration: testcontainers Postgres + Redis.
- Dev loop: Tilt with fake-LLM/fake-TTS modes so request→on-air runs locally
  for $0. The ear test — does it *feel* like radio — stays manual.

## Phases

1. **MVP on air** — library auto-rotation, public HLS stream + Cloudflare
   cache rule, SPA player with now-playing, song requests end to end, basic
   DJ intros.
2. **The heart** — dedications and shoutouts on air, moderation gate,
   cost-aware presence, queue fairness + quotas.
3. **Polish** — jingle package, dayparts, weather color, request-status SSE
   niceties, validate the cache path under load.

**Success criteria:** 7 days on air unattended; request→on-air median under
~10 minutes; a friend sends a dedication and hears it read back; monthly AI
spend inside $10–30; zero manual ops after deploy.

## Not building (deliberately)

Private user-to-user inbox (messages are on-air dedications only), multiple
channels, vote-to-skip, podcast archives of past shows, mobile apps, Spotify
OAuth sync (YouTube-first; Spotify metadata enrichment can come later).

**Status:** spec.

**Platform it rides on:** api.algovn.com gateway (api-control-plane), Zitadel
login, RabbitMQ→SSE push, shared Postgres, Redis, GitOps deploy via `iac` —
all described in [`../ARCHITECTURE.md`](../ARCHITECTURE.md).
