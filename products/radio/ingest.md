# Tần Số 42 — music ingest

How a song gets from YouTube into the library. Subsystem design under
[`../radio.md`](../radio.md).

## Engine: yt-dlp as a managed subprocess

yt-dlp is the only continuously-maintained option (pure-Go YouTube libraries
break monthly and cannot search). The studio image bundles ffmpeg and a
pinned yt-dlp binary.

**Update strategy** — YouTube breaks extractors on their schedule, not our
release schedule, so a broken yt-dlp must not require an image rebuild: at
pod start (and nightly) each pod refreshes the latest yt-dlp release binary
into its own emptyDir (api needs it for search, studio for downloads),
checksum-verified, falling back to the baked-in binary if the fetch fails
or the fetched binary errors. The resilience property that
matters: **broadcast never touches yt-dlp** — when YouTube breaks it, only
new ingests stall; rotation plays on from the library.

**Invocation discipline:** exec-array args (no shell interpolation), JSON
output (`-J`), per-job temp dirs, hard timeouts, one log line per job with
the failure class.

## The five stages

A Postgres `ingest_jobs` table (`FOR UPDATE SKIP LOCKED`), two download
workers, polite pacing.

1. **resolve** — `ytsearch10:` on the parsed song query; rank; top-5 with
   thumbnails to the user; their confirm is ground truth. Dedupe by
   `yt_id`: an already-`ready` track queues instantly.
2. **download** — `-f bestaudio`, stored as-is (opus/m4a — no ingest
   transcode; the renderer re-encodes exactly once). Into `tmp/`, atomic
   rename on success.
3. **analyze** — ffprobe validation (decodable, duration within ±2 s of
   metadata), then two-pass loudnorm measurement; parameters stored as
   JSON, original untouched (normalization applies at render time).
4. **enrich** — one LLM call turns the messy YouTube title into canonical
   *title/artist* plus flags (live/cover/remix) and tags *mood/energy/era*
   for dayparting. This is also the injection scrub: raw YouTube strings
   never enter any later prompt.
5. **ready** — eligible for queue and rotation.

## Candidate ranking (request path)

Score = weighted sum, tuned by tests, human-confirmed so failures are
papercuts:

- **Prefer:** "- Topic" auto-channels (canonical audio, clean metadata),
  official artist channels, titles with "official audio / MV / lyric".
- **Penalize:** `live | cover | remix | karaoke | sped up | nightcore |
  8D | mix | compilation` — *unless the query itself asked for it* (token
  overlap with the parsed query), duration > 8 min or < 60 s, near-zero
  view counts.

## Playlist import (seed path)

Admin submits a **YouTube playlist URL** (the owner's library lives on
YouTube/YouTube Music; Spotify is deliberately out of scope — it would only
add a matching problem we don't have). `--flat-playlist -J` expands entries
without fetching each video page; every entry becomes an ingest job with
`auto` provenance — no per-track confirm, because the owner curated the
exact videos. Public/unlisted playlists keep the happy path cookieless.

**Pacing keeps the account profile boring:** 2 concurrent downloads,
5–15 s jittered gaps between starts, ~200 tracks/day courtesy cap. A
300-track seed takes about a day; the station starts rotating at ~50 ready
tracks.

## Failure taxonomy

| Class | Handling |
| ----- | -------- |
| `removed / private` | terminal |
| `geo_blocked / age_gated` | one retry with the optional cookie file, else terminal |
| `network / timeout / 5xx` | 3 retries, exponential backoff (1 m / 5 m / 25 m) |
| `extractor_broken` | refresh yt-dlp binary, retry once, else park `stalled` + alert — a spike here means YouTube moved |
| `bot_challenge` | global ingest backoff 1 h + alert (rate profile too hot) |

Terminal request-path failures notify the requester (`failed` status); the
DJ never announces a failure. Import-path failures log and skip.

## Cookies and account

Optional Netscape cookie file exported from the owner's browser, stored in
OpenBao → ESO → Secret. **Default off** — cookieless is the designed happy
path (public music is rarely gated); cookies are only the retry path for
age/geo-gated content, refreshed manually when needed. Known escalation
risk: if YouTube widens PO-token/JS-challenge requirements, the mitigation
is bundling a JS runtime for yt-dlp's challenge solver in the studio image —
an accepted, deferred change.

## Storage and eviction

Media PVC (local-path, `algovn-w1`, 20 GB to start):

```
tracks/    <yt_id>.<ext>   originals, as downloaded
voice/     rendered TTS awaiting mix
jingles/   pre-rendered station assets
segments/  published HLS window
tmp/       in-flight downloads and renders
```

Postgres is the index; a weekly sweep deletes orphan files and re-queues or
evicts orphan rows. Disk pressure: stop new downloads, evict never-requested
tracks LRU-first; requested tracks are pinned for 90 days. Losing the PVC
entirely is acceptable — everything re-downloads on demand.
