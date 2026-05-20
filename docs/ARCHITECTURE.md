# Architecture

Component-level design for the briefcast pipeline. Companion to the high-level diagram in the README.

## Runtime model

A single daily cron job runs the full pipeline end-to-end. No persistent server, no queue, no inter-service communication — just a script that fetches, synthesizes, generates audio, and publishes.

Default target: **Cloudflare Workers + R2** (cheap, generous free tier, cron triggers built in). AWS Lambda + S3 is a drop-in alternative if you prefer the AWS ecosystem.

## Pipeline stages

### 1. Ingestion

Three source adapters, all returning a normalized `Item` shape:

```python
@dataclass
class Item:
    source: str          # "gmail" | "substack" | "grok"
    source_detail: str   # newsletter name, publication, or topic
    title: str
    url: str | None
    published_at: datetime
    body_text: str       # cleaned plain text
    body_html: str | None
```

**Gmail adapter** (`src/sources/gmail.py`)
- OAuth2 with read-only scope (`gmail.readonly`).
- Query: `label:{GMAIL_LABEL} newer_than:1d`. User configures Gmail filters once to label newsletters / newspaper digests / Substack emails as `briefing`.
- HTML → text via `readability-lxml` or `trafilatura`.
- One Gmail account, one label, many publications.

**Substack adapter** (`src/sources/substack.py`)
- Reads a hand-curated `profile/substacks.yaml` list of publication slugs.
- Fetches `https://{slug}.substack.com/feed` for each.
- Filters to entries from the last 24h.
- This is the fallback for Substacks the user doesn't get via email.

**Grok adapter** (`src/sources/grok.py`)
- For each "beat" defined in `profile/beats.yaml`, makes one xAI API call.
- Prompt template asks Grok to summarize relevant X posts from the last 24h with links.
- Returns 3–7 bullets per beat, each becoming one `Item`.

### 2. Dedup + filter

`src/pipeline/dedup.py`

- Persistent set of seen URLs and content hashes (stored as JSON in R2/S3).
- Drop items already covered in prior briefings.
- Optionally drop items shorter than N characters (likely promo blurbs).

### 3. Synthesis

`src/pipeline/synthesize.py`

Single Claude call. Inputs:
- All deduped items from today.
- `profile/interests.md` — a hand-maintained markdown profile describing the listener (areas of focus, what to skip, tone preferences, names/orgs to follow).
- Target length in minutes (default 15 → ~2,250 words at ~150 wpm).

Output: a script structured as:

```
[INTRO]    — date, headline tease, length
[SEGMENT]  — repeated per topic, with transitions
[OUTRO]    — sign-off
```

The synthesis prompt is the single biggest quality lever. It lives in `src/prompts/synthesis.md` and is versioned with the repo.

### 4. Audio generation

`src/pipeline/audio.py`

- ElevenLabs Studio API: submit the script, receive an MP3.
- Single voice for v1 (configured via `ELEVENLABS_VOICE_ID`).
- Phase 2: multi-voice dialogue for conversational format.

### 5. Publish

`src/pipeline/publish.py`

- Upload MP3 to R2/S3 at `episodes/{YYYY-MM-DD}.mp3`.
- Rebuild `feed.xml` (RSS 2.0 with iTunes namespace) with the new episode prepended.
- Upload `feed.xml` to a private, unguessable URL.
- The RSS URL is what the user subscribes to in their podcast app.

## Phase 2: companion app

When the feed-only MVP is working well, add a thin frontend that consumes the same artifacts plus a sidecar `episode.json` containing:
- Segment-level timestamps
- Per-segment source URLs
- Full transcript
- Topic tags

This unlocks: skip-segment, save-for-later, transcript search, source-click-through, and per-segment feedback signal that tunes the next day's synthesis prompt.

The app doesn't replace the podcast feed — both stay live, sharing the same backend.

## Configuration files

Lives in `profile/`, gitignored if it contains anything sensitive:

| File             | Purpose                                              |
|------------------|------------------------------------------------------|
| `interests.md`   | Listener profile fed into the synthesis prompt       |
| `substacks.yaml` | Substack publication slugs to pull via RSS           |
| `beats.yaml`     | X topics/users to ask Grok about, one beat per call  |
| `voice.yaml`     | ElevenLabs voice + Studio settings                   |

## Cost model

Rough per-episode cost at 15-minute length, daily cadence:

| Item                  | Per episode | Per month |
|-----------------------|-------------|-----------|
| ElevenLabs (~13.5k chars)  | ~$0.30–0.50 | ~$10–15 |
| Claude synthesis      | ~$0.20–0.50 | ~$8–15  |
| Grok API (3–5 calls)  | ~$0.05–0.20 | ~$2–6   |
| R2/S3 storage + egress| negligible  | ~$1     |
| **Total**             | **~$1–2**   | **~$30–40** |
