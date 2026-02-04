# 03 – Firewall (UFW)

This section configures a basic but secure firewall using UFW (Uncomplicated Firewall).

## Goal

- Block all unsolicited inbound traffic
- Allow only SSH, HTTP, and HTTPS

---

## 1) Install UFW

```bash
sudo apt install -y ufw
````

---

## 2) Set safe defaults

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

---

## 3) Allow required services

Allow SSH (must be done before enabling UFW):

```bash
sudo ufw allow OpenSSH
```

Allow web traffic:

```bash
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
```

If you are running a mail server (Mailcow), allow mail ports:

```bash
sudo ufw allow 25/tcp
sudo ufw allow 465/tcp
sudo ufw allow 587/tcp
sudo ufw allow 993/tcp
sudo ufw allow 995/tcp
sudo ufw allow 4190/tcp
```

---

## 4) Enable the firewall

```bash
sudo ufw enable
```

When prompted, type `y`.

---

## 5) Verify status

```bash
sudo ufw status verbose
```

Expected rules:

- OpenSSH → ALLOW
- 80/tcp → ALLOW
- 443/tcp → ALLOW

---

## Notes & Best Practices

- Do NOT expose databases (PostgreSQL, MySQL) publicly unless absolutely required
- Use SSH tunneling or private networking for admin access
- Any new service must be explicitly allowed through UFW

---

## Result

✅ Firewall enabled  
✅ Default deny inbound  
✅ Only essential ports exposed
