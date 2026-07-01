# Chapter 1: Docker Deep Dive — From docker compose up to a Solid Foundation

**Estimated reading time:** 15 minutes  
**What you'll build:** A fully understood Docker deployment with every service, env var, and volume mapped — plus a working checklist and common first-run fixes.  
**Prerequisites:** Docker and Docker Compose installed on your machine. (If not: `curl -fsSL https://get.docker.com | sh`)

---

Chapter 0 gave you a 2-minute quickstart. This chapter explains every single line you just ran, so when something breaks at 2 AM, you know exactly where to look.

## The Full docker-compose.yml — Line by Line

Here's the actual compose file from lfnovo/open-notebook, annotated:

```yaml
services:
  # ── SERVICE 1: SurrealDB (Database) ──
  surrealdb:
    image: surrealdb/surrealdb:v2
    command: start --log info \
      --user ${SURREAL_USER:-root} \
      --pass ${SURREAL_PASSWORD:-root} \
      rocksdb:/mydata/mydatabase.db
    user: root  # Required for bind mounts on Linux
    ports:
      - "8000:8000"
    volumes:
      - ./surreal_data:/mydata     # All notebooks, notes, chat history
    environment:
      - SURREAL_EXPERIMENTAL_GRAPHQL=true
    restart: always
    pull_policy: always

  # ── SERVICE 2: Open Notebook App ──
  open_notebook:
    image: lfnovo/open_notebook:v1-latest
    ports:
      - "8502:8502"  # Web UI — the only port you browse to
      - "5055:5055"  # REST API — used by automation scripts
    volumes:
      - ./notebook_data:/app/data   # Uploaded files, processed docs
    environment:
      # REQUIRED: Change this to your own secret string
      # This encrypts your API keys in the database
      - OPEN_NOTEBOOK_ENCRYPTION_KEY=change-me-to-a-secret-string

      # Database connection — defaults to root:root for local use
      # Override via .env file before exposing to a network
      - SURREAL_URL=ws://surrealdb:8000/rpc
      - SURREAL_USER=${SURREAL_USER:-root}
      - SURREAL_PASSWORD=${SURREAL_PASSWORD:-root}
      - SURREAL_NAMESPACE=open_notebook
      - SURREAL_DATABASE=open_notebook
    depends_on:
      - surrealdb
    restart: always
    pull_policy: always
```

**Key things to notice vs what you'll see in random blog posts:**

- No `version:` key — Docker Compose V2 doesn't need it.
- DB service starts *first* (`depends_on`), but the app retries internally on startup so order isn't brittle.
- `user: root` on SurrealDB is intentional — Linux bind mounts default to root ownership.
- No AI provider keys here. Configure them via the UI at **Settings → API Keys** instead. (The old env-var approach is deprecated as of v1.8+.)

## What Each Volume Actually Stores

| Path               | Contents                                                                  | Backup Priority |
| ------------------ | ------------------------------------------------------------------------- | --------------- |
| `./surreal_data/`  | Every notebook, note, chat message, source metadata, embedding references | 🔴 CRITICAL     |
| `./notebook_data/` | Raw uploaded files (PDFs, audio), processed chunks, cached thumbnails     | 🟡 HIGH         |

You can delete `./notebook_data/` and re-upload documents without losing your research structure. You absolutely cannot delete `./surreal_data/`.

## The Networking Model

Inside Docker's default bridge network, services reach each other by container name:

```
open_notebook → surrealdb    via ws://surrealdb:8000/rpc    ← Docker DNS
open_notebook → ollama       via http://ollama:11434         ← Docker DNS (if added)
your browser  → open_notebook via http://localhost:8502      ← Host port mapping
```

⚠️ **This is the #1 source of first-time errors.** When adding Ollama in the Settings UI, use `http://ollama:11434` — not `http://localhost:11434`. Inside the container, `localhost` points to the container itself, not your host machine.

The env var for Ollama is `OLLAMA_BASE_URL` (note: not `OLLAMA_API_BASE_URL`). You can set it in your `.env` file or via the UI.

## Adding Ollama to the Stack

Append this to your docker-compose.yml under `services:`:

```yaml
  ollama:
    image: ollama/ollama:latest
    container_name: open-notebook-ollama
    ports:
      - "11434:11434"
    volumes:
      - ./ollama_data:/root/.ollama
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    restart: always
    pull_policy: always
```

If you're on CPU-only or Apple Silicon, remove the entire `deploy.resources` block. Ollama will use CPU inference (slower but functional).

After adding it:

```bash
docker compose up -d ollama
docker exec -it open-notebook-ollama ollama pull gpt-oss:20b
docker exec -it open-notebook-ollama ollama pull nomic-embed-text
```

