# TP2 — Linux Ubuntu · Niveau Intermédiaire

> **Titre :** Mise en service d'un serveur web sécurisé
> **Durée :** 4 à 6 heures
> **Prérequis :** TP1 validé. Ubuntu Server 22.04+ (VM recommandée, snapshot avant le TP).
> **Niveau :** ⭐⭐ Intermédiaire

---

## 🎯 Objectifs

- Installer et configurer **Nginx** comme serveur web.
- Gérer des services avec **systemd**.
- Configurer un **firewall** avec **UFW**.
- Sécuriser **SSH** : clés, désactivation root, port custom.
- Créer un **utilisateur de service** non-privilégié.
- Mettre en place une **tâche cron** d'audit.
- Lire et exploiter les **logs** (`journalctl`, `/var/log`).
- Écrire un script Bash robuste (`set -euo pipefail`).

---

## 📋 Contexte

Votre client souhaite héberger un site statique sur une VM Ubuntu. Vous êtes responsable de :

1. Installer le serveur web Nginx.
2. Sécuriser l'accès SSH et le réseau.
3. Mettre en place une page d'accueil personnalisée.
4. Automatiser un audit quotidien.
5. Documenter votre installation dans un `INSTALLATION.md`.

---

## 🛠️ Mise en place

1. Démarrer une VM Ubuntu Server 22.04 ou 24.04 (VirtualBox / Multipass / cloud).
2. **Faire un snapshot avant de commencer.**
3. Se connecter en SSH depuis votre poste.

---

## 🪜 Étapes

### 🔵 Étape 1 — Mise à jour & utilisateur de service (30 min)

**1.1.** Mettre à jour :

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install ufw fail2ban htop curl net-tools -y
```

**1.2.** Créer un utilisateur `webops` non-privilégié destiné à gérer le site :

```bash
sudo adduser webops
sudo usermod -aG sudo webops
```

**1.3.** Mettre en place l'auth par clé pour `webops`.

Sur votre **poste local** :

```bash
ssh-keygen -t ed25519 -f ~/.ssh/webops_key -C "webops@formation"
ssh-copy-id -i ~/.ssh/webops_key.pub webops@<IP-de-la-VM>
```

Tester :

```bash
ssh -i ~/.ssh/webops_key webops@<IP-de-la-VM>
```

---

### 🔵 Étape 2 — Sécurisation SSH (45 min)

**2.1.** Éditer `/etc/ssh/sshd_config` :

```bash
sudo nano /etc/ssh/sshd_config
```

Modifier les directives suivantes :

```
Port 2222
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
PermitEmptyPasswords no
MaxAuthTries 3
```

**2.2.** Tester la configuration **avant** de recharger :

```bash
sudo sshd -t
```

**2.3.** Recharger SSH :

```bash
sudo systemctl reload ssh
```

⚠️ **Ne fermez pas la session courante** ! Ouvrez un nouveau terminal et testez :

```bash
ssh -p 2222 -i ~/.ssh/webops_key webops@<IP>
```

Si OK, vous pouvez fermer l'ancienne session.

**Question Q2.** Pourquoi changer le port SSH par défaut ne suffit pas à sécuriser le serveur ?

---

### 🔵 Étape 3 — Firewall UFW (30 min)

**3.1.** Configurer UFW :

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 2222/tcp comment 'SSH custom port'
sudo ufw allow 80/tcp comment 'HTTP'
sudo ufw allow 443/tcp comment 'HTTPS'
sudo ufw enable
sudo ufw status verbose
```

**3.2.** Vérifier depuis l'extérieur que le port 22 est **fermé** mais que le 2222 est **ouvert** :

```bash
# Depuis votre poste local
nmap -p 22,2222,80 <IP-de-la-VM>
```

---

### 🔵 Étape 4 — Installation Nginx (30 min)

**4.1.** Installer Nginx :

```bash
sudo apt install nginx -y
sudo systemctl status nginx
sudo systemctl enable nginx
```

**4.2.** Tester depuis votre poste :

```bash
curl http://<IP-de-la-VM>
```

Vous devez voir la page "Welcome to nginx".

**4.3.** Inspecter la configuration :

```bash
ls /etc/nginx/
cat /etc/nginx/sites-available/default
```

---

### 🔵 Étape 5 — Page d'accueil personnalisée (30 min)

**5.1.** Créer une page d'accueil unique :

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

**5.2.** Vérifier :

```bash
curl http://<IP-de-la-VM> | head
```

---

### 🔵 Étape 6 — Configurer un virtual host (45 min)

**6.1.** Créer `/etc/nginx/sites-available/devops-lab` :

```nginx
server {
    listen 80;
    server_name devops-lab.local;

    root /var/www/devops-lab;
    index index.html;

    access_log /var/log/nginx/devops-lab-access.log;
    error_log  /var/log/nginx/devops-lab-error.log;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

**6.2.** Préparer le répertoire :

```bash
sudo mkdir -p /var/www/devops-lab
sudo chown -R webops:webops /var/www/devops-lab
echo "<h1>DevOps Lab v1.0</h1>" | sudo -u webops tee /var/www/devops-lab/index.html
```

**6.3.** Activer le site et tester la config :

```bash
sudo ln -s /etc/nginx/sites-available/devops-lab /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

**6.4.** Tester en ajoutant une ligne dans le `/etc/hosts` de votre poste local :

