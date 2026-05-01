# 14 – Restoring a Hacked Ubuntu VPS

This guide walks you through the **full incident-response and restoration procedure**
for a compromised Ubuntu VPS server — from the moment you suspect a breach through
a hardened, production-ready rebuild.

> Assumes Ubuntu 22.04 / 24.04 LTS.

---

## 1) Immediate isolation

The first priority is to **stop the bleeding** — cut the attacker off before
wiping or changing anything.

### 1a) Disable inbound/outbound network access

If your hosting provider has a control panel (e.g. DigitalOcean, Hetzner, Vultr),
use it to apply an emergency firewall rule that blocks all traffic, or simply
power off the server from the panel.

If you still have SSH access:

```bash
# Drop all inbound traffic except your own management IP
sudo ufw default deny incoming
sudo ufw default deny outgoing
sudo ufw allow from YOUR_MANAGEMENT_IP to any port 22
sudo ufw --force enable
```

> Replace `YOUR_MANAGEMENT_IP` with your current public IP.

### 1b) Notify stakeholders

- Inform your team, manager, and any affected customers.
- Open a dedicated incident channel (Slack, email thread, etc.) and document
  every action taken with timestamps.
- Check whether any data-breach notification obligations apply (GDPR, local law).

---

## 2) Collect evidence before wiping

Do **not** wipe the server until you have preserved key forensic artefacts.
This helps you understand what happened and prevent a recurrence.

### 2a) Copy authentication and system logs off-server

From a **trusted** machine (not the compromised server):

```bash
# Create a local evidence directory
mkdir -p ~/incident-$(date +%Y%m%d)
cd ~/incident-$(date +%Y%m%d)

# Copy logs (adjust paths if needed)
scp -r deploy@SERVER_IP:/var/log/auth.log* .
scp -r deploy@SERVER_IP:/var/log/syslog* .
scp -r deploy@SERVER_IP:/var/log/nginx/ .
scp -r deploy@SERVER_IP:/var/log/postgresql/ .
scp -r deploy@SERVER_IP:/var/log/apt/ .
scp -r deploy@SERVER_IP:/home/ .              # user home directories
```

### 2b) Capture system state

Still from your trusted machine:

```bash
ssh deploy@SERVER_IP "sudo ss -tulpn"            > network-state.txt
ssh deploy@SERVER_IP "sudo ps auxf"              > process-tree.txt
ssh deploy@SERVER_IP "sudo crontab -l"           > root-crontab.txt
ssh deploy@SERVER_IP "sudo cat /etc/crontab"     >> root-crontab.txt
ssh deploy@SERVER_IP "sudo ls /etc/cron.d/"      >> root-crontab.txt
ssh deploy@SERVER_IP "sudo last -a -F"           > last-logins.txt
ssh deploy@SERVER_IP "sudo lastb -a -F"          > failed-logins.txt
ssh deploy@SERVER_IP "sudo find / -path /proc -prune -o -path /sys -prune -o -path /dev -prune -o -path /run -prune -o -mtime -3 -type f -print 2>/dev/null" > recently-modified.txt
ssh deploy@SERVER_IP "sudo dpkg --get-selections" > installed-packages.txt
ssh deploy@SERVER_IP "sudo cat /etc/passwd"      > passwd.txt
ssh deploy@SERVER_IP "sudo cat /etc/sudoers"     > sudoers.txt
```

### 2c) Snapshot / image the disk (optional but recommended)

Use your provider's snapshot feature to preserve the disk state before
destruction. This lets you revisit the image later if needed.

---

## 3) Wipe and reinstall Ubuntu from scratch

**Never attempt to clean a compromised server in-place.** Rootkits and
backdoors are designed to survive disinfection. Always rebuild from scratch.

### 3a) Destroy the existing server or reinstall via the control panel

- **Preferred:** Destroy the VPS and provision a brand-new server with a
  fresh Ubuntu 22.04 or 24.04 LTS image.
- **Alternative:** Use your provider's "Reinstall OS" function to wipe the
  disk and install a clean Ubuntu image.

### 3b) Immediately apply all system updates on the new server

```bash
sudo apt update && sudo apt full-upgrade -y
sudo reboot
```

---

## 4) Restore only clean backups of data and config

Restore **data only** — never restore entire system images or binaries from
the compromised server.

### 4a) Identify a known-good backup

- Use the oldest backup you are confident predates the compromise.
- Cross-check file timestamps against the `recently-modified.txt` log you
  collected in Step 2b.

### 4b) Restore application data

```bash
# Example: restore a PostgreSQL database from a clean dump
psql -U postgres < /path/to/clean_backup.sql

# Example: restore application files (not config or binaries)
rsync -av /path/to/clean/backup/app-data/ /var/www/html/
```

> **Do not** restore `.env` files, SSH keys, or any credentials from the
> compromised server — generate fresh ones in Step 5.

### 4c) Re-apply config files by hand

Recreate Nginx vhosts, PHP-FPM pools, and other configuration from your
documentation (or from this playbook) rather than restoring them from the
compromised server. This prevents you from unknowingly restoring a backdoor.

---

## 5) Rotate all credentials

Every secret that existed on the compromised server must be treated as leaked.

### 5a) SSH keys

