# 08 – Security Baseline

This document summarizes the **minimum security controls** applied on the VPS
and highlights recommended hardening steps.

This baseline is suitable for **production workloads**.

---

## Security controls already implemented

### Access & Identity
- ✅ Non-root user (`deploy`) for daily operations
- ✅ Root SSH login disabled
- ✅ SSH key–based authentication only
- ✅ Password authentication disabled
- ✅ SSH access limited to approved users

### Network
- ✅ UFW firewall enabled
- ✅ Default deny for inbound traffic
- ✅ Only ports 22 (SSH), 80 (HTTP), 443 (HTTPS) exposed

### Web & Transport
- ✅ Nginx reverse proxy
- ✅ HTTPS enforced using Let’s Encrypt
- ✅ Automatic SSL renewal enabled

### Application & Services
- ✅ PHP-FPM isolated from Nginx
- ✅ PostgreSQL bound to localhost by default
- ✅ Role-based database access
- ✅ Correct schema privileges applied

---

## Recommended additional hardening (enable for production)

### 1) Automatic security updates

```bash
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure --priority=low unattended-upgrades
````

Ensures critical security patches are applied automatically.

---

### 2) Fail2ban (brute-force protection)

```bash
sudo apt install -y fail2ban
sudo systemctl enable --now fail2ban
```

Check status:

```bash
sudo fail2ban-client status
```

---

### 3) Audit logging (auditd)

```bash
sudo apt install -y auditd audispd-plugins
sudo systemctl enable --now auditd
```

---

### 4) Log rotation

Ensure log rotation is active:

```bash
sudo systemctl status logrotate.timer --no-pager -l
```

---

### 5) Backups with encryption + restore test

- Encrypt backups at rest
- Store offsite
- Test restore regularly

---

### 6) Web security headers (Nginx)

Enable security headers in Nginx HTTPS blocks:

- HSTS
- X-Frame-Options
- X-Content-Type-Options
- Referrer-Policy
- Permissions-Policy

---

### 7) AppArmor (process confinement)

Ubuntu ships with AppArmor enabled by default. Confirm status:

```bash
sudo aa-status
```

---

### 8) Reduce attack surface

- Avoid exposing databases publicly
- Remove unused packages
- Regularly review open ports:

```bash
sudo ss -tulpn
```

---

## Routine security checks

### Check sudo users

```bash
getent group sudo
```

### Check SSH service

```bash
sudo systemctl status ssh --no-pager -l
```

### Check firewall

```bash
sudo ufw status verbose
```

---

## Result

✅ Strong baseline security applied  
✅ Low attack surface  
✅ Ready for production workloads