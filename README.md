# briefcast

Personalized daily audio news briefings, synthesized from your own sources and delivered as a private podcast feed.

A scheduled pipeline pulls from your Gmail (newsletters + newspaper digests + Substack-by-email), Substack RSS, and Grok summaries of X (Twitter), feeds the raw items into Claude for synthesis, generates audio via ElevenLabs Studio, and publishes the result as a private RSS feed you can subscribe to in any podcast app.

## Status

Pre-MVP. Architecture defined, scaffolding in progress.

## Design goals

- **Single funnel for inputs.** Most paid newspapers and Substacks already arrive as email, so Gmail is one integration covering many sources.
- **Cron, not a server.** A daily cloud job is enough — no always-on backend until there's a real reason for one.
- **Podcast feed first, app later.** Ship audio + RSS as MVP so any podcast app works. Build a custom frontend only once content quality is dialed in.
- **The synthesis prompt is the product.** 80% of perceived quality lives in the Claude prompt that turns raw items into a script. Everything else is plumbing.

## Architecture

```
┌─ Gmail API ──────┐
│  label:briefing   │  newsletters, newspaper digests, Substack-by-email
└──────┬───────────┘
       │
┌─ Substack RSS ───┐
│  per-publication │
└──────┬───────────┘
       │
┌─ Grok API ───────┐
│  per topic/user  │  live X summaries (per "beat")
└──────┬───────────┘
       ▼
   raw_items.json  ── dedup vs. yesterday ──►  Claude (synthesis)
                                                      │
                                                      ▼
                                           script.txt (~2,500 words)
                                                      │
                                                      ▼
                                       ElevenLabs Studio API → MP3
                                                      │
                                                      ▼
                                           S3/R2 + RSS feed XML
                                                      │
                                                      ▼
                                  Apple Podcasts / Overcast / Spotify
```

See [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) for component-by-component detail.

## Repo layout

```
briefcast/
├── README.md
├── .env.example            # template for required API keys
├── .gitignore
├── docs/
│   ├── ARCHITECTURE.md     # component-level design
│   └── ROADMAP.md          # phased plan from MVP to app
├── src/                    # pipeline code (to be scaffolded)
├── profile/                # user interest profile + source lists
└── output/                 # generated scripts and audio (gitignored)
```

## Quick start

Not runnable yet — see [`docs/ROADMAP.md`](docs/ROADMAP.md) for what lands when.

Once scaffolded:

```bash
cp .env.example .env        # fill in API keys
uv sync                     # or: pip install -r requirements.txt
python -m briefcast.run     # one-shot run, writes to output/
```

## Required credentials

| Service        | Used for                        | How to get it |
|----------------|---------------------------------|---------------|
| Anthropic      | Script synthesis (Claude)       | console.anthropic.com |
| ElevenLabs     | Text-to-speech (Studio API)     | elevenlabs.io → API |
| xAI / Grok     | Live X summaries                | x.ai/api |
| Google OAuth   | Gmail read-only access          | console.cloud.google.com |
| S3 or R2       | MP3 + RSS hosting               | AWS or Cloudflare |

All keys live in `.env`. Never commit `.env` — only `.env.example`.

## License

Private project. No license granted.
