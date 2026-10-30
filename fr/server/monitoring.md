# Monitoring & alertes

[🇬🇧 Read in English](../../server/monitoring.md)

Sachez que votre site est en panne **avant** que vos utilisateurs ne vous le disent, et gardez un œil sur le disque, la mémoire et les logs.

- [1. Endpoint de health check](#1-endpoint-de-health-check)
- [2. Surveillance de disponibilité](#2-surveillance-de-disponibilité)
- [3. Alertes disque et mémoire](#3-alertes-disque-et-mémoire)
- [4. Logs](#4-logs)
- [5. Tableaux de bord en temps réel (facultatif)](#5-tableaux-de-bord-en-temps-réel-facultatif)
- [Commandes de vérification rapide](#commandes-de-vérification-rapide)

---

## 1. Endpoint de health check

Donnez aux outils de surveillance quelque chose de parlant à vérifier — pas seulement « le serveur répond », mais « l'app arrive à joindre sa base de données » :

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

Test : `curl -i https://api.yourdomain.com/health`

## 2. Surveillance de disponibilité

### Option A — un service hébergé (recommandé pour la vérification principale)

Un outil de surveillance installé **sur le même VPS** tombe en même temps que lui. Pour la question « mon site est-il en ligne ? », utilisez un service externe — les offres gratuites suffisent : par ex. UptimeRobot, Better Stack, Hetrix Tools. Faites-le pointer vers `https://yourdomain.com` et `https://api.yourdomain.com/health`, avec des alertes par e-mail, Telegram, Discord ou Slack.

### Option B — Uptime Kuma (auto-hébergé)

Joli tableau de bord, pages de statut publiques, plus de 90 canaux de notification. Idéalement installé sur un **autre** petit serveur ; sur le même VPS, il détecte quand même les plantages d'app, les certificats expirés, etc.

Nécessite Docker ([guide d'installation](../deploy/docker-compose-on-vps.md#1-installer-docker-engine)) :

```bash
docker run -d --restart=unless-stopped \
  -p 127.0.0.1:3001:3001 \
  -v uptime-kuma:/app/data \
  --name uptime-kuma louislam/uptime-kuma:2
```

Exposez-le sur `status.yourdomain.com` avec Nginx (`proxy_pass http://127.0.0.1:3001;` + en-têtes WebSocket, voir [Nginx & SSL](nginx-and-ssl.md#4-bloc-serveur--api-nodejs-reverse-proxy)) et obtenez un certificat avec Certbot. Créez le compte admin **tout de suite**.

Sondes utiles à ajouter :

| Sonde | Type |
|---|---|
| `https://yourdomain.com` | HTTP(s) — alerte aussi avant l'expiration du certificat |
| `https://api.yourdomain.com/health` | HTTP(s) — mot-clé `"ok"` |
| MongoDB / PostgreSQL / MySQL / Redis | Types de sondes dédiés aux bases de données |
| Script de sauvegarde | Sonde « Push » — appelez son URL à la fin de `backup-db.sh` pour être alerté s'il ne tourne plus |

Mise à jour : `docker pull louislam/uptime-kuma:2 && docker rm -f uptime-kuma` puis relancez la même commande `docker run` (les données sont conservées dans le volume).

## 3. Alertes disque et mémoire

Un disque plein est le tueur silencieux n° 1 sur un VPS (logs, images Docker, sauvegardes). Ce script envoie un message à un webhook Discord/Slack ou à [ntfy](https://ntfy.sh) quand un seuil est dépassé :

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

Libérer de l'espace si besoin :

```bash
sudo du -xh / --max-depth=2 2>/dev/null | sort -h | tail -20   # biggest folders
sudo journalctl --vacuum-size=200M
docker system prune -af                                        # unused images/containers (if Docker)
sudo apt autoremove --purge -y && sudo apt clean
```

## 4. Logs

Où regarder :

| Quoi | Commande |
|---|---|
| App Node (PM2) | `pm2 logs project-backend --lines 100` |
| Erreurs Nginx | `sudo tail -f /var/log/nginx/error.log` |
| Un service systemd | `sudo journalctl -u mongod -n 100 -f` |
| Connexions / attaques SSH | `sudo journalctl -u ssh -n 50`, `sudo fail2ban-client status sshd` |
| Conteneur Docker | `docker compose logs -f --tail=100 api` |

Empêcher les logs de remplir le disque :

- PM2 : `pm2 install pm2-logrotate` (voir le [guide MERN](../deploy/mern-on-vps.md#4-lancer-le-backend-express-avec-pm2))
- Docker : limites de taille dans `/etc/docker/daemon.json` (voir le [guide Docker](../deploy/docker-compose-on-vps.md#1-installer-docker-engine))
- journald — limitez sa taille :

  ```bash
  sudo mkdir -p /etc/systemd/journald.conf.d
  printf '[Journal]\nSystemMaxUse=500M\n' | sudo tee /etc/systemd/journald.conf.d/size.conf
  sudo systemctl restart systemd-journald
  ```

- Les logs Nginx sont déjà archivés chaque jour par `logrotate`.

**Erreurs applicatives :** pour une app en production, un outil de suivi d'erreurs (Sentry, GlitchTip — auto-hébergeable, Better Stack…) affiche les stack traces et alerte sur les nouvelles erreurs, ce que les logs seuls ne font pas.

## 5. Tableaux de bord en temps réel (facultatif)

- `htop` — processus, CPU, RAM : `sudo apt install -y htop`
- `pm2 monit` — CPU/RAM par processus Node
- `ncdu /` — occupation du disque en interactif : `sudo apt install -y ncdu`
- **Netdata** — tableau de bord complet en temps réel avec alertes intégrées, très léger :

  ```bash
  curl -fsSL https://get.netdata.cloud/kickstart.sh -o /tmp/netdata-kickstart.sh
  sh /tmp/netdata-kickstart.sh --stable-channel
  ```

  Il écoute sur le port 19999. Ne l'ouvrez pas dans UFW — accédez-y par un tunnel SSH : `ssh -N -L 19999:127.0.0.1:19999 deploy@your_vps_ip` puis ouvrez `http://localhost:19999`.

---

## Commandes de vérification rapide

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
