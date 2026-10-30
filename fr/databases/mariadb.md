# MariaDB sur un VPS

[🇬🇧 Read in English](../../databases/mariadb.md)

Installez MariaDB, sécurisez-le, et créez un utilisateur et une base dédiés à votre app.

| Ubuntu | Paquet `mariadb-server` par défaut |
|---|---|
| 26.04 | **11.8 LTS** (supportée jusqu'en juin 2028) |
| 24.04 | 10.11 LTS |
| 22.04 | 10.6 — ⚠️ support communautaire terminé en juillet 2026, utilisez le dépôt MariaDB ci-dessous |

LTS la plus récente depuis le dépôt de MariaDB : **12.3** (supportée jusqu'en juin 2029).

> MariaDB et MySQL **ne peuvent pas être installés ensemble** sur Ubuntu. Choisissez-en un ([guide MySQL](mysql.md)).

- [1. Installation](#1-installation)
- [2. Sécuriser l'installation](#2-sécuriser-linstallation)
- [3. Créer une base et un utilisateur](#3-créer-une-base-et-un-utilisateur)
- [4. Chaînes de connexion](#4-chaînes-de-connexion)
- [5. Accès distant (facultatif)](#5-accès-distant-facultatif)
- [Points clés](#points-clés)

---

## 1. Installation

### Option A — La version d'Ubuntu

```bash
sudo apt update
sudo apt install -y mariadb-server mariadb-client
sudo systemctl enable --now mariadb
sudo systemctl status mariadb
mariadb --version
```

### Option B — Une LTS précise depuis le dépôt de MariaDB

```bash
sudo apt install -y curl
curl -LsS https://r.mariadb.com/downloads/mariadb_repo_setup | \
  sudo bash -s -- --mariadb-server-version="mariadb-11.8"     # or mariadb-12.3
sudo apt update
sudo apt install -y mariadb-server mariadb-client
```

## 2. Sécuriser l'installation

```bash
sudo mariadb-secure-installation
```

Sur Ubuntu, `root` s'authentifie déjà par le **socket unix** (pas de mot de passe, uniquement via `sudo`). Réponses recommandées :

| Question | Réponse |
|---|---|
| Enter current password for root | *Entrée* (aucun) |
| Switch to unix_socket authentication | **n** — c'est déjà activé |
| Change the root password? | **n** — root est déjà protégé par le socket |
| Remove anonymous users? | **Y** |
| Disallow root login remotely? | **Y** |
| Remove test database and access to it? | **Y** |
| Reload privilege tables now? | **Y** |

Connexion en root :

```bash
sudo mariadb
```

## 3. Créer une base et un utilisateur

```bash
sudo mariadb
```

```sql
CREATE DATABASE app_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

CREATE USER 'app_user'@'localhost' IDENTIFIED BY 'Strong_Password_123!';
GRANT ALL PRIVILEGES ON app_db.* TO 'app_user'@'localhost';

EXIT;
```

`FLUSH PRIVILEGES` n'est pas nécessaire après `CREATE USER` / `GRANT`.

Test :

```bash
mariadb -u app_user -p app_db
```

```sql
SHOW DATABASES;
```

## 4. Chaînes de connexion

La plupart des librairies Node.js parlent à MariaDB avec le **protocole MySQL**, donc l'URL commence par `mysql://` :

```bash
# mysql2, Prisma (provider = "mysql"), Sequelize (dialect "mariadb" or "mysql"), TypeORM, Drizzle
mysql://app_user:password@127.0.0.1:3306/app_db

# Official "mariadb" npm connector
mariadb://app_user:password@127.0.0.1:3306/app_db
```

Encodez les caractères spéciaux du mot de passe pour l'URL.

## 5. Accès distant (facultatif)

**Meilleure option : un tunnel SSH** (rien à ouvrir). Depuis votre ordinateur :

```bash
ssh -N -L 3306:127.0.0.1:3306 deploy@your_vps_ip
```

**Accès direct** depuis une IP connue :

1. Écoutez sur le réseau :

   ```bash
   sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
   ```

   ```conf
   bind-address = 0.0.0.0
   ```

2. Créez un utilisateur pour cette IP uniquement (évitez `'%'` = n'importe quel hôte) :

   ```sql
   CREATE USER 'app_user'@'203.0.113.10' IDENTIFIED BY 'Strong_Password_123!';
   GRANT SELECT, INSERT, UPDATE, DELETE ON app_db.* TO 'app_user'@'203.0.113.10';
   ```

3. Pare-feu :

   ```bash
   sudo ufw allow from 203.0.113.10 to any port 3306 proto tcp
   sudo systemctl restart mariadb
   ```

---

## Points clés

**Moindre privilège** — si votre app ne lance pas de migrations :

```sql
GRANT SELECT, INSERT, UPDATE, DELETE ON app_db.* TO 'app_user'@'localhost';
```

**Voir les utilisateurs et leurs droits :**

```sql
SELECT user, host, plugin FROM mysql.user;
SHOW GRANTS FOR 'app_user'@'localhost';
```

**Changer un mot de passe :**

```sql
ALTER USER 'app_user'@'localhost' IDENTIFIED BY 'New_Password_456!';
```

**Petit VPS (1–2 Go de RAM)** — dans `/etc/mysql/mariadb.conf.d/50-server.cnf` :

```conf
innodb_buffer_pool_size = 256M
max_connections = 50
```

**Logs :** `sudo journalctl -u mariadb -n 50`

**Sauvegardes :** voir [Sauvegardes des bases de données](backups.md#mysql--mariadb).
