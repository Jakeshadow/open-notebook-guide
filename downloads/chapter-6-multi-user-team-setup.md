# Chapter 6: Multi-User and Team Setup

**Estimated reading time:** 13 minutes
**What you'll build:** A multi-user Open Notebook instance with workspace isolation, usage monitoring, cost allocation per user, and admin controls.
**Prerequisites:** Chapters 1-3 — working deployment with authentication.

---

Open Notebook v1.x treats all users as equal citizens. No built-in RBAC. This chapter layers isolation, monitoring, and cost tracking on top.

## Pattern 1: Separate Instances Per Team

Run multiple stacks on different ports:

```
/opt/open-notebook/team-alpha/   → :8502
/opt/open-notebook/team-beta/    → :8503
/opt/open-notebook/team-legal/   → :8504  # Ollama-only, zero cloud
```

Each team uses their own API keys → billing goes to their department. nginx routes by subdomain. Complete isolation, simple cost attribution.

## Pattern 2: OAuth2 Proxy with Per-User Identity

```nginx
location / {
    auth_request /oauth2/auth;
    error_page 401 = /oauth2/sign_in;
    auth_request_set $user $upstream_http_x_auth_request_user;
    proxy_set_header X-Forwarded-User $user;
    proxy_set_header X-Forwarded-Email $upstream_http_x_auth_request_email;
    proxy_pass http://127.0.0.1:8502;
}
```

User logs in via Google → email becomes user identifier → each person sees only their notebooks.

## Cost Allocation Per User

```bash
# Extract per-user API usage from nginx logs
grep "$(date +%d/%b/%Y)" /var/log/nginx/access.log | \
    grep "/api/" | awk '{print $1}' | sort | uniq -c | sort -rn

# With OAuth2 Proxy:
grep "X-Forwarded-Email" /var/log/nginx/access.log | \
    awk -F'X-Forwarded-Email: ' '{print $2}' | awk '{print $1}' | \
    sort | uniq -c | sort -rn
```

## Resource Allocation

Limit Ollama concurrency:

```yaml
environment:
  - OLLAMA_NUM_PARALLEL=2
  - OLLAMA_MAX_LOADED_MODELS=1
```

Split GPU: chat vs podcast on separate GPUs:

```yaml
ollama-chat:
    environment: { CUDA_VISIBLE_DEVICES: 0 }
    ports: ["11434:11434"]
ollama-podcast:
    environment: { CUDA_VISIBLE_DEVICES: 1 }
    ports: ["11435:11434"]
```

Rate limit podcasts in nginx:

```nginx
location /api/podcast/ {
    limit_req zone=podcast_limit burst=1 nodelay;
    proxy_pass http://127.0.0.1:5055;
}
```

## Usage Report Script

```bash
#!/bin/bash
echo "=== Usage Report — $(date) ==="
docker compose exec -T surrealdb surreal sql --conn http://localhost:8000 \
    --user root --pass root --ns open_notebook --db open_notebook \
    --query "
    SELECT count() AS notebooks FROM notebook GROUP ALL;
    SELECT count() AS sources FROM source GROUP ALL;
    SELECT type, count() FROM source WHERE created_at > time::now() - 1d GROUP BY type;
    "
du -sh /opt/open-notebook/surreal-data/ /opt/open-notebook/app-data/
nvidia-smi --query-gpu=utilization.gpu,memory.used,memory.total --format=csv,noheader 2>/dev/null
```

## Offboarding

```bash
# Export first, then delete
docker compose exec surrealdb surreal sql ... \
    --query "SELECT * FROM notebook WHERE user = 'departing@company.com'" \
    > /exports/departing-$(date +%Y%m%d).json

# Delete their data
docker compose exec surrealdb surreal sql ... \
    --query "
    DELETE FROM note WHERE notebook.user = 'departing@company.com';
    DELETE FROM source WHERE notebook.user = 'departing@company.com';
    DELETE FROM notebook WHERE user = 'departing@company.com';
    "

sudo htpasswd -D /etc/nginx/.htpasswd departing@company.com
sudo systemctl reload nginx
```

## Alert Thresholds

| Metric         | Warning | Critical      |
| -------------- | ------- | ------------- |
| Disk usage     | >70%    | >85%          |
| GPU memory     | >80%    | >95%          |
| API error rate | >5%     | >10%          |
| Backup age     | >25 hrs | >30 hrs       |
| Health check   | 1 fail  | 3 consecutive |

---

Next: Chapter 7 — API-Driven Workflows and Automation →
