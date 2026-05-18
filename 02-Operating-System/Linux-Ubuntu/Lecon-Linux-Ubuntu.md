# Leçon 2 — Linux Ubuntu pour le DevOps

> Module : Operating System → Linux → Ubuntu / Debian
> Public : étudiants débutants en DevOps
> Durée estimée : **12 à 16 heures** (théorie + TP intensifs)
> Prérequis : Leçon 1 (Python) recommandée mais pas obligatoire

---

## 1. Objectifs pédagogiques

À la fin de cette leçon, l'étudiant sera capable de :

1. Comprendre pourquoi **Linux** domine en production et en DevOps.
2. Installer Ubuntu (VM, WSL2, serveur).
3. Naviguer dans le **système de fichiers** (FHS) et comprendre sa logique.
4. Gérer **utilisateurs, groupes et permissions** (rwx, sudo, chmod, chown).
5. Maîtriser les **commandes essentielles** : fichiers, processus, réseau, texte.
6. Installer / mettre à jour des logiciels avec **APT**.
7. Gérer les **services** avec `systemd` (`systemctl`, `journalctl`).
8. Diagnostiquer **CPU, mémoire, disque, réseau**.
9. Configurer un accès **SSH** sécurisé (clés, pas de root login).
10. Écrire des **scripts Bash** d'automatisation simples.
11. Comprendre les bases de la **sécurité** (firewall UFW, sudoers, mises à jour).

---

## 2. Pourquoi Linux Ubuntu en DevOps ?

- **96 %** des serveurs publics du top 1M tournent sous Linux.
- Tout l'écosystème DevOps moderne est conçu Linux-first : Docker, Kubernetes, Terraform, Ansible…
- **Ubuntu** = distribution la plus utilisée (large communauté, LTS 5 ans, image officielle dans tous les clouds).
- Variantes : **Ubuntu Server** (production), **Ubuntu Desktop** (poste de travail), **WSL2** (Linux sous Windows).

---

## 3. Installation

| Contexte | Solution recommandée |
|---|---|
| Étudiants Windows | **WSL2** + Ubuntu 22.04/24.04 |
| Étudiants Mac | **Multipass** ou Docker Linux |
| Salle dédiée | Dual boot ou VM (VirtualBox / VMware) |
| Cloud (avancé) | AWS EC2 / Hetzner / Contabo |

### Installer WSL2 (Windows)

```powershell
wsl --install -d Ubuntu-24.04
```

### Premier réflexe après installation

```bash
sudo apt update && sudo apt upgrade -y
```

---

## 4. Architecture Linux (vue d'ensemble)

```
+--------------------------------+
|   Utilisateurs / Applications  |  ← bash, python, nginx, docker...
+--------------------------------+
|   Shell (bash, zsh)            |
+--------------------------------+
|   Bibliothèques (glibc, etc.)  |
+--------------------------------+
|   Noyau (kernel Linux)         |  ← gère CPU, mémoire, périphériques
+--------------------------------+
|   Matériel                     |
+--------------------------------+
```

---

## 5. Système de fichiers (FHS)

| Chemin | Rôle |
|---|---|
| `/` | Racine |
| `/home/<user>` | Données utilisateur |
| `/etc` | Fichiers de configuration |
| `/var/log` | Logs |
| `/var/lib` | Données applicatives (Docker, MySQL…) |
| `/usr/bin`, `/usr/local/bin` | Binaires |
| `/opt` | Logiciels tiers |
| `/tmp` | Fichiers temporaires (vidé au reboot) |
| `/proc`, `/sys` | Pseudo-FS du noyau (infos système) |
| `/root` | Home de l'utilisateur root |

**À retenir :** un DevOps passe 80 % de son temps dans `/etc`, `/var/log`, et `/home`.

---

## 6. Commandes essentielles

### 6.1 Navigation et fichiers

```bash
pwd                  # où suis-je ?
ls -lah              # lister (long, all, human)
cd /etc/nginx        # changer de dossier
tree -L 2            # arborescence (à installer)
cp src.txt dst.txt   # copier
mv ancien nouveau    # déplacer / renommer
rm -rf dossier/      # supprimer (DANGER)
mkdir -p a/b/c       # créer dossiers récursivement
touch fichier        # créer un fichier vide
```

