# Docker Deployment Strategy (Laravel + VPS + systemd)

## Architecture overview

Each domain/project runs as:

- Separate Linux user
- Separate home directory
- Separate Docker Compose stack
- Reverse proxied by host Nginx (optional)

### Example projects

| Domain          | Linux User | App Directory      | Internal Port |
| --------------- | ---------- | ------------------ | ------------- |
| techworld.co.ke | tecworld   | /home/tecworld/app | 8001          |
| acquihub.africa | acquihub   | /home/acquihub/app | 8002          |

If you add more dockerized sites, repeat these steps for each app with its own Linux user, home directory, and a unique host port (for example 8003, 8004). Keep the port consistent between `docker-compose.yml` and the host Nginx proxy.

## Prerequisites (Docker install + project environment)

Before Step 1, prepare the VPS host and each application's `.env`.

### A) Install Docker Engine and Compose plugin (Ubuntu 22.04/24.04)

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo \"$VERSION_CODENAME\") stable" \
| sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker
```

Verify:

```bash
docker --version
docker compose version
sudo systemctl status docker --no-pager -l
```

### B) Allow deploy users to run Docker commands

This guide uses per-project Linux users (`tecworld`, `acquihub`) and runs deployment commands as those users. Give each deploy user Docker access:

```bash
sudo usermod -aG docker tecworld
sudo usermod -aG docker acquihub
```

Then log out and log back in (or reboot) so new group membership is applied.

Security note: Docker group access is effectively root-level access on the host. Only add trusted deploy users.

### C) Create the application `.env` before first deploy

In each project directory:

```bash
cd /home/tecworld/app
cp .env.example .env
```

Set production values (minimum):

```env
APP_ENV=production
APP_DEBUG=false
APP_URL=https://techworld.co.ke

DB_CONNECTION=pgsql
DB_HOST=db
DB_PORT=5432
DB_DATABASE=laravel
DB_USERNAME=user
DB_PASSWORD=secret

REDIS_HOST=redis
REDIS_PORT=6379
```

Harden permissions for secrets:

```bash
chmod 600 .env
```

`docker compose` also reads root-level `.env` for variable substitution in `docker-compose.yml`, so this file is required before first `docker compose up`.

## 1) Project structure (per application)

Example for techworld:

```text
/home/tecworld/app
├── Dockerfile
├── docker-compose.yml
├── .dockerignore
├── vite.config.js
├── docker/
│   ├── entrypoint.sh
│   ├── data/
│   │   └── db/
│   └── nginx/
│       └── default.conf
└── Laravel application files
```

## 2) Dockerfile (supports hot reload)

```dockerfile
# Set the base image
FROM php:8.3-fpm

# Set working directory
WORKDIR /var/www/html

# Install dependencies
RUN apt-get update && apt-get install -y \
    build-essential \
    libpng-dev \
    libjpeg62-turbo-dev \
    libfreetype6-dev \
    locales \
    zip \
    jpegoptim optipng pngquant gifsicle \
    vim \
    unzip \
    git \
    curl \
    libonig-dev \
    libzip-dev \
    libpq-dev

# Install Node.js
RUN curl -fsSL https://deb.nodesource.com/setup_20.x | bash - \
    && apt-get install -y nodejs

