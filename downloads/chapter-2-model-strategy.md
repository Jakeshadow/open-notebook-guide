# Chapter 2: Model Strategy — Choosing and Optimizing Your AI Stack

**Estimated reading time:** 18 minutes
**What you'll build:** A complete model configuration optimized for your budget and use case, with per-task model assignments, cost projections, and Ollama tuning.
**Prerequisites:** Chapter 1 — working Docker deployment with at least one provider configured.

---

Open Notebook's killer feature isn't any single model. It's that you can route different tasks through different models. This chapter picks the right models for each task, at your budget.

## The Three Tasks and What They Need

Open Notebook does three fundamentally different AI tasks. Each needs a different kind of model:

| Task            | What it does                                                              | What matters                                        | Ideal model traits           |
| --------------- | ------------------------------------------------------------------------- | --------------------------------------------------- | ---------------------------- |
| Chat / Analysis | Answers questions about your documents, generates summaries, writes notes | Reasoning quality, context length, factual accuracy | Large, smart models          |
| Embedding       | Converts document text into vectors for semantic search                   | Retrieval accuracy, speed, batch throughput         | Specialized embedding models |
| Podcast TTS     | Generates spoken audio for multi-speaker podcasts                         | Voice quality, naturalness, speaker consistency     | TTS-only models (not LLMs)   |

Using a $15/million-token model for embeddings is burning money. Using a $0.02/million-token model for complex document analysis gives garbage answers. Route intentionally.

## Chat/Analysis Models — Picking Your Brain

### Cloud Options (No GPU Required)

| Model             | Provider  | Cost per 1M input tokens | Cost per 1M output tokens | Best for                                       |
| ----------------- | --------- | ------------------------ | ------------------------- | ---------------------------------------------- |
| Claude 3.5 Sonnet | Anthropic | $3.00                    | $15.00                    | Deep analysis, long docs, complex reasoning    |
| GPT-4o            | OpenAI    | $2.50                    | $10.00                    | Best general purpose, strong at everything     |
| GPT-4o-mini       | OpenAI    | $0.15                    | $0.60                     | Budget option, surprisingly good for summaries |
| Gemini 2.0 Flash  | Google    | $0.10                    | $0.40                     | Best price/performance, free tier robust       |
| Llama 3.3 70B     | Groq      | $0.59                    | $0.79                     | Fast, open-source quality, 15K RPM free tier   |
| DeepSeek V3       | DeepSeek  | $0.27                    | $1.10                     | Strong coding/reasoning, very cheap input      |

### Local Options (GPU Required)

| Model         | VRAM needed | Quality equivalent    | Best for                        |
| ------------- | ----------- | --------------------- | ------------------------------- |
| Llama 3.2 3B  | 4 GB        | GPT-3.5-ish           | Quick summaries, testing        |
| Llama 3.1 8B  | 6 GB        | Between GPT-3.5 and 4 | Good all-rounder for 12 GB      |
| GPT-OSS 20B   | 8-10 GB     | Close to GPT-4o-mini  | Best local quality for 12-16 GB |
| Qwen 2.5 32B  | 16-20 GB    | Approaching GPT-4o    | Best local quality for 24 GB    |
| Llama 3.3 70B | 40+ GB      | GPT-4o class          | Dual GPU or Mac Studio          |

### My Recommended Chat Configs

- **$0/month (RTX 3060 12GB):** Primary: `gpt-oss:20b` (fits with `num_ctx=4096`). Fallback: `llama3.2:3b`.
- **$15/month:** Groq `llama-3.3-70b` (free tier). Heavy queries: GPT-4o-mini.
- **$50/month:** GPT-4o for analysis. GPT-4o-mini for summaries. Ollama for sensitive docs.
- **$150/month:** Claude 3.5 Sonnet for analysis, GPT-4o-mini for summaries, Ollama `qwen2.5:32b` on RTX 4090.

