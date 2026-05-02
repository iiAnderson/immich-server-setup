# Immich Restore Procedures

## 1. Restore an individual deleted file

S3 versioning is on; deletes are recoverable for 30 days.

```bash
# List versions of an object
aws s3api list-object-versions \
  --bucket "$S3_BUCKET" \
  --prefix "library/upload/<user-id>/<YYYY>/<MM>/<filename>" \
  --region "$AWS_REGION"

# Restore by deleting the delete-marker
aws s3api delete-object \
  --bucket "$S3_BUCKET" \
  --key "library/upload/.../<filename>" \
  --version-id <delete-marker-version-id> \
  --region "$AWS_REGION"
```

Then trigger an Immich library scan: **Administration > Jobs > Library > Refresh all libraries**.

## 2. Restore the database from a dump

```bash
# Find latest local dump
LATEST_DUMP=$(ls -t /srv/immich/db-backups/immich-*.sql.gz | head -1)

# Stop Immich containers (keep postgres running)
cd /srv/immich
docker compose stop immich-server immich-machine-learning

# Drop and reload (DESTRUCTIVE)
gunzip -c "$LATEST_DUMP" | docker exec -i immich_postgres psql -U postgres

# Start everything
docker compose up -d
```

If the dump is only available in S3:

```bash
rclone copy "s3backup:${S3_BUCKET}/db-backups/immich-YYYYMMDD.sql.gz" \
  /srv/immich/db-backups/
```

## 3. Full disaster recovery (server dies)

On a fresh Ubuntu 24.04 host:

1. Follow SETUP.md Steps 0–7 (Docker CE, filesystem layout, Immich install, rclone setup) **but do not start Immich yet**.

2. Restore the library from S3 **before first start**:
   ```bash
   rclone sync "s3backup:${S3_BUCKET}/library/" /srv/immich/library/ \
     --transfers 8 --checksum
   ```

3. Restore the latest DB dump:
   ```bash
   rclone copy "s3backup:${S3_BUCKET}/db-backups/" /srv/immich/db-backups/ \
     --include "immich-*.sql.gz" --max-age 48h
   ```

4. Start Postgres and Redis only:
   ```bash
   cd /srv/immich && docker compose up -d database redis
   sleep 30
   ```

5. Load the dump:
   ```bash
   LATEST_DUMP=$(ls -t /srv/immich/db-backups/immich-*.sql.gz | head -1)
   gunzip -c "$LATEST_DUMP" | docker exec -i immich_postgres psql -U postgres
   ```

6. Start the rest:
   ```bash
   docker compose up -d
   ```

7. Sign in. Library should be intact, albums and faces preserved. Trigger **Administration > Jobs > Library > Refresh all libraries** if any imports look stale.

## 4. Library files only, DB unrecoverable

Restore the library from S3 as in step 3.2 above, start a fresh Immich, sign in as admin, and run a library import pointing at `/srv/immich/library`. Albums, faces, and ML embeddings will be lost and will need to re-run — slow on modest hardware but it works.