```
<IP-de-la-VM>   devops-lab.local
```

Puis : `curl http://devops-lab.local`

---

### 🔵 Étape 7 — Script d'audit + cron (1h)

**7.1.** Créer `/usr/local/bin/audit.sh` :

```bash
sudo nano /usr/local/bin/audit.sh
```

```bash
#!/usr/bin/env bash
set -euo pipefail

LOG="/var/log/audit-devops.log"
DATE=$(date '+%Y-%m-%d %H:%M:%S')

{
    echo "===== Audit du $DATE ====="
    echo "-- Uptime"
    uptime
    echo "-- Mémoire"
    free -h
    echo "-- Disque"
    df -h /
    echo "-- État Nginx"
    systemctl is-active nginx
    echo "-- Dernières connexions SSH"
    last -n 5
    echo "-- Tentatives échouées (5 dernières)"
    sudo grep "Failed password" /var/log/auth.log 2>/dev/null | tail -n 5 || echo "RAS"
    echo ""
} >> "$LOG" 2>&1
```

**7.2.** Permissions et droits :

```bash
sudo chmod 750 /usr/local/bin/audit.sh
sudo touch /var/log/audit-devops.log
sudo chown root:adm /var/log/audit-devops.log
sudo chmod 640 /var/log/audit-devops.log
```

**7.3.** Tester :

```bash
sudo /usr/local/bin/audit.sh
sudo tail /var/log/audit-devops.log
```

**7.4.** Planifier en cron (toutes les 10 min) :

```bash
sudo crontab -e
```

Ajouter :

```
*/10 * * * * /usr/local/bin/audit.sh
```

**7.5.** Attendre 10 min et vérifier que le log se remplit.

---

### 🔵 Étape 8 — Rotation des logs (30 min)

Le log d'audit va grossir → configurer `logrotate`.

Créer `/etc/logrotate.d/audit-devops` :

```
/var/log/audit-devops.log {
    daily
    rotate 7
    compress
    missingok
    notifempty
    create 640 root adm
}
```

Tester :

```bash
sudo logrotate -d /etc/logrotate.d/audit-devops
sudo logrotate -f /etc/logrotate.d/audit-devops
ls -lh /var/log/audit-devops*
```

---

### 🔵 Étape 9 — Fail2Ban (45 min)

**9.1.** Vérifier que fail2ban est actif :

```bash
sudo systemctl status fail2ban
```

**9.2.** Créer `/etc/fail2ban/jail.local` :

```ini
[DEFAULT]
bantime = 1h
findtime = 10m
maxretry = 3

[sshd]
enabled = true
port = 2222
```

```bash
sudo systemctl restart fail2ban
sudo fail2ban-client status sshd
```

**9.3.** Simuler 3 connexions échouées depuis votre poste et vérifier le bannissement :

```bash
sudo fail2ban-client status sshd
```

---

### 🔵 Étape 10 — Documentation (30 min)

Créer `INSTALLATION.md` dans `~/INSTALLATION.md` :

- Étapes réalisées
- Choix de configuration (port SSH, ports UFW)
- Comment se connecter (commande SSH complète)
- Comment vérifier la santé du serveur
- Comment déployer une nouvelle version du site

---

## ✅ Critères de réussite

| Critère | Points |
|---|---:|
| Utilisateur `webops` + sudo + clé SSH | 2 |
| SSH sur port 2222, root désactivé, password désactivé | 3 |
| `sudo sshd -t` passe sans erreur | 1 |
| UFW configuré et actif | 2 |
| Nginx actif, page personnalisée servie | 2 |
| Virtual host `devops-lab` fonctionnel | 2 |
| Script `audit.sh` robuste (`set -euo pipefail`) | 2 |
| Cron actif, log alimenté | 2 |
| logrotate configuré | 2 |
| fail2ban actif sur port 2222 | 2 |
| `INSTALLATION.md` complet et clair | 2 |
| **Bonus** : HTTPS avec Let's Encrypt (certbot) | +4 |
| **Bonus** : page d'erreur 404 personnalisée | +1 |
| **Bonus** : limiter le débit (rate-limit Nginx) | +2 |
| **Total** | **/22** |

---

## 🐛 Pièges classiques

- Modifier `sshd_config` **sans** tester `sudo sshd -t` → on se ferme dehors. Toujours garder une session ouverte.
- Activer UFW **sans** autoriser SSH d'abord → idem.
- `chmod 777` "pour que ça marche" → JAMAIS. Comprendre ce qu'on autorise.
- Mettre un script en cron sans rediriger `stderr` → on perd les erreurs.
- Logs énormes parce que pas de logrotate → disque plein.

---

## 🎓 Évaluation orale (10 min)

1. Pourquoi avez-vous mis SSH sur le port 2222 ? Est-ce vraiment plus sûr ?
2. Que se passerait-il si quelqu'un faisait `chmod 777` sur `/var/www/devops-lab` ?
3. Comment ajouter un deuxième site (`api.local`) sans interrompre celui-ci ?
4. Comment savoir si fail2ban a banni une IP ?
5. Quelle commande pour voir les 50 dernières lignes du journal nginx ?

---

## 📚 Pour la prochaine fois

➡️ **TP3 (Avancé)** : durcissement complet (CIS), monitoring custom, sauvegardes automatisées, alerting.
