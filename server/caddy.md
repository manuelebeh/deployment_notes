# Caddy: an Alternative to Nginx (Automatic HTTPS)

[🇫🇷 Lire en français](../fr/server/caddy.md)

[Caddy](https://caddyserver.com) is a web server that gets and renews HTTPS certificates **by itself** — no Certbot. The config for a whole MERN app fits in about 10 lines.

Use **either** Caddy **or** Nginx on a server, not both: they both need ports 80 and 443.

| | Nginx + Certbot | Caddy |
|---|---|---|
| HTTPS | Certbot, separate step | Automatic, built in |
| Config | Verbose, very flexible | Short, sensible defaults |
| Reverse proxy headers, WebSockets | Write them yourself | Automatic |
| Docs & tutorials online | Huge | Smaller but good |
| Performance | Excellent | Excellent for this kind of app |

- [1. Install](#1-install)
- [2. Caddyfile for a MERN app](#2-caddyfile-for-a-mern-app)
- [3. Next.js app](#3-nextjs-app)
- [4. Apply & check](#4-apply--check)
- [5. Useful extras](#5-useful-extras)

---

## 1. Install

If Nginx is installed, stop it first:

```bash
sudo systemctl disable --now nginx
```

Install Caddy from its official repository:

```bash
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https curl
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo chmod o+r /usr/share/keyrings/caddy-stable-archive-keyring.gpg
sudo chmod o+r /etc/apt/sources.list.d/caddy-stable.list
sudo apt update
sudo apt install -y caddy
```

Open the web ports (Caddy also uses UDP 443 for HTTP/3):

```bash
sudo ufw allow 80/tcp
sudo ufw allow 443
```

Make sure your DNS records already point to the VPS (see [Nginx & SSL, step 2](nginx-and-ssl.md#2-point-your-domain-to-the-vps)) — Caddy requests certificates as soon as it starts.

## 2. Caddyfile for a MERN app

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

That's all: HTTPS, HTTP → HTTPS redirect, certificate renewal, `X-Forwarded-*` headers and WebSockets are handled automatically.

<details>
<summary>Single domain: frontend + <code>/api</code></summary>

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

## 3. Next.js app

```caddy
yourdomain.com {
    encode zstd gzip
    reverse_proxy 127.0.0.1:3000 {
        flush_interval -1    # stream responses (Suspense, loading.js)
    }
}
```

See [Next.js on a VPS](../deploy/nextjs-on-vps.md) for running the app itself.

## 4. Apply & check

```bash
caddy fmt --overwrite /etc/caddy/Caddyfile   # format
caddy validate --config /etc/caddy/Caddyfile # check syntax
sudo systemctl reload caddy
```

Logs (certificate issuance shows up here):

```bash
sudo journalctl -u caddy -f
```

In Express, keep `app.set('trust proxy', 1)` so `req.ip` and `req.secure` are correct.

## 5. Useful extras

**Password-protect a staging site:**

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

**Security headers:**

```caddy
header {
    X-Content-Type-Options nosniff
    X-Frame-Options SAMEORIGIN
    Referrer-Policy strict-origin-when-cross-origin
    -Server
}
```

**Behind Cloudflare:** use SSL mode **Full (strict)** — Caddy's own certificate is valid. For real visitor IPs, add in the global options block:

```caddy
{
    servers {
        trusted_proxies static 173.245.48.0/20 103.21.244.0/22 103.22.200.0/22 103.31.4.0/22 141.101.64.0/18 108.162.192.0/18 190.93.240.0/20 188.114.96.0/20 197.234.240.0/22 198.41.128.0/17 162.158.0.0/15 104.16.0.0/13 104.24.0.0/14 172.64.0.0/13 131.0.72.0/22 2400:cb00::/32 2606:4700::/32 2803:f800::/32 2405:b500::/32 2405:8100::/32 2a06:98c0::/29 2c0f:f248::/32
        client_ip_headers CF-Connecting-IP
    }
}
```

More in the [Cloudflare guide](cloudflare.md).
