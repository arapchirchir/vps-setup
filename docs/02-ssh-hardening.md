# 02 – SSH Hardening

This section secures SSH access by:

- Enforcing SSH key authentication
- Disabling root login
- Disabling password-based login
- Restricting SSH access to a specific user

> ⚠️ **Warning:** Do NOT close your current SSH session until you confirm a new one works.

---

## 1) Copy SSH key to the server

Run this on your **local machine**:

```bash
ssh-copy-id deploy@SERVER_IP
```

Test login:

```bash
ssh deploy@SERVER_IP
```

You should log in **without a password**.

---

## 2) Harden SSH configuration

On the server (logged in as `deploy`):

```bash
sudo nano /etc/ssh/sshd_config
```

Ensure the following settings exist (add or update them):

```ini
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
X11Forwarding no
AllowUsers deploy
PermitEmptyPasswords no
KbdInteractiveAuthentication no
MaxAuthTries 3
MaxSessions 2
LoginGraceTime 30
ClientAliveInterval 300
ClientAliveCountMax 2
Port XXX
```

Save and exit.

---

## 3) Restart SSH service

```bash
sudo systemctl restart ssh
```

---

## 4) Validate before closing your session

Open a **new terminal window** and test:

```bash
ssh deploy@SERVER_IP
```

Confirm root login is blocked:

```bash
ssh root@SERVER_IP
```

Expected result: **access denied**

---

## 5) Common recovery (if locked out)

If you lose SSH access:

- Use your VPS provider web console
- Revert changes in `/etc/ssh/sshd_config`
- Restart SSH:

```bash
sudo systemctl restart ssh
```
## 6) PORT fixation if changing port does not work
```
sudo systemctl edit ssh.socket
```
Add the port override

```
[Socket]
ListenStream=
ListenStream=2001
```
Reload and Restart
```
sudo systemctl daemon-reload
sudo systemctl restart ssh.socket
```
---

## Result

✅ Root SSH login disabled  
✅ Password authentication disabled  
✅ SSH keys enforced  
✅ Access limited to approved users only
