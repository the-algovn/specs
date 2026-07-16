# Tần Số 42 — the DJ brain

How Tiểu Dương Dương thinks and speaks. Subsystem design under
[`../radio.md`](../radio.md).

## The director schedules; the brain writes

Scheduling is deterministic code, testable without an LLM. In chatty mode:
a talk slot every 1–2 songs, at least one musing per hour, the hourly
station ID, dedications aired within ~15 minutes of queueing (fused into
the intro when their song is next). The brain receives a **typed brief**
and owns only the words. Segment types and their phases:

| Type | What it is | Phase |
| ---- | ---------- | ----- |
| `intro` | front-announce the next track; may carry dedications | 1 |
| `backsell` | "vừa rồi là…" after a track | 1 |
| `station_id` | hourly, time + station name | 1 |
| `dedication_read` | solo slot for a heavy dedication | 2 |
| `wake_greeting` | return from music-only mode | 2 |
| `musing` | weather color, a tiny story, a question into the night | 2 |
| `daypart_transition` | "đã chín giờ tối rồi đó…" | 3 |

## The brief (input)

One JSON object per segment:

- type, clock ("hai mươi ba giờ mười hai, thứ Năm"), daypart
- weather (Hà Nội + TP.HCM, Open-Meteo, Phase 2)
- now/next track metadata: canonical title, artist, mood, era
- dedications: `[{from, to, digest, weight_hint}]` — the moderation call
  already judged each `casual | warm | heavy` at intake
- show memory: last 5 segment self-summaries + a recent-phrases ring buffer
  (last ~20 idioms) with a don't-reuse instruction
- constraints: hard character budget derived from target seconds

## The output (schema-validated JSON)

- `script` — the words she says
- `summary` — one line; becomes show memory
- `used_phrases` — feeds the ring buffer
- per-dedication `treatment` chosen: `quick` (a warm line inside the intro)
  or `full` (30–60 s, she reflects on the message, speaks to both people,
  lets it breathe) — guided by the weight hint. Deterministic guard: max
  one full treatment per slot; the rest carry over to the next slot.

**Post-validation:** char cap enforced at sentence boundary; digit-lint —
every number spelled out in words ("chín giờ tối", never "21:00"); one
retry with feedback on violation, then fallback.

## The persona bible

A versioned file, ConfigMap-mounted (tweakable without code changes):
identity and backstory, voice register (soft-spoken, warm, playful at the
edges), self-reference style ("Dương Dương"), address style ("bạn nghe
đài"), catchphrase pool, daypart tone map (brighter mornings, intimate
after 21:00), and taboos: no negative gossip about real people, never read
raw user text, no numerals, no fake urgency. Trivia about songs and artists
is **allowed from model knowledge, unverified** — an accepted hallucination
risk on a personal station — bounded by the neutral/positive-only rail.

System prompt = persona bible + output contract. The brain has no tools;
user and YouTube content arrive only as delimited, pre-sanitized data
blocks (the parse/moderation call sanitizes user text; ingest's enrich
stage canonicalizes YouTube titles).

## The parse/moderation call (the brain's sibling)

One LLM call per call-in submission returns
`{song_query, recipient, message, verdict, sanitized_digest, weight}`.
Rejections (abuse, doxxing, spam, ads) surface to the user with the reason.
The sanitized digest is the only form of user text that ever reaches an
on-air prompt; the weight (`casual | warm | heavy`) becomes the brief's
hint.

## Model and cost

Provider interface (`ScriptModel`): Gemini-Flash-class default,
Anthropic-Haiku-class alternate, chosen by config. Per segment ≈ 1.2k input
+ 250 output tokens ≈ $0.0002 — irrelevant next to TTS.

## Voice (TTS)

Primary: Google `vi-VN` Chirp3-HD; Neural2 as the cheaper fallback tier;
FPT.AI as a configured alternative. One fixed voice, fixed rate — the
station's sound. Chirp3-HD has no SSML: pacing is punctuation-driven
(ellipses and commas do the breathing; the prompt teaches "viết như nói").
English titles stay as-is — the voice code-switches. Output WAV → the
renderer resamples, normalizes voice to −16 LUFS against music at −14, and
mixes per [`audio-path.md`](audio-path.md).

**Honest chatty-mode math:** ~10–12 segments/hour listened ≈ 6.5k
chars/hour → at ~4 h of listened airtime daily ≈ 780k chars/month:
**~$23/mo on Chirp3-HD, ~$12.50 on Neural2**. The $1/day default cap
sustains ~5 h of chatty listening daily. Start Chirp3-HD; drop a tier if
the bill annoys.

## Fallback ladder

1. LLM error or 8 s timeout → **canned template pool** per segment type:
   10–15 handwritten, time-aware variants each (Go text/template), still
   TTS'd live.
2. TTS down → **pre-rendered canned audio** (station IDs, generic sweepers
   — a fixed set generated once at build time for cents) or skip to
   jingle.
3. Both down → music-only. Never dead air.

## Example (to nail the voice)

Brief: `intro`, 23:12, dedication Đức→Ngọc weight `heavy`, next:
*Em Của Ngày Hôm Qua* — Sơn Tùng M-TP.

> "Mười một giờ mười hai phút rồi đó, Sài Gòn vẫn còn mưa lất phất… Khuya
> nay có một bạn tên Đức gửi đến Ngọc một bài hát, kèm một lời nhắn mà
> Dương Dương đọc xong cứ mỉm cười mãi — Đức nói rằng, có những ngày chỉ
> muốn quay về hôm qua, để được gặp lại nụ cười đó thêm một lần. Ngọc ơi,
> nếu em đang thức… bài này là dành cho em. *Em Của Ngày Hôm Qua* — Sơn
> Tùng M-TP, trên Tần Số Bốn Mươi Hai."
