# 07 – Let’s Encrypt SSL (Certbot + Nginx)

This section secures your domains with **free SSL certificates** from Let’s Encrypt
using Certbot’s Nginx integration.

---

## Prerequisites (must be true)

Before continuing, confirm:

- Domain DNS A records point to the VPS IP
- Nginx server blocks are working on port 80
- Firewall allows ports 80 and 443

Check DNS:

```bash
dig +short example.com
dig +short app.example.com
````

They must return your server IP.

---

## 1) Install Certbot

```bash
sudo apt install -y certbot python3-certbot-nginx
```

Verify installation:

```bash
certbot --version
```

---

## 2) Issue SSL certificate for a domain

Example for main domain:

```bash
sudo certbot --nginx -d example.com -d www.example.com
```

Follow prompts:

- Enter email
- Agree to terms
- Allow HTTP → HTTPS redirect (recommended)

---

## 3) Issue SSL certificate for another domain

```bash
sudo certbot --nginx -d app.example.com
```

---

## 4) Verify HTTPS is working

```bash
curl -I https://example.com
curl -I https://app.example.com
```

You should see `HTTP/2 200` or `HTTP/1.1 200`.

---

## 5) Test automatic renewal

```bash
sudo certbot renew --dry-run
```

Certbot installs a systemd timer automatically.

Check it:

```bash
systemctl list-timers | grep certbot
```

---

## Troubleshooting

### Certbot fails domain validation

- DNS not propagated
- Wrong server block
- Port 80 blocked by firewall

### View Certbot logs

```bash
sudo journalctl -u certbot --no-pager -n 200
```

---

## Result

✅ HTTPS enabled  
✅ Certificates auto-renew  
✅ HTTP redirected to HTTPS