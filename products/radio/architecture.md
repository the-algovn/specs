# Tần Số 42 — system architecture

The detailed technical architecture. Subsystem design under
[`../radio.md`](../radio.md); siblings: [`audio-path.md`](audio-path.md),
[`ingest.md`](ingest.md), [`dj-brain.md`](dj-brain.md).

## Context

```
                        Cloudflare edge (TLS, segment cache) ── tunnel ──▶ cloudflared ──▶ Kong
                                                                                            │
              ┌──────────────────────────────┬──────────────────────────────────────────────┤
              ▼                              ▼                                              ▼
   algovn.com/radio/              algovn.com/radio/stream/                        api.algovn.com
   radio SPA (nginx static)       studio nginx sidecar                          api-control-plane
                                  live.m3u8 + seg-*.ts                       /radio/* ──gRPC──▶ radio-api
                                        ▲                                    /events/radio.*  ◀── RabbitMQ "events"
                                        │ shared emptyDir                                          ▲
                                   radio-studio ───────── publishes nowplaying/queue ──────────────┘
                                        │
                       Postgres `radio` ┴ Redis ─── also read/written by radio-api
```

One repo `radio-service`, **one container image, two entrypoints**
(`cmd/radio-api`, `cmd/radio-studio`). Both need yt-dlp (api searches,
studio downloads), ffmpeg lives in the same image anyway, and one image
means one build, one scan, one version to reason about. Each pod refreshes
yt-dlp into its own emptyDir at start (checksum-verified, baked binary as
fallback) — the media PVC is not involved in binary management.

## Components and internal modules

Go package boundaries (each unit: one purpose, testable alone):

```
radio-service/
  cmd/radio-api/ cmd/radio-studio/
  internal/
    api/        thin gRPC handlers; claims decode (read-only, never re-verify)
    callin/     parse+moderation LLM call → verdict, digest, weight
    catalog/    tracks, search resolution, candidate ranking
    ingest/     job runner (SKIP LOCKED), yt-dlp exec, 5 stages
    queue/      requests, round-robin fairness, quotas, dedication merge
    director/   the clock: slot scheduling, selection, wake/sleep
    brain/      briefs, prompts, ScriptModel providers (gemini|anthropic)
    voice/      TTS providers (google|fpt), canned-audio fallback
    render/     ffmpeg filtergraphs → normalized, pre-segmented items
    publish/    manifest scheduler: wall-clock segment publishing, HLS state
    presence/   heartbeat ZSET, listener count, wake signal
    spend/      budget meter: price-at-write, daily cap enforcement
    events/     RabbitMQ topic publisher (fire-and-forget)
    store/      Postgres repos + migrations (embedded, run by api at start)
  assets/       jingles, canned audio   persona/  the bible
```

**radio-api** (2 replicas, stateless): serves the gRPC surface; owns
moderation, resolution, request/shoutout intake, quota enforcement, reads.
Publishes `radio.queue` events on queue-changing writes.

**radio-studio** (1 replica, `strategy: Recreate`): owns the media PVC and
everything that renders or airs. Internal loop: presence watcher → director
picks next item → brain/voice (if awake) → render → publish. Publishes
`radio.nowplaying` when the playhead crosses an item boundary. Ingest
workers run here too (they own the disk). A deploy or crash means a ~15 s
stream gap — accepted for a single-host home cluster; hls.js rides through
the discontinuity.

**nginx sidecar**: serves `live.m3u8` + segments from the shared emptyDir
with the exact Cache-Control split (segments immutable, manifest no-store).

## Key flows

**Call-in → on air**

1. SPA `POST /radio/resolve {text}` → gateway → `ResolveCallIn`: one LLM
   call (parse + moderate + weight) then `ytsearch10` + ranking. Returns
   parsed fields + top-5 candidates. This is the one slow synchronous
   route (~3–6 s): its registration gets a generous timeout and the SPA
   shows a tuning-dial spinner.
2. User confirms (fields editable) → `POST /radio/requests {yt_id, …}` →
   `CreateRequest`: quota check (Redis), dedupe (queued track → merge
   dedication; aired < 2 h → friendly reject), INSERT `requests` row
   (`approved`), INSERT `ingest_jobs` if the track isn't `ready`, publish
   `radio.queue`.
3. Studio ingest workers: download → analyze → enrich → track `ready` →
   request `ready`.
4. Director (buffer low): next slot goes to the oldest `ready` request in
   round-robin order → brain brief (with dedication digest) → TTS →
   render (intro + track, pre-segmented) → request `queued`.
5. Publisher: segments air on the wall clock → `play_history` row, request
   `aired`, `radio.nowplaying` published → SPA flips the card when the
   listener's `PROGRAM-DATE-TIME` catches up.

