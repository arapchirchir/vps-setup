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

### Create a sudoers file for the user

```bash
sudo visudo -f /etc/sudoers.d/deploy
```

Add the following line:

```text
deploy ALL=(ALL:ALL) ALL
```

### Set correct permissions

```bash
sudo chmod 440 /etc/sudoers.d/deploy
```

### Validate sudoers configuration

```bash
sudo visudo -c
```