## Embedding Models — The Engine Nobody Talks About

Bad embedding = right documents not found = AI gives wrong answers even with excellent chat model.

### Cloud Embedding

| Model                  | Provider  | Cost per 1M tokens | Dimensions    | Notes                    |
| ---------------------- | --------- | ------------------ | ------------- | ------------------------ |
| text-embedding-3-small | OpenAI    | $0.02              | 512/1536      | Default recommendation   |
| text-embedding-3-large | OpenAI    | $0.13              | 256/1024/3072 | When quality matters     |
| voyage-3-lite          | Voyage AI | $0.02              | 512           | Good with technical text |
| voyage-3               | Voyage AI | $0.06              | 1024          | Better for code/tech     |

Cost reality: 1000 PDFs (~50M tokens) ≈ $1.00 one-time.

### Local Embedding (Ollama)

| Model             | VRAM | Quality                       |
| ----------------- | ---- | ----------------------------- |
| nomic-embed-text  | 1 GB | Good                          |
| mxbai-embed-large | 2 GB | Better for tech/scientific    |
| bge-m3            | 3 GB | Multilingual, highest quality |

> **⚠️ Esperanto library bug (Issue #655):** Ollama embeddings may fail with "Connection error" even when chat works. Workaround: use OpenAI `text-embedding-3-small` ($0.02/1M) and keep Ollama for chat only.

## Podcast TTS Models

| ElevenLabs Model | Cost per 1K chars | Quality                |
| ---------------- | ----------------- | ---------------------- |
| Multilingual v2  | ~$0.30            | Professional           |
| Turbo v2.5       | ~$0.05            | Very good (sweet spot) |
| Flash v2.5       | ~$0.01            | Acceptable, fast       |

15-min podcast (~20K chars): Multilingual ≈ $6.00, Turbo ≈ $1.00, Flash ≈ $0.20.

## The Model Routing Pattern

```
Chat (analysis)     → GPT-4o or Claude     (expensive, high quality)
Title generation    → GPT-4o-mini          (cheap, simple)
Summary generation  → GPT-4o-mini          (cheap, frequent)
Embedding           → text-embed-3-small   (cheapest, specialized)
Podcast TTS         → ElevenLabs Turbo     (pay per podcast)
Sensitive chat      → Ollama gpt-oss:20b   (local, zero-cost)
```

## Cost Projections

- **Solo:** ~$4/month (Groq free + ElevenLabs Turbo)
- **Small team** (500 queries, 200 docs): ~$16/month (GPT-4o-mini + Turbo + $5 VPS)
- **Research team** (2000 queries, 1000 docs): ~$90/month (GPT-4o + mini + Turbo + GPU server)

## Ollama Optimization

```bash
# 1. Context window by VRAM (num_ctx in Settings):
# 8 GB → 2048-4096 | 12 GB → 4096-8192
# 16 GB → 8192-16384 | 24 GB → 16384-32768

# 2. Check loaded models and VRAM:
curl http://localhost:11434/api/ps

# 3. Unload unused models:
curl http://localhost:11434/api/generate -d '{
  "model": "gpt-oss:20b", "keep_alive": 0
}'

# 4. Multi-GPU:
environment:
  - CUDA_VISIBLE_DEVICES=0,1
```

## Provider Gotchas

- **Groq:** 15K RPM free tier but 30 RPM rate limit. Chat only, NOT batch embedding.
- **Anthropic:** Rate limits per-org, not per-key. Teams need separate keys.
- **OpenAI:** Tier 1 = 500 RPM. Batch embeddings 50 at a time.
- **ElevenLabs:** Free tier = 10K chars (~7 min podcast). Starter $5/mo = 30K. Creator $22/mo = 100K.
- **DeepSeek:** Short 8K context. Unsuitable for long-document chat.

---

Next: Chapter 3 — Reverse Proxy, TLS, and Authentication →
