# MySQL on a VPS

Install MySQL, secure it, and create a dedicated user and database for your app.

| Ubuntu | Default `mysql-server` package |
|---|---|
| 26.04 | **8.4 LTS** |
| 24.04 | 8.0 |
| 22.04 | 8.0 |

> MySQL **8.0 reached upstream end of life in April 2026**. Ubuntu keeps patching its own 8.0 package on 22.04/24.04, but for a new project prefer **8.4 LTS** (Ubuntu 26.04, or Oracle's repository below).

> MySQL and MariaDB **can't be installed together** on Ubuntu. Pick one ([MariaDB guide](mariadb.md)).

- [1. Install](#1-install)
- [2. Secure the installation](#2-secure-the-installation)
- [3. Create a database and a user](#3-create-a-database-and-a-user)
- [4. Connection strings](#4-connection-strings)
- [5. Remote access (optional)](#5-remote-access-optional)
- [Key points](#key-points)

---

## 1. Install

```bash
sudo apt update
sudo apt install -y mysql-server
sudo systemctl enable --now mysql
sudo systemctl status mysql
mysql --version
```

<details>
<summary>Newer version from Oracle's APT repository (8.4 LTS / 9.7 LTS)</summary>

Useful on Ubuntu 24.04 if you want 8.4 LTS or 9.7 LTS instead of 8.0. Oracle's repository only provides **amd64** packages (no ARM).

1. Download the latest `mysql-apt-config_*.deb` from <https://dev.mysql.com/downloads/repo/apt/>
2. Install it and choose the server version in the menu:

   ```bash
   sudo dpkg -i mysql-apt-config_*_all.deb
   sudo apt update
   sudo apt install -y mysql-server
   ```

Upgrading an existing server between major versions (8.0 → 8.4 → 9.7) must be done one LTS at a time, with a backup first.
</details>

## 2. Secure the installation

```bash
sudo mysql_secure_installation
```

Recommended answers:

- **VALIDATE PASSWORD component** → Yes, level 1 (MEDIUM) or 2 (STRONG)
- **Remove anonymous users** → Yes
- **Disallow root login remotely** → Yes
- **Remove test database** → Yes
- **Reload privilege tables** → Yes

On Ubuntu, `root` uses **`auth_socket`**: it logs in without a password, but only from the Linux `root` account (`sudo`). The script will skip setting a root password — that's expected and secure. Connect as root with:

```bash
sudo mysql
```

## 3. Create a database and a user

```bash
sudo mysql
```

```sql
CREATE DATABASE app_db CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci;

CREATE USER 'app_user'@'localhost' IDENTIFIED BY 'Strong_Password_123!';
GRANT ALL PRIVILEGES ON app_db.* TO 'app_user'@'localhost';

EXIT;
```

- `utf8mb4` supports all characters, including emojis (MySQL's old `utf8` doesn't).
- `FLUSH PRIVILEGES` is **not needed** after `CREATE USER` / `GRANT` — only after editing the `mysql.*` tables directly.
- `ALL PRIVILEGES ON app_db.*` is limited to that one database; that's fine for an app that runs its own migrations. For tighter control, see [Key points](#key-points).

Test:

```bash
mysql -u app_user -p app_db
```

```sql
SHOW DATABASES;
```

## 4. Connection strings

```bash
mysql://app_user:password@127.0.0.1:3306/app_db
```

- Works with `mysql2`, Prisma (`provider = "mysql"`), Sequelize, TypeORM, Drizzle.
- URL-encode special characters in the password (`@` → `%40`, `#` → `%23`…).
- MySQL 8.4+ disables the old `mysql_native_password` plugin by default; users use `caching_sha2_password`. Recent drivers support it — if an old driver fails with *"auth method unknown"*, update the driver rather than re-enabling the old plugin.

## 5. Remote access (optional)

**Best option: SSH tunnel** (nothing to open). From your computer:

```bash
ssh -N -L 3306:127.0.0.1:3306 deploy@your_vps_ip
```

**Direct access** from a known IP:

1. Listen on the network:

   ```bash
   sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
   ```

   ```conf
   bind-address = 0.0.0.0
   ```

2. Create a user for that IP only (avoid `'%'` = any host):

   ```sql
   CREATE USER 'app_user'@'203.0.113.10' IDENTIFIED BY 'Strong_Password_123!' REQUIRE SSL;
   GRANT SELECT, INSERT, UPDATE, DELETE ON app_db.* TO 'app_user'@'203.0.113.10';
   ```

3. Firewall:

   ```bash
   sudo ufw allow from 203.0.113.10 to any port 3306 proto tcp
   sudo systemctl restart mysql
   ```

---

## Key points

**Least privilege** — if your app doesn't run migrations, give it only data rights:

```sql
GRANT SELECT, INSERT, UPDATE, DELETE ON app_db.* TO 'app_user'@'localhost';
```

**Check users and their rights:**

```sql
SELECT user, host, plugin FROM mysql.user;
SHOW GRANTS FOR 'app_user'@'localhost';
```

**Change a password:**

```sql
ALTER USER 'app_user'@'localhost' IDENTIFIED BY 'New_Password_456!';
```

**Small VPS (1–2 GB RAM)** — limit memory in `/etc/mysql/mysql.conf.d/mysqld.cnf`:

```conf
innodb_buffer_pool_size = 256M
max_connections = 50
```

**Logs:** `sudo tail -f /var/log/mysql/error.log`

**Backups:** see [Database backups](backups.md#mysql--mariadb).
