# TP3 — Linux Ubuntu · Niveau Avancé

> **Titre :** Durcissement, monitoring custom et sauvegardes
> **Durée :** 8 à 12 heures (étalable sur 2 séances)
> **Prérequis :** TP1 et TP2 validés
> **Niveau :** ⭐⭐⭐ Avancé

---

## 🎯 Objectifs

- Appliquer un **durcissement** (hardening) cohérent : sysctl, sudo, audit.
- Mettre en place un **monitoring custom** avec scripts Bash + métriques exposées.
- Industrialiser les **sauvegardes** (`rsync`, `tar`, rotation, vérification).
- Configurer un **alerting** par email ou webhook.
- Comprendre **AppArmor** et les **capabilities**.
- Diagnostiquer un incident à partir des **logs et métriques**.
- Automatiser tout cela avec des **scripts Bash idempotents** et `systemd timers` (alternative moderne à cron).

---

## 📋 Contexte

Vous êtes responsable d'une **flotte de serveurs Ubuntu**. Vous devez livrer un **modèle d'image** durci, prêt à être cloné, avec :

- Un **profil de sécurité** appliqué automatiquement.
- Un **agent de monitoring** local qui expose des métriques HTTP.
- Un **système de sauvegarde** quotidien vers un autre serveur.
- Un **système d'alerting** en cas de problème.

---

## 🏗️ Architecture cible

```
              ┌──────────────────────┐
              │   Poste admin (vous) │
              └──────────┬───────────┘
                         │ SSH 2222
                         ▼
   ┌───────────────────────────────────────────┐
   │           VM cible (à durcir)              │
   │                                            │
   │  • SSH durci, fail2ban, UFW                │
   │  • Audit sysctl + auditd                   │
   │  • AppArmor en mode enforce                │
   │  • metrics-agent (port 9100)               │
   │  • backup.sh (timer systemd quotidien)     │
   │  • alert.sh (webhook si erreur)            │
   └───────────────┬────────────────────────────┘
                   │ rsync over SSH
                   ▼
   ┌───────────────────────────────────────────┐
   │         VM de sauvegarde (NAS)             │
   │  /var/backups/<hostname>/AAAA-MM-JJ/       │
   └────────────────────────────────────────────┘
```

Vous aurez donc **deux VMs** : la cible à durcir et la VM de sauvegarde.

---

## 🪜 Étapes

### 🟣 Étape 1 — Inventaire avant durcissement (45 min)

**1.1.** Faire un état des lieux et le sauvegarder :

```bash
sudo apt install lynis -y
sudo lynis audit system --quiet > ~/avant-hardening.txt
```

Lire le rapport. Noter les **warnings** et **suggestions**.

**1.2.** Snapshot de la VM (vital).

**Question Q1.** Quels sont les 3 risques les plus importants identifiés par Lynis ?

---

### 🟣 Étape 2 — Durcissement sysctl (1h)

Créer `/etc/sysctl.d/99-hardening.conf` :

```
# Désactiver le forwarding IP (sauf si routeur)
net.ipv4.ip_forward = 0

# Anti-spoofing
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1

# Ignorer les broadcast ICMP
net.ipv4.icmp_echo_ignore_broadcasts = 1

# Anti-SYN flood
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 2048

# Empêcher le source routing
net.ipv4.conf.all.accept_source_route = 0

# Refuser les redirects ICMP
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0

# Logger les paquets martiens
net.ipv4.conf.all.log_martians = 1

# ASLR full
kernel.randomize_va_space = 2

# Restreindre dmesg
kernel.dmesg_restrict = 1
```

Appliquer :

```bash
sudo sysctl --system
```

Vérifier :

```bash
sysctl net.ipv4.tcp_syncookies
```

---

### 🟣 Étape 3 — auditd (45 min)

```bash
sudo apt install auditd audispd-plugins -y
sudo systemctl enable --now auditd
```

Ajouter des règles dans `/etc/audit/rules.d/hardening.rules` :

```
# Surveiller les modifications de fichiers sensibles
-w /etc/passwd -p wa -k passwd_changes
-w /etc/shadow -p wa -k shadow_changes
-w /etc/sudoers -p wa -k sudoers_changes
-w /etc/ssh/sshd_config -p wa -k ssh_changes

# Surveiller les exécutions sudo
-a always,exit -F arch=b64 -S execve -F euid=0 -k root_commands
```

Recharger :

```bash
sudo augenrules --load
sudo systemctl restart auditd
```

Tester :

```bash
sudo touch /etc/passwd
sudo ausearch -k passwd_changes -ts recent
```

---

### 🟣 Étape 4 — AppArmor (45 min)

```bash
sudo aa-status
```

S'assurer que tous les profils Nginx sont en `enforce` :

```bash
sudo apt install apparmor-utils -y
sudo aa-enforce /etc/apparmor.d/usr.sbin.nginx
sudo systemctl reload apparmor
```

**Question Q4.** Quelle est la différence entre `complain`, `enforce` et `disable` ?

---