**Playlist import**: admin `POST /radio/admin/import {url}` →
`--flat-playlist -J` expands entries → one `ingest_job` per entry
(provenance `auto`, paced 2-concurrent/jittered/≤200 per day) → tracks flow
through the same stages into rotation.

**Tune-in / wake**: player fetches manifest+segments (possibly entirely
from CF cache); SPA begins 30 s heartbeats with a session UUID →
`radio:presence` ZSET → director's presence watcher sees count 0→1: at the
next slot boundary, wake jingle + `wake_greeting` brief. Sleep is the
mirror: count 0 for 10 min → music-only (brain/voice idle).

**Budget trip**: every brain/voice call runs through `spend` — price
computed at write, `INCRBY radio:spend:<date>` (atomic; midnight key
rollover). Meter ≥ cap → director flag: canned/music-only until tomorrow;
alert at 80% and at trip.

## Contract sketch — `algovn.radio.v1`

```proto
service RadioService {
  // anonymous
  rpc GetNowPlaying(GetNowPlayingRequest) returns (GetNowPlayingResponse);
  rpc GetQueue(GetQueueRequest) returns (GetQueueResponse);
  rpc GetHistory(GetHistoryRequest) returns (GetHistoryResponse);
  rpc Heartbeat(HeartbeatRequest) returns (HeartbeatResponse);          // session_id
  // authenticated
  rpc ResolveCallIn(ResolveCallInRequest) returns (ResolveCallInResponse);
  rpc CreateRequest(CreateRequestRequest) returns (CreateRequestResponse);
  rpc CreateShoutout(CreateShoutoutRequest) returns (CreateShoutoutResponse);
  rpc ListMyRequests(ListMyRequestsRequest) returns (ListMyRequestsResponse);
  // role:radio-admin
  rpc ImportPlaylist(ImportPlaylistRequest) returns (ImportPlaylistResponse);
  rpc SkipCurrent(SkipCurrentRequest) returns (SkipCurrentResponse);
  rpc RemoveRequest(RemoveRequestRequest) returns (RemoveRequestResponse);
}
```

Key messages: `TrackCandidate {yt_id, title, channel, duration_s,
thumbnail_url}`, `ParsedCallIn {song_query, recipient, message}`,
`RequestStatus` enum mirroring the state machine below. Buf-managed in
`protos`, `buf breaking`-gated, generated Go consumed as a module — a
protos release must ship before the service can build outside the
workspace (platform convention).

**SSE payloads** (JSON, snapshots, no replay):

- `radio.nowplaying`: `{kind: track|dj|jingle, track: {title, artist,
  thumbnail_url} | null, dedication: {from, to} | null, started_at,
  program_date_time}`
- `radio.queue`: `{items: [{position, title, artist, requested_by,
  has_dedication}], updated_at}`

## Data architecture

Schema (key columns; all tables get `id bigserial`, `created_at`):

- `tracks` — `yt_id text UNIQUE`, canonical `title`/`artist`, `duration_s`,
  `file_path`, `loudnorm jsonb`, `mood`, `energy int`, `era`,
  `flags jsonb` (live/cover/remix), `status`, `provenance` (request|auto),
  `last_aired_at`. Index: `(status, last_aired_at)` for rotation picks.
- `requests` — `user_sub`, `track_id FK NULL until resolved`, `raw_text`,
  `parsed jsonb`, `recipient`, `message`, `digest`, `weight`, `status`,
  `fail_reason`, `aired_at`. Indexes: `(status)`, `(user_sub, created_at)`.
- `shoutouts` — as requests minus track fields.
- `play_history` — `kind`, `track_id`, `dj_segment_id`, `started_at`,
  `duration_s`.
- `dj_segments` — `type`, `brief jsonb`, `script`, `summary`, `model`,
  `in_tokens`, `out_tokens`, `tts_chars`, `cost_usd numeric`, `aired_at`.
- `ingest_jobs` — `kind`, `payload jsonb`, `status`, `attempts`,
  `locked_by`, `locked_at`, `run_at`, `last_error`. Index:
  `(status, run_at)` for the SKIP LOCKED poll.

**State machines**

```
request:  pending ─▶ rejected                    track:  ingesting ─▶ ready ─▶ evicted
          pending ─▶ approved ─▶ ready ─▶ queued ─▶ aired         ingesting ─▶ failed
                      approved ─▶ failed
job:      queued ─▶ running ─▶ done
                    running ─▶ failed ─▶ queued (retryable, backoff)
                    running ─▶ stalled (extractor_broken, alerts)
```

