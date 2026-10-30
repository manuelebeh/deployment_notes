# Initial Server Setup (Ubuntu 24.04 / 26.04)

Do this **once**, right after buying your VPS, before deploying anything. It takes ~15 minutes and protects you from the bots that start scanning every new IP within minutes.

- [1. First login & updates](#1-first-login--updates)
- [2. Create a sudo user](#2-create-a-sudo-user)
- [3. Log in with SSH keys](#3-log-in-with-ssh-keys)
- [4. Harden SSH](#4-harden-ssh)
- [5. Firewall (UFW)](#5-firewall-ufw)
- [6. Fail2ban](#6-fail2ban)
- [7. Automatic security updates](#7-automatic-security-updates)
- [8. Swap (small VPS)](#8-swap-small-vps)
- [Checklist](#checklist)

> Need a VPS? See the [Hostinger links in the README](../README.md#get-a-vps). Choose a plain **Ubuntu 24.04** or **26.04** image.

---

## 1. First login & updates

From your computer:

```bash
ssh root@your_vps_ip
```

Update everything:

```bash
apt update && apt upgrade -y
```

If the output mentions a new kernel, reboot (`reboot`) and log in again.

Set the timezone (optional, makes logs easier to read):

```bash
timedatectl set-timezone Europe/Paris   # list them with: timedatectl list-timezones
```

## 2. Create a sudo user

Working as `root` all the time is risky. Create a regular user (here `deploy`) with `sudo` rights:

```bash
adduser deploy
usermod -aG sudo deploy
```

## 3. Log in with SSH keys

**On your computer** (not on the server), create a key if you don't already have one:

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
```

Copy the public key to the new user:

```bash
ssh-copy-id deploy@your_vps_ip
```

> If you already log in to `root` with a key (password login disabled), copy root's keys to the new user instead, from the root session:
> `rsync --archive --chown=deploy:deploy ~/.ssh /home/deploy`
>
> On Windows without `ssh-copy-id`, paste the content of `~/.ssh/id_ed25519.pub` into `/home/deploy/.ssh/authorized_keys` on the server (permissions: `700` for `.ssh`, `600` for the file, owned by `deploy`).

Test it in a **new terminal** — you should get in without the account password:

```bash
ssh deploy@your_vps_ip
```

From now on, work as `deploy` and use `sudo`.

## 4. Harden SSH

Only do this once key login works, and **keep your current session open** until you've tested a new one.

Look at the existing drop-in files (cloud images often ship one that re-enables passwords):

```bash
ls /etc/ssh/sshd_config.d/
```

For each option, OpenSSH uses the **first** value it reads and files are read in alphabetical order, so name your file `00-…` to make sure it wins:

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

Check the syntax and the effective values, then restart SSH:

```bash
sudo sshd -t
sudo sshd -T | grep -Ei 'permitrootlogin|passwordauthentication'
sudo systemctl restart ssh
```

Open a **new** terminal and confirm `ssh deploy@your_vps_ip` still works before closing the old one.

> If you lock yourself out, use your provider's browser console (Hostinger: hPanel → VPS → *Browser terminal*) to fix the file.

## 5. Firewall (UFW)

Allow SSH **before** enabling the firewall, otherwise you'll cut your own connection:

```bash
sudo ufw allow OpenSSH
sudo ufw enable
sudo ufw status verbose
```

Later, open only what the outside world needs — typically just web traffic:

```bash
sudo ufw allow 'Nginx Full'   # ports 80 and 443, after installing Nginx
```

Do **not** open your Node.js port (e.g. 4000) or database ports (27017, 5432, 3306, 6379). Nginx and your app talk to them locally. If you really need remote database access, allow only your own IP:

```bash
sudo ufw allow from 203.0.113.10 to any port 5432 proto tcp
```

> Docker publishes container ports by editing iptables directly and **bypasses UFW**. See the [Docker guide](../deploy/docker-compose-on-vps.md#firewall-warning).

Some providers (Hostinger included) also offer a firewall in their dashboard. You can use both; just make sure SSH, 80 and 443 are allowed in each.

## 6. Fail2ban

Bans IPs that keep failing SSH logins.

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

## 7. Automatic security updates

Usually installed by default on Ubuntu; make sure it's on:

```bash
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades   # answer "Yes"
```

Check from time to time whether a reboot is pending:

```bash
[ -f /var/run/reboot-required ] && echo "Reboot needed"
```

## 8. Swap (small VPS)

`npm install` and `npm run build` can run out of memory on 1–4 GB plans. A swap file prevents the build from being killed:

```bash
free -h                                  # check if swap already exists
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

Use swap only as a last resort:

```bash
echo 'vm.swappiness=10' | sudo tee /etc/sysctl.d/99-swappiness.conf
sudo sysctl --system
```

---

## Checklist

- [ ] System updated
- [ ] `deploy` user with `sudo`
- [ ] Key-based SSH login works
- [ ] Root login and password login disabled (`sshd -T` confirms)
- [ ] UFW enabled with only `OpenSSH` (and later `Nginx Full`)
- [ ] Fail2ban running
- [ ] Unattended upgrades enabled
- [ ] Swap configured (small plans)

**Next:** install a [database](../README.md#3-databases), then [deploy your MERN app](../deploy/mern-on-vps.md).
