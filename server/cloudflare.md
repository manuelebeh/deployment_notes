# Cloudflare in Front of Your VPS

[🇫🇷 Lire en français](../fr/server/cloudflare.md)

Cloudflare (free plan) gives you fast DNS, a global CDN, DDoS protection and hides your server's IP. It also comes with a few classic traps: redirect loops, wrong visitor IPs, and failed certificate renewals. This guide avoids them.

```text
Browser ──HTTPS──▶ Cloudflare (proxy) ──HTTPS──▶ Nginx on your VPS ──▶ your app
```

- [1. Add your domain to Cloudflare](#1-add-your-domain-to-cloudflare)
- [2. DNS records: proxied or DNS only](#2-dns-records-proxied-or-dns-only)
- [3. SSL/TLS mode: Full (strict)](#3-ssltls-mode-full-strict)
- [4. Certificate on the server](#4-certificate-on-the-server)
- [5. See the real visitor IP](#5-see-the-real-visitor-ip)
- [6. (Optional) Only accept traffic from Cloudflare](#6-optional-only-accept-traffic-from-cloudflare)
- [7. Caching](#7-caching)
- [Limits & troubleshooting](#limits--troubleshooting)

---

## 1. Add your domain to Cloudflare

1. Create a free account at <https://dash.cloudflare.com> → *Add a domain*.
2. Cloudflare imports your existing DNS records — check them.
3. At your registrar, replace the nameservers with the two Cloudflare gives you.
4. Wait for the "Active" status (minutes to a few hours).

## 2. DNS records: proxied or DNS only

| Record | Proxy status | Why |
|---|---|---|
| `A @` → VPS IP | 🟠 Proxied | Website through Cloudflare |
| `A www` → VPS IP | 🟠 Proxied | |
| `A api` → VPS IP | 🟠 Proxied | |
| `MX`, mail-related records | ⚪ DNS only | Email can't go through the proxy |
| `A ssh` / panel subdomains on non-web ports | ⚪ DNS only | The proxy only forwards HTTP(S) on standard ports |

Once proxied, `yourdomain.com` resolves to Cloudflare's IPs. **Keep connecting over SSH with your VPS IP** (or a DNS-only record).

## 3. SSL/TLS mode: Full (strict)

*SSL/TLS → Overview* → choose **Full (strict)**.

| Mode | What happens | Verdict |
|---|---|---|
| Off | No HTTPS | ❌ |
| Flexible | HTTPS to Cloudflare, **plain HTTP** to your server | ❌ insecure, and causes `ERR_TOO_MANY_REDIRECTS` when the server redirects to HTTPS |
| Full | HTTPS to your server, certificate **not checked** | ⚠️ |
| **Full (strict)** | HTTPS to your server, valid certificate required | ✅ |

Also enable *SSL/TLS → Edge Certificates → Always Use HTTPS*.

## 4. Certificate on the server

Full (strict) needs a valid certificate on Nginx. Two options:

### Option A — Let's Encrypt (Certbot), as usual

The HTTP challenge works through the Cloudflare proxy. For the **first** certificate, the simplest is:

1. Set the records to ⚪ DNS only.
2. Run Certbot as in [Nginx & SSL](nginx-and-ssl.md#7-https-with-certbot).
3. Switch the records back to 🟠 Proxied, then set Full (strict).

Renewals then work with the proxy on. Check with `sudo certbot renew --dry-run`.

### Option B — Cloudflare Origin Certificate (valid 15 years)

*SSL/TLS → Origin Server → Create Certificate* → keep the defaults (`yourdomain.com`, `*.yourdomain.com`, 15 years). Save both parts on the server:

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

No renewal to manage, but the certificate is **only trusted by Cloudflare**: if you ever turn the proxy off, browsers will show a warning. Switch back to Let's Encrypt in that case.

## 5. See the real visitor IP

Behind Cloudflare, Nginx sees Cloudflare's IPs instead of your visitors'. Logs, rate limiting and Fail2ban become useless. Fix it with Nginx's `realip` module:

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

`$remote_addr`, the access log, and the `X-Real-IP` / `X-Forwarded-For` headers sent to your app now contain the visitor's IP. Keep `app.set('trust proxy', 1)` in Express.

The list rarely changes, but you can regenerate it automatically:

<details>
<summary>Script to refresh the list (weekly cron)</summary>

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

## 6. (Optional) Only accept traffic from Cloudflare

Your IP may still be known (old DNS history, emails…). Attackers can then bypass Cloudflare by hitting the IP directly. Allow ports 80/443 **only from Cloudflare**:

```bash
sudo ufw delete allow 'Nginx Full'
for ip in $(curl -fsS https://www.cloudflare.com/ips-v4) $(curl -fsS https://www.cloudflare.com/ips-v6); do
  sudo ufw allow proto tcp from "$ip" to any port 80,443 comment 'Cloudflare'
done
sudo ufw status numbered
```

> SSH stays open (`OpenSSH` rule). If you later disable the Cloudflare proxy, re-add `sudo ufw allow 'Nginx Full'` or your site will be unreachable.

## 7. Caching

By default Cloudflare caches **static files by extension** (JS, CSS, images, fonts) and **not** HTML or API responses — that's the right default for React/Next.js apps.

- After a deploy, hashed assets (`/assets/index-a1b2c3.js`, `/_next/static/...`) get new names, so no purge is needed.
- For non-hashed files you changed (e.g. `/logo.png`), use *Caching → Configuration → Purge Cache*.
- Never create a "Cache Everything" rule on API routes or pages showing user-specific data.

---

## Limits & troubleshooting

| Problem | Cause / fix |
|---|---|
| `ERR_TOO_MANY_REDIRECTS` | SSL mode is Flexible → set **Full (strict)** |
| Error **526** *Invalid SSL certificate* | No valid cert on the server in Full (strict) → Certbot or Origin Certificate |
| Error **521** *Web server is down* | Nginx stopped, or UFW blocks Cloudflare IPs |
| Error **522** *Connection timed out* | Server unreachable on 443 (firewall, provider firewall) |
| Error **524** *A timeout occurred* | Your app took > 100 s to answer (free plan limit) → make long tasks asynchronous |
| `413` on upload | Free plan limit is **100 MB** per request |
| Logs show only Cloudflare IPs | Step 5 not done |
| Certbot fails with proxy on | Issue the first certificate in DNS-only mode, or use an Origin Certificate |

WebSockets work through Cloudflare on all plans.