### 6.2 Lire et chercher dans des fichiers

```bash
cat /etc/os-release       # afficher
less /var/log/syslog      # paginer (q pour quitter)
head -n 20 fichier.log    # 20 premières lignes
tail -f /var/log/nginx/access.log   # suivi temps réel
grep "ERROR" app.log      # chercher
grep -r "TODO" .          # récursif
find / -name "*.conf" 2>/dev/null
```

### 6.3 Manipulation de texte (essentiel DevOps)

```bash
awk '{print $1}' access.log         # 1ère colonne
sed -i 's/8080/9090/g' config.yaml  # remplacer en place
cut -d':' -f1 /etc/passwd           # 1er champ
sort | uniq -c | sort -rn           # top occurrences
wc -l fichier                        # nombre de lignes
```

### 6.4 Processus

```bash
ps aux                    # tous les processus
top                       # temps réel (q pour quitter)
htop                      # version améliorée (à installer)
kill -9 <PID>             # tuer
pkill nginx               # tuer par nom
jobs ; fg ; bg            # gestion des jobs
nohup ./script.sh &       # lancer en arrière-plan
```

### 6.5 Réseau

```bash
ip a                      # interfaces réseau
ss -tulnp                 # ports en écoute
ping 8.8.8.8
curl -I https://google.com
wget https://...
dig example.com
traceroute google.com
```

---

## 7. Utilisateurs, groupes, permissions

### 7.1 Comptes

```bash
sudo adduser alice
sudo usermod -aG sudo alice    # ajouter au groupe sudo
sudo usermod -aG docker alice  # ajouter au groupe docker
sudo deluser alice
id alice
groups alice
```

### 7.2 Permissions Unix (rwx)

```
-rwxr-xr--  1 alice devs  1024 May 18 10:00 deploy.sh
 ↑↑↑↑↑↑↑↑↑
 │└─┬─┘└─┬─┘└─┬─┘
 │  │     │     └── autres (read)
 │  │     └──────── groupe (read + execute)
 │  └────────────── propriétaire (read + write + execute)
 └───────────────── type (- fichier, d dossier, l lien)
```

```bash
chmod 750 deploy.sh         # rwxr-x---
chmod u+x script.sh         # ajouter exécution au propriétaire
chown alice:devs fichier    # changer propriétaire/groupe
```

### 7.3 sudo

```bash
sudo cat /etc/shadow
sudo -i                    # shell root
sudo visudo                # éditer /etc/sudoers en sécurité
```

---

## 8. Gestion de paquets — APT

```bash
sudo apt update                  # rafraîchir l'index
sudo apt upgrade -y              # mettre à jour
sudo apt install nginx -y        # installer
sudo apt remove nginx            # désinstaller
sudo apt purge nginx             # + supprimer la conf
sudo apt search docker           # chercher
apt list --installed             # lister installés
```

**À retenir :** `apt` = APT en ligne de commande moderne ; `apt-get` = ancien, utilisé en script.

---

## 9. Services avec `systemd`

```bash
systemctl status nginx
sudo systemctl start nginx
sudo systemctl stop nginx
sudo systemctl restart nginx
sudo systemctl reload nginx       # recharger sans couper
sudo systemctl enable nginx       # démarrage automatique
sudo systemctl disable nginx
```

### Lire les logs (`journalctl`)

```bash
journalctl -u nginx              # logs du service nginx
journalctl -u nginx -f           # temps réel
journalctl -u nginx --since "1 hour ago"
journalctl -p err -b             # erreurs depuis ce boot
```

### Créer un service custom

`/etc/systemd/system/monapp.service` :

```ini
[Unit]
Description=Mon application Python
After=network.target

[Service]
Type=simple
User=alice
WorkingDirectory=/opt/monapp
ExecStart=/opt/monapp/.venv/bin/python app.py
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now monapp
```

---

## 10. Surveillance système

```bash
uptime               # charge moyenne
free -h              # mémoire
df -h                # disque
du -sh /var/log/*    # taille par dossier
iostat               # I/O disque (paquet sysstat)
vmstat 2 5           # mémoire/CPU toutes les 2s
```