### 🟣 Étape 5 — Sudo : commandes restreintes (45 min)

Créer un fichier `/etc/sudoers.d/webops-restricted` avec `sudo visudo -f /etc/sudoers.d/webops-restricted` :

```
# webops peut redémarrer nginx mais rien d'autre en root
webops ALL=(root) NOPASSWD: /bin/systemctl restart nginx
webops ALL=(root) NOPASSWD: /bin/systemctl reload nginx
webops ALL=(root) NOPASSWD: /usr/bin/nginx -t
```

Et retirer `webops` du groupe `sudo` "tout puissant" :

```bash
sudo deluser webops sudo
```

Tester :

```bash
sudo systemctl restart nginx     # doit fonctionner
sudo apt update                  # doit être refusé
```

---

### 🟣 Étape 6 — Agent de métriques (2h)

Écrire un service qui expose des métriques au format Prometheus sur `http://localhost:9100/metrics`.

`/usr/local/bin/metrics-agent.sh` :

```bash
#!/usr/bin/env bash
set -euo pipefail

while true; do
    {
        # Lecture des métriques
        load=$(awk '{print $1}' /proc/loadavg)
        mem_total=$(awk '/MemTotal/ {print $2}' /proc/meminfo)
        mem_avail=$(awk '/MemAvailable/ {print $2}' /proc/meminfo)
        disk_used=$(df --output=pcent / | tail -1 | tr -d ' %')
        nginx_up=$(systemctl is-active nginx >/dev/null 2>&1 && echo 1 || echo 0)

        # Sortie Prometheus
        cat <<EOF
# HELP node_load1 Charge moyenne sur 1 minute
# TYPE node_load1 gauge
node_load1 $load
# HELP node_memory_total_kb Mémoire totale en kB
# TYPE node_memory_total_kb gauge
node_memory_total_kb $mem_total
# HELP node_memory_available_kb Mémoire disponible
# TYPE node_memory_available_kb gauge
node_memory_available_kb $mem_avail
# HELP node_disk_root_used_percent Pourcentage d'utilisation /
# TYPE node_disk_root_used_percent gauge
node_disk_root_used_percent $disk_used
# HELP nginx_up Nginx actif (1/0)
# TYPE nginx_up gauge
nginx_up $nginx_up
EOF
    } > /var/lib/metrics-agent/metrics.txt.tmp
    mv /var/lib/metrics-agent/metrics.txt.tmp /var/lib/metrics-agent/metrics.txt
    sleep 15
done
```

Servir le fichier via un petit serveur HTTP (avec `socat`, `python -m http.server`, ou Nginx).

Solution simple avec Nginx :

```nginx
# /etc/nginx/sites-available/metrics
server {
    listen 127.0.0.1:9100;
    location /metrics {
        alias /var/lib/metrics-agent/metrics.txt;
        add_header Content-Type text/plain;
        allow 127.0.0.1;
        deny all;
    }
}
```

Créer le service systemd `/etc/systemd/system/metrics-agent.service` :

```ini
[Unit]
Description=Agent de métriques DevOps
After=network.target

[Service]
Type=simple
User=nobody
ExecStartPre=/bin/mkdir -p /var/lib/metrics-agent
ExecStart=/usr/local/bin/metrics-agent.sh
Restart=always
RestartSec=5

# Sécurité
ProtectSystem=strict
ReadWritePaths=/var/lib/metrics-agent
NoNewPrivileges=true
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now metrics-agent
curl http://127.0.0.1:9100/metrics
```

---

### 🟣 Étape 7 — Sauvegarde (1h30)

**7.1.** Préparer la VM de sauvegarde : créer un utilisateur `backup` avec un home dédié et autoriser SSH par clé.

**7.2.** Sur la VM cible, `/usr/local/bin/backup.sh` :

```bash
#!/usr/bin/env bash
set -euo pipefail

REMOTE_USER="backup"
REMOTE_HOST="<IP-VM-NAS>"
REMOTE_DIR="/var/backups/$(hostname)"
LOCAL_DIRS=(/etc /var/www /var/log)
DATE=$(date '+%Y-%m-%d')
TMP=$(mktemp -d)

trap 'rm -rf "$TMP"' EXIT

# Snapshot tar
ARCHIVE="$TMP/backup-${DATE}.tar.gz"
tar --warning=no-file-changed -czf "$ARCHIVE" "${LOCAL_DIRS[@]}" 2>/dev/null

# Calcul SHA-256
sha256sum "$ARCHIVE" > "${ARCHIVE}.sha256"

# Envoi
rsync -avz --partial \
    -e "ssh -p 2222 -i /root/.ssh/backup_key -o StrictHostKeyChecking=accept-new" \
    "$ARCHIVE" "${ARCHIVE}.sha256" \
    "${REMOTE_USER}@${REMOTE_HOST}:${REMOTE_DIR}/"

# Rotation locale : garder 7 jours
find /var/backups/local -name 'backup-*.tar.gz' -mtime +7 -delete 2>/dev/null || true

logger -t backup "Backup ${DATE} envoyé vers ${REMOTE_HOST}"
```

