# PostgreSQL sur un VPS

[🇬🇧 Read in English](../../databases/postgresql.md)

Installez PostgreSQL, créez un utilisateur et une base dédiés à votre app, et (si besoin) autorisez un accès distant sécurisé.

| Ubuntu | Paquet `postgresql` par défaut | Dernière version via le dépôt PGDG |
|---|---|---|
| 26.04 | **18** | 18 |
| 24.04 | 16 | 18 |
| 22.04 | 14 | 18 |

- [1. Installation](#1-installation)
- [2. Créer un utilisateur et une base](#2-créer-un-utilisateur-et-une-base)
- [3. Chaînes de connexion](#3-chaînes-de-connexion)
- [4. Accès distant (facultatif)](#4-accès-distant-facultatif)
- [5. Commandes utiles](#5-commandes-utiles)
- [Conseils de sécurité](#conseils-de-sécurité)

---

## 1. Installation

### Option A — La version d'Ubuntu (la plus simple)

```bash
sudo apt update
sudo apt install -y postgresql postgresql-contrib
```

### Option B — La dernière version depuis le dépôt officiel PostgreSQL (PGDG)

Recommandé sur 24.04 / 22.04 si vous voulez PostgreSQL 18 :

```bash
sudo apt install -y postgresql-common
sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh   # press Enter to confirm
sudo apt install -y postgresql-18
```

Vous recevrez ensuite toutes les mises à jour mineures (18.x) via `apt upgrade`.

### Vérifier le service

```bash
sudo systemctl status postgresql
psql --version
```

Par défaut, PostgreSQL écoute uniquement sur `localhost:5432`.

## 2. Créer un utilisateur et une base

Sur Ubuntu, le super-utilisateur `postgres` n'a **pas de mot de passe** : on s'y connecte en tant qu'utilisateur Linux `postgres` (authentification « peer »). Gardez-le ainsi et créez un utilisateur séparé pour votre app.

```bash
sudo -u postgres psql
```

```sql
CREATE ROLE app_user WITH LOGIN PASSWORD 'strong_password_here';
CREATE DATABASE app_db OWNER app_user;
\q
```

Faire de `app_user` le **propriétaire** de la base lui permet de créer des tables dans le schéma `public` (depuis PostgreSQL 15, les utilisateurs ordinaires ne le peuvent plus par défaut).

Testez la connexion en TCP (comme le fera votre app) :

```bash
psql "postgresql://app_user@127.0.0.1:5432/app_db"
```

<details>
<summary>Alternative interactive</summary>

```bash
sudo -u postgres createuser --pwprompt app_user
sudo -u postgres createdb --owner=app_user app_db
```

</details>

<details>
<summary>Utilisateur en lecture seule (par ex. pour des outils d'analyse / BI)</summary>

```sql
\c app_db
CREATE ROLE readonly_user WITH LOGIN PASSWORD 'another_password';
GRANT CONNECT ON DATABASE app_db TO readonly_user;
GRANT USAGE ON SCHEMA public TO readonly_user;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly_user;
ALTER DEFAULT PRIVILEGES FOR ROLE app_user IN SCHEMA public GRANT SELECT ON TABLES TO readonly_user;
```

</details>

## 3. Chaînes de connexion

```bash
postgresql://app_user:password@127.0.0.1:5432/app_db
```

- Prisma / Drizzle / `pg` / Sequelize acceptent tous ce format (`postgres://` fonctionne aussi).
- Encodez les caractères spéciaux du mot de passe pour l'URL.
- Pour une connexion distante via Internet, ajoutez `?sslmode=require` (voir ci-dessous).

## 4. Accès distant (facultatif)

**Meilleure option : un tunnel SSH** — aucune modification de config, rien d'exposé. Depuis votre ordinateur :

```bash
ssh -N -L 5432:127.0.0.1:5432 deploy@your_vps_ip
```

Connectez ensuite pgAdmin/DBeaver/TablePlus à `127.0.0.1:5432`.

**Accès direct** (un autre serveur doit se connecter) :

1. Écoutez sur l'interface réseau :

   ```bash
   sudo nano /etc/postgresql/*/main/postgresql.conf
   ```

   ```conf
   listen_addresses = '*'
   ```

2. N'autorisez qu'une IP précise, un utilisateur et une base précis, avec `scram-sha-256` :

   ```bash
   sudo nano /etc/postgresql/*/main/pg_hba.conf
   ```

   ```conf
   # TYPE    DATABASE  USER      ADDRESS            METHOD
   hostssl   app_db    app_user  203.0.113.10/32    scram-sha-256
   ```

   `hostssl` impose le chiffrement — le paquet Ubuntu est livré avec un certificat auto-signé, donc `sslmode=require` fonctionne directement.

3. Pare-feu — même IP uniquement :

   ```bash
   sudo ufw allow from 203.0.113.10 to any port 5432 proto tcp
   ```

4. Redémarrez :

   ```bash
   sudo systemctl restart postgresql
   ```

> Utilisez **`scram-sha-256`**, pas `md5` : les mots de passe MD5 sont dépréciés depuis PostgreSQL 18 et seront supprimés dans une future version. N'utilisez jamais `trust` ni `0.0.0.0/0`.

## 5. Commandes utiles

Dans `psql` :

| Commande | Signification |
|---|---|
| `\l` | lister les bases |
| `\c app_db` | se connecter à une base |
| `\dt` | lister les tables |
| `\du` | lister les utilisateurs/rôles |
| `\d table_name` | décrire une table |
| `\q` | quitter |

Changer le mot de passe d'un utilisateur :

```sql
ALTER ROLE app_user WITH PASSWORD 'new_password';
```

Logs :

```bash
sudo journalctl -u postgresql -n 50
sudo tail -f /var/log/postgresql/postgresql-*-main.log
```

---

## Conseils de sécurité

- Un utilisateur par application, propriétaire de sa propre base — ne connectez jamais votre app en tant que `postgres`.
- Gardez `postgres` en authentification peer (pas de mot de passe, pas de connexion distante).
- Préférez les tunnels SSH à l'ouverture du port 5432.
- Gardez le serveur à jour (`sudo apt upgrade` apporte les versions mineures avec les correctifs de sécurité).
- Mettez en place des [sauvegardes automatiques](backups.md#postgresql) et testez une restauration.
