# 10 – Mailcow Installation Behind Nginx Reverse Proxy (Certbot TLS)

This is a public, reusable guide for installing **Mailcow** on a server already running **Nginx** for other websites.

## 0) Assumptions

- Domain: `example.com`
- Mail host: `mail.example.com`
- Autoconfig/autodiscover:
  - `autoconfig.example.com`
  - `autodiscover.example.com`
- You have root access and control of DNS
- You can set PTR (reverse DNS) at your VPS provider
- Optional: you will host multiple domains on the same Mailcow instance

---

## 1) DNS Setup (Do this first)

Create records (DNS-only if using Cloudflare):

| Type | Name | Value | Notes |
|---|---|---|---|
| A | mail | SERVER_IP | DNS only |
| A | autodiscover | SERVER_IP | DNS only |
| A | autoconfig | SERVER_IP | DNS only |
| MX | @ | mail.example.com | Priority 10 |
| TXT | @ | v=spf1 ip4:SERVER_IP -all | SPF baseline |

> Later you will add DKIM and DMARC.
> If you plan to host multiple domains, each additional domain will get its own MX/SPF/DKIM/DMARC.

---

## 2) Install Docker (Official Repository)

> Don't use Snap.

```bash
apt update -y
apt install -y ca-certificates curl gnupg lsb-release

install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
chmod a+r /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" \
> /etc/apt/sources.list.d/docker.list

apt update -y
apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
systemctl enable docker
systemctl start docker

docker --version
docker compose version
```

> Security note: avoid adding non-admin users to the `docker` group. Docker group membership is equivalent to root.

---

---

## 3) Clone Mailcow

```bash
cd /opt
git clone https://github.com/mailcow/mailcow-dockerized.git
cd mailcow-dockerized
```

---

## 4) Generate Mailcow Config

```bash
./generate_config.sh
```

When prompted, set:

- Hostname: `mail.example.com`

---

## 5) Configure Mailcow for Reverse Proxy

Host Nginx already uses 80/443. Mailcow UI must bind to alternate ports.

Edit:

```bash
nano mailcow.conf
```

Set/confirm:

```ini
MAILCOW_HOSTNAME=mail.example.com

# Choose ports that are not used on the host
HTTP_PORT=8180
HTTPS_PORT=18443

# TLS is handled by host Nginx + Certbot
SKIP_LETS_ENCRYPT=y
ENABLE_SSL_SNI=n
```

> If `8180` is used on your host, change it (e.g. `8280`).

---

## 6) Start Mailcow

```bash
docker compose pull
docker compose up -d
docker compose ps
```

Confirm:

- `nginx-mailcow` is running
- ports `8180` and `18443` are listening

---

## 7) Install Certbot (Nginx Plugin)

```bash
apt update -y
apt install -y certbot python3-certbot-nginx
certbot --version
```

---

## 8) Temporary Nginx HTTP vhost (for Certbot issuance)

Create:

```bash
nano /etc/nginx/sites-available/mail.example.com
```

Content:

```nginx
server {
    listen 80;
    server_name mail.example.com autodiscover.example.com autoconfig.example.com;

    location / {
        return 200 "OK\n";
    }
}
```

Enable + reload:

```bash
ln -s /etc/nginx/sites-available/mail.example.com /etc/nginx/sites-enabled/
nginx -t && systemctl reload nginx
```

---

## 9) Issue Let's Encrypt Certificates (Certbot)

```bash
certbot --nginx \
  -d mail.example.com \
  -d autodiscover.example.com \
  -d autoconfig.example.com
```

Choose redirect: **Yes**.

Verify:

```bash
ls -lah /etc/letsencrypt/live/mail.example.com/
```

---

## 10) Replace vhost with Reverse Proxy

Edit:

```bash
nano /etc/nginx/sites-available/mail.example.com
```