```bash
# On your local machine — generate a new ED25519 key pair
ssh-keygen -t ed25519 -C "deploy-$(date +%Y%m%d)" -f ~/.ssh/id_ed25519_new

# Copy the new public key to the rebuilt server
ssh-copy-id -i ~/.ssh/id_ed25519_new.pub deploy@NEW_SERVER_IP
```

Remove any old public keys from `~/.ssh/authorized_keys` on the server:

```bash
cat ~/.ssh/authorized_keys   # review each entry
nano ~/.ssh/authorized_keys  # delete any keys you don't recognise
```

### 5b) System and application passwords

```bash
# Change the deploy user password
sudo passwd deploy

# Change the root password (even if root SSH login is disabled)
sudo passwd root
```

### 5c) Database credentials

```bash
# PostgreSQL example — rotate every application user
sudo -u postgres psql -c "ALTER USER appuser WITH PASSWORD 'NEW_STRONG_PASSWORD';"
```

Update the new password in your application's `.env` file.

### 5d) Application secrets and third-party tokens

- Regenerate `APP_KEY` / secret keys in every `.env` file.
- Revoke and re-issue all API keys, OAuth secrets, and webhook tokens.
- Update any secrets stored in GitHub Actions, CI/CD pipelines, or secret
  managers.

### 5e) TLS / SSL certificates

If the private key material for your TLS certificates may have been exposed,
revoke and reissue them:

```bash
# Re-issue Let's Encrypt certificates on the rebuilt server
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d example.com -d www.example.com
```

See [docs/07-letsencrypt.md](07-letsencrypt.md) for full details.

---

## 6) Apply system and application security updates

### 6a) Full system upgrade

```bash
sudo apt update && sudo apt full-upgrade -y
sudo reboot
```

### 6b) Enable automatic security updates

```bash
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

### 6c) Update application dependencies

```bash
# PHP / Composer example
composer update --no-dev

# Node.js example
npm audit fix

# Python example
pip install --upgrade -r requirements.txt
```

---

## 7) Post-incident report and lessons learned

Write a brief incident report while events are fresh. Store it in your team
wiki or repository. A good report includes:

| Section | Content |
|---------|---------|
| **Timeline** | When was the breach first suspected? When was it confirmed? When was the server isolated/rebuilt? |
| **Root cause** | How did the attacker get in? (weak password, unpatched CVE, leaked key, etc.) |
| **Impact** | What data or services were affected? Duration of outage? |
| **Actions taken** | List every step performed, with timestamps. |
| **Improvements** | What changes will prevent a recurrence? |
| **Lessons learned** | What worked well? What would you do differently? |

Create an `INCIDENT-YYYY-MM-DD.md` file and commit it to a **private** repository.

---

## 8) Harden and monitor the reinstalled server

Use the rest of this playbook to put the rebuilt server into a hardened state.

### 8a) Baseline security setup

Follow the full security baseline:
→ [docs/08-security-baseline.md](08-security-baseline.md)

Key controls to confirm:

```bash
# SSH hardening — no root login, key-only auth
sudo sshd -T | grep -iE "permitrootlogin|passwordauthentication|pubkeyauthentication"
```

Expected output:

```
permitrootlogin no
passwordauthentication no
pubkeyauthentication yes
```

See [docs/02-ssh-hardening.md](02-ssh-hardening.md) for the full SSH config.

### 8b) Firewall

```bash
sudo ufw status verbose
```

Only your SSH port, 80, and 443 should be open.
See [docs/03-firewall.md](03-firewall.md) for the full firewall setup.

### 8c) Fail2ban (brute-force protection)

```bash
sudo apt install -y fail2ban
sudo systemctl enable --now fail2ban
sudo fail2ban-client status
```

### 8d) Intrusion detection with rkhunter

```bash
sudo apt install -y rkhunter
sudo rkhunter --update
sudo rkhunter --check --skip-keypress
```

Store the baseline for future comparisons:

```bash
sudo rkhunter --propupd
```

### 8e) Audit logging (auditd)

```bash
sudo apt install -y auditd audispd-plugins
sudo systemctl enable --now auditd
```

### 8f) Ongoing monitoring

Review these logs regularly (see [docs/09-operations-and-maintenance.md](09-operations-and-maintenance.md)):

```bash
# Authentication events
sudo tail -n 100 /var/log/auth.log

# Fail2ban activity
sudo fail2ban-client status sshd

# Open ports (check for unexpected listeners)
sudo ss -tulpn

# Disk usage
df -h
```

Set up alerting on:

- Failed SSH logins (> 5 in a minute) — fail2ban handles this automatically once installed (see Step 8c above)
- Unexpected new listening ports — compare `ss -tulpn` output daily or use auditd socket-bind rules (Step 8e)
- Disk usage > 80 % — check with `df -h` or configure a cron alert
- Any `sudo` usage outside business hours — auditd logs all sudo activity to `/var/log/audit/audit.log`

See [docs/09-operations-and-maintenance.md](09-operations-and-maintenance.md) for routine monitoring commands.

---

## Result

✅ Server isolated and evidence preserved  
✅ Clean OS reinstalled from scratch  
✅ Only known-good data restored  
✅ All credentials rotated  
✅ System and apps fully patched  
✅ Incident documented and lessons recorded  
✅ Hardened and monitored server back in production
