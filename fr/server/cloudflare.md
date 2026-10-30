# Cloudflare devant votre VPS

[🇬🇧 Read in English](../../server/cloudflare.md)

Cloudflare (offre gratuite) vous apporte un DNS rapide, un CDN mondial, une protection DDoS, et masque l'IP de votre serveur. Mais il y a quelques pièges classiques : boucles de redirection, mauvaise IP des visiteurs, renouvellement de certificat qui échoue. Ce guide les évite.

```text
Browser ──HTTPS──▶ Cloudflare (proxy) ──HTTPS──▶ Nginx on your VPS ──▶ your app
```

- [1. Ajouter votre domaine à Cloudflare](#1-ajouter-votre-domaine-à-cloudflare)
- [2. Enregistrements DNS : proxifiés ou DNS uniquement](#2-enregistrements-dns--proxifiés-ou-dns-uniquement)
- [3. Mode SSL/TLS : Full (strict)](#3-mode-ssltls--full-strict)
- [4. Certificat sur le serveur](#4-certificat-sur-le-serveur)
- [5. Voir la vraie IP des visiteurs](#5-voir-la-vraie-ip-des-visiteurs)
- [6. (Facultatif) N'accepter que le trafic de Cloudflare](#6-facultatif-naccepter-que-le-trafic-de-cloudflare)
- [7. Cache](#7-cache)
- [Limites et dépannage](#limites-et-dépannage)

---

## 1. Ajouter votre domaine à Cloudflare

1. Créez un compte gratuit sur <https://dash.cloudflare.com> → *Add a domain*.
2. Cloudflare importe vos enregistrements DNS existants — vérifiez-les.
3. Chez votre registrar, remplacez les serveurs de noms (nameservers) par les deux que Cloudflare vous donne.
4. Attendez le statut « Active » (de quelques minutes à quelques heures).

## 2. Enregistrements DNS : proxifiés ou DNS uniquement

| Enregistrement | Statut du proxy | Pourquoi |
|---|---|---|
| `A @` → IP du VPS | 🟠 Proxied | Site web via Cloudflare |
| `A www` → IP du VPS | 🟠 Proxied | |
| `A api` → IP du VPS | 🟠 Proxied | |
| `MX`, enregistrements liés aux e-mails | ⚪ DNS only | Les e-mails ne passent pas par le proxy |
| `A ssh` / sous-domaines de panels sur des ports non web | ⚪ DNS only | Le proxy ne transmet que le HTTP(S) sur les ports standards |

Une fois proxifié, `yourdomain.com` pointe vers les IP de Cloudflare. **Continuez à vous connecter en SSH avec l'IP du VPS** (ou un enregistrement en DNS only).

## 3. Mode SSL/TLS : Full (strict)

*SSL/TLS → Overview* → choisissez **Full (strict)**.

| Mode | Ce qui se passe | Verdict |
|---|---|---|
| Off | Pas de HTTPS | ❌ |
| Flexible | HTTPS jusqu'à Cloudflare, puis **HTTP en clair** jusqu'à votre serveur | ❌ non sécurisé, et provoque `ERR_TOO_MANY_REDIRECTS` quand le serveur redirige vers HTTPS |
| Full | HTTPS jusqu'à votre serveur, certificat **non vérifié** | ⚠️ |
| **Full (strict)** | HTTPS jusqu'à votre serveur, certificat valide exigé | ✅ |

Activez aussi *SSL/TLS → Edge Certificates → Always Use HTTPS*.

## 4. Certificat sur le serveur

Le mode Full (strict) exige un certificat valide sur Nginx. Deux options :

### Option A — Let's Encrypt (Certbot), comme d'habitude

Le challenge HTTP fonctionne à travers le proxy Cloudflare. Pour le **premier** certificat, le plus simple est :

1. Passez les enregistrements en ⚪ DNS only.
2. Lancez Certbot comme dans [Nginx & SSL](nginx-and-ssl.md#7-https-avec-certbot).
3. Repassez les enregistrements en 🟠 Proxied, puis choisissez Full (strict).

Les renouvellements fonctionnent ensuite avec le proxy activé. Vérifiez avec `sudo certbot renew --dry-run`.

### Option B — Certificat d'origine Cloudflare (valable 15 ans)

*SSL/TLS → Origin Server → Create Certificate* → gardez les valeurs par défaut (`yourdomain.com`, `*.yourdomain.com`, 15 ans). Enregistrez les deux parties sur le serveur :

```bash
sudo mkdir -p /etc/ssl/cloudflare
sudo nano /etc/ssl/cloudflare/yourdomain.com.pem    # paste the "Origin Certificate"
sudo nano /etc/ssl/cloudflare/yourdomain.com.key    # paste the "Private key"
sudo chmod 600 /etc/ssl/cloudflare/yourdomain.com.key
```

```nginx
server {
    listen 443 ssl http2;        # Nginx ≥ 1.25.1 (Ubuntu 26.04): "listen 443 ssl;" + "http2 on;"
    listen [::]:443 ssl http2;
    server_name yourdomain.com www.yourdomain.com;

    ssl_certificate     /etc/ssl/cloudflare/yourdomain.com.pem;
    ssl_certificate_key /etc/ssl/cloudflare/yourdomain.com.key;

    # ... root / location blocks as before
}

server {
    listen 80;
    listen [::]:80;
    server_name yourdomain.com www.yourdomain.com;
    return 301 https://$host$request_uri;
}
```

Aucun renouvellement à gérer, mais ce certificat n'est **reconnu que par Cloudflare** : si vous désactivez un jour le proxy, les navigateurs afficheront un avertissement. Repassez alors à Let's Encrypt.

## 5. Voir la vraie IP des visiteurs

Derrière Cloudflare, Nginx voit les IP de Cloudflare au lieu de celles de vos visiteurs. Les logs, la limitation de débit et Fail2ban deviennent inutiles. Corrigez-le avec le module `realip` de Nginx :

```bash
sudo nano /etc/nginx/conf.d/cloudflare-realip.conf
```

```nginx
# Cloudflare IP ranges — https://www.cloudflare.com/ips/
set_real_ip_from 173.245.48.0/20;
set_real_ip_from 103.21.244.0/22;
set_real_ip_from 103.22.200.0/22;
set_real_ip_from 103.31.4.0/22;
set_real_ip_from 141.101.64.0/18;
set_real_ip_from 108.162.192.0/18;
set_real_ip_from 190.93.240.0/20;
set_real_ip_from 188.114.96.0/20;
set_real_ip_from 197.234.240.0/22;
set_real_ip_from 198.41.128.0/17;
set_real_ip_from 162.158.0.0/15;
set_real_ip_from 104.16.0.0/13;
set_real_ip_from 104.24.0.0/14;
set_real_ip_from 172.64.0.0/13;
set_real_ip_from 131.0.72.0/22;
set_real_ip_from 2400:cb00::/32;
set_real_ip_from 2606:4700::/32;
set_real_ip_from 2803:f800::/32;
set_real_ip_from 2405:b500::/32;
set_real_ip_from 2405:8100::/32;
set_real_ip_from 2a06:98c0::/29;
set_real_ip_from 2c0f:f248::/32;

real_ip_header CF-Connecting-IP;
```

```bash
sudo nginx -t && sudo systemctl reload nginx
```

`$remote_addr`, le log d'accès et les en-têtes `X-Real-IP` / `X-Forwarded-For` envoyés à votre app contiennent maintenant l'IP du visiteur. Gardez `app.set('trust proxy', 1)` dans Express.

La liste change rarement, mais vous pouvez la régénérer automatiquement :

<details>
<summary>Script de mise à jour de la liste (cron hebdomadaire)</summary>

```bash
sudo tee /usr/local/bin/update-cloudflare-realip.sh >/dev/null <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
OUT=/etc/nginx/conf.d/cloudflare-realip.conf
TMP=$(mktemp)
{
  echo "# Generated $(date -I) from https://www.cloudflare.com/ips/"
  for ip in $(curl -fsS https://www.cloudflare.com/ips-v4) $(curl -fsS https://www.cloudflare.com/ips-v6); do
    echo "set_real_ip_from $ip;"
  done
  echo "real_ip_header CF-Connecting-IP;"
} > "$TMP"
install -m 644 "$TMP" "$OUT" && rm -f "$TMP"
nginx -t && systemctl reload nginx
EOF
sudo chmod +x /usr/local/bin/update-cloudflare-realip.sh
echo '0 4 * * 1 root /usr/local/bin/update-cloudflare-realip.sh' | sudo tee /etc/cron.d/cloudflare-realip
```

</details>

## 6. (Facultatif) N'accepter que le trafic de Cloudflare

Votre IP peut rester connue (anciens historiques DNS, e-mails…). Un attaquant peut alors contourner Cloudflare en visant directement l'IP. N'autorisez les ports 80/443 **que depuis Cloudflare** :

```bash
sudo ufw delete allow 'Nginx Full'
for ip in $(curl -fsS https://www.cloudflare.com/ips-v4) $(curl -fsS https://www.cloudflare.com/ips-v6); do
  sudo ufw allow proto tcp from "$ip" to any port 80,443 comment 'Cloudflare'
done
sudo ufw status numbered
```

> SSH reste ouvert (règle `OpenSSH`). Si vous désactivez plus tard le proxy Cloudflare, rajoutez `sudo ufw allow 'Nginx Full'`, sinon votre site sera inaccessible.

## 7. Cache

Par défaut, Cloudflare met en cache les **fichiers statiques selon leur extension** (JS, CSS, images, polices) et **pas** le HTML ni les réponses d'API — c'est le bon comportement pour des apps React/Next.js.

- Après un déploiement, les fichiers versionnés par hash (`/assets/index-a1b2c3.js`, `/_next/static/...`) changent de nom : pas besoin de purger le cache.
- Pour un fichier non versionné que vous avez modifié (par ex. `/logo.png`), utilisez *Caching → Configuration → Purge Cache*.
- Ne créez jamais de règle « Cache Everything » sur des routes d'API ou des pages affichant des données propres à un utilisateur.

---

## Limites et dépannage

| Problème | Cause / solution |
|---|---|
| `ERR_TOO_MANY_REDIRECTS` | Le mode SSL est Flexible → passez en **Full (strict)** |
| Erreur **526** *Invalid SSL certificate* | Pas de certificat valide sur le serveur en Full (strict) → Certbot ou certificat d'origine |
| Erreur **521** *Web server is down* | Nginx arrêté, ou UFW bloque les IP de Cloudflare |
| Erreur **522** *Connection timed out* | Serveur injoignable sur le port 443 (pare-feu, pare-feu de l'hébergeur) |
| Erreur **524** *A timeout occurred* | Votre app a mis plus de 100 s à répondre (limite de l'offre gratuite) → rendez les tâches longues asynchrones |
| `413` lors d'un upload | Limite de **100 Mo** par requête sur l'offre gratuite |
| Les logs n'affichent que des IP Cloudflare | L'étape 5 n'a pas été faite |
| Certbot échoue avec le proxy activé | Émettez le premier certificat en mode DNS only, ou utilisez un certificat d'origine |

Les WebSockets fonctionnent à travers Cloudflare sur toutes les offres.