Replace the **entire file** with the following complete vhost (both HTTP and HTTPS blocks).
Certbot has already placed the certificate files; we now configure Nginx to
proxy both plain HTTP and HTTPS traffic to Mailcow.

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name mail.example.com autodiscover.example.com autoconfig.example.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    listen [::]:443 ssl;
    http2 on;
    server_name mail.example.com autodiscover.example.com autoconfig.example.com;

    ssl_certificate     /etc/letsencrypt/live/mail.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/mail.example.com/privkey.pem;
    include             /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam         /etc/letsencrypt/ssl-dhparams.pem;

    location / {
        proxy_pass http://127.0.0.1:8180;
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Required for Mailcow / SOGo WebSocket and long-polling
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        # Allow large email attachments
        client_max_body_size 0;
    }
}
```

Reload:

```bash
nginx -t && systemctl reload nginx
```

---

## 11) Verify Mail Ports

```bash
ss -tulpen | grep -E ':(25|465|587|993|995|4190|8180|18443)\s'
```

Expected open ports:

- 25, 465, 587 (SMTP)
- 993 (IMAPS)
- 995 (POP3S)
- 4190 (Sieve)
- 8180/18443 (Mailcow UI internal)

---

## 12) Firewall rules (UFW)

Allow mail-related ports:

```bash
sudo ufw allow 25/tcp
sudo ufw allow 465/tcp
sudo ufw allow 587/tcp
sudo ufw allow 993/tcp
sudo ufw allow 995/tcp
sudo ufw allow 4190/tcp
```

Verify:

```bash
sudo ufw status verbose
```

---

## 13) Access Mailcow UI

Open:

- `https://mail.example.com`

---

## 14) Reset Admin Password

```bash
cd /opt/mailcow-dockerized
./helper-scripts/mailcow-reset-admin.sh
```

Login:

- Username: `admin`
- Password: your new password

---

## 15) Add Domain + Mailboxes

Mailcow UI:

- Configuration → Mail Setup → Domains → Add `example.com`
- Then add mailboxes

### Additional domains (shared MX)

You can host multiple domains on the same Mailcow instance by pointing them at the same mail host.

Mailcow UI:

- Configuration → Mail Setup → Domains → Add `second-domain.tld`
- Enable DKIM for each domain

DNS for the additional domain:

```dns
MX  second-domain.tld  -> mail.example.com
TXT second-domain.tld  -> v=spf1 mx a:mail.example.com ip4:SERVER_IP -all
TXT dkim._domainkey.second-domain.tld -> (Mailcow DKIM key)
TXT _dmarc.second-domain.tld -> v=DMARC1; p=reject; rua=mailto:dmarc@second-domain.tld; adkim=s; aspf=s
```

---

## 16) DKIM + SPF + DMARC (Deliverability)

Mailcow UI:

- Configuration → ARC/DKIM → Generate DKIM
- Add DKIM TXT to DNS

DMARC TXT (recommended baseline):

```txt
v=DMARC1; p=quarantine; rua=mailto:dmarc@example.com; ruf=mailto:dmarc@example.com; fo=1
```

After confidence, move to `p=reject`:

```txt
v=DMARC1; p=reject; rua=mailto:dmarc@example.com; fo=1
```

---

## 17) PTR (Reverse DNS)

Set PTR at your VPS provider:

- PTR: `mail.example.com`
- A record must resolve `mail.example.com -> SERVER_IP`

---

## 18) Updates

Mailcow:

```bash
cd /opt/mailcow-dockerized
./update.sh
```

System:

```bash
apt update && apt upgrade -y
```

Certbot renew test:

```bash
certbot renew --dry-run
```

---

## 19) Backups

```bash
cd /opt/mailcow-dockerized
./helper-scripts/backup.sh
```

Restore:

```bash
./helper-scripts/restore.sh
```

For a full disaster recovery playbook and offsite strategy, see [09-operations-and-maintenance.md](09-operations-and-maintenance.md).

---

## 20) Troubleshooting (quick)

- Port conflicts: `ss -tulpen | grep :PORT`
- Containers: `docker compose ps`
- Logs: `docker compose logs -f postfix-mailcow`

---

## Result

✅ Mailcow installed and running  
✅ Behind Nginx reverse proxy  
✅ HTTPS with Certbot SSL  
✅ Mail ports configured  
✅ Ready for multiple domains