**Redis keys**: `radio:presence` (ZSET, member=session UUID, score=last
beat; count = range over 90 s), `radio:nowplaying` (JSON snapshot),
`radio:quota:<sub>:<yyyymmdd>` (INCR, TTL 48 h), `radio:spend:<yyyymmdd>`
(FLOAT-cents INCRBY, TTL 48 h), `radio:hls:seq` (last media sequence — keeps
manifest sequence numbers monotonic across studio restarts).

**Consistency rules**: Postgres is the only source of truth. SSE events are
best-effort snapshots — no outbox, no replay; a reconnecting client
re-reads via GET. Ingest effects are idempotent by `yt_id` (re-download
overwrites), renders idempotent by item id (re-render replaces files in
`tmp/` before atomic move). The studio reconciles at start: re-lease
expired job locks, mark the interrupted on-air item `aired`, resume the
manifest from `radio:hls:seq` + discontinuity, re-render the buffer.

## Deployment inventory (`iac/apps/radio/`)

| Object | Notes |
| ------ | ----- |
| Deployment `radio-api` | 2 replicas, stateless; headless Service :9090 `grpc`, :9091 `metrics`; native gRPC health probes |
| Deployment `radio-studio` | 1 replica, `Recreate`; media PVC (RWO, local-path, 20 Gi); HLS emptyDir shared with nginx sidecar; Service :8080 for the stream |
| Ingress `radio-stream` | `algovn.com/radio/stream/*` → nginx sidecar; Kong rate-limit tier sized for manifest polling |
| SPA | `web/apps/radio` → nginx static image, Ingress `algovn.com/radio/`, CF cache rule on `assets/*` (mandatory) + `stream/seg-*` (this product's addition) |
| Registration YAML | `iac/apps/api-control-plane/registrations/radio.yaml` — the routes table in [`../radio.md`](../radio.md); generous timeout on `/radio/resolve`; dev copy in `dev/registrations` (both must change together) |
| ExternalSecrets | `radio-llm` (Gemini/Anthropic key), `radio-tts` (GCP SA), `radio-youtube-cookies` (optional, default absent) — all from OpenBao via ESO |
| ConfigMaps | `radio-config` (knobs below), `radio-persona` (the bible) |
| CNPG | database `radio` + owner role, declarative onboarding |
| VMServiceScrape ×2, VMRule | metrics + the alert set (staleness, underrun, spend, ingest spikes) |
| Argo Application | app-of-apps entry; merge to `main` is the only deploy path |

## Config surface (`radio-config`)

| Key | Default |
| --- | ------- |
| `talk.min_songs` / `talk.max_songs` | 1 / 2 (chatty) |
| `talk.musings_per_hour_min` | 1 |
| `buffer.target_seconds` | 180 |
| `hls.segment_seconds` / `hls.window_segments` | 4 / 15 |
| `presence.sleep_after_seconds` | 600 |
| `presence.heartbeat_ttl_seconds` | 90 |
| `spend.daily_cap_usd` | 1.00 |
| `rotation.no_repeat_hours` / `rotation.replay_cooldown_hours` | 4 / 2 |
| `quota.pending` / `quota.daily` | 3 / 10 |
| `ingest.concurrency` / `ingest.daily_import_cap` | 2 / 200 |
| `llm.provider` / `llm.model` | gemini / flash-class |
| `tts.provider` / `tts.voice` | google / `vi-VN` Chirp3-HD |
| `weather.cities` | Hà Nội, TP.HCM |

## Capacity and performance

Renders: a 4-minute track ≈ 5–15 s of one core, ~15 items/hour — bursts,
not load. TTS+brain latency (3–6 s/segment) hides entirely inside the
2–5 min render-ahead buffer. Origin stream traffic: one ~64 KB segment per
4 s (CF absorbs the fan-out) + ~1 KB manifest per listener per poll — tens
of listeners ≈ single-digit KB/s on the uplink. Postgres < 5 QPS; RabbitMQ
< 1 msg/s. Cold start to audible: ~10–20 s (yt-dlp fetch is lazy, first
render dominates). Everything fits inside `algovn-w1`'s existing headroom;
the studio's 1 Gi memory limit is the only new reservation that matters.

## Security boundaries

AuthN terminates at the control plane (JWKS-verified JWT); `radio-api`
does a read-only claims decode and never re-verifies (platform contract).
`role:radio-admin` guards admin routes at the gateway; the service
double-checks the role claim on admin methods (belt and suspenders, free).
The studio accepts no inbound traffic except nginx's stream files; its
egress (YouTube, LLM, TTS, Open-Meteo) rides the home residential IP.
Injection rails per [`dj-brain.md`](dj-brain.md); moderation and quota
enforcement in `radio-api`; presence endpoint is anonymous by design and
rate-limited at Kong. The stream path is deliberately unauthenticated —
public listening is the product.
