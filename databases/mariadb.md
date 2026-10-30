# MariaDB on a VPS

Install MariaDB, secure it, and create a dedicated user and database for your app.

| Ubuntu | Default `mariadb-server` package |
|---|---|
| 26.04 | **11.8 LTS** (supported until June 2028) |
| 24.04 | 10.11 LTS |
| 22.04 | 10.6 — ⚠️ community support ended in July 2026, use the MariaDB repo below |

Newest LTS from MariaDB's own repository: **12.3** (supported until June 2029).

> MariaDB and MySQL **can't be installed together** on Ubuntu. Pick one ([MySQL guide](mysql.md)).

- [1. Install](#1-install)
- [2. Secure the installation](#2-secure-the-installation)
- [3. Create a database and a user](#3-create-a-database-and-a-user)
- [4. Connection strings](#4-connection-strings)
- [5. Remote access (optional)](#5-remote-access-optional)
- [Key points](#key-points)

---

## 1. Install

### Option A — Ubuntu's version

```bash
sudo apt update
sudo apt install -y mariadb-server mariadb-client
sudo systemctl enable --now mariadb
sudo systemctl status mariadb
mariadb --version
```

### Option B — A specific LTS from MariaDB's repository

```bash
sudo apt install -y curl
curl -LsS https://r.mariadb.com/downloads/mariadb_repo_setup | \
  sudo bash -s -- --mariadb-server-version="mariadb-11.8"     # or mariadb-12.3
sudo apt update
sudo apt install -y mariadb-server mariadb-client
```

## 2. Secure the installation

```bash
sudo mariadb-secure-installation
```

On Ubuntu, `root` already authenticates through the **unix socket** (no password, only from `sudo`). Recommended answers:

| Question | Answer |
|---|---|
| Enter current password for root | *Enter* (none) |
| Switch to unix_socket authentication | **n** — it's already enabled |
| Change the root password? | **n** — root is already protected by the socket |
| Remove anonymous users? | **Y** |
| Disallow root login remotely? | **Y** |
| Remove test database and access to it? | **Y** |
| Reload privilege tables now? | **Y** |

Log in as root:

```bash
sudo mariadb
```

## 3. Create a database and a user

```bash
sudo mariadb
```

```sql
CREATE DATABASE app_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

CREATE USER 'app_user'@'localhost' IDENTIFIED BY 'Strong_Password_123!';
GRANT ALL PRIVILEGES ON app_db.* TO 'app_user'@'localhost';

EXIT;
```

`FLUSH PRIVILEGES` is not needed after `CREATE USER` / `GRANT`.

Test:

```bash
mariadb -u app_user -p app_db
```

```sql
SHOW DATABASES;
```

## 4. Connection strings

Most Node.js libraries talk to MariaDB with the **MySQL protocol**, so the URL starts with `mysql://`:

```bash
# mysql2, Prisma (provider = "mysql"), Sequelize (dialect "mariadb" or "mysql"), TypeORM, Drizzle
mysql://app_user:password@127.0.0.1:3306/app_db

# Official "mariadb" npm connector
mariadb://app_user:password@127.0.0.1:3306/app_db
```

URL-encode special characters in the password.

## 5. Remote access (optional)

**Best option: SSH tunnel** (nothing to open). From your computer:

```bash
ssh -N -L 3306:127.0.0.1:3306 deploy@your_vps_ip
```

**Direct access** from a known IP:

1. Listen on the network:

   ```bash
   sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
   ```

   ```conf
   bind-address = 0.0.0.0
   ```

2. Create a user for that IP only (avoid `'%'` = any host):

   ```sql
   CREATE USER 'app_user'@'203.0.113.10' IDENTIFIED BY 'Strong_Password_123!';
   GRANT SELECT, INSERT, UPDATE, DELETE ON app_db.* TO 'app_user'@'203.0.113.10';
   ```

3. Firewall:

   ```bash
   sudo ufw allow from 203.0.113.10 to any port 3306 proto tcp
   sudo systemctl restart mariadb
   ```

---

## Key points

**Least privilege** — if your app doesn't run migrations:

```sql
GRANT SELECT, INSERT, UPDATE, DELETE ON app_db.* TO 'app_user'@'localhost';
```

**Check users and rights:**

```sql
SELECT user, host, plugin FROM mysql.user;
SHOW GRANTS FOR 'app_user'@'localhost';
```

**Change a password:**

```sql
ALTER USER 'app_user'@'localhost' IDENTIFIED BY 'New_Password_456!';
```

**Small VPS (1–2 GB RAM)** — in `/etc/mysql/mariadb.conf.d/50-server.cnf`:

```conf
innodb_buffer_pool_size = 256M
max_connections = 50
```

**Logs:** `sudo journalctl -u mariadb -n 50`

**Backups:** see [Database backups](backups.md#mysql--mariadb).
