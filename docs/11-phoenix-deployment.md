# 11 – Phoenix Production Deployment on VPS (Nginx + Cloudflare + GitHub Actions)

This document is a **copy‑paste friendly, battle‑tested guide** based on a real Phoenix application deployed successfully to a VPS.

It covers:

- Elixir/Phoenix production setup
- GitHub Actions CI/CD with SSH deploy
- asdf for version alignment
- PostgreSQL configuration
- systemd service
- Nginx reverse proxy
- Cloudflare + Origin / TLS handling

---

## 1) Server prerequisites

### OS

- Ubuntu 22.04 (works the same on 20.04 / 24.04)

### Required packages

```bash
sudo apt update
sudo apt install -y git curl unzip build-essential ca-certificates \
  libssl-dev libncurses-dev openssl nginx postgresql postgresql-contrib
```

---

## 2) Install asdf (Elixir + Erlang version alignment)

### Why asdf

- CI and server must use **exact same OTP + Elixir versions**
- Avoids apt version drift

### Install asdf

```bash
git clone https://github.com/asdf-vm/asdf.git ~/.asdf --branch v0.14.1

echo '. "$HOME/.asdf/asdf.sh"' >> ~/.bashrc
echo '. "$HOME/.asdf/completions/asdf.bash"' >> ~/.bashrc
source ~/.bashrc
```

### Install Erlang + Elixir

```bash
asdf plugin add erlang https://github.com/asdf-vm/asdf-erlang.git
asdf plugin add elixir https://github.com/asdf-vm/asdf-elixir.git

asdf install erlang 26.0
asdf global erlang 26.0

asdf install elixir 1.15.2-otp-26
asdf global elixir 1.15.2-otp-26

elixir -v
mix -v
```

---

## 3) PostgreSQL setup

### Create database and role

```bash
sudo -u postgres psql
```

```sql
CREATE ROLE tecworld WITH LOGIN PASSWORD '@Techworld2026.';
CREATE DATABASE tecworld_prod OWNER tecworld;
GRANT ALL PRIVILEGES ON DATABASE tecworld_prod TO tecworld;
\q
```

---

## 4) Phoenix `.env` (production)

Location:

```bash
/home/tecworld/tecworld/.env
```

```env
# Phoenix
MIX_ENV=prod
PHX_SERVER=true
PHX_HOST=techworld.co.ke
PORT=4000

# Secrets
SECRET_KEY_BASE=GENERATE_WITH_mix_phx.gen.secret

# Database (URL‑encoded password)
DATABASE_URL=postgres://tecworld:%40Techworld2026.@localhost:5432/tecworld_prod
```

Generate secret:

```bash
MIX_ENV=prod mix phx.gen.secret
```

---

## 5) Phoenix runtime config (`config/runtime.exs`)

Ensure **server is enabled via env**:

```elixir
if System.get_env("PHX_SERVER") do
  config :tecworld, TecworldWeb.Endpoint, server: true
end
```

Database config:

```elixir
database_url = System.get_env("DATABASE_URL") ||
  raise "DATABASE_URL is missing"

config :tecworld, Tecworld.Repo,
  url: database_url,
  pool_size: String.to_integer(System.get_env("POOL_SIZE") || "10")
```

---

## 6) Build & test release manually (first time)

```bash
cd /home/tecworld/tecworld
set -a
. .env
set +a

MIX_ENV=prod mix deps.get --only prod
MIX_ENV=prod mix compile
MIX_ENV=prod mix assets.deploy
MIX_ENV=prod mix ecto.migrate
MIX_ENV=prod mix release --overwrite
```

---

## 7) systemd service

Create service:

```bash
sudo nano /etc/systemd/system/tecworld.service
```

```ini
[Unit]
Description=Tecworld Phoenix App
After=network.target

[Service]
Type=simple
User=tecworld
Group=tecworld
WorkingDirectory=/home/tecworld/tecworld
EnvironmentFile=/home/tecworld/tecworld/.env
ExecStart=/home/tecworld/tecworld/_build/prod/rel/tecworld/bin/tecworld foreground
ExecStop=/home/tecworld/tecworld/_build/prod/rel/tecworld/bin/tecworld stop
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Enable & start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable tecworld
sudo systemctl start tecworld
```

Verify:

```bash
sudo ss -lntp | grep 4000
```

---

## 8) Nginx (Phoenix behind Cloudflare)

File:

```bash
sudo nano /etc/nginx/sites-available/techworld.co.ke
```

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name techworld.co.ke www.techworld.co.ke;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name techworld.co.ke www.techworld.co.ke;

    ssl_certificate     /etc/letsencrypt/live/techworld.co.ke/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/techworld.co.ke/privkey.pem;

    client_max_body_size 20m;

    location / {
        proxy_pass http://127.0.0.1:4000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $http_cf_connecting_ip;
        proxy_set_header X-Forwarded-For $http_cf_connecting_ip;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

Enable:

```bash
sudo ln -s /etc/nginx/sites-available/techworld.co.ke /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

---

## 9) GitHub Actions CI/CD (SSH deploy)

Key lessons:

- **Always load asdf manually** in SSH step
- Fail fast (`set -euo pipefail`)

```yaml
- name: Deploy via SSH
  uses: appleboy/ssh-action@v1.0.3
  with:
    host: ${{ secrets.SSH_HOST }}
    username: ${{ secrets.SSH_USERNAME }}
    key: ${{ secrets.SSH_PRIVATE_KEY }}
    port: ${{ secrets.SSH_PORT }}
    script: |
      cd /home/tecworld/tecworld
      set -euo pipefail

      export ASDF_DIR="$HOME/.asdf"
      . "$ASDF_DIR/asdf.sh"
      hash -r

      git pull origin master

      set -a
      . .env
      set +a

      MIX_ENV=prod mix deps.get --only prod
      MIX_ENV=prod mix compile
      MIX_ENV=prod mix assets.deploy
      MIX_ENV=prod mix ecto.migrate
      MIX_ENV=prod mix release --overwrite

      sudo systemctl restart tecworld
```

---

## 10) Verification checklist

```bash
sudo systemctl status tecworld
sudo ss -lntp | grep 4000
curl -I http://127.0.0.1:4000
curl -I https://techworld.co.ke
```

---

## Final notes

✅ This setup supports **multiple Phoenix apps on one VPS**  
✅ Cloudflare handles public TLS  
✅ systemd keeps the app alive  
✅ CI/CD is deterministic and repeatable  

This document is safe to **copy & reuse** for future Phoenix deployments.
