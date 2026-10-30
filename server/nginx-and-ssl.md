# Nginx & SSL (Let's Encrypt)

Serve your frontends and proxy your API with Nginx, then add free HTTPS certificates that renew themselves.

- [1. Install Nginx](#1-install-nginx)
- [2. Point your domain to the VPS](#2-point-your-domain-to-the-vps)
- [3. Server block: React / Vite SPA](#3-server-block-react--vite-spa)
- [4. Server block: Node.js API (reverse proxy)](#4-server-block-nodejs-api-reverse-proxy)
- [5. Alternative: one domain, frontend + `/api`](#5-alternative-one-domain-frontend--api)
- [6. Enable sites](#6-enable-sites)
- [7. HTTPS with Certbot](#7-https-with-certbot)
- [8. Recommended extras](#8-recommended-extras)
- [Troubleshooting](#troubleshooting)

---

## 1. Install Nginx

```bash
sudo apt update
sudo apt install -y nginx
sudo ufw allow 'Nginx Full'
```

Visit `http://your_vps_ip` — you should see the Nginx welcome page.

Remove the default site so it doesn't catch your domains:

```bash
sudo rm /etc/nginx/sites-enabled/default
```

## 2. Point your domain to the VPS

In your domain registrar's DNS manager, create:

| Type | Name | Value |
|---|---|---|
| A | `@` | `your_vps_ip` |
| A | `www` | `your_vps_ip` |
| A | `api` | `your_vps_ip` |
| AAAA *(optional)* | same names | your VPS IPv6 |

Check propagation (can take a few minutes to a few hours):

```bash
dig +short yourdomain.com
dig +short api.yourdomain.com
```

> Only add AAAA records if your VPS really has IPv6 configured — a wrong AAAA record will make Let's Encrypt validation fail.

## 3. Server block: React / Vite SPA

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

Repeat with another file for each additional frontend (`yourdomain2.com.conf`, `admin.yourdomain.com.conf`, …).

## 4. Server block: Node.js API (reverse proxy)

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

The `$connection_upgrade` variable must be defined once, globally:

```bash
sudo nano /etc/nginx/conf.d/websocket-upgrade.conf
```

```nginx
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}
```

In Express, trust the proxy so `req.ip`, `req.secure` and rate limiters see the real client:

```js
app.set('trust proxy', 1);
```

## 5. Alternative: one domain, frontend + `/api`

If you'd rather avoid a subdomain (and CORS), serve both from the same domain:

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

## 6. Enable sites

```bash
sudo ln -s /etc/nginx/sites-available/yourdomain.com.conf /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/api.yourdomain.com.conf /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

Always run `sudo nginx -t` before reloading — a typo would otherwise take every site down.

## 7. HTTPS with Certbot

Certbot's developers (EFF) recommend the **snap** package. Ubuntu servers ship with `snapd`.

```bash
sudo apt remove -y certbot            # only if you installed the apt version before
sudo snap install --classic certbot
sudo ln -sf /snap/bin/certbot /usr/local/bin/certbot
```

<details>
<summary>Alternative: apt package</summary>

```bash
sudo apt install -y certbot python3-certbot-nginx
```

Works fine too, but it's usually an older version.
</details>

Get certificates for all your domains — Certbot edits the Nginx files and adds the HTTP → HTTPS redirect for you:

```bash
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com -d api.yourdomain.com
```

Renewal is automatic (systemd timer). Test it:

```bash
sudo certbot renew --dry-run
systemctl list-timers | grep -i certbot
```

Good to know (2025–2028 changes at Let's Encrypt):

- Let's Encrypt **no longer sends expiry reminder emails** (since June 2025) — rely on auto-renewal and check it with the dry run.
- Default certificate lifetime drops from 90 to **64 days in February 2027**, then **45 days in 2028**. Certbot's automatic renewal handles this; just never renew by hand with a cron of your own every 60+ days.
- Port **80 must stay open**: it's used to validate renewals.

## 8. Recommended extras

Enable compression for text assets in `/etc/nginx/nginx.conf` (inside `http { … }`, most lines are already there commented out):

```nginx
gzip on;
gzip_vary on;
gzip_proxied any;
gzip_comp_level 5;
gzip_types text/plain text/css application/json application/javascript text/xml application/xml image/svg+xml;
```

Basic security headers (add inside each HTTPS `server { … }` block):

```nginx
add_header X-Content-Type-Options "nosniff" always;
add_header X-Frame-Options "SAMEORIGIN" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
# Once HTTPS is confirmed working everywhere:
# add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
```

Hide the Nginx version (`http { … }` block):

```nginx
server_tokens off;
```

Then `sudo nginx -t && sudo systemctl reload nginx`.

---

## Troubleshooting

| Symptom | Likely cause / fix |
|---|---|
| **502 Bad Gateway** | The Node app isn't running or not on that port → `pm2 status`, `pm2 logs`, `curl -I http://127.0.0.1:4000` |
| **404 on page refresh** (React Router) | Missing `try_files $uri $uri/ /index.html;` |
| **403 Forbidden** | Wrong `root` path, or Nginx (`www-data`) can't read the folder → `namei -l /var/www/your-repo/frontend/dist/index.html` |
| **Welcome to nginx!** instead of your site | `default` site still enabled, or `server_name` doesn't match |
| Certbot: *Timeout during connect* | DNS not propagated yet, port 80 closed in UFW or in the provider's firewall |
| `413 Request Entity Too Large` | Increase `client_max_body_size` |

Logs:

```bash
sudo tail -f /var/log/nginx/error.log
sudo tail -f /var/log/nginx/access.log
```
