# Chapter 4: Production Deployment Patterns

**Estimated reading time:** 18 minutes
**What you'll build:** Production-ready Open Notebook deployments on a single machine, a split Ollama+App architecture, Kubernetes manifests, and cloud-specific recipes for AWS, GCP, and Hetzner.
**Prerequisites:** Chapters 1-3 — working local deployment with nginx + TLS.

---

Your Open Notebook works on localhost. This chapter gets it running like infrastructure — auto-restarting, surviving reboots, scaling across machines, and deployable through CI/CD.

## Pattern 1: Single Docker Host (systemd)

```ini
# /etc/systemd/system/open-notebook.service
[Unit]
Description=Open Notebook Docker Compose Stack
Requires=docker.service
After=docker.service network-online.target

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/opt/open-notebook
ExecStart=/usr/bin/docker compose up -d
ExecStop=/usr/bin/docker compose down
ExecReload=/usr/bin/docker compose up -d --force-recreate
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable open-notebook --now
```

Add resource limits and log rotation to docker-compose.yml:

```yaml
services:
  open-notebook:
    deploy:
      resources:
        limits: { cpus: '2.0', memory: 2G }
        reservations: { cpus: '0.5', memory: 512M }
    logging:
      driver: "json-file"
      options: { max-size: "50m", max-file: "3" }
```

## Pattern 2: Split Architecture — Ollama on Separate GPU

```
VPS ($5-10/mo)              GPU Machine (your hardware)
┌──────────────┐            ┌──────────────┐
│ Open Notebook │──Tailscale─→│ Ollama :11434 │
│ SurrealDB     │            └──────────────┘
└──────────────┘
```

Install Tailscale on both machines, then configure:

```yaml
# VPS docker-compose.yml — NO ollama service
environment:
  - OLLAMA_BASE_URL=http://gpu-machine.tailnet.ts.net:11434
```

On GPU machine:

```bash
curl -fsSL https://ollama.com/install.sh | sh
ollama pull gpt-oss:20b
echo 'Environment="OLLAMA_HOST=0.0.0.0:11434"' | \
  sudo tee /etc/systemd/system/ollama.service.d/override.conf
sudo systemctl daemon-reload && sudo systemctl restart ollama
```

## Pattern 3: Kubernetes

For Kubernetes deployments, adapt the official manifests from the Open Notebook GitHub repository to your cluster. Key considerations:

- **Replicas:** 1 (stateful — SurrealDB does not support multi-writer)
- **PersistentVolumeClaim:** For `surreal-data` and `app-data` volumes
- **cert-manager:** For automatic TLS certificate management
- **Ingress:** Route `/api/` to port 5055, everything else to port 8502

See `lfnovo/open-notebook` on GitHub for the latest K8s manifest examples.

## Cloud Recipes

| Provider          | CPU-only             | GPU      | Monthly |
| ----------------- | -------------------- | -------- | ------- |
| Hetzner CX22      | 2 vCPU, 4 GB, 40 GB  | —        | $4.50   |
| Hetzner CPX31     | 4 vCPU, 8 GB, 160 GB | —        | $14.00  |
| Hetzner Auction   | —                    | RTX 3060 | ~$50    |
| Hetzner Auction   | —                    | RTX 4090 | ~$200   |
| AWS g4dn.xlarge   | 4 vCPU, 16 GB        | T4 16 GB | ~$380   |
| AWS t3.medium     | 2 vCPU, 4 GB         | —        | ~$30    |
| GCP g2-standard-4 | 4 vCPU, 16 GB        | L4 24 GB | ~$500   |

## CI/CD (GitHub Actions)

```yaml
# .github/workflows/deploy.yml
on:
  push:
    branches: [main]
    paths: ['docker-compose.yml', 'nginx/**']
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Deploy
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SSH_HOST }}
          script: |
            cd /opt/open-notebook
            docker compose exec -T surrealdb surreal export \
              --conn http://localhost:8000 --user root --pass root \
              --ns open_notebook --db open_notebook \
              > /backups/pre-deploy-$(date +%Y%m%d-%H%M).surql
            docker compose pull
            docker compose up -d --force-recreate
            sleep 15
            curl -sf http://localhost:5055/health || exit 1
```

## Health Check Cron

```cron
# /etc/cron.d/open-notebook-health
*/5 * * * * root curl -sf http://localhost:5055/health || \
  echo "ALERT: Open Notebook Down at $(date)" | \
  mail -s "ALERT" admin@yourdomain.com
```

---

Next: Chapter 5 — Data Management and Backup →
