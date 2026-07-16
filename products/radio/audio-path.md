# Tần Số 42 — the audio path

How rendered audio becomes a live stream. Subsystem design under
[`../radio.md`](../radio.md).

## The packager is a Go scheduler, not a process

Each program item is rendered ahead of air time by one ffmpeg invocation
into final form: loudness-normalized (music −14 LUFS integrated, voice −16,
parameters measured at ingest), AAC-LC 128k, **already cut into ~4 s `.ts`
segments**. A Go goroutine then publishes segments into `live.m3u8` on the
wall clock — when a segment's air time arrives, it appears in the manifest.

- Manifest window ~60 s; accurate `EXTINF`; `EXT-X-PROGRAM-DATE-TIME` on
  every segment.
- Segment names unique across restarts: `seg-<epoch>-<n>.ts`.
- There is no always-on encoder: steady-state CPU is nginx serving small
  files. Renders are serialized (one ffmpeg at a time).
- A studio restart resumes at "next unpublished segment" with one
  `EXT-X-DISCONTINUITY`. State is a manifest position, not an encoder
  mid-stream.

## The program director

Keeps a render-ahead buffer of ~2–5 minutes of finished segments in front of
the playhead. When it runs low it picks the next item:

1. Pending requests first — round-robin across requesters, FIFO within one.
2. Otherwise auto-rotation from the library — no-repeat window; mood/era
   tags exist from ingest; daypart-aware *selection* lands in Phase 3.
3. Talk slots per the chatty clock: every 1–2 songs (type chosen by the
   director — see [`dj-brain.md`](dj-brain.md)), at least one musing per
   hour, the hourly station ID.

A crash never catches the stream mid-word — the "live" is rendered seconds
ahead, like real radio automation.

## Transitions

MVP joins items with short fade-out/fade-in and lets DJ talk and jingle
stings cover the seams — which is how old-school radio actually sounded.
True crossfades are Phase 3, baked at render time (item N+1 renders mixed
with item N's tail) so the scheduler stays a dumb concatenator forever.
Voice-over-intro ducking (sidechain) applies where the next track has a
usable instrumental lead-in.

## Now-playing follows the ear

HLS listeners run ~12–20 s behind the studio playhead. The SPA maps SSE
events onto the player's `PROGRAM-DATE-TIME` position (hls.js exposes it),
so the card flips when *you hear* the change, not when the server made it.

## Presence and cost-aware broadcasting

SPA players send an anonymous heartbeat (session UUID) every 30 s →
`radio:presence` ZSET scored by timestamp; the listener count is a range
count over the last 90 s. Kong rate-limits the endpoint per-IP.

- Zero listeners for 10 minutes → **music-only mode**: no LLM/TTS calls,
  rotation continues, jingles still air.
- First heartbeat → the DJ returns at the next slot boundary with a jingle
  and a proper wake greeting. She never greets individual joins.
- Requesting a song implies presence: the requester's own heartbeat wakes
  the station.
- The daily budget cap (see [`../radio.md`](../radio.md), Abuse & safety)
  forces music-only mode when hit, independent of presence.

## Delivery and caching

- `algovn.com/radio/stream/live.m3u8` + segments served by the studio pod's
  nginx sidecar from the shared HLS volume, routed via Kong.
- Cloudflare cache rule (mandatory before launch): segments
  `public, max-age=86400, immutable`; manifest cache-bypassed
  (`no-store`). N listeners ≈ one origin fetch per segment + manifest polls
  (~1 KB per listener per 4–6 s).
- Latency posture: ~12–20 s behind live is irrelevant for radio and fully
  absorbed by the ear-sync above.

## Studio resources

Requests/limits around 512 Mi–1 Gi memory, 250m–1 CPU; ffmpeg bursts are
short (one item at a time). HLS volume is an emptyDir (the stream is
ephemeral by nature); the media library PVC is separate — layout in
[`ingest.md`](ingest.md).

## Stream observability

`buffer_ahead_seconds` (underrun alert), manifest staleness (the
stream-frozen canary: manifest older than 2× segment duration), render
failure counter, listener_count gauge. All on the station's Grafana
dashboard, alerts via Alertmanager → Telegram.
