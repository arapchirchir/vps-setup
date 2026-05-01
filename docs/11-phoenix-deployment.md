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
CREATE ROLE appuser WITH LOGIN PASSWORD 'StrongPasswordHere';
CREATE DATABASE appname_prod OWNER appuser;
GRANT ALL PRIVILEGES ON DATABASE appname_prod TO appuser;
\q
```

> Replace `appuser`, `StrongPasswordHere`, and `appname_prod` with your actual values.

---

## 4) Phoenix `.env` (production)

Location:

```bash
/home/appuser/appname/.env
```

```env
# Phoenix
MIX_ENV=prod
PHX_SERVER=true
PHX_HOST=yourdomain.com
PORT=4000

# Secrets
SECRET_KEY_BASE=GENERATE_WITH_mix_phx.gen.secret

# Database (URL-encoded password)
DATABASE_URL=postgresql://appuser:StrongPasswordHere@localhost:5432/appname_prod
```

> Replace `appuser`, `appname`, `yourdomain.com`, and `StrongPasswordHere` with your actual values.

Generate secret:

```bash
MIX_ENV=prod mix phx.gen.secret
```

---

## 5) Phoenix runtime config (`config/runtime.exs`)

Ensure **server is enabled via env**:

```elixir
if System.get_env("PHX_SERVER") do
  config :myapp, MyAppWeb.Endpoint, server: true
end
```

Database config:

```elixir
database_url = System.get_env("DATABASE_URL") ||
  raise "DATABASE_URL is missing"

config :myapp, MyApp.Repo,
  url: database_url,
  pool_size: String.to_integer(System.get_env("POOL_SIZE") || "10")
```

> Replace `:myapp`, `MyAppWeb.Endpoint`, and `MyApp.Repo` with your actual OTP app name and module names.

---

## 6) Build & test release manually (first time)

```bash
cd /home/appuser/appname
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
sudo nano /etc/systemd/system/appname.service
```

```ini
[Unit]
Description=MyApp Phoenix App
After=network.target

[Service]
Type=simple
User=appuser
Group=appuser
WorkingDirectory=/home/appuser/appname
EnvironmentFile=/home/appuser/appname/.env
ExecStart=/home/appuser/appname/_build/prod/rel/appname/bin/appname foreground
ExecStop=/home/appuser/appname/_build/prod/rel/appname/bin/appname stop
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

> Replace `appuser`, `appname`, and `MyApp` with your actual values.

Enable & start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable appname
sudo systemctl start appname
```

Verify:

```bash
sudo ss -lntp | grep 4000
```

---

## 8) Nginx (Phoenix behind Cloudflare)

Use [04-nginx-multi-domain.md](04-nginx-multi-domain.md) as the shared
reference for Cloudflare Origin CA certificate creation and the base TLS layout.
The Phoenix-specific details here are the upstream port and websocket upgrade
headers.

File:

```bash
sudo nano /etc/nginx/sites-available/yourdomain.com
```

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name yourdomain.com www.yourdomain.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    listen [::]:443 ssl;
    http2 on;
    server_name yourdomain.com www.yourdomain.com;

    ssl_certificate     /etc/nginx/ssl/yourdomain.com.crt;
    ssl_certificate_key /etc/nginx/ssl/yourdomain.com.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    client_max_body_size 20m;

    location / {
        proxy_pass http://127.0.0.1:4000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $http_cf_connecting_ip;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Port 443;
        # Phoenix LiveView / WebSocket support
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

> Replace `yourdomain.com` and the certificate paths with your actual domain and certificate locations.
> See [04-nginx-multi-domain.md](04-nginx-multi-domain.md) for the full Nginx version note on `http2 on;`.

Enable:

```bash
sudo ln -s /etc/nginx/sites-available/yourdomain.com /etc/nginx/sites-enabled/
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
  # Pin to a full commit SHA in production to prevent supply-chain attacks.
  # Check the latest SHA at: https://github.com/appleboy/ssh-action/releases
  uses: appleboy/ssh-action@v1.0.3
  with:
    host: ${{ secrets.SSH_HOST }}
    username: ${{ secrets.SSH_USERNAME }}
    key: ${{ secrets.SSH_PRIVATE_KEY }}
    port: ${{ secrets.SSH_PORT }}
    script: |
      cd /home/appuser/appname
      set -euo pipefail

      export ASDF_DIR="$HOME/.asdf"
      . "$ASDF_DIR/asdf.sh"
      hash -r

      git pull origin main

      set -a
      . .env
      set +a

      MIX_ENV=prod mix deps.get --only prod
      MIX_ENV=prod mix compile
      MIX_ENV=prod mix assets.deploy
      MIX_ENV=prod mix ecto.migrate
      MIX_ENV=prod mix release --overwrite

      sudo systemctl restart appname
```

---

## 10) Verification checklist

```bash
sudo systemctl status appname
sudo ss -lntp | grep 4000
curl -I http://127.0.0.1:4000
curl -I https://yourdomain.com
```

---

## Final notes

✅ This setup supports **multiple Phoenix apps on one VPS**  
✅ Cloudflare handles public TLS  
✅ systemd keeps the app alive  
✅ CI/CD is deterministic and repeatable  

This document is safe to **copy & reuse** for future Phoenix deployments.
