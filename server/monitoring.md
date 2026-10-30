# Monitoring & Alerts

[🇫🇷 Lire en français](../fr/server/monitoring.md)

Know when your site is down **before** your users tell you, and keep an eye on disk, memory and logs.

- [1. Health check endpoint](#1-health-check-endpoint)
- [2. Uptime monitoring](#2-uptime-monitoring)
- [3. Disk & memory alerts](#3-disk--memory-alerts)
- [4. Logs](#4-logs)
- [5. Real-time dashboards (optional)](#5-real-time-dashboards-optional)
- [Quick health check commands](#quick-health-check-commands)

---

## 1. Health check endpoint

Give monitors something meaningful to check — not just "the server answers", but "the app can reach its database":

```js
// Express
import mongoose from 'mongoose';

app.get('/health', async (req, res) => {
  try {
    await mongoose.connection.db.admin().ping();   // or: await pool.query('SELECT 1')
    res.json({ status: 'ok', uptime: process.uptime() });
  } catch {
    res.status(503).json({ status: 'error', db: 'unreachable' });
  }
});
```

Test: `curl -i https://api.yourdomain.com/health`

## 2. Uptime monitoring

### Option A — a hosted service (recommended for the main check)

A monitor running **on the same VPS** goes down with it. For the "is my site up?" check, use an external service — the free plans are enough: e.g. UptimeRobot, Better Stack, Hetrix Tools. Point it at `https://yourdomain.com` and `https://api.yourdomain.com/health`, with alerts by email, Telegram, Discord or Slack.

### Option B — Uptime Kuma (self-hosted)

Nice dashboard, public status pages, 90+ notification channels. Best installed on **another** small server; on the same VPS it still catches app crashes, expired certificates, etc.

Requires Docker ([install guide](../deploy/docker-compose-on-vps.md#1-install-docker-engine)):

```bash
docker run -d --restart=unless-stopped \
  -p 127.0.0.1:3001:3001 \
  -v uptime-kuma:/app/data \
  --name uptime-kuma louislam/uptime-kuma:2
```

Expose it on `status.yourdomain.com` with Nginx (`proxy_pass http://127.0.0.1:3001;` + WebSocket headers, see [Nginx & SSL](nginx-and-ssl.md#4-server-block-nodejs-api-reverse-proxy)) and get a certificate with Certbot. Create the admin account **immediately**.

Monitors worth adding:

| Monitor | Type |
|---|---|
| `https://yourdomain.com` | HTTP(s) — also alerts before the certificate expires |
| `https://api.yourdomain.com/health` | HTTP(s) — keyword `"ok"` |
| MongoDB / PostgreSQL / MySQL / Redis | Database-specific monitor types |
| Backup script | Push monitor — call its URL at the end of `backup-db.sh`, get alerted if it stops |

Update: `docker pull louislam/uptime-kuma:2 && docker rm -f uptime-kuma` then run the same `docker run` command again (data is kept in the volume).

## 3. Disk & memory alerts

A full disk is the #1 silent killer on a VPS (logs, Docker images, backups). This script sends a message to a Discord/Slack webhook or to [ntfy](https://ntfy.sh) when thresholds are crossed:

```bash
sudo nano /usr/local/bin/server-alerts.sh
```

```bash
#!/usr/bin/env bash
set -euo pipefail

WEBHOOK_URL="https://discord.com/api/webhooks/XXX/YYY"   # or https://ntfy.sh/your-secret-topic
DISK_LIMIT=85   # %
MEM_LIMIT=90    # %

HOST=$(hostname)
DISK=$(df --output=pcent / | tail -1 | tr -dc '0-9')
MEM=$(free | awk '/Mem:/ {printf "%d", ($2-$7)/$2*100}')

notify() {
  if [[ "$WEBHOOK_URL" == *discord.com* ]]; then
    curl -fsS -H 'Content-Type: application/json' -d "{\"content\":\"$1\"}" "$WEBHOOK_URL" >/dev/null
  else
    curl -fsS -d "$1" "$WEBHOOK_URL" >/dev/null
  fi
}

(( DISK >= DISK_LIMIT )) && notify "⚠️ $HOST: disk at ${DISK}%"
(( MEM  >= MEM_LIMIT  )) && notify "⚠️ $HOST: memory at ${MEM}%"
exit 0
```

```bash
sudo chmod 700 /usr/local/bin/server-alerts.sh
sudo /usr/local/bin/server-alerts.sh                       # test (lower the limits to try it)
echo '*/10 * * * * root /usr/local/bin/server-alerts.sh' | sudo tee /etc/cron.d/server-alerts
```

Free up space when needed:

```bash
sudo du -xh / --max-depth=2 2>/dev/null | sort -h | tail -20   # biggest folders
sudo journalctl --vacuum-size=200M
docker system prune -af                                        # unused images/containers (if Docker)
sudo apt autoremove --purge -y && sudo apt clean
```

## 4. Logs

Where to look:

| What | Command |
|---|---|
| Node app (PM2) | `pm2 logs project-backend --lines 100` |
| Nginx errors | `sudo tail -f /var/log/nginx/error.log` |
| A systemd service | `sudo journalctl -u mongod -n 100 -f` |
| SSH logins / attacks | `sudo journalctl -u ssh -n 50`, `sudo fail2ban-client status sshd` |
| Docker container | `docker compose logs -f --tail=100 api` |

Keep logs from filling the disk:

- PM2: `pm2 install pm2-logrotate` (see [MERN guide](../deploy/mern-on-vps.md#4-run-the-express-backend-with-pm2))
- Docker: size limits in `/etc/docker/daemon.json` (see [Docker guide](../deploy/docker-compose-on-vps.md#1-install-docker-engine))
- journald — cap its size:

  ```bash
  sudo mkdir -p /etc/systemd/journald.conf.d
  printf '[Journal]\nSystemMaxUse=500M\n' | sudo tee /etc/systemd/journald.conf.d/size.conf
  sudo systemctl restart systemd-journald
  ```

- Nginx logs are rotated daily by `logrotate` out of the box.

**Application errors:** for production apps, an error tracker (Sentry, GlitchTip — self-hostable, Better Stack…) shows stack traces and alerts on new errors, which logs alone don't do.

## 5. Real-time dashboards (optional)

- `htop` — processes, CPU, RAM: `sudo apt install -y htop`
- `pm2 monit` — CPU/RAM per Node process
- `ncdu /` — interactive disk usage: `sudo apt install -y ncdu`
- **Netdata** — full real-time dashboard with built-in alerts, very light:

  ```bash
  curl -fsSL https://get.netdata.cloud/kickstart.sh -o /tmp/netdata-kickstart.sh
  sh /tmp/netdata-kickstart.sh --stable-channel
  ```

  It listens on port 19999. Don't open it in UFW — access it via an SSH tunnel: `ssh -N -L 19999:127.0.0.1:19999 deploy@your_vps_ip` then open `http://localhost:19999`.

---

## Quick health check commands

```bash
uptime                       # load average
free -h                      # memory & swap
df -h                        # disk
pm2 status                   # Node processes
systemctl --failed           # crashed services
sudo ss -ltnp                # what listens on which port
sudo ufw status              # firewall
sudo certbot certificates    # certificate expiry dates
```
