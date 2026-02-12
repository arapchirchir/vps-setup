# Docker Deployment Strategy (Laravel + VPS + systemd)

## Architecture overview

Each domain/project runs as:

- Separate Linux user
- Separate home directory
- Separate Docker Compose stack
- Reverse proxied by host Nginx

### Example projects

| Domain          | Linux User | App Directory      | Internal Port |
| --------------- | ---------- | ------------------ | ------------- |
| techworld.co.ke | tecworld   | /home/tecworld/app | 8081          |
| acquihub.africa | acquihub   | /home/acquihub/app | 8082          |

If you add more dockerized sites, repeat these steps for each app with its own Linux user, home directory, and a unique localhost port (for example 8083, 8084). Keep the port consistent between docker-compose and the host Nginx proxy.

## 1) Project structure (per application)

Example for techworld:

```
/home/tecworld/app
├── Dockerfile
├── docker-compose.yml
├── docker/
│   └── nginx/
│       └── default.conf
└── Laravel application files
```

## 2) Dockerfile (Laravel production image)

```dockerfile
FROM php:8.3-fpm-alpine

RUN apk add --no-cache \
    bash curl git unzip icu-dev oniguruma-dev libzip-dev \
    postgresql-dev \
  && docker-php-ext-install \
    intl mbstring zip pdo pdo_pgsql opcache \
  && rm -rf /var/cache/apk/*

COPY --from=composer:2 /usr/bin/composer /usr/bin/composer

WORKDIR /var/www/html

COPY composer.json composer.lock ./
RUN composer install --no-interaction --no-progress --prefer-dist --optimize-autoloader

COPY . .

RUN chmod -R 775 storage bootstrap/cache || true

EXPOSE 9000
CMD ["php-fpm"]
```

## 3) docker-compose.yml (production)

Example for techworld (port 8081):

```yaml
services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: techworld_app
    restart: unless-stopped
    working_dir: /var/www/html
    volumes:
      - /home/tecworld/app:/var/www/html
    environment:
      APP_ENV: production
      APP_DEBUG: "false"
      DB_CONNECTION: pgsql
      DB_HOST: db
      DB_PORT: 5432
      DB_DATABASE: techworld
      DB_USERNAME: techworld
      DB_PASSWORD: change_me_strong
    depends_on:
      - db
    networks:
      - techworld_net

  nginx:
    image: nginx:1.27-alpine
    container_name: techworld_nginx
    restart: unless-stopped
    ports:
      - "127.0.0.1:8081:80"
    volumes:
      - /home/tecworld/app:/var/www/html
      - ./docker/nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      - app
    networks:
      - techworld_net

  db:
    image: postgres:15-alpine
    container_name: techworld_db
    restart: unless-stopped
    environment:
      POSTGRES_DB: techworld
      POSTGRES_USER: techworld
      POSTGRES_PASSWORD: change_me_strong
    volumes:
      - techworld_pg:/var/lib/postgresql/data
    networks:
      - techworld_net

networks:
  techworld_net:

volumes:
  techworld_pg:
```

Port binding breakdown for "127.0.0.1:8081:80":

- 127.0.0.1: bind only on localhost (not public)
- 8081: host port the reverse proxy connects to
- 80: container port where Nginx listens inside the container

## 4) Host Nginx reverse proxy

File:

```
/etc/nginx/sites-available/techworld.co.ke
```

```nginx
server {
    listen 80;
    server_name techworld.co.ke www.techworld.co.ke;

    location / {
        proxy_pass http://127.0.0.1:8081;
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

## 5) systemd auto-start (reusable template)

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

## 6) Deployment strategy (GitHub Actions + appleboy/ssh-action)

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

## 7) GitHub Actions example

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

## 8) Post-deployment checks

```bash
docker compose ps
docker compose logs -f
curl -I http://127.0.0.1:8081
```

## 9) Key principles

- One Linux user per project
- One compose stack per project
- Containers bound to 127.0.0.1
- Host Nginx handles public traffic and SSL
- systemd ensures auto-start on reboot
- Deploy via rolling replace (no docker compose down)

## 10) Notes on downtime

Current strategy provides:

- Minimal downtime (1-3 seconds)
- No blue/green complexity
- Safe and simple GitHub-based deployment

For strict zero-downtime, blue/green or load balancing is required.
