# 09 – Operations & Maintenance

This section covers **day-to-day operations**, service management, logs,
updates, and routine health checks for the VPS.

---

## Service management

### Nginx

Test configuration and reload:

```bash
sudo nginx -t && sudo systemctl reload nginx
```

Restart (only if needed):

```bash
sudo systemctl restart nginx
```

---

### PHP-FPM (PHP 8.3)

Restart service:

```bash
sudo systemctl restart php8.3-fpm
```

Check status:

```bash
sudo systemctl status php8.3-fpm --no-pager -l
```

---

### PostgreSQL

Restart service:

```bash
sudo systemctl restart postgresql
```

Check status:

```bash
sudo systemctl status postgresql --no-pager -l
```

---

## Logs

### Nginx logs

Global logs:

```bash
sudo tail -n 200 /var/log/nginx/error.log
sudo tail -n 200 /var/log/nginx/access.log
```

Per-site logs (if configured):

```bash
sudo tail -n 200 /var/log/nginx/example.com.error.log
sudo tail -n 200 /var/log/nginx/example.com.access.log
```

---

### Systemd logs

```bash
sudo journalctl -u nginx --no-pager -n 200
sudo journalctl -u php8.3-fpm --no-pager -n 200
sudo journalctl -u postgresql --no-pager -n 200
```

---

## Log rotation

Ensure log rotation runs daily:

```bash
sudo systemctl status logrotate.timer --no-pager -l
```

---

## Backups and restore testing

- Encrypt backups at rest
- Store offsite
- Perform a test restore monthly

---

## Monitoring (minimum)

- Review authentication failures in `/var/log/auth.log`
- Review Nginx error logs for spikes or 4xx/5xx bursts
- Alert on disk usage > 80%

---

## System updates

### Safe upgrade procedure

```bash
sudo apt update && sudo apt upgrade -y
sudo reboot
```

After reboot, verify services:

```bash
sudo systemctl status nginx php8.3-fpm postgresql --no-pager -l
```

---

## SSL / Certbot maintenance

List installed certificates:

```bash
sudo certbot certificates
```

Test renewal:

```bash
sudo certbot renew --dry-run
```

---

## Health checks

### Open ports

```bash
sudo ss -tulpn
```

### Disk usage

```bash
df -h
```

### Memory usage

```bash
free -h
```

---

## Quick availability checks

```bash
curl -I https://example.com
curl -I https://app.example.com
```

---

## Result

✅ Predictable operations  
✅ Easy troubleshooting  
✅ Safe upgrade and maintenance workflow