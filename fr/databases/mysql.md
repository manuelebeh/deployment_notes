# MySQL sur un VPS

[🇬🇧 Read in English](../../databases/mysql.md)

Installez MySQL, sécurisez-le, et créez un utilisateur et une base dédiés à votre app.

| Ubuntu | Paquet `mysql-server` par défaut |
|---|---|
| 26.04 | **8.4 LTS** |
| 24.04 | 8.0 |
| 22.04 | 8.0 |

> MySQL **8.0 est arrivé en fin de vie en avril 2026**. Ubuntu continue de corriger son propre paquet 8.0 sur 22.04/24.04, mais pour un nouveau projet préférez la **8.4 LTS** (Ubuntu 26.04, ou le dépôt d'Oracle ci-dessous).

> MySQL et MariaDB **ne peuvent pas être installés ensemble** sur Ubuntu. Choisissez-en un ([guide MariaDB](mariadb.md)).

- [1. Installation](#1-installation)
- [2. Sécuriser l'installation](#2-sécuriser-linstallation)
- [3. Créer une base et un utilisateur](#3-créer-une-base-et-un-utilisateur)
- [4. Chaînes de connexion](#4-chaînes-de-connexion)
- [5. Accès distant (facultatif)](#5-accès-distant-facultatif)
- [Points clés](#points-clés)

---

## 1. Installation

```bash
sudo apt update
sudo apt install -y mysql-server
sudo systemctl enable --now mysql
sudo systemctl status mysql
mysql --version
```

<details>
<summary>Version plus récente depuis le dépôt APT d'Oracle (8.4 LTS / 9.7 LTS)</summary>

Utile sur Ubuntu 24.04 si vous voulez la 8.4 LTS ou la 9.7 LTS au lieu de la 8.0. Le dépôt d'Oracle ne fournit que des paquets **amd64** (pas d'ARM).

1. Téléchargez le dernier `mysql-apt-config_*.deb` sur <https://dev.mysql.com/downloads/repo/apt/>
2. Installez-le et choisissez la version du serveur dans le menu :

   ```bash
   sudo dpkg -i mysql-apt-config_*_all.deb
   sudo apt update
   sudo apt install -y mysql-server
   ```

La mise à niveau d'un serveur existant entre versions majeures (8.0 → 8.4 → 9.7) doit se faire une LTS à la fois, avec une sauvegarde avant.

</details>

## 2. Sécuriser l'installation

```bash
sudo mysql_secure_installation
```

Réponses recommandées :

- **VALIDATE PASSWORD component** → Oui, niveau 1 (MEDIUM) ou 2 (STRONG)
- **Remove anonymous users** → Oui
- **Disallow root login remotely** → Oui
- **Remove test database** → Oui
- **Reload privilege tables** → Oui

Sur Ubuntu, `root` utilise **`auth_socket`** : il se connecte sans mot de passe, mais uniquement depuis le compte Linux `root` (`sudo`). Le script ne proposera donc pas de définir un mot de passe root — c'est normal et sécurisé. Connectez-vous en root avec :

```bash
sudo mysql
```

## 3. Créer une base et un utilisateur

```bash
sudo mysql
```

```sql
CREATE DATABASE app_db CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci;

CREATE USER 'app_user'@'localhost' IDENTIFIED BY 'Strong_Password_123!';
GRANT ALL PRIVILEGES ON app_db.* TO 'app_user'@'localhost';

EXIT;
```

- `utf8mb4` gère tous les caractères, emojis compris (l'ancien `utf8` de MySQL non).
- `FLUSH PRIVILEGES` n'est **pas nécessaire** après `CREATE USER` / `GRANT` — seulement après une modification directe des tables `mysql.*`.
- `ALL PRIVILEGES ON app_db.*` est limité à cette seule base ; c'est adapté à une app qui gère ses propres migrations. Pour un contrôle plus fin, voir les [points clés](#points-clés).

Test :

```bash
mysql -u app_user -p app_db
```

```sql
SHOW DATABASES;
```

## 4. Chaînes de connexion

```bash
mysql://app_user:password@127.0.0.1:3306/app_db
```

- Fonctionne avec `mysql2`, Prisma (`provider = "mysql"`), Sequelize, TypeORM, Drizzle.
- Encodez les caractères spéciaux du mot de passe pour l'URL (`@` → `%40`, `#` → `%23`…).
- MySQL 8.4+ désactive par défaut l'ancien plugin `mysql_native_password` ; les utilisateurs utilisent `caching_sha2_password`. Les drivers récents le gèrent — si un vieux driver échoue avec *« auth method unknown »*, mettez à jour le driver plutôt que de réactiver l'ancien plugin.

## 5. Accès distant (facultatif)

**Meilleure option : un tunnel SSH** (rien à ouvrir). Depuis votre ordinateur :

```bash
ssh -N -L 3306:127.0.0.1:3306 deploy@your_vps_ip
```

**Accès direct** depuis une IP connue :

1. Écoutez sur le réseau :

   ```bash
   sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
   ```

   ```conf
   bind-address = 0.0.0.0
   ```

2. Créez un utilisateur pour cette IP uniquement (évitez `'%'` = n'importe quel hôte) :

   ```sql
   CREATE USER 'app_user'@'203.0.113.10' IDENTIFIED BY 'Strong_Password_123!' REQUIRE SSL;
   GRANT SELECT, INSERT, UPDATE, DELETE ON app_db.* TO 'app_user'@'203.0.113.10';
   ```

3. Pare-feu :

   ```bash
   sudo ufw allow from 203.0.113.10 to any port 3306 proto tcp
   sudo systemctl restart mysql
   ```

---

## Points clés

**Moindre privilège** — si votre app ne lance pas de migrations, donnez-lui seulement les droits sur les données :

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

**Petit VPS (1–2 Go de RAM)** — limitez la mémoire dans `/etc/mysql/mysql.conf.d/mysqld.cnf` :

```conf
innodb_buffer_pool_size = 256M
max_connections = 50
```

**Logs :** `sudo tail -f /var/log/mysql/error.log`

**Sauvegardes :** voir [Sauvegardes des bases de données](backups.md#mysql--mariadb).