## The First-Run Checklist

Before you upload your first document, verify these 7 things:

```bash
# 1. All services running?
docker compose ps
# Expected: 2-3 services all "Up"

# 2. Web UI accessible?
curl -I http://localhost:8502
# Expected: HTTP 200

# 3. API responding?
curl http://localhost:5055/api/health
# Expected: {"status": "ok"}

# 4. Database connected?
docker compose logs surrealdb | grep -i "started"
# Expected: confirmation line from SurrealDB

# 5. Ollama model loaded (if using Ollama)?
curl http://localhost:11434/api/tags
# Expected: JSON list including your pulled models

# 6. ENCRYPTION_KEY changed from default?
docker compose exec open_notebook env | grep ENCRYPTION_KEY
# Expected: NOT "change-me-to-a-secret-string"

# 7. Disk space adequate?
df -h ./surreal_data ./notebook_data ./ollama_data
# Each needs at least 2 GB free for growth
```

## Five Common First-Run Errors (And Their Exact Fixes)

### Error 1: `docker: Error response from daemon: Ports are not available`

```bash
# Check what's using port 8502
sudo lsof -i :8502
# If another service — change the host port:
# In docker-compose.yml: "8503:8502" and access on :8503
```

### Error 2: `PermissionError: [Errno 13] Permission denied` (SurrealDB)

The SurrealDB container runs as root by default (`user: root` in compose), which is correct for Linux bind mounts. If you've changed this or your host has strict permissions:

```bash
sudo chown -R 1000:1000 ./surreal_data
# Or leave as-is if running the default root config
```

### Error 3: Ollama embedding — `Connection error`, but chat works fine

This is the Esperanto library bug (GitHub Issue #655). The embedding client may ignore the base URL and connect to a hardcoded default. The fix:

In **Settings → API Keys → Ollama**, verify the Base URL is set correctly (`http://ollama:11434`). If the embedding-specific URL field is available, set it explicitly — don't rely on it inheriting from the LLM setting.

If that doesn't work, use a cloud embedding provider as a workaround (OpenAI `text-embedding-3-small` or Voyage AI). It costs < $0.01 per 1000 documents and avoids the bug entirely.

### Error 4: `RuntimeError: CUDA out of memory`

You're running too large a model for your GPU. Quick fixes:

```bash
# Check what's using GPU memory
nvidia-smi

# Pull a smaller model
docker exec open-notebook-ollama ollama pull llama3.2:3b
```

For context window sizing in the Settings UI, use these safe starting points:

| GPU VRAM | Model size | Recommended num_ctx |
| -------- | ---------- | ------------------- |
| 12 GB    | 20B        | 2048 (model fills most VRAM) |
| 24 GB    | 20B        | 8192 |
| 12 GB    | 8B         | 8192 (recommended — model uses ~5 GB) |
| 8 GB     | 8B         | 4096 |

### Error 5: `WebSocket connection to 'ws://surrealdb:8000/rpc' failed`

The app container can't reach SurrealDB:

```bash
# From inside the app container, test DB reachability
docker compose exec open_notebook sh -c "wget -qO- http://surrealdb:8000/health"

# If it fails, check both are on the same network
docker network ls | grep open-notebook
docker compose ps  # Verify both services are Up
```

## SurrealDB: What You Need to Know

SurrealDB is a multi-model database that Open Notebook uses differently from PostgreSQL:

- **Namespace** = `open_notebook` — top-level isolation unit
- **Database** = `open_notebook` — within that namespace
- **Tables** created automatically — `notebook`, `note`, `source`, `chat`, `user_config`

### Exploring the database

```bash
# Connect from host
docker compose exec surrealdb surreal sql \
  --conn http://localhost:8000 \
  --user root --pass root \
  --ns open_notebook --db open_notebook

# See all notebooks:
SELECT * FROM notebook;

# See all sources (uploaded documents):
SELECT id, title, type, status FROM source;

# Count notes per notebook:
SELECT notebook, count() FROM note GROUP BY notebook;
```

### Quick backup/restore

```bash
# Backup
docker compose exec surrealdb surreal export \
  --conn http://localhost:8000 --user root --pass root \
  --ns open_notebook --db open_notebook \
  > backup-$(date +%Y%m%d).surql

# Restore
cat backup-20260625.surql | docker compose exec -T surrealdb surreal import \
  --conn http://localhost:8000 --user root --pass root \
  --ns open_notebook --db open_notebook
```

---

Your instance is running. You have a database with persistent storage and auto-restart. You can now upload documents, chat with them, and generate podcasts on localhost. In Chapter 2, we'll pick the right models for your workload and budget.

---

*Next: Chapter 2 — Model Strategy: Choosing and Optimizing Your AI Stack →*
