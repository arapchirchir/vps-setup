# 04 – Nginx Multi-Domain Setup

This section sets up **Nginx** to host multiple domains on a single VPS.

Example domains used:

- `example.com`
- `app.example.com`

> Replace these with your real domain names.
>
> This is the shared reference for **host-level Nginx**, **TLS termination**, and
> **Cloudflare origin certificates**. App-specific deployment guides should only
> override upstream ports and runtime-specific headers.

---

## 1) Install and start Nginx

```bash
sudo apt install -y nginx
sudo systemctl enable nginx
sudo systemctl start nginx
```

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

The examples in this section use Let's Encrypt certificate paths. If you use
Cloudflare Origin CA instead, keep the same server blocks and swap only the
certificate paths as shown in section 8.

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

Use this when the domain is proxied through **Cloudflare** and you want
Cloudflare to present the public certificate while Nginx uses an
**Origin CA certificate** for the Cloudflare-to-origin connection.

This is the canonical Cloudflare TLS section for this repository. Reuse it for
Laravel, Docker, and Phoenix deployments instead of repeating the same TLS steps
in each deployment document.

### 8.1) Prerequisites

- Your domain is proxied through Cloudflare
- You have access to the Cloudflare dashboard
- Cloudflare **SSL/TLS** mode is set to **Full (strict)**

### 8.2) Generate the Origin CA certificate

1. Log into **Cloudflare Dashboard**
2. Open **SSL/TLS** → **Origin Server**
3. Click **Create Certificate**
4. Keep the default settings unless you need custom hostnames
5. Include the hostnames you will serve, for example `example.com` and `*.example.com`
6. Copy the generated **Origin Certificate** and **Private Key**

### 8.3) Save the certificate on the VPS

Create a dedicated directory for origin certificates:

```bash
sudo mkdir -p /etc/nginx/ssl
```

Save the certificate:

```bash
sudo nano /etc/nginx/ssl/example.com.crt
```

Save the private key:

```bash
sudo nano /etc/nginx/ssl/example.com.key
```

Set the correct permissions:

```bash
sudo chmod 644 /etc/nginx/ssl/example.com.crt
sudo chmod 600 /etc/nginx/ssl/example.com.key
```

### 8.4) Update Nginx to use the Cloudflare certificate

For local PHP-FPM sites, only the certificate lines change:

```nginx
listen 443 ssl http2;
listen [::]:443 ssl http2;
ssl_certificate     /etc/nginx/ssl/example.com.crt;
ssl_certificate_key /etc/nginx/ssl/example.com.key;
```

For applications that Nginx proxies to a local upstream port, use this shared pattern:

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name example.com www.example.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name example.com www.example.com;

    ssl_certificate     /etc/nginx/ssl/example.com.crt;
    ssl_certificate_key /etc/nginx/ssl/example.com.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    location / {
        proxy_pass http://127.0.0.1:8001;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $http_cf_connecting_ip;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Port 443;
    }
}
```

Notes:

- Replace `127.0.0.1:8001` with the correct local port for the application
- Keep any app-specific headers such as `Upgrade` and `Connection` in the app deployment document
- If you are not using Cloudflare, use your normal upstream headers instead of `CF-Connecting-IP`

### 8.5) Laravel HTTPS awareness

If Laravel still generates `http://` asset or redirect URLs behind Cloudflare,
force the scheme outside local development:

```php
use Illuminate\Support\Facades\URL;

public function boot(): void
{
    if (config('app.env') !== 'local') {
        URL::forceScheme('https');
    }
}
```

Apply this once in `app/Providers/AppServiceProvider.php`.

### 8.6) Test and reload Nginx

```bash
sudo nginx -t
sudo systemctl reload nginx
```

### 8.7) Certificate renewal

Cloudflare Origin CA certificates are long-lived, but renewal is still manual:

- Before expiry, return to the Cloudflare dashboard
- Create a new Origin CA certificate
- Replace `/etc/nginx/ssl/example.com.crt` and `/etc/nginx/ssl/example.com.key`
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
