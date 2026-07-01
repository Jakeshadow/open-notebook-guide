# Chapter 3: Reverse Proxy, TLS, and Authentication

**Estimated reading time:** 15 minutes
**What you'll build:** A production-grade nginx reverse proxy with Let's Encrypt TLS, firewall hardening, and multi-layer authentication protecting your Open Notebook instance.
**Prerequisites:** Chapter 1 — working local deployment. A domain name pointing to your server's IP.

---

Right now your Open Notebook runs on `http://localhost:8502`. It's accessible only from your machine. This chapter makes it accessible from the internet — securely.

> **⚠️ Do not skip TLS.** Exposing Open Notebook over plain HTTP means API keys, document contents, and chat history travel in cleartext. Anyone on the same network can sniff them.

## Complete nginx Configuration

Install nginx and certbot:

```bash
sudo apt update && sudo apt install -y nginx certbot python3-certbot-nginx
```

Create `/etc/nginx/sites-available/open-notebook`:

```nginx
# ── HTTP → HTTPS Redirect ──
server {
    listen 80;
    server_name notebook.yourdomain.com;
    location /.well-known/acme-challenge/ {
        root /var/www/html;
    }
    location / {
        return 301 https://$host$request_uri;
    }
}

# ── HTTPS Main Server ──
server {
    listen 443 ssl http2;
    server_name notebook.yourdomain.com;

    ssl_certificate     /etc/letsencrypt/live/notebook.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/notebook.yourdomain.com/privkey.pem;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
    ssl_prefer_server_ciphers on;
    ssl_session_cache   shared:SSL:10m;
    ssl_session_timeout 10m;

    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    client_max_body_size 500M;  # For large PDFs

    # ── Web UI with WebSocket support ──
    location / {
        proxy_pass http://127.0.0.1:8502;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;      # WebSocket
        proxy_set_header Connection "upgrade";        # WebSocket
        proxy_read_timeout 86400s;
    }

    # ── REST API ──
    location /api/ {
        rewrite ^/api/(.*) /$1 break;
        proxy_pass http://127.0.0.1:5055;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_http_version 1.1;
        proxy_read_timeout 300s;
    }

    # ── Health check (no auth) ──
    location /health {
        proxy_pass http://127.0.0.1:5055/health;
        access_log off;
    }
}
```

Enable site, get TLS:

```bash
sudo ln -s /etc/nginx/sites-available/open-notebook /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default
sudo nginx -t && sudo systemctl reload nginx
sudo certbot --nginx -d notebook.yourdomain.com
sudo certbot renew --dry-run  # Verify auto-renewal
```

Update docker-compose.yml:

```yaml
environment:
  - API_URL=https://notebook.yourdomain.com
```

## Firewall Hardening

```bash
sudo apt install -y ufw
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
```

> **⚠️ Do NOT open ports 8502, 5055, 8000, or 11434.** Users access through nginx on 443 only.

## Authentication — Three Layers

### Layer 1: Instance Password

```yaml
environment:
  - OPEN_NOTEBOOK_PASSWORD=your-strong-password
```

### Layer 2: nginx Basic Auth

```nginx
auth_basic "Open Notebook — Authentication Required";
auth_basic_user_file /etc/nginx/.htpasswd;
```

```bash
sudo apt install -y apache2-utils
sudo htpasswd -c /etc/nginx/.htpasswd alice
sudo htpasswd /etc/nginx/.htpasswd bob
sudo systemctl reload nginx
```

### Layer 3: OAuth2 Proxy (Google/GitHub)

```bash
sudo apt install -y oauth2-proxy
```

`/etc/oauth2-proxy/oauth2-proxy.cfg`:

```ini
provider = "google"
email_domains = ["yourcompany.com"]
upstreams = ["http://127.0.0.1:8502"]
```

Then point nginx to `http://127.0.0.1:4180` instead of `:8502`.

**Cloudflare Access** (easiest for Cloudflare users): Zero software install. Add application in Cloudflare Zero Trust, restrict by email domain. Free for ≤50 users.

## Rate Limiting

```nginx
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;
limit_req_zone $binary_remote_addr zone=web_limit:10m rate=30r/s;

# In location /api/:  limit_req zone=api_limit burst=20 nodelay;
# In location /:      limit_req zone=web_limit burst=50 nodelay;
```

## fail2ban Brute-Force Protection

```bash
sudo apt install -y fail2ban
sudo tee /etc/fail2ban/filter.d/open-notebook.conf << 'EOF'
[Definition]
failregex = ^<HOST> -.*"(GET|POST).*/api/.*" 401
            ^<HOST> -.*"(GET|POST).*/api/.*" 403
ignoreregex =
EOF
sudo tee /etc/fail2ban/jail.d/open-notebook.conf << 'EOF'
[open-notebook]
enabled = true
filter = open-notebook
logpath = /var/log/nginx/access.log
maxretry = 10
bantime = 3600
findtime = 600
EOF
sudo systemctl restart fail2ban
```

## Security Verification

```bash
curl -I https://notebook.yourdomain.com | grep -E "HTTP|Strict-Transport"
curl -I http://notebook.yourdomain.com | grep "301"
nmap -p 8502,5055,8000,11434 your-server-ip  # Should all be filtered/closed
curl -I https://notebook.yourdomain.com | grep -E "X-Frame|X-Content|X-XSS"
```

Your Open Notebook is now accessible from anywhere, with valid TLS, firewall, authentication, rate limiting, and brute-force protection. In Chapter 4, we scale for production workloads.

---

Next: Chapter 4 — Production Deployment Patterns →
