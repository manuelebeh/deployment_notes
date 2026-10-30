# Database Backups

[🇫🇷 Lire en français](../fr/databases/backups.md)

A database without tested backups is a database you'll lose one day. This guide sets up **daily compressed dumps**, keeps the last 7 days, copies them **off the server**, and shows how to restore.

- [0. Folder & credentials](#0-folder--credentials)
- [MongoDB](#mongodb)
- [PostgreSQL](#postgresql)
- [MySQL / MariaDB](#mysql--mariadb)
- [Schedule it (cron)](#schedule-it-cron)
- [Off-site copy (rclone)](#off-site-copy-rclone)
- [Checklist](#checklist)

> Your provider's VPS snapshots (e.g. Hostinger weekly backups) are a good extra, but not a replacement: they're stored with the same provider and restoring one rolls back the **whole** server.

---

## 0. Folder & credentials

```bash
sudo mkdir -p /var/backups/db
sudo chown $USER:$USER /var/backups/db
chmod 700 /var/backups/db
```

Store credentials in files only your user can read, never in the script or the crontab.

## MongoDB

`~/.mongo-backup.env`:

```bash
MONGO_URI="mongodb://admin:password@127.0.0.1:27017/?authSource=admin"
```

```bash
chmod 600 ~/.mongo-backup.env
```

Backup:

```bash
source ~/.mongo-backup.env
mongodump --uri="$MONGO_URI" --archive="/var/backups/db/mongo-$(date +%F).archive.gz" --gzip
```

Restore (all databases in the archive; `--drop` replaces existing collections):

```bash
mongorestore --uri="$MONGO_URI" --archive=/var/backups/db/mongo-2026-10-30.archive.gz --gzip --drop
```

Restore a single database: add `--nsInclude='app_db.*'`.

## PostgreSQL

`~/.pgpass` (format `host:port:database:user:password`):

```text
127.0.0.1:5432:app_db:app_user:strong_password_here
```

```bash
chmod 600 ~/.pgpass
```

Backup (custom format = compressed and restorable table by table):

```bash
pg_dump -h 127.0.0.1 -U app_user -Fc app_db -f "/var/backups/db/pg-app_db-$(date +%F).dump"
```

Restore into an existing (empty) database:

```bash
pg_restore -h 127.0.0.1 -U app_user -d app_db --clean --if-exists --no-owner /var/backups/db/pg-app_db-2026-10-30.dump
```

All databases + roles at once (run as `postgres`): `sudo -u postgres pg_dumpall | gzip > /var/backups/db/pg-all-$(date +%F).sql.gz`

## MySQL / MariaDB

`~/.my.cnf`:

```ini
[client]
user=app_user
password=Strong_Password_123!
host=127.0.0.1
```

```bash
chmod 600 ~/.my.cnf
```

Backup (`--single-transaction` = consistent dump without locking InnoDB tables):

```bash
mysqldump --single-transaction --routines --triggers app_db | gzip > "/var/backups/db/mysql-app_db-$(date +%F).sql.gz"
```

> On MariaDB the command is `mariadb-dump` (`mysqldump` still works as an alias on most versions). If `app_user` lacks rights for `--routines`, add `--no-tablespaces` or run the dump as root with `sudo`.

Restore:

```bash
gunzip < /var/backups/db/mysql-app_db-2026-10-30.sql.gz | mysql app_db
```

---

## Schedule it (cron)

Create `~/backup-db.sh` and keep only the parts for the databases you use:

```bash
#!/usr/bin/env bash
set -euo pipefail

DIR=/var/backups/db
DATE=$(date +%F)

# MongoDB
source ~/.mongo-backup.env
mongodump --uri="$MONGO_URI" --archive="$DIR/mongo-$DATE.archive.gz" --gzip --quiet

# PostgreSQL
pg_dump -h 127.0.0.1 -U app_user -Fc app_db -f "$DIR/pg-app_db-$DATE.dump"

# MySQL / MariaDB
mysqldump --single-transaction --routines --triggers app_db | gzip > "$DIR/mysql-app_db-$DATE.sql.gz"

# Keep 7 days locally
find "$DIR" -type f -mtime +7 -delete

# Off-site copy (see next section)
# rclone copy "$DIR" remote:my-bucket/db-backups --max-age 25h

echo "$(date -Is) backup OK"
```

```bash
chmod 700 ~/backup-db.sh
~/backup-db.sh          # run once by hand and check the files
ls -lh /var/backups/db
```

Run every night at 3:00:

```bash
crontab -e
```

```cron
0 3 * * * /home/deploy/backup-db.sh >> /home/deploy/backup-db.log 2>&1
```

## Off-site copy (rclone)

Backups stored only on the VPS disappear with the VPS. [rclone](https://rclone.org) copies them to S3, Backblaze B2, Cloudflare R2, Google Drive, etc.

```bash
sudo apt install -y rclone
rclone config            # create a remote, e.g. "remote" → Backblaze B2 / S3 / R2
rclone lsd remote:       # test
```

Then uncomment the `rclone copy` line in `backup-db.sh`. Enable lifecycle rules / retention on the bucket (e.g. keep 30 days).

> Use a bucket key that can **write but not delete** if your provider supports it — that way ransomware on the server can't wipe your backups.

---

## Checklist

- [ ] Backup script runs without errors by hand
- [ ] Cron job scheduled, log file checked the next morning
- [ ] Files copied off the server
- [ ] **Restore tested** at least once on another database/machine
- [ ] Credential files are `chmod 600`
