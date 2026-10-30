# Sauvegardes des bases de données

[🇬🇧 Read in English](../../databases/backups.md)

Une base sans sauvegardes testées est une base que vous perdrez un jour. Ce guide met en place des **dumps compressés quotidiens**, garde les 7 derniers jours, les copie **hors du serveur**, et montre comment restaurer.

- [0. Dossier et identifiants](#0-dossier-et-identifiants)
- [MongoDB](#mongodb)
- [PostgreSQL](#postgresql)
- [MySQL / MariaDB](#mysql--mariadb)
- [Planifier (cron)](#planifier-cron)
- [Copie hors serveur (rclone)](#copie-hors-serveur-rclone)
- [Checklist](#checklist)

> Les snapshots VPS de votre hébergeur (par ex. les sauvegardes hebdomadaires Hostinger) sont un bon complément, mais pas un remplacement : ils sont stockés chez le même hébergeur, et en restaurer un ramène **tout** le serveur en arrière.

---

## 0. Dossier et identifiants

```bash
sudo mkdir -p /var/backups/db
sudo chown $USER:$USER /var/backups/db
chmod 700 /var/backups/db
```

Stockez les identifiants dans des fichiers lisibles uniquement par votre utilisateur, jamais dans le script ni dans la crontab.

## MongoDB

`~/.mongo-backup.env` :

```bash
MONGO_URI="mongodb://admin:password@127.0.0.1:27017/?authSource=admin"
```

```bash
chmod 600 ~/.mongo-backup.env
```

Sauvegarde :

```bash
source ~/.mongo-backup.env
mongodump --uri="$MONGO_URI" --archive="/var/backups/db/mongo-$(date +%F).archive.gz" --gzip
```

Restauration (toutes les bases de l'archive ; `--drop` remplace les collections existantes) :

```bash
mongorestore --uri="$MONGO_URI" --archive=/var/backups/db/mongo-2026-10-30.archive.gz --gzip --drop
```

Restaurer une seule base : ajoutez `--nsInclude='app_db.*'`.

## PostgreSQL

`~/.pgpass` (format `host:port:database:user:password`) :

```text
127.0.0.1:5432:app_db:app_user:strong_password_here
```

```bash
chmod 600 ~/.pgpass
```

Sauvegarde (format custom = compressé et restaurable table par table) :

```bash
pg_dump -h 127.0.0.1 -U app_user -Fc app_db -f "/var/backups/db/pg-app_db-$(date +%F).dump"
```

Restauration dans une base existante (vide) :

```bash
pg_restore -h 127.0.0.1 -U app_user -d app_db --clean --if-exists --no-owner /var/backups/db/pg-app_db-2026-10-30.dump
```

Toutes les bases + les rôles d'un coup (en tant que `postgres`) : `sudo -u postgres pg_dumpall | gzip > /var/backups/db/pg-all-$(date +%F).sql.gz`

## MySQL / MariaDB

`~/.my.cnf` :

```ini
[client]
user=app_user
password=Strong_Password_123!
host=127.0.0.1
```

```bash
chmod 600 ~/.my.cnf
```

Sauvegarde (`--single-transaction` = dump cohérent sans verrouiller les tables InnoDB) :

```bash
mysqldump --single-transaction --routines --triggers app_db | gzip > "/var/backups/db/mysql-app_db-$(date +%F).sql.gz"
```

> Sur MariaDB, la commande est `mariadb-dump` (`mysqldump` fonctionne encore comme alias sur la plupart des versions). Si `app_user` n'a pas les droits pour `--routines`, ajoutez `--no-tablespaces` ou lancez le dump en root avec `sudo`.

Restauration :

```bash
gunzip < /var/backups/db/mysql-app_db-2026-10-30.sql.gz | mysql app_db
```

---

## Planifier (cron)

Créez `~/backup-db.sh` et ne gardez que les parties correspondant à vos bases :

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

Lancement chaque nuit à 3 h :

```bash
crontab -e
```

```cron
0 3 * * * /home/deploy/backup-db.sh >> /home/deploy/backup-db.log 2>&1
```

## Copie hors serveur (rclone)

Des sauvegardes stockées uniquement sur le VPS disparaissent avec le VPS. [rclone](https://rclone.org) les copie vers S3, Backblaze B2, Cloudflare R2, Google Drive, etc.

```bash
sudo apt install -y rclone
rclone config            # create a remote, e.g. "remote" → Backblaze B2 / S3 / R2
rclone lsd remote:       # test
```

Décommentez ensuite la ligne `rclone copy` dans `backup-db.sh`. Activez des règles de cycle de vie / de rétention sur le bucket (par ex. garder 30 jours).

> Utilisez une clé de bucket qui peut **écrire mais pas supprimer** si votre fournisseur le permet — ainsi, un ransomware sur le serveur ne pourra pas effacer vos sauvegardes.

---

## Checklist

- [ ] Le script de sauvegarde tourne sans erreur à la main
- [ ] La tâche cron est planifiée, fichier de log vérifié le lendemain matin
- [ ] Les fichiers sont copiés hors du serveur
- [ ] **Restauration testée** au moins une fois sur une autre base/machine
- [ ] Les fichiers d'identifiants sont en `chmod 600`
