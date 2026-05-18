# Corrigé — TP2 Ubuntu (Intermédiaire)

> Toutes les configurations + scripts attendus.

---

## Étape 1 — Mise à jour & utilisateur

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install ufw fail2ban htop curl net-tools -y

sudo adduser webops
sudo usermod -aG sudo webops
```

Sur le **poste local** :

```bash
ssh-keygen -t ed25519 -f ~/.ssh/webops_key -C "webops@formation"
ssh-copy-id -i ~/.ssh/webops_key.pub webops@<IP>
ssh -i ~/.ssh/webops_key webops@<IP>
```

---

## Étape 2 — SSH sécurisé

### `/etc/ssh/sshd_config` (extraits modifiés)

```
Port 2222
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
PermitEmptyPasswords no
MaxAuthTries 3
LoginGraceTime 30
ClientAliveInterval 300
ClientAliveCountMax 2
Protocol 2
```

```bash
sudo sshd -t                # vérifier la syntaxe
sudo systemctl reload ssh
```

Sur Ubuntu 22.04+, le service peut s'appeler `ssh.socket` :

```bash
sudo systemctl restart ssh.socket
```

### Réponse Q2 — pourquoi changer le port ne suffit pas

Changer le port SSH **réduit le bruit** des bots automatisés mais ne **protège pas** contre :

- Un attaquant qui scan tous les ports (`nmap -p-`).
- Une attaque par force brute sur le bon port.
- Une vulnérabilité dans le démon SSH lui-même.
- Une fuite de clé privée.

C'est de la **sécurité par obscurité** — utile en couche supplémentaire, jamais comme seule défense.
Les vraies protections : clés fortes (ed25519), désactivation root et mot de passe, fail2ban, MFA, restriction par IP.

---

## Étape 3 — UFW

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 2222/tcp comment 'SSH custom'
sudo ufw allow 80/tcp comment 'HTTP'
sudo ufw allow 443/tcp comment 'HTTPS'
sudo ufw enable
sudo ufw status verbose
```

Depuis le poste local :

```bash
nmap -p 22,2222,80,443 <IP>
# 22  → filtered/closed
# 2222 → open
# 80   → open
```

---

## Étape 4 — Nginx

```bash
sudo apt install nginx -y
sudo systemctl enable --now nginx
systemctl status nginx --no-pager
curl -I http://<IP>           # HTTP/1.1 200 OK
```

---

## Étape 5 — Page personnalisée

```bash
sudo tee /var/www/html/index.html > /dev/null <<'EOF'
<!doctype html>
<html lang="fr">
<head>
    <meta charset="utf-8">
    <title>Bienvenue chez DevOps Lab</title>
</head>
<body>
    <h1>🚀 Serveur configuré par <em>webops</em></h1>
    <p>Date du déploiement : <span id="now"></span></p>
    <script>document.getElementById("now").textContent = new Date().toISOString();</script>
</body>
</html>
EOF
```

---

## Étape 6 — Virtual host

### `/etc/nginx/sites-available/devops-lab`

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name devops-lab.local;

    root /var/www/devops-lab;
    index index.html;

    access_log /var/log/nginx/devops-lab-access.log;
    error_log  /var/log/nginx/devops-lab-error.log;

    location / {
        try_files $uri $uri/ =404;
    }

    # Bonus : page 404 custom
    error_page 404 /404.html;
    location = /404.html {
        internal;
    }
}
```

```bash
sudo mkdir -p /var/www/devops-lab
sudo chown -R webops:webops /var/www/devops-lab
echo "<h1>DevOps Lab v1.0</h1>" | sudo -u webops tee /var/www/devops-lab/index.html
echo "<h1>404 — page introuvable</h1>" | sudo -u webops tee /var/www/devops-lab/404.html

sudo ln -sf /etc/nginx/sites-available/devops-lab /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

Sur le poste local, dans `/etc/hosts` (ou `C:\Windows\System32\drivers\etc\hosts`) :

```
<IP-VM>   devops-lab.local
```

Test :

```bash
curl -H 'Host: devops-lab.local' http://<IP>
```

---

## Étape 7 — Script d'audit + cron

### `/usr/local/bin/audit.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

LOG="/var/log/audit-devops.log"
DATE=$(date '+%Y-%m-%d %H:%M:%S')
HOST=$(hostname)

