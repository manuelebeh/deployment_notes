# PostgreSQL on a VPS

[🇫🇷 Lire en français](../fr/databases/postgresql.md)

Install PostgreSQL, create a dedicated user and database for your app, and (optionally) allow secure remote access.

| Ubuntu | Default `postgresql` package | Latest via PGDG repo |
|---|---|---|
| 26.04 | **18** | 18 |
| 24.04 | 16 | 18 |
| 22.04 | 14 | 18 |

- [1. Install](#1-install)
- [2. Create a user and a database](#2-create-a-user-and-a-database)
- [3. Connection strings](#3-connection-strings)
- [4. Remote access (optional)](#4-remote-access-optional)
- [5. Useful commands](#5-useful-commands)
- [Security tips](#security-tips)

---

## 1. Install

### Option A — Ubuntu's version (simplest)

```bash
sudo apt update
sudo apt install -y postgresql postgresql-contrib
```

### Option B — Latest version from the official PostgreSQL repo (PGDG)

Recommended on 24.04 / 22.04 if you want PostgreSQL 18:

```bash
sudo apt install -y postgresql-common
sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh   # press Enter to confirm
sudo apt install -y postgresql-18
```

You'll then get every minor update (18.x) through `apt upgrade`.

### Check the service

```bash
sudo systemctl status postgresql
psql --version
```

PostgreSQL listens on `localhost:5432` only, by default.

## 2. Create a user and a database

On Ubuntu, the `postgres` superuser has **no password**: you log in as the Linux user `postgres` ("peer" authentication). Keep it that way and create a separate user for your app.

```bash
sudo -u postgres psql
```

```sql
CREATE ROLE app_user WITH LOGIN PASSWORD 'strong_password_here';
CREATE DATABASE app_db OWNER app_user;
\q
```

Making `app_user` the **owner** of the database lets it create tables in the `public` schema (since PostgreSQL 15, ordinary users no longer can by default).

Test the login over TCP (like your app will):

```bash
psql "postgresql://app_user@127.0.0.1:5432/app_db"
```

<details>
<summary>Interactive alternative</summary>

```bash
sudo -u postgres createuser --pwprompt app_user
sudo -u postgres createdb --owner=app_user app_db
```

</details>

<details>
<summary>Read-only user (e.g. for analytics / BI tools)</summary>

```sql
\c app_db
CREATE ROLE readonly_user WITH LOGIN PASSWORD 'another_password';
GRANT CONNECT ON DATABASE app_db TO readonly_user;
GRANT USAGE ON SCHEMA public TO readonly_user;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly_user;
ALTER DEFAULT PRIVILEGES FOR ROLE app_user IN SCHEMA public GRANT SELECT ON TABLES TO readonly_user;
```

</details>

## 3. Connection strings

```bash
postgresql://app_user:password@127.0.0.1:5432/app_db
```

- Prisma / Drizzle / `pg` / Sequelize all accept this format (`postgres://` works too).
- URL-encode special characters in the password.
- Remote connections over the internet should add `?sslmode=require` (see below).

## 4. Remote access (optional)

**Best option: SSH tunnel** — no config change, nothing exposed. From your computer:

```bash
ssh -N -L 5432:127.0.0.1:5432 deploy@your_vps_ip
```

Then connect pgAdmin/DBeaver/TablePlus to `127.0.0.1:5432`.

**Direct access** (another server needs to connect):

1. Listen on the network interface:

   ```bash
   sudo nano /etc/postgresql/*/main/postgresql.conf
   ```

   ```conf
   listen_addresses = '*'
   ```

2. Allow only a specific IP, a specific user and database, with `scram-sha-256`:

   ```bash
   sudo nano /etc/postgresql/*/main/pg_hba.conf
   ```

   ```conf
   # TYPE    DATABASE  USER      ADDRESS            METHOD
   hostssl   app_db    app_user  203.0.113.10/32    scram-sha-256
   ```

   `hostssl` forces encryption — Ubuntu's package ships with a self-signed certificate, so `sslmode=require` works out of the box.

3. Firewall — same IP only:

   ```bash
   sudo ufw allow from 203.0.113.10 to any port 5432 proto tcp
   ```

4. Restart:

   ```bash
   sudo systemctl restart postgresql
   ```

> Use **`scram-sha-256`**, not `md5`: MD5 passwords are deprecated since PostgreSQL 18 and will be removed in a future version. Never use `trust` or `0.0.0.0/0`.

## 5. Useful commands

Inside `psql`:

| Command | Meaning |
|---|---|
| `\l` | list databases |
| `\c app_db` | connect to a database |
| `\dt` | list tables |
| `\du` | list users/roles |
| `\d table_name` | describe a table |
| `\q` | quit |

Change a user's password:

```sql
ALTER ROLE app_user WITH PASSWORD 'new_password';
```

Logs:

```bash
sudo journalctl -u postgresql -n 50
sudo tail -f /var/log/postgresql/postgresql-*-main.log
```

---

## Security tips

- One user per application, owner of its own database — never connect your app as `postgres`.
- Keep `postgres` on peer authentication (no password, no remote login).
- Prefer SSH tunnels over opening port 5432.
- Keep the server updated (`sudo apt upgrade` brings minor releases with security fixes).
- Set up [automatic backups](backups.md#postgresql) and test a restore.
