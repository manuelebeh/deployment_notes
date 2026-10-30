# Caddy : une alternative à Nginx (HTTPS automatique)

[🇬🇧 Read in English](../../server/caddy.md)

[Caddy](https://caddyserver.com) est un serveur web qui obtient et renouvelle les certificats HTTPS **tout seul** — pas besoin de Certbot. La config d'une app MERN complète tient en une dizaine de lignes.

Utilisez **soit** Caddy **soit** Nginx sur un serveur, pas les deux : ils ont tous les deux besoin des ports 80 et 443.

| | Nginx + Certbot | Caddy |
|---|---|---|
| HTTPS | Certbot, étape séparée | Automatique, intégré |
| Config | Verbeuse, très flexible | Courte, bons réglages par défaut |
| En-têtes de reverse proxy, WebSockets | À écrire soi-même | Automatiques |
| Docs et tutos en ligne | Énormément | Moins, mais de bonne qualité |
| Performances | Excellentes | Excellentes pour ce type d'app |

- [1. Installation](#1-installation)
- [2. Caddyfile pour une app MERN](#2-caddyfile-pour-une-app-mern)
- [3. App Next.js](#3-app-nextjs)
- [4. Appliquer et vérifier](#4-appliquer-et-vérifier)
- [5. Extras utiles](#5-extras-utiles)

---

## 1. Installation

Si Nginx est installé, arrêtez-le d'abord :

```bash
sudo systemctl disable --now nginx
```

Installez Caddy depuis son dépôt officiel :

```bash
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https curl
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo chmod o+r /usr/share/keyrings/caddy-stable-archive-keyring.gpg
sudo chmod o+r /etc/apt/sources.list.d/caddy-stable.list
sudo apt update
sudo apt install -y caddy
```

Ouvrez les ports web (Caddy utilise aussi l'UDP 443 pour HTTP/3) :

```bash
sudo ufw allow 80/tcp
sudo ufw allow 443
```

Vérifiez que vos enregistrements DNS pointent déjà vers le VPS (voir [Nginx & SSL, étape 2](nginx-and-ssl.md#2-pointer-votre-domaine-vers-le-vps)) — Caddy demande les certificats dès son démarrage.

## 2. Caddyfile pour une app MERN

```bash
sudo nano /etc/caddy/Caddyfile
```

```caddy
{
    email your_email@example.com
}

# React / Vite frontend
yourdomain.com {
    root * /var/www/your-repo/frontend/dist
    encode zstd gzip

    @assets path /assets/*
    header @assets Cache-Control "public, max-age=31536000, immutable"

    try_files {path} /index.html
    file_server
}

# Redirect www → apex
www.yourdomain.com {
    redir https://yourdomain.com{uri} permanent
}

# Express API
api.yourdomain.com {
    encode zstd gzip
    reverse_proxy 127.0.0.1:4000
}
```

C'est tout : HTTPS, redirection HTTP → HTTPS, renouvellement des certificats, en-têtes `X-Forwarded-*` et WebSockets sont gérés automatiquement.

<details>
<summary>Un seul domaine : frontend + <code>/api</code></summary>

```caddy
yourdomain.com {
    encode zstd gzip

    handle /api/* {
        reverse_proxy 127.0.0.1:4000
    }

    handle {
        root * /var/www/your-repo/frontend/dist
        try_files {path} /index.html
        file_server
    }
}
```

</details>

## 3. App Next.js

```caddy
yourdomain.com {
    encode zstd gzip
    reverse_proxy 127.0.0.1:3000 {
        flush_interval -1    # stream responses (Suspense, loading.js)
    }
}
```

Voir [Next.js sur un VPS](../deploy/nextjs-on-vps.md) pour faire tourner l'app elle-même.

## 4. Appliquer et vérifier

```bash
caddy fmt --overwrite /etc/caddy/Caddyfile   # format
caddy validate --config /etc/caddy/Caddyfile # check syntax
sudo systemctl reload caddy
```

Logs (l'émission des certificats apparaît ici) :

```bash
sudo journalctl -u caddy -f
```

Dans Express, gardez `app.set('trust proxy', 1)` pour que `req.ip` et `req.secure` soient corrects.

## 5. Extras utiles

**Protéger un site de staging par mot de passe :**

```bash
caddy hash-password    # type the password, copy the hash
```

```caddy
staging.yourdomain.com {
    basic_auth {
        admin <paste-the-hash-here>
    }
    reverse_proxy 127.0.0.1:4100
}
```

**En-têtes de sécurité :**

```caddy
header {
    X-Content-Type-Options nosniff
    X-Frame-Options SAMEORIGIN
    Referrer-Policy strict-origin-when-cross-origin
    -Server
}
```

**Derrière Cloudflare :** utilisez le mode SSL **Full (strict)** — le certificat de Caddy est valide. Pour avoir la vraie IP des visiteurs, ajoutez dans le bloc d'options globales :

```caddy
{
    servers {
        trusted_proxies static 173.245.48.0/20 103.21.244.0/22 103.22.200.0/22 103.31.4.0/22 141.101.64.0/18 108.162.192.0/18 190.93.240.0/20 188.114.96.0/20 197.234.240.0/22 198.41.128.0/17 162.158.0.0/15 104.16.0.0/13 104.24.0.0/14 172.64.0.0/13 131.0.72.0/22 2400:cb00::/32 2606:4700::/32 2803:f800::/32 2405:b500::/32 2405:8100::/32 2a06:98c0::/29 2c0f:f248::/32
        client_ip_headers CF-Connecting-IP
    }
}
```

Plus de détails dans le [guide Cloudflare](cloudflare.md).