**7.3.** Créer un **systemd timer** quotidien (à 2h00) :

`/etc/systemd/system/backup.service` :

```ini
[Unit]
Description=Sauvegarde quotidienne

[Service]
Type=oneshot
ExecStart=/usr/local/bin/backup.sh
```

`/etc/systemd/system/backup.timer` :

```ini
[Unit]
Description=Timer backup quotidien

[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now backup.timer
systemctl list-timers | grep backup
```

**7.4.** Tester un **restore** : décompresser une archive dans `/tmp/restore` et vérifier que tout est là.

---

### 🟣 Étape 8 — Alerting (1h)

`/usr/local/bin/alert.sh` :

```bash
#!/usr/bin/env bash
set -euo pipefail

WEBHOOK="${ALERT_WEBHOOK:-}"        # à définir dans /etc/default/alert
SEVERITY="${1:-info}"
MESSAGE="${2:-(no message)}"
HOST=$(hostname)

if [[ -z "$WEBHOOK" ]]; then
    logger -t alert "[$SEVERITY] $MESSAGE (no webhook)"
    exit 0
fi

curl -sS -X POST -H 'Content-Type: application/json' \
    --max-time 10 \
    -d "{\"host\":\"$HOST\",\"severity\":\"$SEVERITY\",\"message\":\"$MESSAGE\"}" \
    "$WEBHOOK" || logger -t alert "Échec envoi alerte"
```

L'intégrer dans `audit.sh` (TP2) :

```bash
# Si disque > 85%, alerter
USAGE=$(df --output=pcent / | tail -1 | tr -d ' %')
if (( USAGE > 85 )); then
    /usr/local/bin/alert.sh critical "Disque / à ${USAGE}%"
fi
```

Tester avec un webhook de test (https://webhook.site).

---

### 🟣 Étape 9 — Re-audit Lynis (30 min)

```bash
sudo lynis audit system --quiet > ~/apres-hardening.txt
diff <(grep "Hardening index" ~/avant-hardening.txt) <(grep "Hardening index" ~/apres-hardening.txt)
```

L'objectif est de **gagner au moins 20 points** sur l'index Lynis.

---

### 🟣 Étape 10 — Document de livraison (45 min)

Produire `LIVRAISON.md` contenant :

1. Architecture (schéma).
2. Procédure de provisionnement (étapes pour reproduire).
3. Comptes et accès (sans secrets en clair).
4. Liste des services critiques et leurs ports.
5. Procédure de restauration depuis une sauvegarde.
6. Procédure de tests post-déploiement (smoke tests).
7. Avant/après Lynis.

---

## ✅ Critères de réussite

| Critère | Points |
|---|---:|
| sysctl hardening appliqué et persistant | 3 |
| auditd configuré, règles actives, événements visibles | 3 |
| AppArmor en enforce sur Nginx | 2 |
| sudo restreint pour `webops` | 3 |
| `metrics-agent` exposé et données cohérentes | 4 |
| systemd unit avec restrictions (`ProtectSystem`, `NoNewPrivileges`...) | 3 |
| Sauvegarde fonctionnelle avec checksum | 4 |
| Rotation locale et systemd timer actif | 2 |
| Restauration testée | 3 |
| Alerting webhook opérationnel | 3 |
| Gain Lynis ≥ 20 points | 3 |
| Document `LIVRAISON.md` complet | 3 |
| Réponses aux questions Q1 et Q4 | 2 |
| **Bonus** : Ansible playbook qui automatise tout le TP | +5 |
| **Bonus** : conteneurisation du metrics-agent | +3 |
| **Bonus** : grafana branché sur les métriques | +3 |
| **Total** | **/38** |

---

## 🎓 Évaluation finale (30 min)

**Mise en situation :** l'instructeur "casse" la VM (modifie un fichier critique, ouvre un port, supprime une règle UFW). L'étudiant dispose de 20 minutes pour :

1. **Détecter** l'anomalie (logs, audit, monitoring).
2. **Diagnostiquer** la cause.
3. **Corriger** sans abimer le reste.
4. **Expliquer** ce qu'il aurait dû faire pour éviter cela.

**Questions transverses :**

- Quelle est la différence entre `cron` et `systemd timer` ? Pourquoi avoir choisi le second ?
- Comment géreriez-vous le secret webhook si vous deviez déployer cette image sur 100 serveurs ?
- Quels indicateurs surveilleriez-vous en priorité pour ce serveur ? Pourquoi ?
- Vous découvrez un fichier `/tmp/.x` exécutable, propriétaire `nobody`, qui ouvre une connexion vers une IP inconnue. Que faites-vous, dans quel ordre ?

---

## 📚 Prochaine étape dans la roadmap

➡️ **Terminal Knowledge** approfondi (scripting Bash avancé, traps, IPC).
➡️ **Configuration Management** : tout ce TP peut être encodé dans un playbook **Ansible**. Vous aurez alors une **image durcie reproductible** en 1 commande.
