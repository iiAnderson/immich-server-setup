# Immich Server Setup

Step-by-step guide for deploying a self-hosted Immich instance on an Intel NUC (or similar small server) running Ubuntu 24.04, with nightly S3 backups, push notifications via ntfy.sh, and documented restore procedures.

---

## Overview

| Item | Decision |
|---|---|
| Host | Intel NUC (or similar), 16 GB RAM, 1 TB NVMe, Ubuntu 24.04 |
| Container runtime | Docker CE (official apt repo) |
| Immich data path | `/srv/immich/library` |
| Postgres data path | `/srv/immich/postgres` |
| Alerts | ntfy.sh (push to phone) |
| Backup target | AWS S3 Standard, single bucket, versioned, lifecycle expires non-current versions after 30 days |
| Backup cadence | Nightly 03:30 UTC (library + DB dump) |
| Remote access | LAN-only (Tailscale / reverse proxy deferred) |

---

## Prerequisites

Gather these before starting:

1. **Reserved IP** for the server on your home router.
2. **AWS S3 bucket** with:
   - Versioning enabled
   - Default encryption (SSE-S3)
   - Block all public access on
   - Lifecycle rule: non-current versions expire after 30 days
3. **IAM user** with programmatic credentials and the policy from [Appendix A](#appendix-a--iam-policy).
4. **ntfy topic** — a hard-to-guess string (ntfy.sh is public). Subscribe in the ntfy mobile app.
5. **Variables:**

```text
LINUX_USER=<user that will own /srv/immich, e.g. robbie>
SERVER_IP=<reserved IP of the server>
S3_BUCKET=<bucket name>
AWS_REGION=<e.g. eu-west-1>
AWS_ACCESS_KEY_ID=<...>
AWS_SECRET_ACCESS_KEY=<...>
NTFY_TOPIC=<e.g. myserver-immich-abc123>
TZ=Europe/London
POSTGRES_PASSWORD=<generate: openssl rand -base64 32 | tr -d '/+=' | head -c 32>
```

---

## Step 0 — Install Docker CE

Snap Docker has known issues with bind mounts outside `/home` and networking. Install Docker CE from the official repo instead.

```bash
# Remove snap docker if present
sudo snap stop docker || true
sudo snap remove --purge docker

# Install Docker CE
sudo apt update
sudo apt install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu noble stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin

sudo usermod -aG docker "$LINUX_USER"

# Verify
docker --version
docker compose version
```

> Docker group membership requires a fresh login shell. Run remaining docker commands via `sudo` or re-login first.

---

## Step 1 — Filesystem layout

```bash
sudo mkdir -p /srv/immich/{library,postgres,db-backups,scripts}
sudo mkdir -p /var/log/immich
sudo chown -R "$LINUX_USER:$LINUX_USER" /srv/immich /var/log/immich
sudo chmod 750 /srv/immich
```

---

## Step 2 — Install Immich

```bash
cd /srv/immich

# Determine latest release tag
LATEST_IMMICH=$(curl -fsSL https://api.github.com/repos/immich-app/immich/releases/latest \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['tag_name'])")
echo "Latest Immich: $LATEST_IMMICH"

# Fetch official compose file
curl -fsSL "https://github.com/immich-app/immich/releases/download/${LATEST_IMMICH}/docker-compose.yml" \
  -o docker-compose.yml
```

Create `/srv/immich/.env`:

```ini
IMMICH_VERSION=<value of $LATEST_IMMICH, e.g. v2.7.5>
UPLOAD_LOCATION=/srv/immich/library
DB_DATA_LOCATION=/srv/immich/postgres
DB_PASSWORD=<value of $POSTGRES_PASSWORD>
DB_USERNAME=postgres
DB_DATABASE_NAME=immich
TZ=Europe/London
```

Lock it down and start:

```bash
sudo chmod 600 /srv/immich/.env
sudo chown "$LINUX_USER:$LINUX_USER" /srv/immich/.env /srv/immich/docker-compose.yml

cd /srv/immich
docker compose pull
docker compose up -d
```

Wait for healthy (up to 5 minutes):

```bash
for i in $(seq 1 60); do
  status=$(docker inspect --format='{{.State.Health.Status}}' immich_server 2>/dev/null || echo "starting")
  echo "[$i] immich_server: $status"
  [ "$status" = "healthy" ] && break
  sleep 5
done
```

Open firewall if needed:

```bash
sudo ufw allow 2283/tcp
```

---

## Step 3 — Admin user creation

1. Open `http://<SERVER_IP>:2283` in a browser
2. Click **Getting Started** → create the admin account
3. Go to **Administration → Users → Create User** for additional accounts

---

## Step 4 — Install and configure rclone

```bash
curl https://rclone.org/install.sh | sudo bash
```

Configure the S3 remote:

```bash
sudo -u "$LINUX_USER" mkdir -p "/home/$LINUX_USER/.config/rclone"
sudo -u "$LINUX_USER" tee "/home/$LINUX_USER/.config/rclone/rclone.conf" > /dev/null <<EOF
[s3backup]
type = s3
provider = AWS
access_key_id = ${AWS_ACCESS_KEY_ID}
secret_access_key = ${AWS_SECRET_ACCESS_KEY}
region = ${AWS_REGION}
location_constraint = ${AWS_REGION}
storage_class = STANDARD
no_check_bucket = true
EOF
sudo chmod 600 "/home/$LINUX_USER/.config/rclone/rclone.conf"
```

Verify:

```bash
sudo -u "$LINUX_USER" rclone lsd "s3backup:${S3_BUCKET}"
```

---

## Step 5 — Backup environment file

```bash
sudo tee /srv/immich/.backup.env > /dev/null <<EOF
NTFY_TOPIC=${NTFY_TOPIC}
S3_BUCKET=${S3_BUCKET}
AWS_REGION=${AWS_REGION}
LIBRARY_PATH=/srv/immich/library
DB_BACKUP_DIR=/srv/immich/db-backups
LOG_DIR=/var/log/immich
EOF
sudo chown "$LINUX_USER:$LINUX_USER" /srv/immich/.backup.env
sudo chmod 600 /srv/immich/.backup.env
```

---

## Step 6 — ntfy notification helper

```bash
sudo tee /srv/immich/scripts/notify.sh > /dev/null <<'EOF'
#!/usr/bin/env bash
# Usage: notify.sh <priority> <title> <tags> <message>
set -euo pipefail
: "${NTFY_TOPIC:?NTFY_TOPIC must be set}"
priority="${1:-default}"
title="${2:-Immich}"
tags="${3:-floppy_disk}"
message="${4:-}"
curl -fsSL --max-time 10 \
  -H "Title: ${title}" \
  -H "Priority: ${priority}" \
  -H "Tags: ${tags}" \
  -d "${message}" \
  "https://ntfy.sh/${NTFY_TOPIC}" > /dev/null
EOF
sudo chmod +x /srv/immich/scripts/notify.sh
sudo chown "$LINUX_USER:$LINUX_USER" /srv/immich/scripts/notify.sh
```

Test it:

```bash
sudo -u "$LINUX_USER" bash -c "set -a; source /srv/immich/.backup.env; set +a; \
  /srv/immich/scripts/notify.sh default 'Immich setup' 'wrench' 'Test alert from setup'"
```

Confirm the notification arrives on your phone before continuing.

---

## Step 7 — Storage usage alert

Warns via ntfy when disk usage hits 83%.

```bash
sudo tee /srv/immich/scripts/check-storage.sh > /dev/null <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
: "${NTFY_TOPIC:?}"; : "${LIBRARY_PATH:?}"

THRESHOLD_PCT=83
MOUNT="$LIBRARY_PATH"

usage=$(df --output=pcent "$MOUNT" | tail -1 | tr -dc '0-9')
free=$(df -h --output=avail "$MOUNT" | tail -1 | tr -d ' ')

if [ "$usage" -ge "$THRESHOLD_PCT" ]; then
  /srv/immich/scripts/notify.sh high "Immich disk warning" "warning" \
    "Disk at ${usage}% on ${MOUNT} — ${free} free"
fi
EOF
sudo chmod +x /srv/immich/scripts/check-storage.sh
sudo chown "$LINUX_USER:$LINUX_USER" /srv/immich/scripts/check-storage.sh
```

Systemd service + timer (hourly):

```bash
sudo tee /etc/systemd/system/immich-storage-check.service > /dev/null <<EOF
[Unit]
Description=Immich storage usage check

[Service]
Type=oneshot
User=${LINUX_USER}
EnvironmentFile=/srv/immich/.backup.env
ExecStart=/srv/immich/scripts/check-storage.sh
EOF

sudo tee /etc/systemd/system/immich-storage-check.timer > /dev/null <<'EOF'
[Unit]
Description=Run Immich storage check hourly

[Timer]
OnCalendar=hourly
Persistent=true
RandomizedDelaySec=10min

[Install]
WantedBy=timers.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now immich-storage-check.timer
```

---

## Step 8 — Nightly backup script and timer

```bash
sudo tee /srv/immich/scripts/backup.sh > /dev/null <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
: "${NTFY_TOPIC:?}"; : "${S3_BUCKET:?}"; : "${LIBRARY_PATH:?}"
: "${DB_BACKUP_DIR:?}"; : "${LOG_DIR:?}"

LOG="${LOG_DIR}/backup-$(date +%Y%m%d-%H%M%S).log"
mkdir -p "$LOG_DIR" "$DB_BACKUP_DIR"
exec >>"$LOG" 2>&1

trap '/srv/immich/scripts/notify.sh urgent "Immich backup FAILED" "x,rotating_light" "See ${LOG} on the server"' ERR

echo "=== $(date -Iseconds) backup start ==="

# 1. Postgres dump
echo "--- pg_dumpall ---"
DUMP="${DB_BACKUP_DIR}/immich-$(date +%Y%m%d).sql.gz"
docker exec -t immich_postgres pg_dumpall --clean --if-exists -U postgres \
  | gzip -9 > "$DUMP"
ls -lh "$DUMP"

# Prune local DB dumps older than 30 days
find "$DB_BACKUP_DIR" -name 'immich-*.sql.gz' -mtime +30 -delete

# 2. Library sync to S3
echo "--- rclone sync library ---"
rclone sync "$LIBRARY_PATH" "s3backup:${S3_BUCKET}/library/" \
  --transfers 4 \
  --checksum \
  --stats-one-line \
  --stats 5m

# 3. DB dumps to S3
echo "--- rclone sync db-backups ---"
rclone sync "$DB_BACKUP_DIR" "s3backup:${S3_BUCKET}/db-backups/" \
  --transfers 2 \
  --stats-one-line

echo "=== $(date -Iseconds) backup OK ==="

/srv/immich/scripts/notify.sh low "Immich backup OK" "white_check_mark,floppy_disk" \
  "Library + DB synced to S3"
EOF
sudo chmod +x /srv/immich/scripts/backup.sh
sudo chown "$LINUX_USER:$LINUX_USER" /srv/immich/scripts/backup.sh
```

Service + timer:

```bash
sudo tee /etc/systemd/system/immich-backup.service > /dev/null <<EOF
[Unit]
Description=Immich nightly backup
After=docker.service
Requires=docker.service

[Service]
Type=oneshot
User=${LINUX_USER}
EnvironmentFile=/srv/immich/.backup.env
ExecStart=/srv/immich/scripts/backup.sh
TimeoutStartSec=6h
EOF

sudo tee /etc/systemd/system/immich-backup.timer > /dev/null <<'EOF'
[Unit]
Description=Run Immich backup nightly

[Timer]
OnCalendar=*-*-* 03:30:00
Persistent=true
RandomizedDelaySec=20min

[Install]
WantedBy=timers.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now immich-backup.timer
```

---

## Step 9 — Log rotation

```bash
sudo tee /etc/logrotate.d/immich-backup > /dev/null <<'EOF'
/var/log/immich/*.log {
    weekly
    rotate 8
    compress
    delaycompress
    missingok
    notifempty
}
EOF
```

---

## Step 10 — Validation

Trigger a backup and verify end-to-end:

```bash
# Run backup now
sudo systemctl start immich-backup.service
sudo journalctl -u immich-backup.service -f
```

Then verify:

```bash
# Local dump exists
ls -lh /srv/immich/db-backups/

# S3 contents
rclone ls "s3backup:${S3_BUCKET}/db-backups/" | head
rclone size "s3backup:${S3_BUCKET}/library/"

# Timers scheduled
sudo systemctl list-timers immich-*.timer

# Confirm ntfy notification arrived on phone
```

Restore smoke test (does not modify production data):

```bash
mkdir -p /tmp/restore-test
rclone copy "s3backup:${S3_BUCKET}/db-backups/" /tmp/restore-test/ \
  --include "immich-*.sql.gz" --max-age 24h
ls -lh /tmp/restore-test/
gunzip -t /tmp/restore-test/*.sql.gz && echo "Dump file is a valid gzip"
rm -rf /tmp/restore-test
```

---

## Notes

- **Snap Docker won't work** — snap-confined Docker cannot access `/srv`. Use Docker CE from the official apt repo.
- **rclone `no_check_bucket = true`** is required unless the IAM user has `s3:CreateBucket` permission. Since the bucket is pre-created, skipping the check is correct.
- **rclone `location_constraint`** must match `region` for non-us-east-1 buckets.
- **Brave browser** may block plain HTTP on local IPs. Use another browser or disable HTTPS-only mode for the server address.
- **Docker group** membership requires a fresh login shell after `usermod -aG docker`.

---

## Appendix A — IAM policy

Replace `BUCKET_NAME` with the actual bucket name:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "BucketLevel",
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket",
        "s3:GetBucketLocation",
        "s3:GetBucketVersioning",
        "s3:ListBucketVersions"
      ],
      "Resource": "arn:aws:s3:::BUCKET_NAME"
    },
    {
      "Sid": "ObjectLevel",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:DeleteObject",
        "s3:GetObjectVersion",
        "s3:DeleteObjectVersion"
      ],
      "Resource": "arn:aws:s3:::BUCKET_NAME/*"
    }
  ]
}
```

---

## Appendix B — External USB drive (not yet actioned)

A second local copy on a USB HDD completes the 3-2-1 backup strategy. Purchase a 2 TB+ USB HDD, then:

1. Format as ext4 with label `IMMICH_BACKUP`.
2. Add fstab entry: `LABEL=IMMICH_BACKUP /mnt/usb-backup ext4 defaults,nofail,noauto 0 2`
3. Create `/srv/immich/scripts/backup-usb.sh` that mounts, rsyncs library + db-backups, unmounts, and notifies via ntfy.
4. Add systemd timer `immich-backup-usb.timer` running daily, 30 min after the S3 backup.
