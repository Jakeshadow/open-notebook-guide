# Chapter 0: Is Open Notebook Right for You?

**Estimated reading time:** 7 minutes  
**What you'll build:** A working mental model of what Open Notebook is, who should self-host it, and whether it's worth your time.  
**Prerequisites:** None — this is the decision chapter.

---

Open Notebook is an open-source, self-hosted alternative to Google NotebookLM. It ingests documents (PDFs, URLs, YouTube transcripts, audio), indexes them with embeddings, and lets you chat with your content — plus generate multi-speaker AI podcasts. It supports 18+ AI providers instead of just Gemini, has no 50-source limit, and can run entirely offline.

The question isn't whether it's good. The question is whether you should deploy it.

## The 2-Minute Test

Paste this into a terminal:

```bash
mkdir open-notebook && cd open-notebook
curl -O https://raw.githubusercontent.com/lfnovo/open-notebook/main/docker-compose.yml
# Change the encryption key to your own secret string
sed -i 's/change-me-to-a-secret-string/a-random-string-you-pick-now/' docker-compose.yml
docker compose up -d
```

Open http://localhost:8502. On first run, Docker downloads the images (~30 seconds to 2 minutes depending on your connection). Once the interface appears, you've deployed Open Notebook. You can stop reading here and go play with it — or keep reading to understand whether you should invest more time.

## The Honest Comparison

### Open Notebook vs NotebookLM

| Dimension          | Open Notebook                                                        | NotebookLM                                      |
| ------------------ | -------------------------------------------------------------------- | ----------------------------------------------- |
| Data privacy       | Full local mode (Ollama). Cloud APIs see query text only.            | Everything on Google's servers.                 |
| Setup time         | 2 min Docker (cloud API). 15-30 min (Ollama). 1-2 hrs (production).  | 0 min. Sign in.                                 |
| AI providers       | 18+ (OpenAI, Claude, Ollama, Groq, DeepSeek, ElevenLabs...)          | Gemini only.                                    |
| Source limits      | No hard cap. Storage + context window only.                          | 50 sources per notebook. 500K words per source. |
| Podcast generation | 1-4 speakers. Custom voice profiles. Editable prompts.               | 2 speakers. "Live host" mode.                   |
| API access         | Full REST API on port 5055.                                          | None.                                           |
| Citations          | Basic (acknowledged weakness).                                       | Excellent. Exact passages highlighted.          |
| Cost               | $0-$30/user/month (API) or $0 (Ollama) + infra.                      | Free tier. Plus: ~$20/month/user.               |
| Offline            | Yes, with Ollama.                                                    | No.                                             |

### Open Notebook vs AnythingLLM

AnythingLLM (also open source) is similar — but Open Notebook's podcast/audio overview feature is the differentiator. If you don't need podcast generation, AnythingLLM has a more polished UI and easier setup. If you do need podcasts, Open Notebook is the clear winner.

### Open Notebook vs PrivateGPT

PrivateGPT focuses on document Q&A with guaranteed local processing. It's lighter but has no podcast feature, no multi-model routing, and a more basic UI. Open Notebook is the full research workspace; PrivateGPT is the focused document chatbot.

## Who Should Self-Host

**Do it if:**

- You handle sensitive documents (legal, medical, proprietary code) and can't upload them to Google
- You want API access to automate document processing in your workflow
- You've hit NotebookLM's 50-source cap and still have 100 more papers to ingest
- You need podcasts with more than 2 speakers or custom voice profiles
- You want to use Claude or GPT-4o for analysis quality that Gemini can't match
- You're building a product and need a white-label document AI backend

**Skip it if:**

- You use NotebookLM casually and have no compliance requirements
- You don't want to spend 1-2 hours on initial setup and ongoing maintenance
- You need production-quality citations (Open Notebook's citations are basic)
- You just need a chatbot for a few PDFs (use ChatGPT file upload or Claude Projects)

## What You'll Build by the End of This Guide

```
                         ┌──────────────────┐
                         │   Your Browser    │
                         │  https://nb.your- │
                         │  domain.com       │
                         └────────┬─────────┘
                                  │ HTTPS
                         ┌────────▼─────────┐
                         │  nginx / Traefik  │
                         │  (TLS + auth)     │
                         └───┬─────────┬─────┘
                             │         │
                    ┌────────▼──┐  ┌──▼──────────┐
                    │  Open      │  │  Ollama      │
                    │  Notebook  │  │  (GPU)       │
                    │  :8502     │  │  :11434      │
                    └───┬────────┘  └──────────────┘
                        │
               ┌────────▼──────────┐
               │  SurrealDB :8000  │
               │  (or external)    │
               └───────────────────┘
```

A production deployment with TLS encryption, authentication, automated backups, and monitoring — serving multiple users on your team.

## The $0 vs $30 vs $150 Setup

|             | $0/month                     | $30/month                     | $150/month                |
| ----------- | ---------------------------- | ----------------------------- | ------------------------- |
| Chat model  | Ollama (gpt-oss:20b)         | Groq (llama-3.3-70b)          | Claude Sonnet             |
| Embedding   | Ollama (nomic-embed-text)    | OpenAI (text-embedding-3-small) | Same                    |
| Podcast TTS | Offline (no podcasts)        | ElevenLabs                    | ElevenLabs                |
| Good for    | Solo privacy use             | Solo/2-person, good quality   | Full team, best quality   |
| GPU needed  | Yes (12GB+ VRAM)             | No                            | No                        |

*Note: Groq offers a generous free tier — most solo users won't hit $30/month under normal usage. The $30 figure is a conservative ceiling.*

---

*Next: Chapter 1 — Docker Deep Dive: From docker compose up to a Solid Foundation →*
