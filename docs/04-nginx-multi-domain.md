# 04 – Nginx Multi-Domain Setup

This section sets up **Nginx** to host multiple domains on a single VPS.

Example domains used:

- `example.com`
- `app.example.com`

> Replace these with your real domain names.

---

## 1) Install and start Nginx

```bash
sudo apt install -y nginx
sudo systemctl enable nginx
sudo systemctl start nginx
````

Verify Nginx is running:

```bash
curl -I http://SERVER_IP
```

---

## 2) Create web root directories and set permissions

For each application user, create the project directory:

```bash
sudo mkdir -p /home/exampleuser/exampleapp/public
```

Set ownership and permissions:

```bash
sudo chown -R exampleuser:www-data /home/exampleuser/exampleapp
sudo find /home/exampleuser/exampleapp -type d -exec chmod 755 {} \;
sudo find /home/exampleuser/exampleapp -type f -exec chmod 644 {} \;
sudo chmod 755 /home/exampleuser
```

Make storage and cache writable by both user and Nginx:

```bash
sudo chown -R exampleuser:www-data /home/exampleuser/exampleapp/storage /home/exampleuser/exampleapp/bootstrap/cache
sudo chmod -R 775 /home/exampleuser/exampleapp/storage /home/exampleuser/exampleapp/bootstrap/cache
```

> Replace `exampleuser` and `exampleapp` with your actual application username and directory.

---

## 3) Create Nginx server blocks

### 3.1 example.com

```bash
sudo nano /etc/nginx/sites-available/example.com
```

```nginx
server {
    server_name example.com www.example.com;
    client_max_body_size 500M;

    root /home/exampleuser/exampleapp/public;
    index index.php index.html index.htm;

    access_log /var/log/nginx/example.com.access.log;
    error_log  /var/log/nginx/example.com.error.log;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.3-fpm.sock;
    }

    location ~ /\. {
        deny all;
    }

    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Permissions-Policy "geolocation=(), microphone=(), camera=()" always;

    if ($host = www.example.com) {
        return 301 https://example.com$request_uri;
    }
}

server {
    listen 80;
    listen [::]:80;
    server_name example.com www.example.com;
    return 301 https://example.com$request_uri;
}
```

Enable site:

```bash
sudo ln -s /etc/nginx/sites-available/example.com /etc/nginx/sites-enabled/
```

---

### 3.2 app.example.com

```bash
sudo nano /etc/nginx/sites-available/app.example.com
```

```nginx
server {
    server_name app.example.com;
    client_max_body_size 500M;

    root /home/exampleuser/exampleapp/public;
    index index.php index.html index.htm;

    access_log /var/log/nginx/app.example.com.access.log;
    error_log  /var/log/nginx/app.example.com.error.log;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.3-fpm.sock;
    }

    location ~ /\. {
        deny all;
    }

    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    ssl_certificate /etc/letsencrypt/live/app.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/app.example.com/privkey.pem;

    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Permissions-Policy "geolocation=(), microphone=(), camera=()" always;
}

server {
    listen 80;
    listen [::]:80;
    server_name app.example.com;
    return 301 https://app.example.com$request_uri;
}
```

Enable site:

```bash
sudo ln -s /etc/nginx/sites-available/app.example.com /etc/nginx/sites-enabled/
```

---

## 4) Disable default Nginx site (recommended)

```bash
sudo rm -f /etc/nginx/sites-enabled/default
```

---

## 5) Test and reload Nginx

```bash
sudo nginx -t
sudo systemctl reload nginx
```

---

## 6) DNS prerequisite (important)

Before SSL:

- Domain A records **must** point to the server IP
- Test with:

```bash
dig +short example.com
dig +short app.example.com
```

---

## 7) SSL Certificates: Option A - Let's Encrypt (Certbot)

See [07-letsencrypt.md](07-letsencrypt.md) for full Let's Encrypt setup with Certbot.

---

## 8) SSL Certificates: Option B - Cloudflare Origin CA

This option allows you to use **Cloudflare-issued certificates** instead of Let's Encrypt.

### Prerequisites

- Your domain is proxied through Cloudflare
- You have access to Cloudflare dashboard
- Cloudflare SSL/TLS is set to **Full** or **Full (strict)**

### 8.1) Generate Cloudflare Origin CA Certificate

1. Log into **Cloudflare Dashboard**
2. Navigate to **SSL/TLS** → **Origin Server**
3. Click **Create Certificate**
4. Select:
   - Certificate format: **PEM** (for Nginx)
   - Hostname: `example.com` and `*.example.com` (wildcard for all subdomains)
5. Copy the **certificate** and **private key**

### 8.2) Save certificates on server

```bash
sudo mkdir -p /etc/cloudflare/certs
```

Create certificate file:

```bash
sudo nano /etc/cloudflare/certs/example.com.crt
```

Paste the **certificate** content, save and exit.

Create private key file:

```bash
sudo nano /etc/cloudflare/certs/example.com.key
```

Paste the **private key** content, save and exit.

Set permissions:

```bash
sudo chmod 600 /etc/cloudflare/certs/example.com.key
sudo chmod 644 /etc/cloudflare/certs/example.com.crt
```

### 8.3) Update Nginx server blocks for Cloudflare certificates

Edit your Nginx config:

```bash
sudo nano /etc/nginx/sites-available/example.com
```

Update the SSL directives to use Cloudflare certificates:

```nginx
listen 443 ssl http2;
listen [::]:443 ssl http2;
ssl_certificate     /etc/cloudflare/certs/example.com.crt;
ssl_certificate_key /etc/cloudflare/certs/example.com.key;
```

### 8.4) Test and start Nginx

```bash
sudo nginx -t
sudo systemctl reload nginx
```

### 8.5) Certificate renewal

Cloudflare Origin CA certificates are valid for **15 years**. Renewal is manual:

- Before expiry, return to Cloudflare dashboard
- Create a new certificate
- Update `/etc/cloudflare/certs/` files
- Reload Nginx

---

## 9) Queue Worker Service (Optional - for Laravel)

If your application uses Laravel queues, create a systemd service file:

```bash
sudo nano /etc/systemd/system/exampleapp-queue.service
```

Add:

```ini
[Unit]
Description=Laravel Queue Worker
After=network.target

[Service]
User=exampleuser
Group=www-data
Restart=always
WorkingDirectory=/home/exampleuser/exampleapp
ExecStart=/usr/bin/php artisan queue:work --sleep=3 --tries=3

[Install]
WantedBy=multi-user.target
```

Enable and start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable exampleapp-queue
sudo systemctl start exampleapp-queue
sudo systemctl status exampleapp-queue
```

---

## 10) Laravel Scheduler (Optional - via Cron)

If your application uses Laravel scheduler:

```bash
crontab -e
```

Add:

```cron
* * * * * cd /home/exampleuser/exampleapp && php artisan schedule:run >> /dev/null 2>&1
```

---

## Result

✅ Multiple domains hosted on one VPS  
✅ Clean per-domain logs  
✅ HTTPS with automatic redirects  
✅ Production-ready Nginx configuration  
✅ Ready for Laravel queue workers (optional)