# Clear cache
RUN apt-get clean && rm -rf /var/lib/apt/lists/*

# Install extensions
RUN docker-php-ext-install pdo_mysql pdo_pgsql mbstring zip exif pcntl
RUN docker-php-ext-configure gd --with-freetype --with-jpeg
RUN docker-php-ext-install gd

# Install Composer
RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer

# Add user for laravel application
RUN groupadd -g 1000 www
RUN useradd -u 1000 -ms /bin/bash -g www www

# Copy existing application directory contents
COPY . .

# Copy existing application directory permissions
COPY --chown=www:www . /var/www/html

# Copy entrypoint script
COPY docker/entrypoint.sh /usr/local/bin/docker-entrypoint.sh
RUN chmod +x /usr/local/bin/docker-entrypoint.sh

# Change current user to www
USER www

# Expose port 9000 and start php-fpm server
EXPOSE 9000
ENTRYPOINT ["docker-entrypoint.sh"]
CMD ["php-fpm"]
```

## 3) docker/entrypoint.sh (development only)

Use this to auto-start Vite in the background only for development:

```bash
#!/bin/bash
set -e

if [ "${APP_ENV:-production}" = "development" ] || [ "${APP_ENV:-production}" = "local" ]; then
    # Install Node dependencies
    npm install

    # Start Vite in the background
    npm run dev &
fi

# Execute the main container command
exec "$@"
```

Set `APP_ENV=development` (or `APP_ENV=local`) in development, and keep `APP_ENV=production` on server deployments.

## 4) docker-compose.yml (slim)

```yaml
services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    image: tail-tap-app
    container_name: tail-tap-app
    restart: unless-stopped
    working_dir: /var/www/html
    volumes:
      - .:/var/www/html
    ports:
      - "5173:5173"
    networks:
      - tail-tap

  nginx:
    image: nginx:alpine
    container_name: tail-tap-nginx
    restart: unless-stopped
    ports:
      - "8001:80"
    volumes:
      - .:/var/www/html
      - ./docker/nginx/default.conf:/etc/nginx/conf.d/default.conf
    networks:
      - tail-tap

  db:
    image: postgres:16
    container_name: tail-tap-db
    restart: unless-stopped
    environment:
      POSTGRES_DB: ${DB_DATABASE:-laravel}
      POSTGRES_USER: ${DB_USERNAME:-user}
      POSTGRES_PASSWORD: ${DB_PASSWORD:-secret}
    volumes:
      - ./docker/data/db:/var/lib/postgresql/data
    ports:
      - "54321:5432"
    networks:
      - tail-tap

  redis:
    image: redis:alpine
    container_name: tail-tap-redis
    restart: unless-stopped
    ports:
      - "63791:6379"
    networks:
      - tail-tap

networks:
  tail-tap:
    driver: bridge
```

### Laravel `.env` for Docker networking

When Laravel runs inside the `app` container, use Docker service names:

```env
DB_HOST=db
DB_PORT=5432
REDIS_HOST=redis
REDIS_PORT=6379
```

Run Laravel maintenance commands from inside the container:

```bash
docker compose exec -T app php artisan migrate --force
docker compose exec -T app php artisan optimize
```

If you run `php artisan ...` from the VPS host directly, use mapped host ports instead:

```env
DB_HOST=127.0.0.1
DB_PORT=54321
REDIS_HOST=127.0.0.1
REDIS_PORT=63791
```

## 5) docker/nginx/default.conf

```nginx
server {
    listen 80;
    server_name _;
    root /var/www/html/public;

    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-XSS-Protection "1; mode=block";
    add_header X-Content-Type-Options "nosniff";

    index index.php;

    charset utf-8;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }

    error_page 404 /index.php;

    location ~ \.php$ {
        fastcgi_pass app:9000;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }
}
```

## 6) .dockerignore

```text
.env
.env.backup
.env.production
.git
.gitignore
.idea
.vscode
docker-compose.yml
docker/data/
node_modules
public/storage
storage/framework/cache/data/
storage/framework/sessions/
storage/framework/testing/
storage/framework/views/
storage/logs/
vendor
.DS_Store
npm-debug.log
yarn-error.log
/bootstrap/cache/packages.php
/bootstrap/cache/services.php
```

### Why `docker/data/db` is needed

Inside the project `docker/` folder, each path has a specific role:

- `docker/data/db`: PostgreSQL persistent data on the host
- `docker/entrypoint.sh`: app container startup logic (including dev-only Vite start)
- `docker/nginx/default.conf`: Nginx virtual host config for the container

For PostgreSQL persistence, this mapping in `docker-compose.yml` is critical:

```yaml
db:
  volumes:
    - ./docker/data/db:/var/lib/postgresql/data
```

- `./docker/data/db` is the host folder in your project.
- `/var/lib/postgresql/data` is PostgreSQL's data directory inside the container.
- If the container is recreated (for example `docker compose down` then `up`), database files remain in `./docker/data/db`, so data is not lost.

`docker/data/` is intentionally listed in `.dockerignore` so database files are never copied into Docker image builds. This keeps images small and avoids shipping or overwriting live database data during build.

### Fixing permission errors for storage and cache directories

When using volume mounts (`.:/var/www/html`), you may encounter this error:

```
file_put_contents(...): Permission denied
```

This happens because the `www` user inside the container (UID 1000) does not have permission to write to the `storage` and `bootstrap/cache` directories on your host machine.

#### Solution: Set proper ownership from the host

Run these commands from your host machine's terminal (not inside the container) in your project directory:

```bash
# Navigate to your project directory (replace with your actual path)
cd /home/tecworld/app

# Set ownership to UID 1000 and GID 33 (www-data group)
sudo chown -R 1000:33 storage bootstrap/cache

# Keep group write and setgid so new files inherit group
sudo find storage bootstrap/cache -type d -exec chmod 2775 {} \;
sudo find storage bootstrap/cache -type f -exec chmod 664 {} \;
```

**Why UID 1000 and GID 33?**

The Dockerfile creates the `www` user with UID 1000:

```dockerfile
RUN groupadd -g 1000 www
RUN useradd -u 1000 -ms /bin/bash -g www www
```

When volumes are mounted from the host, the container uses numeric IDs to determine file access. UID `1000` matches the `www` user in the container, and GID `33` maps to `www-data` on Ubuntu. This allows both the app container user and Nginx/PHP host group workflows to write logs and cache files safely.

#### Alternative: Run commands inside the container

If you prefer to set permissions from inside the container:

```bash
docker compose exec -T -u root app sh -lc "chown -R 1000:33 storage bootstrap/cache && find storage bootstrap/cache -type d -exec chmod 2775 {} + && find storage bootstrap/cache -type f -exec chmod 664 {} +"
```

**Important:** After running these commands, restart your containers to ensure changes take effect:

```bash
docker compose restart
```

## 7) Vite config (TailwindCSS + hot reload)

The `server` block below is required:

```js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';

export default defineConfig({
    server: {
        host: '0.0.0.0',
        hmr: {
            host: 'localhost',
        },
    },
    plugins: [
        laravel({
            input: ['resources/css/app.css', 'resources/js/app.js'],
            refresh: true,
        }),
    ],
});
```

## 8) Host Nginx reverse proxy

File:

```text
/etc/nginx/sites-available/techworld.co.ke
```

```nginx
server {
    listen 80;
    server_name techworld.co.ke www.techworld.co.ke;

    location / {
        proxy_pass http://127.0.0.1:8001;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

Enable:

```bash
sudo ln -s /etc/nginx/sites-available/techworld.co.ke /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

## 9) systemd auto-start (reusable template)

Create once:

```bash
sudo nano /etc/systemd/system/compose@.service
```

```ini
[Unit]
Description=Docker Compose Stack for %i
Requires=docker.service
After=docker.service network-online.target
Wants=network-online.target

[Service]
Type=oneshot
RemainAfterExit=yes

WorkingDirectory=/home/%i/app
User=%i
Group=%i

ExecStart=/usr/bin/docker compose up -d --remove-orphans
ExecStop=/usr/bin/docker compose down

TimeoutStartSec=0
TimeoutStopSec=120

[Install]
WantedBy=multi-user.target
```

Reload:

```bash
sudo systemctl daemon-reload
```

Enable per project:

```bash
sudo systemctl enable --now compose@tecworld
sudo systemctl enable --now compose@acquihub
```

## 10) Deployment strategy (GitHub Actions + appleboy/ssh-action)

We use rolling replace (minimal downtime).

### SSH deploy commands

```bash
cd /home/tecworld/app

git pull

docker compose up -d --build --remove-orphans

docker compose exec -T app php artisan migrate --force
docker compose exec -T app php artisan config:cache
docker compose exec -T app php artisan route:cache

docker image prune -f
```

Repeat for acquihub.

## 11) GitHub Actions example

```yaml
- name: Deploy to VPS
  uses: appleboy/ssh-action@v1.0.3
  with:
    host: ${{ secrets.SERVER_HOST }}
    username: tecworld
    key: ${{ secrets.SERVER_SSH_KEY }}
    script: |
      cd /home/tecworld/app
      git pull
      docker compose up -d --build --remove-orphans
      docker compose exec -T app php artisan migrate --force
      docker compose exec -T app php artisan optimize
      docker image prune -f
```

## 12) Post-deployment checks

```bash
docker compose ps
docker compose logs -f
curl -I http://127.0.0.1:8001
```

## 13) Key principles

- One Linux user per project
- One compose stack per project
- Host Nginx handles public traffic and SSL
- systemd ensures auto-start on reboot
- Deploy via rolling replace (no `docker compose down`)

## 14) Notes on downtime

Current strategy provides:

- Minimal downtime (1-3 seconds)
- No blue/green complexity
- Safe and simple GitHub-based deployment

For strict zero-downtime, blue/green or load balancing is required.

## 15) Full Docker reset (delete all containers and images)

Use this only when you want to completely wipe Docker on the host.

It can reclaim large space (for example about `9.38GB`), but Docker will not delete anything still marked as in use.

### 1. Stop and kill all running containers

This terminates active processes for all stacks (for example `tail-tap` and `social-app`).

```bash
docker stop $(docker ps -q)
```

### 2. Remove all containers

This deletes container instances.

```bash
docker rm $(docker ps -aq)
```

### 3. Force remove all images

This removes all local images (Postgres, Nginx, Node, and others).

```bash
docker rmi -f $(docker images -q)
```

### Why a previous cleanup may not clear everything

If a container is running (for example `Up 4 hours`), Docker keeps its image locked and cannot remove it.

After stopping and removing containers first, the images can be deleted.

### Clean up leftovers (volumes and networks)

Run this to remove dangling resources, including volumes:

```bash
docker system prune -a --volumes -f
```

After these commands, `docker ps` and `docker images` should be empty.

### Delete only one resource (safe targeted cleanup)

If you only want to remove one container/image instead of everything, use these:

1. Stop and remove one container by name:

```bash
docker stop tail-tap-nginx
docker rm tail-tap-nginx
```

2. Remove one image by name or ID:

```bash
docker rmi nginx:alpine
# or
docker rmi <image_id>
```

3. Force remove one image if it is still referenced:

```bash
docker rmi -f nginx:alpine
```

4. Remove one volume or network (if no container is using it):

```bash
docker volume rm <volume_name>
docker network rm <network_name>
```

Tip: list exact names first with `docker ps -a`, `docker images`, `docker volume ls`, and `docker network ls`.

## 16) Troubleshooting

### Permission denied errors

If you see `file_put_contents(...): Permission denied` or similar errors:

- **Cause:** The `www` user inside the container (UID 1000) doesn't have write access to `storage` or `bootstrap/cache` on the host.
- **Solution:** See section 6 "Fixing permission errors for storage and cache directories" above.

Quick fix:

```bash
sudo chown -R 1000:33 storage bootstrap/cache
sudo find storage bootstrap/cache -type d -exec chmod 2775 {} \;
sudo find storage bootstrap/cache -type f -exec chmod 664 {} \;
docker compose restart
```

### Container won't start or exits immediately

Check logs:

```bash
docker compose logs app
```

Common causes:

- Missing `.env` file
- Invalid environment variables
- PHP syntax errors
- Port conflicts (another process using port 9000)

### Database connection refused

- Check if the `db` container is running: `docker compose ps`
- Verify database credentials in `.env` match `docker-compose.yml`
- If running Laravel inside Docker, ensure `DB_HOST=db` and `DB_PORT=5432`
- If running Laravel from host, use `DB_HOST=127.0.0.1` and `DB_PORT=54321`
- If you see `could not translate host name "db"`, run Artisan via `docker compose exec -T app ...` or switch host DB values as above

### Vite not accessible or hot reload not working

- Confirm port 5173 is exposed in `docker-compose.yml`
- Check if `APP_ENV=development` or `APP_ENV=local` is set
- Verify `vite.config.js` has `host: '0.0.0.0'` and `hmr.host: 'localhost'`
- Restart containers: `docker compose restart`
