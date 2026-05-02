# Immich Server Setup

A complete guide for deploying a self-hosted [Immich](https://immich.app) instance on a home server running Ubuntu 24.04, with automated S3 backups and push notifications.

## What's included

- **Docker CE installation** — replaces snap Docker to avoid bind-mount and path restrictions
- **Immich deployment** — pinned version via official compose file, data under `/srv/immich`
- **Nightly S3 backups** — library files and Postgres dumps synced via rclone at 03:30 UTC
- **Push notifications** — success/failure alerts via [ntfy.sh](https://ntfy.sh) to your phone
- **Disk monitoring** — hourly storage check with ntfy alert at 83% usage
- **Log rotation** — weekly rotation of backup logs, 8 weeks retained
- **Restore procedures** — from single-file recovery to full disaster recovery on a new host

## Documents

| File | Description |
|---|---|
| [SETUP.md](SETUP.md) | Step-by-step installation and configuration guide |
| [RESTORE.md](RESTORE.md) | Restore procedures for four recovery scenarios |

## Architecture

```
Server (Ubuntu 24.04)
├── Docker CE
│   ├── immich_server        (web UI + API on port 2283)
│   ├── immich_postgres      (database)
│   ├── immich_machine_learning
│   └── immich_redis
├── Systemd timers
│   ├── immich-backup.timer  (nightly S3 sync)
│   └── immich-storage-check.timer (hourly disk check)
└── /srv/immich
    ├── library/             (uploaded photos & videos)
    ├── postgres/            (database files)
    ├── db-backups/          (local .sql.gz dumps, 30 days)
    └── scripts/             (backup.sh, notify.sh, check-storage.sh)
```

## Prerequisites

- Ubuntu 24.04 server with a reserved LAN IP
- AWS S3 bucket (versioned, encrypted, public access blocked)
- IAM user scoped to the bucket (policy in [SETUP.md](SETUP.md#appendix-a--iam-policy))
- [ntfy](https://ntfy.sh) mobile app with a subscribed topic

## Quick start

1. Gather your variables (see [Prerequisites](SETUP.md#prerequisites))
2. Follow [SETUP.md](SETUP.md) steps 0–10
3. Keep [RESTORE.md](RESTORE.md) handy for when you need it

## License

MIT
