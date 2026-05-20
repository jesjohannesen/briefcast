# Roadmap

Phased plan from empty repo to working briefing, in dependency order. Each phase ends with something testable.

## Phase 0 — Repo scaffold ✅

- README, ARCHITECTURE, ROADMAP
- `.env.example`, `.gitignore`
- GitHub remote, private

## Phase 1 — Source adapters (offline)

Get clean data flowing without worrying about audio or publishing yet. Outputs go to `output/raw_items.json` for inspection.

- [ ] Project scaffold (`pyproject.toml`, `src/briefcast/`, basic logging)
- [ ] Gmail OAuth flow + `src/sources/gmail.py`
- [ ] Set up Gmail filters → label `briefing` (one-time manual step, documented)
- [ ] Substack RSS adapter + `profile/substacks.yaml`
- [ ] Grok adapter + `profile/beats.yaml`
- [ ] Item normalization + dedup with on-disk seen-set
- [ ] `python -m briefcast.fetch` writes a single normalized `raw_items.json`

**Done when:** running `fetch` end-to-end produces a JSON file with items from all three sources, deduped against yesterday.

## Phase 2 — Synthesis

- [ ] `profile/interests.md` — initial hand-written listener profile
- [ ] `src/prompts/synthesis.md` — versioned synthesis prompt
- [ ] `src/pipeline/synthesize.py` — single Claude call, structured output
- [ ] Word-count check / length tuning against `BRIEFING_TARGET_MINUTES`
- [ ] `python -m briefcast.script` writes `output/script-{date}.txt`

**Done when:** the script reads like something you'd actually want to listen to. Iterate the prompt and `interests.md` until it does. Expect this phase to take longer than feels reasonable — it's the product.

## Phase 3 — Audio + local playback

- [ ] `src/pipeline/audio.py` — ElevenLabs Studio API integration
- [ ] Local MP3 output at `output/{date}.mp3`
- [ ] Manually AirDrop / drag into a podcast app to test listening UX

**Done when:** you've listened to three real briefings on your phone and they're worth your time.

## Phase 4 — Cloud cron + RSS feed

- [ ] Pick storage: R2 (default) or S3
- [ ] Upload MP3 to bucket at `episodes/{date}.mp3`
- [ ] Build / update `feed.xml` (RSS 2.0 + iTunes namespace)
- [ ] Deploy to Cloudflare Workers (or Lambda) with cron trigger
- [ ] Subscribe to the private feed URL in your podcast app

**Done when:** new episode appears in Overcast / Apple Podcasts automatically every morning.

## Phase 5 — Quality iteration

Long-running, no fixed scope. Examples:

- [ ] Per-segment timestamps (write a sidecar `episode.json`)
- [ ] Smarter dedup (semantic, not just URL)
- [ ] Source diversity guard (don't let one newsletter dominate)
- [ ] "What changed since yesterday" framing for ongoing stories
- [ ] Cold-open A/B testing

## Phase 6 — Companion app

Only after Phase 5 has plateaued and a richer UI is the real bottleneck.

- [ ] Decide stack (likely React Native + Expo, sharing transcript JSON with the feed pipeline)
- [ ] Per-segment skip / save / source-link
- [ ] Feedback signal piped back into synthesis prompt
- [ ] Auth (you're the only user — magic link is fine)

## Non-goals

- Multi-tenant / public product. This is a single-user tool.
- Real-time / breaking news. Daily batch is the whole point.
- Replacing reading. The briefing is a complement to deep reading, not a substitute.
