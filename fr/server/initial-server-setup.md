# Configuration initiale du serveur (Ubuntu 24.04 / 26.04)

[🇬🇧 Read in English](../../server/initial-server-setup.md)

À faire **une seule fois**, juste après l'achat du VPS, avant de déployer quoi que ce soit. Comptez environ 15 minutes : cela vous protège des robots qui scannent chaque nouvelle IP dans les minutes qui suivent sa mise en ligne.

- [1. Première connexion et mises à jour](#1-première-connexion-et-mises-à-jour)
- [2. Créer un utilisateur sudo](#2-créer-un-utilisateur-sudo)
- [3. Se connecter avec des clés SSH](#3-se-connecter-avec-des-clés-ssh)
- [4. Durcir SSH](#4-durcir-ssh)
- [5. Pare-feu (UFW)](#5-pare-feu-ufw)
- [6. Fail2ban](#6-fail2ban)
- [7. Mises à jour de sécurité automatiques](#7-mises-à-jour-de-sécurité-automatiques)
- [8. Swap (petit VPS)](#8-swap-petit-vps)
- [Checklist](#checklist)

> Besoin d'un VPS ? Voir les [liens Hostinger dans le README](../README.md#obtenir-un-vps). Choisissez une image **Ubuntu 24.04** ou **26.04** simple.

---

## 1. Première connexion et mises à jour

Depuis votre ordinateur :

```bash
ssh root@your_vps_ip
```

Mettez tout à jour :

```bash
apt update && apt upgrade -y
```

Si la sortie parle d'un nouveau noyau, redémarrez (`reboot`) puis reconnectez-vous.

Réglez le fuseau horaire (facultatif, mais les logs sont plus lisibles) :

```bash
timedatectl set-timezone Europe/Paris   # list them with: timedatectl list-timezones
```

## 2. Créer un utilisateur sudo

Travailler en permanence en `root` est risqué. Créez un utilisateur normal (ici `deploy`) avec les droits `sudo` :

```bash
adduser deploy
usermod -aG sudo deploy
```

## 3. Se connecter avec des clés SSH

**Sur votre ordinateur** (pas sur le serveur), créez une clé si vous n'en avez pas déjà une :

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
```

Copiez la clé publique vers le nouvel utilisateur :

```bash
ssh-copy-id deploy@your_vps_ip
```

> Si vous vous connectez déjà à `root` avec une clé (connexion par mot de passe désactivée), copiez plutôt les clés de root vers le nouvel utilisateur, depuis la session root :
> `rsync --archive --chown=deploy:deploy ~/.ssh /home/deploy`
>
> Sous Windows sans `ssh-copy-id`, collez le contenu de `~/.ssh/id_ed25519.pub` dans `/home/deploy/.ssh/authorized_keys` sur le serveur (permissions : `700` pour `.ssh`, `600` pour le fichier, propriétaire `deploy`).

Testez dans un **nouveau terminal** — vous devez entrer sans le mot de passe du compte :

```bash
ssh deploy@your_vps_ip
```

À partir de maintenant, travaillez en tant que `deploy` et utilisez `sudo`.

## 4. Durcir SSH

À faire uniquement quand la connexion par clé fonctionne, et **gardez votre session actuelle ouverte** tant que vous n'avez pas testé une nouvelle connexion.

Regardez les fichiers de configuration existants (les images cloud en contiennent souvent un qui réactive les mots de passe) :

```bash
ls /etc/ssh/sshd_config.d/
```

Pour chaque option, OpenSSH garde la **première** valeur lue, et les fichiers sont lus par ordre alphabétique : nommez donc votre fichier `00-…` pour qu'il soit prioritaire :

```bash
sudo nano /etc/ssh/sshd_config.d/00-hardening.conf
```

```conf
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
MaxAuthTries 3
```

Vérifiez la syntaxe et les valeurs effectives, puis redémarrez SSH :

```bash
sudo sshd -t
sudo sshd -T | grep -Ei 'permitrootlogin|passwordauthentication'
sudo systemctl restart ssh
```

Ouvrez un **nouveau** terminal et vérifiez que `ssh deploy@your_vps_ip` fonctionne toujours avant de fermer l'ancien.

> Si vous êtes bloqué dehors, utilisez la console web de votre hébergeur (Hostinger : hPanel → VPS → *Terminal du navigateur*) pour corriger le fichier.

## 5. Pare-feu (UFW)

Autorisez SSH **avant** d'activer le pare-feu, sinon vous coupez votre propre connexion :

```bash
sudo ufw allow OpenSSH
sudo ufw enable
sudo ufw status verbose
```

Plus tard, n'ouvrez que ce dont l'extérieur a besoin — en général seulement le trafic web :

```bash
sudo ufw allow 'Nginx Full'   # ports 80 and 443, after installing Nginx
```

N'ouvrez **pas** le port de votre app Node.js (par ex. 4000) ni les ports des bases de données (27017, 5432, 3306, 6379). Nginx et votre app y accèdent en local. Si vous avez vraiment besoin d'un accès distant à une base, autorisez uniquement votre propre IP :

```bash
sudo ufw allow from 203.0.113.10 to any port 5432 proto tcp
```

> Docker publie les ports des conteneurs en modifiant directement iptables et **contourne UFW**. Voir le [guide Docker](../deploy/docker-compose-on-vps.md#avertissement-pare-feu).

Certains hébergeurs (dont Hostinger) proposent aussi un pare-feu dans leur tableau de bord. Vous pouvez utiliser les deux ; vérifiez simplement que SSH, 80 et 443 sont autorisés dans chacun.

## 6. Fail2ban

Bannit les IP qui échouent à se connecter en SSH à répétition.

```bash
sudo apt install -y fail2ban
sudo nano /etc/fail2ban/jail.local
```

```ini
[DEFAULT]
bantime  = 1h
findtime = 10m
maxretry = 5
backend  = systemd

[sshd]
enabled = true
```

```bash
sudo systemctl enable --now fail2ban
sudo systemctl restart fail2ban
sudo fail2ban-client status sshd
```

## 7. Mises à jour de sécurité automatiques

Généralement installé par défaut sur Ubuntu ; vérifiez qu'il est actif :

```bash
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades   # answer "Yes"
```

Vérifiez de temps en temps si un redémarrage est nécessaire :

```bash
[ -f /var/run/reboot-required ] && echo "Reboot needed"
```

## 8. Swap (petit VPS)

`npm install` et `npm run build` peuvent manquer de mémoire sur les offres de 1 à 4 Go. Un fichier de swap évite que le build soit tué :

```bash
free -h                                  # check if swap already exists
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

N'utiliser le swap qu'en dernier recours :

```bash
echo 'vm.swappiness=10' | sudo tee /etc/sysctl.d/99-swappiness.conf
sudo sysctl --system
```

---

## Checklist

- [ ] Système à jour
- [ ] Utilisateur `deploy` avec `sudo`
- [ ] Connexion SSH par clé fonctionnelle
- [ ] Connexion root et par mot de passe désactivées (confirmé par `sshd -T`)
- [ ] UFW actif avec seulement `OpenSSH` (et plus tard `Nginx Full`)
- [ ] Fail2ban en marche
- [ ] Mises à jour automatiques activées
- [ ] Swap configuré (petites offres)

**Ensuite :** installez une [base de données](../README.md#3-bases-de-données), puis [déployez votre app MERN](../deploy/mern-on-vps.md).
