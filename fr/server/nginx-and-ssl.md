# Nginx & SSL (Let's Encrypt)

[🇬🇧 Read in English](../../server/nginx-and-ssl.md)

Servez vos frontends et faites passer votre API par Nginx, puis ajoutez des certificats HTTPS gratuits qui se renouvellent tout seuls.

> Vous préférez un HTTPS automatique avec une config beaucoup plus courte ? Voir [Caddy](caddy.md). Vous utilisez Cloudflare ? Lisez aussi [Cloudflare devant votre VPS](cloudflare.md).

- [1. Installer Nginx](#1-installer-nginx)
- [2. Pointer votre domaine vers le VPS](#2-pointer-votre-domaine-vers-le-vps)
- [3. Bloc serveur : SPA React / Vite](#3-bloc-serveur--spa-react--vite)
- [4. Bloc serveur : API Node.js (reverse proxy)](#4-bloc-serveur--api-nodejs-reverse-proxy)
- [5. Variante : un seul domaine, frontend + `/api`](#5-variante--un-seul-domaine-frontend--api)
- [6. Activer les sites](#6-activer-les-sites)
- [7. HTTPS avec Certbot](#7-https-avec-certbot)
- [8. Réglages recommandés](#8-réglages-recommandés)
- [Dépannage](#dépannage)

---

## 1. Installer Nginx

```bash
sudo apt update
sudo apt install -y nginx
sudo ufw allow 'Nginx Full'
```

Ouvrez `http://your_vps_ip` — la page d'accueil de Nginx doit s'afficher.

Supprimez le site par défaut pour qu'il n'intercepte pas vos domaines :

```bash
sudo rm /etc/nginx/sites-enabled/default
```

## 2. Pointer votre domaine vers le VPS

Dans le gestionnaire DNS de votre registrar, créez :

| Type | Nom | Valeur |
|---|---|---|
| A | `@` | `your_vps_ip` |
| A | `www` | `your_vps_ip` |
| A | `api` | `your_vps_ip` |
| AAAA *(facultatif)* | mêmes noms | l'IPv6 de votre VPS |

Vérifiez la propagation (de quelques minutes à quelques heures) :

```bash
dig +short yourdomain.com
dig +short api.yourdomain.com
```

> N'ajoutez des enregistrements AAAA que si l'IPv6 est vraiment configurée sur votre VPS — un AAAA incorrect fera échouer la validation Let's Encrypt.

## 3. Bloc serveur : SPA React / Vite

```bash
sudo nano /etc/nginx/sites-available/yourdomain.com.conf
```

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name yourdomain.com www.yourdomain.com;

    root /var/www/your-repo/frontend/dist;   # Vite → dist, Create React App → build
    index index.html;

    # Send every unknown route to index.html so React Router works on refresh
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Long cache for hashed build assets
    location /assets/ {
        expires 1y;
        add_header Cache-Control "public, immutable";
        try_files $uri =404;
    }
}
```

Répétez avec un autre fichier pour chaque frontend supplémentaire (`yourdomain2.com.conf`, `admin.yourdomain.com.conf`, …).

## 4. Bloc serveur : API Node.js (reverse proxy)

```bash
sudo nano /etc/nginx/sites-available/api.yourdomain.com.conf
```

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name api.yourdomain.com;

    client_max_body_size 20M;   # raise if you accept file uploads

    location / {
        proxy_pass http://127.0.0.1:4000;
        proxy_http_version 1.1;

        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # WebSockets (Socket.IO, etc.)
        proxy_set_header Upgrade    $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_read_timeout 60s;
    }
}
```

La variable `$connection_upgrade` doit être définie une fois, globalement :

```bash
sudo nano /etc/nginx/conf.d/websocket-upgrade.conf
```

```nginx
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}
```

Dans Express, faites confiance au proxy pour que `req.ip`, `req.secure` et les limiteurs de débit voient le vrai client :

```js
app.set('trust proxy', 1);
```

## 5. Variante : un seul domaine, frontend + `/api`

Si vous préférez éviter un sous-domaine (et les problèmes de CORS), servez les deux depuis le même domaine :

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name yourdomain.com www.yourdomain.com;

    root /var/www/your-repo/frontend/dist;
    index index.html;

    location /api/ {
        proxy_pass http://127.0.0.1:4000;   # no trailing slash: /api/users → /api/users
        proxy_http_version 1.1;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Upgrade           $http_upgrade;
        proxy_set_header Connection        $connection_upgrade;
    }

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

## 6. Activer les sites

```bash
sudo ln -s /etc/nginx/sites-available/yourdomain.com.conf /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/api.yourdomain.com.conf /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

Lancez toujours `sudo nginx -t` avant de recharger — sinon une simple faute de frappe ferait tomber tous les sites.

## 7. HTTPS avec Certbot

Les développeurs de Certbot (EFF) recommandent le paquet **snap**. Les serveurs Ubuntu sont livrés avec `snapd`.

```bash
sudo apt remove -y certbot            # only if you installed the apt version before
sudo snap install --classic certbot
sudo ln -sf /snap/bin/certbot /usr/local/bin/certbot
```

<details>
<summary>Alternative : paquet apt</summary>

```bash
sudo apt install -y certbot python3-certbot-nginx
```

Fonctionne aussi très bien, mais c'est en général une version plus ancienne.

</details>

Obtenez les certificats pour tous vos domaines — Certbot modifie les fichiers Nginx et ajoute la redirection HTTP → HTTPS à votre place :

```bash
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com -d api.yourdomain.com
```

Le renouvellement est automatique (timer systemd). Testez-le :

```bash
sudo certbot renew --dry-run
systemctl list-timers | grep -i certbot
```

Bon à savoir (changements chez Let's Encrypt entre 2025 et 2028) :

- Let's Encrypt **n'envoie plus d'e-mails de rappel d'expiration** (depuis juin 2025) — comptez sur le renouvellement automatique et vérifiez-le avec le `--dry-run`.
- La durée de vie par défaut des certificats passe de 90 à **64 jours en février 2027**, puis à **45 jours en 2028**. Le renouvellement automatique de Certbot s'en charge ; ne programmez simplement jamais votre propre cron de renouvellement tous les 60 jours ou plus.
- Le **port 80 doit rester ouvert** : il sert à valider les renouvellements.

## 8. Réglages recommandés

Activez la compression des ressources texte dans `/etc/nginx/nginx.conf` (dans `http { … }` — la plupart des lignes y sont déjà, en commentaire) :

```nginx
gzip on;
gzip_vary on;
gzip_proxied any;
gzip_comp_level 5;
gzip_types text/plain text/css application/json application/javascript text/xml application/xml image/svg+xml;
```

En-têtes de sécurité de base (à ajouter dans chaque bloc `server { … }` HTTPS) :

```nginx
add_header X-Content-Type-Options "nosniff" always;
add_header X-Frame-Options "SAMEORIGIN" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
# Once HTTPS is confirmed working everywhere:
# add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
```

Masquer la version de Nginx (bloc `http { … }`) :

```nginx
server_tokens off;
```

Puis `sudo nginx -t && sudo systemctl reload nginx`.

---

## Dépannage

| Symptôme | Cause probable / solution |
|---|---|
| **502 Bad Gateway** | L'app Node ne tourne pas ou pas sur ce port → `pm2 status`, `pm2 logs`, `curl -I http://127.0.0.1:4000` |
| **404 au rafraîchissement** (React Router) | Il manque `try_files $uri $uri/ /index.html;` |
| **403 Forbidden** | Mauvais chemin `root`, ou Nginx (`www-data`) ne peut pas lire le dossier → `namei -l /var/www/your-repo/frontend/dist/index.html` |
| **Welcome to nginx!** au lieu de votre site | Site `default` encore actif, ou `server_name` qui ne correspond pas |
| Certbot : *Timeout during connect* | DNS pas encore propagé, port 80 fermé dans UFW ou dans le pare-feu de l'hébergeur |
| `413 Request Entity Too Large` | Augmentez `client_max_body_size` |

Logs :

```bash
sudo tail -f /var/log/nginx/error.log
sudo tail -f /var/log/nginx/access.log
```
