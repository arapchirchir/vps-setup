# 06 – PostgreSQL Setup

This section installs PostgreSQL, creates a database and user, and applies
the required permissions to avoid common `permission denied for schema public`
errors (especially with Laravel migrations).

---

## 1) Install PostgreSQL

```bash
sudo apt install -y postgresql postgresql-contrib
```

Verify service status:

```bash
sudo systemctl status postgresql --no-pager -l
```

---

## 2) Create database and user

Switch to the postgres system user:

```bash
sudo -u postgres psql
```

Create database and user (example values):

```sql
CREATE DATABASE exampledb;
CREATE USER exampleuser WITH PASSWORD 'StrongPasswordHere';

-- Set ownership to avoid permission issues
ALTER DATABASE exampledb OWNER TO exampleuser;

GRANT ALL PRIVILEGES ON DATABASE exampledb TO exampleuser;
\q
```
### Changing a password

```sql
\password db_user
```

---

## 3) Fix public schema permissions (IMPORTANT)

This step prevents migration failures.

Connect to the database:

```bash
sudo -u postgres psql exampledb
```

Run:

```sql
GRANT ALL ON SCHEMA public TO exampleuser;

ALTER DEFAULT PRIVILEGES IN SCHEMA public
GRANT ALL ON TABLES TO exampleuser;

ALTER DEFAULT PRIVILEGES IN SCHEMA public
GRANT ALL ON SEQUENCES TO exampleuser;
\q
```

---

## 4) Application connection variables (example)

```env
DB_CONNECTION=pgsql
DB_HOST=127.0.0.1
DB_PORT=5432
DB_DATABASE=exampledb
DB_USERNAME=exampleuser
DB_PASSWORD=StrongPasswordHere
```

---

## 5) PostgreSQL hardening (recommended)

First, check your installed PostgreSQL version:

```bash
ls /etc/postgresql/
```

Use the version number shown (e.g., `14`, `15`, or `16`) in the paths below.

Edit the main config:

```bash
sudo nano /etc/postgresql/VERSION/main/postgresql.conf
```

Set or confirm:

```ini
listen_addresses = 'localhost'
ssl = on
password_encryption = scram-sha-256
```

If you need remote access, change `listen_addresses` to include external connections (for example `'*'`).

Edit client authentication:

```bash
sudo nano /etc/postgresql/VERSION/main/pg_hba.conf
```

Ensure local connections use scram:

```text
local   all             all                                     scram-sha-256
host    all             all             127.0.0.1/32            scram-sha-256
host    all             all             ::1/128                 scram-sha-256
```

To allow a specific external IP, add this at the end of `/etc/postgresql/VERSION/main/pg_hba.conf`:

```text
host    all             all             YOUR_IP_ADDRESS/32            scram-sha-256
```

## Allow an external IP to connect to the database

```bash
sudo ufw allow from YOUR_IP_ADDRESS to any port 5432 proto tcp
```

Reload PostgreSQL:

```bash
sudo systemctl reload postgresql
```

---

## Troubleshooting

### Laravel migration fails with schema error

Re-run **Step 3**.

### Test database login

```bash
psql "host=127.0.0.1 port=5432 dbname=exampledb user=exampleuser"
```

---

## Result

✅ PostgreSQL installed  
✅ Database and user created  
✅ Correct schema permissions applied