{
    echo "===== Audit du $DATE — $HOST ====="

    echo "[uptime]"
    uptime

    echo "[memoire]"
    free -h

    echo "[disque /]"
    df -h /

    echo "[nginx]"
    if systemctl is-active --quiet nginx; then
        echo "OK"
    else
        echo "❌ NGINX INACTIF"
    fi

    echo "[ssh]"
    systemctl is-active ssh || true

    echo "[dernieres connexions]"
    last -n 5 -F | head -n 6

    echo "[tentatives echouees]"
    grep "Failed password" /var/log/auth.log 2>/dev/null | tail -n 5 || echo "RAS"

    echo "[ports ouverts]"
    ss -tulnp | tail -n +2 | awk '{print $1, $5}'

    echo ""
} >> "$LOG" 2>&1
```

```bash
sudo install -m 0750 -o root -g root audit.sh /usr/local/bin/audit.sh
sudo touch /var/log/audit-devops.log
sudo chown root:adm /var/log/audit-devops.log
sudo chmod 640 /var/log/audit-devops.log
sudo /usr/local/bin/audit.sh
sudo tail /var/log/audit-devops.log
```

### Cron

```bash
sudo crontab -e
```

```
*/10 * * * * /usr/local/bin/audit.sh
```

**Note pédagogique :** dans le TP3 on remplace ça par un `systemd timer` (plus moderne, logs intégrés à journald).

---

## Étape 8 — logrotate

### `/etc/logrotate.d/audit-devops`

```
/var/log/audit-devops.log {
    daily
    rotate 7
    compress
    delaycompress
    missingok
    notifempty
    create 640 root adm
    sharedscripts
}
```

```bash
sudo logrotate -d /etc/logrotate.d/audit-devops        # dry-run
sudo logrotate -f /etc/logrotate.d/audit-devops        # force
ls -lh /var/log/audit-devops*
```

---

## Étape 9 — fail2ban

### `/etc/fail2ban/jail.local`

```ini
[DEFAULT]
bantime  = 1h
findtime = 10m
maxretry = 3
banaction = ufw

[sshd]
enabled  = true
port     = 2222
filter   = sshd
logpath  = %(sshd_log)s
backend  = systemd
```

```bash
sudo systemctl restart fail2ban
sudo fail2ban-client status sshd
sudo fail2ban-client status                       # liste des jails
# Pour débannir une IP :
sudo fail2ban-client set sshd unbanip <IP>
```

---

## Étape 10 — `INSTALLATION.md` (modèle)

```markdown
# Serveur DevOps Lab — Installation

## 1. Accès
- IP : 192.168.1.42
- Port SSH : 2222
- Utilisateur : `webops`
- Clé privée : `~/.ssh/webops_key`

```bash
ssh -p 2222 -i ~/.ssh/webops_key webops@192.168.1.42
```

## 2. Services en place
| Service | Port | Statut |
|---|---|---|
| SSH | 2222/tcp | actif |
| Nginx | 80, 443/tcp | actif |
| fail2ban | — | actif (jail sshd) |

## 3. Sites hébergés
- `devops-lab.local` → `/var/www/devops-lab`

## 4. Vérification rapide
```bash
sudo systemctl status nginx ssh fail2ban
sudo ufw status
sudo /usr/local/bin/audit.sh && sudo tail /var/log/audit-devops.log
```

## 5. Déployer une nouvelle version
```bash
ssh -p 2222 -i ~/.ssh/webops_key webops@192.168.1.42
sudo tee /var/www/devops-lab/index.html < nouvelle-page.html
sudo nginx -t && sudo systemctl reload nginx
```

## 6. Procédure d'urgence
- Ports SSH bloqués : ouvrir la console VM directement.
- IP bannie par fail2ban : `sudo fail2ban-client set sshd unbanip <IP>`.
```

---

## 🎁 Bonus — HTTPS avec Let's Encrypt

```bash
sudo snap install --classic certbot
sudo ln -sf /snap/bin/certbot /usr/bin/certbot
sudo certbot --nginx -d devops-lab.exemple.com
```

Renouvellement automatique par défaut via systemd timer (`systemctl list-timers | grep certbot`).

---

## 🎁 Bonus — Rate limit Nginx

Dans `/etc/nginx/nginx.conf`, section `http {}` :

```
limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
```

Dans le virtual host :

```nginx
location /api/ {
    limit_req zone=api burst=20 nodelay;
    proxy_pass http://localhost:8080;
}
```

---

## ✅ Points pédagogiques importants

1. **Toujours** tester `sudo sshd -t` avant de recharger SSH.
2. **Garder une session SSH ouverte** pendant qu'on modifie sshd_config.
3. Autoriser le **port SSH dans UFW AVANT** d'activer UFW (sinon on se ferme dehors).
4. `set -euo pipefail` = obligatoire dans tout script de prod.
5. **logrotate `delaycompress`** : permet aux processus qui écrivent dans le log de finir avant la compression.
6. fail2ban + `banaction = ufw` : utilise UFW pour bannir, plus propre que iptables direct.
