# Chapter 5: Data Management and Backup

**Estimated reading time:** 14 minutes
**What you'll build:** Automated daily backups with off-site storage, a tested disaster recovery procedure, migration scripts, and data pruning strategies.
**Prerequisites:** Chapter 1 — working deployment.

---

Open Notebook's data lives in three places. You need to back up two of them.

| Path            | Content                                        | Backup?     |
| --------------- | ---------------------------------------------- | ----------- |
| ./surreal-data/ | All notebooks, notes, chat history, embeddings | 🔴 MUST     |
| ./app-data/     | Raw uploaded files, processed chunks           | 🟡 SHOULD   |
| ./app-config/   | UI preferences                                 | 🟢 OPTIONAL |
| ./ollama-data/  | Model weights (huge, reproducible)             | ❌ SKIP      |

## Automated Backup Script

Create `/opt/open-notebook/backup.sh`:

```bash
#!/bin/bash
set -euo pipefail
BACKUP_ROOT="/backups/open-notebook"
TIMESTAMP=$(date +%Y%m%d-%H%M)
BACKUP_DIR="${BACKUP_ROOT}/${TIMESTAMP}"
RETENTION_DAYS=30

mkdir -p "$BACKUP_DIR"
cd /opt/open-notebook

# Stop app for consistent DB snapshot (brief downtime)
docker compose stop open-notebook
sleep 2

# Export SurrealDB
docker compose exec -T surrealdb surreal export \
    --conn http://localhost:8000 --user root --pass root \
    --ns open_notebook --db open_notebook \
    > "${BACKUP_DIR}/database-export.surql"

# Verify export
if grep -q "DEFINE TABLE" "${BACKUP_DIR}/database-export.surql"; then
    RECORD_COUNT=$(grep -c "CREATE" "${BACKUP_DIR}/database-export.surql" || echo "0")
    echo "Export valid: ${RECORD_COUNT} records"
else
    echo "ERROR: Export failed" && docker compose start open-notebook && exit 1
fi

# Copy app data
tar -czf "${BACKUP_DIR}/app-data.tar.gz" ./app-data/ 2>/dev/null || true

# Restart
docker compose start open-notebook

# Compress
ARCHIVE_NAME="open-notebook-backup-${TIMESTAMP}.tar.gz"
tar -czf "${BACKUP_ROOT}/${ARCHIVE_NAME}" -C "$BACKUP_DIR" .

# Optional: Encrypt with GPG
if command -v gpg &> /dev/null && [ -n "${GPG_RECIPIENT:-}" ]; then
    gpg --encrypt --recipient "$GPG_RECIPIENT" \
        --output "${BACKUP_ROOT}/${ARCHIVE_NAME}.gpg" \
        "${BACKUP_ROOT}/${ARCHIVE_NAME}"
    mv "${BACKUP_ROOT}/${ARCHIVE_NAME}.gpg" "${BACKUP_ROOT}/${ARCHIVE_NAME}"
fi

# Upload off-site (rclone)
if command -v rclone &> /dev/null && [ -n "${RCLONE_REMOTE:-}" ]; then
    rclone copy "${BACKUP_ROOT}/${ARCHIVE_NAME}" "${RCLONE_REMOTE}:/open-notebook-backups/"
fi

# Clean old backups
find "$BACKUP_ROOT" -name "open-notebook-backup-*.tar.gz*" -mtime +${RETENTION_DAYS} -delete
find "$BACKUP_ROOT" -maxdepth 1 -type d -name "202*" -mtime +${RETENTION_DAYS} -exec rm -rf {} \;

FINAL_SIZE=$(du -h "${BACKUP_ROOT}/${ARCHIVE_NAME}" | cut -f1)
echo "Backup complete: ${ARCHIVE_NAME} (${FINAL_SIZE})"
rm -rf "$BACKUP_DIR"
```

Schedule:

```bash
chmod +x /opt/open-notebook/backup.sh
echo "0 3 * * * root /opt/open-notebook/backup.sh" | \
    sudo tee /etc/cron.d/open-notebook-backup
```

> **Note:** The backup script stops the app for ~5-10 seconds to get a consistent database snapshot. Schedule it during low-usage hours.

## Off-Site Storage (rclone)

- **Backblaze B2:** `rclone config` → b2 → 10 GB free tier (~3 GB for 30 daily backups)
- **AWS S3:** `rclone config` → s3 → Standard tier
- **Google Drive:** `rclone config` → drive → 15 GB free

Set `export RCLONE_REMOTE="b2"` (or s3/drive) in the backup script.

## Disaster Recovery: Full Restore

```bash
# Fresh server with Docker installed
cd /opt/open-notebook
tar -xzf /path/to/backup.tar.gz -C /tmp/restore/

# Fresh deploy
curl -O https://raw.githubusercontent.com/lfnovo/open-notebook/main/docker-compose.yml
sed -i 's/change-me-to-a-secret-string/YOUR-ORIGINAL-SECRET-KEY/' docker-compose.yml
docker compose up -d surrealdb
sleep 10

# Restore database
cat /tmp/restore/database-export.surql | docker compose exec -T surrealdb \
    surreal import --conn http://localhost:8000 \
    --user root --pass root --ns open_notebook --db open_notebook

# Restore files
tar -xzf /tmp/restore/app-data.tar.gz -C ./

# Start and verify
docker compose up -d
curl http://localhost:5055/health
```

> **Test your restore procedure monthly.** An untested backup is a prayer.

## Migration: Moving Servers

```bash
# Old server
docker compose stop
tar -czf /tmp/migration.tar.gz ./surreal-data/ ./app-data/ ./app-config/ docker-compose.yml
scp /tmp/migration.tar.gz user@new-server:/tmp/

# New server
cd /opt/open-notebook && tar -xzf /tmp/migration.tar.gz && docker compose up -d
```

## Data Pruning & Storage Growth

| Usage             | Monthly growth |
| ----------------- | -------------- |
| Light (1-2 users) | 1-2 GB         |
| Medium (5 users)  | 5-10 GB        |
| Heavy (20+ users) | 30-50 GB       |

```sql
-- Prune old sources
DELETE FROM source WHERE created_at < time::now() - 6w;
```

Export old notebooks to JSON before deleting for cold storage.

## Monthly Maintenance Script

```bash
#!/bin/bash
docker compose exec surrealdb surreal sql --conn http://localhost:8000 \
    --user root --pass root --ns open_notebook --db open_notebook \
    --query "
    SELECT 'notebooks', count() FROM notebook GROUP ALL;
    SELECT 'sources', count() FROM source GROUP ALL;
    "
du -sh /opt/open-notebook/surreal-data/ /opt/open-notebook/app-data/
```

---

Next: Chapter 6 — Multi-User and Team Setup →
