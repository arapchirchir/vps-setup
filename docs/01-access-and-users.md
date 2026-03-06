# 01 – Initial Access & Users

This section covers the first-time login to a fresh VPS, basic system prep, and creating a non-root sudo user.

> Use placeholder values (SERVER_IP, usernames) and replace them with your real values.

---

## 1) First login as root (initial only)

From your local machine:

```bash
ssh root@SERVER_IP
```

---

## 2) Update the server

On the server:

```bash
apt update && apt upgrade -y
```

Install a few helpful basics you’ll use later:

```bash
apt install -y curl unzip git ca-certificates lsb-release software-properties-common
```

---

## 3) Set timezone (recommended)

```bash
timedatectl set-timezone Africa/Nairobi
timedatectl status
timedatectl set-ntp true
```

---

## 4) Create a non-root user

We’ll use `deploy` as an example username.

```bash
adduser deploy
```

Add the user to the sudo group:

```bash
usermod -aG sudo deploy
```

Verify group membership:

```bash
id deploy
getent group sudo
```

---

## 5) Test sudo works

Switch into the new user:

```bash
su - deploy
```

Test sudo:

```bash
sudo whoami
```

Expected output:

```text
root
```

## 6) Explicit sudo permissions (sudoers.d)

Instead of relying only on group membership, we define sudo access explicitly
using `/etc/sudoers.d`. This approach is safer, clearer, and easier to audit.

### Option A: Full sudo access

```bash
sudo visudo -f /etc/sudoers.d/deploy
```

Add the following line for unrestricted access:

```text
deploy ALL=(ALL:ALL) ALL
```

This grants the user full sudo access to all commands (with password prompt).

### Option B: Partial sudo access (recommended for specific tasks)

For security, you can limit sudo to specific commands without a password prompt:

```bash
sudo visudo -f /etc/sudoers.d/deploy
```

Add:

```text
deploy ALL=(ALL:ALL) NOPASSWD: /bin/systemctl
```

Or for even more granular control, specific commands only:

```text
deploy ALL=(ALL) NOPASSWD: /usr/bin/systemctl nginx reload
deploy ALL=(ALL) NOPASSWD: /usr/bin/systemctl nginx status
deploy ALL=(ALL) NOPASSWD: /usr/bin/systemctl php8.3-fpm restart
deploy ALL=(ALL) NOPASSWD: /usr/bin/systemctl postgresql restart
```

### Set correct permissions

```bash
sudo chown root:root /etc/sudoers.d/deploy
sudo chmod 440 /etc/sudoers.d/deploy
```

> `chown` sets ownership to root:root, `chmod` sets permissions to 440 (read-only for owner/group)

### Optional sudo hardening

Limit how long sudo stays authenticated (in minutes):

```text
Defaults:deploy timestamp_timeout=5
```

### Validate sudoers configuration

```bash
sudo visudo -c
```

---

## 7) Sudo usage examples

### Full access (Option A)

```bash
sudo whoami
# Prompts for password
```

### Partial access (Option B)

```bash
sudo /bin/systemctl nginx reload
# No password prompt (if configured with NOPASSWD)

sudo /bin/systemctl nginx status
# No password prompt (if configured with NOPASSWD)

sudo apt install -y somepackage
# Denied (not in allowed list)
```