---

## 11. SSH — accès distant sécurisé

### Générer une clé

```bash
ssh-keygen -t ed25519 -C "alice@formation"
```

### Copier la clé sur un serveur

```bash
ssh-copy-id alice@192.168.1.10
ssh alice@192.168.1.10
```

### Bonnes pratiques (`/etc/ssh/sshd_config`)

```
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
```

```bash
sudo systemctl restart ssh
```

---

## 12. Bases du scripting Bash

```bash
#!/usr/bin/env bash
set -euo pipefail        # mode strict (recommandé)

NOM="${1:-monde}"
echo "Bonjour $NOM"

for srv in web-1 web-2 db-1; do
    if ping -c 1 -W 1 "$srv" > /dev/null; then
        echo "✅ $srv OK"
    else
        echo "❌ $srv DOWN"
    fi
done
```

**À retenir :** `set -euo pipefail` est obligatoire dans un script de prod.

---

## 13. Sécurité de base

### Pare-feu UFW

```bash
sudo ufw allow 22/tcp
sudo ufw allow 80,443/tcp
sudo ufw enable
sudo ufw status verbose
```

### Mises à jour automatiques

```bash
sudo apt install unattended-upgrades
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

### Audit rapide

```bash
last -n 10                # dernières connexions
sudo lastb -n 10          # tentatives échouées
sudo cat /var/log/auth.log | grep "Failed password"
```

---

## 14. TP final — Configurer un mini-serveur

**Objectif :** depuis une Ubuntu 24.04 fraîche, l'étudiant doit produire un serveur web minimal :

1. Mettre à jour le système (`apt update && upgrade`).
2. Créer un utilisateur `webops` avec sudo, accès SSH par clé uniquement.
3. Désactiver `PermitRootLogin` et l'auth par mot de passe.
4. Installer et démarrer **Nginx**, vérifier `systemctl status`.
5. Configurer UFW : autoriser 22, 80, 443.
6. Modifier la page d'accueil par défaut (`/var/www/html/index.html`).
7. Écrire un script `audit.sh` qui affiche : uptime, mémoire libre, disque, état Nginx.
8. Planifier le script en cron toutes les 10 minutes, sortie vers `/var/log/audit.log`.

**Critères d'évaluation :**
- Connexion root impossible : ✅ / ❌
- Connexion par mot de passe impossible : ✅ / ❌
- Nginx répond sur le port 80 : ✅ / ❌
- Script idempotent et avec `set -euo pipefail` : ✅ / ❌
- Logs cron lisibles : ✅ / ❌

---

## 15. Évaluation — questions de contrôle

1. Donnez le rôle des dossiers : `/etc`, `/var/log`, `/usr/local/bin`, `/proc`.
2. Quelle commande pour suivre en temps réel un fichier log ?
3. Différence entre `apt update` et `apt upgrade` ?
4. Quels droits représente `chmod 644` ?
5. Pourquoi désactiver `PermitRootLogin` en SSH ?
6. À quoi sert `systemctl daemon-reload` ?
7. Quelle commande liste les ports en écoute ?
8. Pourquoi `set -euo pipefail` dans un script Bash ?
9. Comment ajouter un utilisateur au groupe `docker` ?
10. Différence entre `kill -9` et `kill -15` ?

---

## 16. Ressources

- 📘 *The Linux Command Line* — William Shotts (gratuit, https://linuxcommand.org/tlcl.php)
- 📘 *How Linux Works* — Brian Ward
- 🌐 Ubuntu Server Guide : https://ubuntu.com/server/docs
- 🧪 Exercices : https://overthewire.org/wargames/bandit/ (excellent pour la ligne de commande)
- 🎓 Linux Foundation — *Introduction to Linux* (gratuit sur edX)

---

## 17. Étapes suivantes dans la roadmap

➡️ **Leçon 3** : Terminal Knowledge approfondi (Bash, scripting, monitoring, networking tools)
➡️ **Leçon 4** : Git & Version Control
➡️ **Projet transversal** : combiner Python (Leçon 1) + Linux (Leçon 2) pour automatiser un audit multi-serveurs via SSH.
