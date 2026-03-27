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
docker image ls --format '{{.Repository}}:{{.Tag}}\t{{.ID}}' | rg '^(social-|sail-8\.5/app:)'
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
docker network ls --format '{{.Name}}' | rg '(social|sail)'
docker volume ls --format '{{.Name}}' | rg '(social|sail)'
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
docker ps -a --format '{{.Image}}\t{{.Names}}' | rg '(social|sail)'
docker image ls --format '{{.Repository}}:{{.Tag}}' | rg '(social|sail)'
docker network ls --format '{{.Name}}' | rg '(social|sail)'
docker volume ls --format '{{.Name}}' | rg '(social|sail)'
```

Important: steps 7 and 8 permanently delete Docker volumes (database/cache/session data).


# 10) Load live credentials from .envkdc
```
set -a
source .envkdc
set +a
```

# 11) If running on host (not inside Docker network), force forwarded DB endpoint
```
# DB_HOST=127.0.0.1
# DB_PORT="${FORWARD_DB_PORT:-5433}"
```

# 12) Backup PostgreSQL + storage
```
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
```

# 13) Restore PostgreSQL + storage later
```
set -a
source .envkdc
set +a

# DB_HOST=127.0.0.1
# DB_PORT="${FORWARD_DB_PORT:-5433}"

BACKUP_DIR="$HOME/backups/kdc/<TS>"

PGPASSWORD="$DB_PASSWORD" pg_restore \
  -h "$DB_HOST" -p "${DB_PORT:-5432}" \
  -U "$DB_USERNAME" -d "$DB_DATABASE" \
  --clean --if-exists --no-owner --no-privileges \
  "$BACKUP_DIR/${DB_DATABASE}.dump"

rm -rf storage
tar -xzf "$BACKUP_DIR/storage.tar.gz"

php artisan storage:link
php artisan optimize:clear

echo "Restore complete from: $BACKUP_DIR"
```
