# Docker Social/Sail Cleanup

Use this exact sequence (the same flow I used), in order.


# 1) Inspect current containers and images
```
docker ps -a --format '{{.ID}}\t{{.Image}}\t{{.Names}}\t{{.Status}}'
docker image ls --format '{{.Repository}}:{{.Tag}}\t{{.ID}}\t{{.CreatedSince}}'
```

# 2) List containers belonging to the social compose project
```
docker ps -aq --filter label=com.docker.compose.project=social
```

# 3) List only social/sail images
```
docker image ls --format '{{.Repository}}:{{.Tag}}\t{{.ID}}' | grep -E '^(social-|sail-8\.5/app:)'
```

# 4) Remove social project containers (replace IDs with your output from step 2)
```
docker rm -f <id1> <id2> <id3> <id4> <id5> <id6> <id7> <id8>
```

# 5) Remove social/sail images
```
docker image rm social-nginx:latest social-app:latest sail-8.5/app:latest
```

# 6) Check remaining social/sail networks and volumes
```
docker network ls --format '{{.Name}}' | grep -E '(social|sail)'
docker volume ls --format '{{.Name}}' | grep -E '(social|sail)'
```

# 7) Remove remaining sail-specific network/volumes
```
docker network rm social_sail
docker volume rm social_sail-pgsql social_sail-redis
```

# 8) Remove remaining social network/volumes
```
docker network rm social_default
docker volume rm social_laravel-cache social_laravel-storage social_redis-data
```

# 9) Final verification (should return no lines)
```
docker ps -a --format '{{.Image}}\t{{.Names}}' | grep -E '(social|sail)'
docker image ls --format '{{.Repository}}:{{.Tag}}' | grep -E '(social|sail)'
docker network ls --format '{{.Name}}' | grep -E '(social|sail)'
docker volume ls --format '{{.Name}}' | grep -E '(social|sail)'
```

Important: steps 7 and 8 permanently delete Docker volumes (database/cache/session data).


# 10) Go to your Laravel project root
```
cd /path/to/your/laravel/project
```

# 11) Create scripts folder
```
mkdir -p scripts
```

# 12) Create script files explicitly
```
touch scripts/backup-kdc.sh scripts/restore-kdc.sh
```

# 13) Paste full backup script into scripts/backup-kdc.sh (copy and paste exactly)
```
cat > scripts/backup-kdc.sh <<'SH'
#!/usr/bin/env bash
set -euo pipefail

set -a
source .envkdc
set +a

# If running from host (outside docker network), uncomment these two lines:
# DB_HOST=127.0.0.1
# DB_PORT="${FORWARD_DB_PORT:-5433}"

TS=$(date +%F-%H%M%S)
BACKUP_DIR="$HOME/backups/kdc/$TS"
mkdir -p "$BACKUP_DIR"

PGPASSWORD="$DB_PASSWORD" pg_dump \
  -h "$DB_HOST" -p "${DB_PORT:-5432}" \
  -U "$DB_USERNAME" -d "$DB_DATABASE" \
  -Fc --no-owner --no-privileges \
  > "$BACKUP_DIR/${DB_DATABASE}.dump"

tar -czf "$BACKUP_DIR/storage.tar.gz" storage
cp .envkdc "$BACKUP_DIR/.envkdc.backup"
sha256sum "$BACKUP_DIR/${DB_DATABASE}.dump" "$BACKUP_DIR/storage.tar.gz" > "$BACKUP_DIR/SHA256SUMS"

echo "Backup complete: $BACKUP_DIR"
SH
```

# 14) Paste full restore script into scripts/restore-kdc.sh (copy and paste exactly)
```
cat > scripts/restore-kdc.sh <<'SH'
#!/usr/bin/env bash
set -euo pipefail

if [ "${1:-}" = "" ]; then
  echo "Usage: ./scripts/restore-kdc.sh <backup_timestamp>"
  echo "Example: ./scripts/restore-kdc.sh 2026-03-27-221500"
  exit 1
fi

TS="$1"
BACKUP_DIR="$HOME/backups/kdc/$TS"

set -a
source .envkdc
set +a

# If running from host (outside docker network), uncomment these two lines:
# DB_HOST=127.0.0.1
# DB_PORT="${FORWARD_DB_PORT:-5433}"

if [ ! -f "$BACKUP_DIR/${DB_DATABASE}.dump" ]; then
  echo "Missing dump file: $BACKUP_DIR/${DB_DATABASE}.dump"
  exit 1
fi

if [ ! -f "$BACKUP_DIR/storage.tar.gz" ]; then
  echo "Missing storage archive: $BACKUP_DIR/storage.tar.gz"
  exit 1
fi

# Verify checksums before making any destructive changes
if [ -f "$BACKUP_DIR/SHA256SUMS" ]; then
  echo "Verifying backup checksums..."
  (cd "$BACKUP_DIR" && sha256sum -c SHA256SUMS)
  echo "Checksums OK."
fi

# Restore database
PGPASSWORD="$DB_PASSWORD" pg_restore \
  -h "$DB_HOST" -p "${DB_PORT:-5432}" \
  -U "$DB_USERNAME" -d "$DB_DATABASE" \
  --clean --if-exists --no-owner --no-privileges \
  "$BACKUP_DIR/${DB_DATABASE}.dump"

# Extract storage to a temporary directory first, then swap atomically
STORAGE_TMP="storage.restore.$$"
tar -xzf "$BACKUP_DIR/storage.tar.gz" --one-top-level="$STORAGE_TMP"

# Only remove the existing storage directory after the archive is confirmed good
rm -rf storage
mv "$STORAGE_TMP/storage" storage

php artisan storage:link
php artisan optimize:clear

echo "Restore complete from: $BACKUP_DIR"
SH
```

# 15) Make scripts executable
```
chmod +x scripts/backup-kdc.sh scripts/restore-kdc.sh
```

# 16) Run backup
```
./scripts/backup-kdc.sh
```

# 17) Run restore
```
./scripts/restore-kdc.sh <backup_timestamp>
```
