# MongoDB sur un VPS (Community 9.0)

[🇬🇧 Read in English](../../databases/mongodb.md)

Installez MongoDB depuis le dépôt officiel, activez l'authentification et créez un utilisateur pour votre app.

Compatible : **Ubuntu 22.04 (jammy), 24.04 (noble), 26.04 (resolute)** — x86_64 et ARM64.

- [1. Installation](#1-installation)
- [2. Démarrer le service](#2-démarrer-le-service)
- [3. Activer l'authentification (à ne pas sauter)](#3-activer-lauthentification-à-ne-pas-sauter)
- [4. Créer un utilisateur pour votre app](#4-créer-un-utilisateur-pour-votre-app)
- [5. Chaînes de connexion](#5-chaînes-de-connexion)
- [6. Accès distant (seulement si nécessaire)](#6-accès-distant-seulement-si-nécessaire)
- [Dépannage](#dépannage)

---

## 1. Installation

Vérifiez que le processeur supporte AVX (exigé par MongoDB 5.0+ sur x86_64 — c'est le cas de la plupart des VPS) :

```bash
grep -q avx /proc/cpuinfo && echo "AVX OK" || echo "No AVX: use MongoDB Atlas or Docker on another host"
```

Supprimez le paquet non officiel `mongodb` d'Ubuntu s'il est installé (il entre en conflit) :

```bash
sudo apt remove -y mongodb mongodb-server 2>/dev/null || true
```

Importez la clé de MongoDB :

```bash
sudo apt install -y gnupg curl
curl -fsSL https://pgp.mongodb.com/server-9.asc | \
  sudo gpg -o /usr/share/keyrings/mongodb-server-9.gpg --dearmor
```

Ajoutez le dépôt — cette ligne détecte automatiquement le nom de code de votre version d'Ubuntu :

```bash
echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-9.gpg ] https://repo.mongodb.org/apt/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME")/mongodb-org/9.0 multiverse" | \
  sudo tee /etc/apt/sources.list.d/mongodb-org-9.0.list
```

Installez (comprend le serveur `mongod`, le shell `mongosh` et les outils de base de données) :

```bash
sudo apt update
sudo apt install -y mongodb-org
```

<details>
<summary>Besoin de MongoDB 8.0 ? (par ex. pour correspondre à un cluster existant)</summary>

Remplacez chaque `9` / `9.0` ci-dessus par `8.0` :

```bash
curl -fsSL https://www.mongodb.org/static/pgp/server-8.0.asc | \
  sudo gpg -o /usr/share/keyrings/mongodb-server-8.0.gpg --dearmor
echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-8.0.gpg ] https://repo.mongodb.org/apt/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME")/mongodb-org/8.0 multiverse" | \
  sudo tee /etc/apt/sources.list.d/mongodb-org-8.0.list
```

Vous mettez à niveau un serveur existant de 8.0 vers 9.0 ? Faites d'abord une sauvegarde, puis suivez les notes de mise à niveau de MongoDB (le `featureCompatibilityVersion` doit être à `8.0` avant de changer de binaires).

</details>

## 2. Démarrer le service

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now mongod
sudo systemctl status mongod
```

Test rapide :

```bash
mongosh --eval 'db.runCommand({ ping: 1 })'
```

À ce stade, MongoDB écoute uniquement sur `127.0.0.1:27017`, **sans authentification** : n'importe quel programme du serveur peut tout lire et tout modifier.

## 3. Activer l'authentification (à ne pas sauter)

Créez un utilisateur administrateur :

```bash
mongosh
```

```js
use admin
db.createUser({
  user: "admin",
  pwd: passwordPrompt(),          // asks for the password, keeps it out of your shell history
  roles: [ { role: "root", db: "admin" } ]
})
exit
```

Activez l'autorisation :

```bash
sudo nano /etc/mongod.conf
```

```yaml
net:
  port: 27017
  bindIp: 127.0.0.1

security:
  authorization: enabled
```

> Le YAML est sensible à l'indentation : utilisez **2 espaces**, pas de tabulations. `security:` est commenté (`#security:`) par défaut — retirez le `#`.

```bash
sudo systemctl restart mongod
```

Connectez-vous désormais en admin :

```bash
mongosh -u admin --authenticationDatabase admin
```

## 4. Créer un utilisateur pour votre app

N'utilisez jamais l'utilisateur `admin`/`root` dans votre application. Créez un utilisateur par app, avec accès à sa propre base uniquement :

```js
use app_db
db.createUser({
  user: "app_user",
  pwd: passwordPrompt(),
  roles: [ { role: "readWrite", db: "app_db" } ]
})
exit
```

Testez-le :

```bash
mongosh "mongodb://app_user@127.0.0.1:27017/app_db?authSource=app_db"
```

## 5. Chaînes de connexion

```bash
# Without auth (local testing only)
mongodb://127.0.0.1:27017/app_db

# App user created in app_db (step 4)
mongodb://app_user:password@127.0.0.1:27017/app_db?authSource=app_db
```

- Utilisez `127.0.0.1` plutôt que `localhost` (évite les problèmes de résolution IPv6 `::1` avec Node.js).
- Si le mot de passe contient `@ : / ? # %`, encodez-le pour l'URL : `node -e 'console.log(encodeURIComponent(process.argv[1]))' 'p@ss:word'`

## 6. Accès distant (seulement si nécessaire)

Préférez un **tunnel SSH** — rien à ouvrir sur le serveur. Depuis votre ordinateur :

```bash
ssh -N -L 27017:127.0.0.1:27017 deploy@your_vps_ip
```

Connectez ensuite MongoDB Compass à `mongodb://admin:password@127.0.0.1:27017/?authSource=admin`.

Si vous avez vraiment besoin d'un accès direct (par ex. depuis un autre serveur) : seulement avec l'authentification activée, et seulement depuis des IP connues :

```yaml
# /etc/mongod.conf
net:
  bindIp: 127.0.0.1,your_vps_private_or_public_ip
```

```bash
sudo ufw allow from 203.0.113.10 to any port 27017 proto tcp
sudo systemctl restart mongod
```

> Ne faites jamais `ufw allow 27017` pour tout le monde. Les instances MongoDB exposées sont scannées et rançonnées en quelques heures.

---

## Dépannage

| Problème | Solution |
|---|---|
| `Illegal instruction (core dumped)` | Processeur sans AVX — voir l'étape 1 |
| `mongod.service not found` | `sudo systemctl daemon-reload` |
| Le service ne démarre plus après modification de `mongod.conf` | Indentation YAML → `sudo journalctl -u mongod -n 50` |
| `Authentication failed` | Mauvais `authSource` — c'est la base dans laquelle l'utilisateur a été créé |
| `ECONNREFUSED ::1:27017` dans Node | Utilisez `127.0.0.1` au lieu de `localhost` |
| Erreurs de permissions sur `/var/lib/mongodb` | `sudo chown -R mongodb:mongodb /var/lib/mongodb /var/log/mongodb` |

Logs : `sudo tail -f /var/log/mongodb/mongod.log`

**Ensuite :** [sauvegardes automatiques](backups.md#mongodb).
