# Corrigé — TP3 Ubuntu (Avancé)

> Tous les fichiers de configuration, scripts et services.

---

## Étape 1 — Inventaire Lynis

```bash
sudo apt install lynis -y
sudo lynis audit system --quiet > ~/avant-hardening.txt
grep -E "(Warning|Suggestion|Hardening index)" ~/avant-hardening.txt | head -30
```

### Réponse Q1 (exemple typique)

Sur une Ubuntu fraîche, les 3 risques majeurs habituels :

1. `sysctl` : protections noyau non durcies (kernel.randomize_va_space, kernel.dmesg_restrict).
2. `auditd` non installé → pas de traçabilité des actions sensibles.
3. AppArmor partiellement actif → profils en `complain` au lieu d'`enforce`.

---

## Étape 2 — sysctl

### `/etc/sysctl.d/99-hardening.conf`

```
# --- Réseau IPv4 ---
net.ipv4.ip_forward = 0
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1
net.ipv4.icmp_echo_ignore_broadcasts = 1
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 2048
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0
net.ipv4.conf.all.log_martians = 1

# --- Réseau IPv6 ---
net.ipv6.conf.all.accept_redirects = 0
net.ipv6.conf.default.accept_redirects = 0

# --- Noyau ---
kernel.randomize_va_space = 2
kernel.dmesg_restrict = 1
kernel.kptr_restrict = 2
kernel.yama.ptrace_scope = 1
fs.protected_hardlinks = 1
fs.protected_symlinks = 1
fs.protected_fifos = 2
fs.protected_regular = 2
fs.suid_dumpable = 0
```

```bash
sudo sysctl --system
sysctl net.ipv4.tcp_syncookies kernel.randomize_va_space kernel.dmesg_restrict
```

---

## Étape 3 — auditd

```bash
sudo apt install auditd audispd-plugins -y
sudo systemctl enable --now auditd
```

### `/etc/audit/rules.d/hardening.rules`

```
# Buffer
-b 8192

# Modifications de comptes & sécurité
-w /etc/passwd -p wa -k passwd_changes
-w /etc/shadow -p wa -k shadow_changes
-w /etc/group -p wa -k group_changes
-w /etc/sudoers -p wa -k sudoers_changes
-w /etc/sudoers.d/ -p wa -k sudoers_changes
-w /etc/ssh/sshd_config -p wa -k ssh_changes
-w /etc/hosts -p wa -k hosts_changes

# Commandes lancées en root
-a always,exit -F arch=b64 -S execve -F euid=0 -k root_commands

# Modifications du firewall et systemd
-w /etc/ufw/ -p wa -k ufw_changes
-w /etc/systemd/ -p wa -k systemd_changes

# Rendre les règles immuables (à mettre à la fin)
-e 2
```

```bash
sudo augenrules --load
sudo systemctl restart auditd
sudo auditctl -l                          # vérifier que les règles sont chargées

# Tester
sudo touch /etc/passwd                    # déclenchera passwd_changes
sudo ausearch -k passwd_changes -ts recent
```

---

## Étape 4 — AppArmor

```bash
sudo apt install apparmor-utils -y
sudo aa-status                            # voir les profils actifs

# Forcer le mode enforce sur Nginx
sudo aa-enforce /etc/apparmor.d/usr.sbin.nginx 2>/dev/null || \
    echo "Profil non présent — installer apparmor-profiles-extra"

sudo apt install apparmor-profiles apparmor-profiles-extra -y
sudo systemctl reload apparmor
sudo aa-status | head
```

### Réponse Q4

| Mode | Effet |
|---|---|
| `complain` | Audit-only : viole-> logge dans `/var/log/audit.log` mais autorise. Utile en phase de mise au point. |
| `enforce` | Bloque réellement toute action interdite par le profil. C'est le mode production. |
| `disable` | Profil non chargé du tout. |

---

## Étape 5 — sudo restreint

### `/etc/sudoers.d/webops-restricted`

```
# webops peut piloter Nginx uniquement
Cmnd_Alias NGINX_OPS = /bin/systemctl restart nginx, \
                       /bin/systemctl reload nginx, \
                       /usr/sbin/nginx -t

webops ALL=(root) NOPASSWD: NGINX_OPS

# Refus implicite de tout le reste
```

Édition obligatoire avec `visudo` (vérifie la syntaxe) :

```bash
sudo visudo -f /etc/sudoers.d/webops-restricted
sudo chmod 0440 /etc/sudoers.d/webops-restricted
sudo deluser webops sudo
```

Test :

```bash
sudo -u webops sudo systemctl restart nginx       # OK
sudo -u webops sudo apt update                    # Sorry, user webops is not allowed...
```

---

## Étape 6 — Agent de métriques

### `/usr/local/bin/metrics-agent.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

DIR="/var/lib/metrics-agent"
OUT="${DIR}/metrics.txt"
TMP="${OUT}.tmp"

mkdir -p "$DIR"

while true; do
    load=$(awk '{print $1}' /proc/loadavg)
    mem_total=$(awk '/MemTotal/ {print $2}' /proc/meminfo)
    mem_avail=$(awk '/MemAvailable/ {print $2}' /proc/meminfo)
    disk_used=$(df --output=pcent / | tail -1 | tr -d ' %')

    if systemctl is-active --quiet nginx; then nginx_up=1; else nginx_up=0; fi
    if systemctl is-active --quiet ssh 2>/dev/null || systemctl is-active --quiet ssh.socket; then
        ssh_up=1
    else
        ssh_up=0
    fi

    {
        cat <<EOF
# HELP node_load1 Charge moyenne 1 minute
# TYPE node_load1 gauge
node_load1 ${load}
# HELP node_memory_total_kb Mémoire totale (kB)
# TYPE node_memory_total_kb gauge
node_memory_total_kb ${mem_total}
# HELP node_memory_available_kb Mémoire disponible (kB)
# TYPE node_memory_available_kb gauge
node_memory_available_kb ${mem_avail}
# HELP node_disk_root_used_percent Pourcentage utilisé /
# TYPE node_disk_root_used_percent gauge
node_disk_root_used_percent ${disk_used}
# HELP nginx_up Nginx actif
# TYPE nginx_up gauge
nginx_up ${nginx_up}
# HELP ssh_up SSH actif
# TYPE ssh_up gauge
ssh_up ${ssh_up}
EOF
    } > "$TMP"
    mv "$TMP" "$OUT"
    sleep 15
done
```

```bash
sudo install -m 0755 metrics-agent.sh /usr/local/bin/metrics-agent.sh
sudo mkdir -p /var/lib/metrics-agent
sudo chown nobody:nogroup /var/lib/metrics-agent
```

### `/etc/systemd/system/metrics-agent.service`

```ini
[Unit]
Description=Agent de métriques DevOps
After=network.target

[Service]
Type=simple
User=nobody
Group=nogroup
ExecStart=/usr/local/bin/metrics-agent.sh
Restart=always
RestartSec=5

# --- Durcissement ---
NoNewPrivileges=true
ProtectSystem=strict
ReadWritePaths=/var/lib/metrics-agent
ProtectHome=true
PrivateTmp=true
PrivateDevices=true
ProtectKernelTunables=true
ProtectKernelModules=true
ProtectControlGroups=true
RestrictNamespaces=true
RestrictSUIDSGID=true
LockPersonality=true
MemoryDenyWriteExecute=true

[Install]
WantedBy=multi-user.target
```

### `/etc/nginx/sites-available/metrics`

```nginx
server {
    listen 127.0.0.1:9100;

    location /metrics {
        alias /var/lib/metrics-agent/metrics.txt;
        default_type text/plain;
        allow 127.0.0.1;
        deny all;
    }
}
```

```bash
sudo ln -sf /etc/nginx/sites-available/metrics /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
sudo systemctl daemon-reload
sudo systemctl enable --now metrics-agent
curl http://127.0.0.1:9100/metrics
```

---

## Étape 7 — Sauvegarde

Sur la **VM NAS** :

```bash
sudo adduser backup
sudo mkdir -p /var/backups
sudo chown backup:backup /var/backups
sudo -u backup mkdir -p /home/backup/.ssh
sudo -u backup tee /home/backup/.ssh/authorized_keys < clé_publique_de_la_VM_cible
sudo chmod 700 /home/backup/.ssh
sudo chmod 600 /home/backup/.ssh/authorized_keys
```

Sur la **VM cible** :

```bash
sudo ssh-keygen -t ed25519 -f /root/.ssh/backup_key -N ""
sudo cat /root/.ssh/backup_key.pub        # à copier vers la VM NAS
```

### `/usr/local/bin/backup.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

REMOTE_USER="backup"
REMOTE_HOST="${BACKUP_HOST:?BACKUP_HOST manquant}"
REMOTE_PORT="2222"
REMOTE_DIR="/var/backups/$(hostname)"
LOCAL_DIRS=(/etc /var/www /var/log/audit-devops.log)
DATE=$(date '+%Y-%m-%d')
TMP=$(mktemp -d /var/tmp/backup.XXXXXX)

trap 'rm -rf "$TMP"' EXIT

ARCHIVE="$TMP/backup-${DATE}.tar.gz"
SUMS="${ARCHIVE}.sha256"

logger -t backup "Démarrage backup $DATE"

tar --warning=no-file-changed --warning=no-file-removed \
    -czf "$ARCHIVE" "${LOCAL_DIRS[@]}" 2>/dev/null

sha256sum "$ARCHIVE" > "$SUMS"

ssh -p "$REMOTE_PORT" -i /root/.ssh/backup_key \
    -o StrictHostKeyChecking=accept-new \
    "${REMOTE_USER}@${REMOTE_HOST}" "mkdir -p '$REMOTE_DIR'"

rsync -az --partial --timeout=60 \
    -e "ssh -p ${REMOTE_PORT} -i /root/.ssh/backup_key" \
    "$ARCHIVE" "$SUMS" \
    "${REMOTE_USER}@${REMOTE_HOST}:${REMOTE_DIR}/"

# Rotation locale (7j)
mkdir -p /var/backups/local
cp "$ARCHIVE" /var/backups/local/
find /var/backups/local -name 'backup-*.tar.gz' -mtime +7 -delete

logger -t backup "Backup $DATE OK -> ${REMOTE_HOST}:${REMOTE_DIR}"
```

```bash
sudo install -m 0750 backup.sh /usr/local/bin/backup.sh
echo "BACKUP_HOST=<IP-NAS>" | sudo tee /etc/default/backup
```

### `/etc/systemd/system/backup.service`

```ini
[Unit]
Description=Sauvegarde quotidienne
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
EnvironmentFile=/etc/default/backup
ExecStart=/usr/local/bin/backup.sh
Nice=10
IOSchedulingClass=best-effort
IOSchedulingPriority=7
```

### `/etc/systemd/system/backup.timer`

```ini
[Unit]
Description=Timer backup quotidien

[Timer]
OnCalendar=*-*-* 02:00:00
RandomizedDelaySec=10m
Persistent=true
Unit=backup.service

[Install]
WantedBy=timers.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now backup.timer
systemctl list-timers --all | grep backup
sudo systemctl start backup.service       # test immédiat
sudo journalctl -u backup.service -n 50
```

### Test de restauration

```bash
mkdir -p /tmp/restore && cd /tmp/restore
scp -P 2222 -i /root/.ssh/backup_key \
    backup@<NAS>:/var/backups/$(hostname)/backup-*.tar.gz .
sha256sum -c backup-*.tar.gz.sha256
tar -xzf backup-*.tar.gz
ls -la
```

---

## Étape 8 — Alerting

### `/etc/default/alert`

```
ALERT_WEBHOOK=https://webhook.site/xxxxx-xxxx-xxxx
```

### `/usr/local/bin/alert.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

# shellcheck disable=SC1091
[ -f /etc/default/alert ] && . /etc/default/alert

SEVERITY="${1:-info}"
MESSAGE="${2:-(no message)}"
HOST=$(hostname)
DATE=$(date -Is)

if [[ -z "${ALERT_WEBHOOK:-}" ]]; then
    logger -t alert "[$SEVERITY] $MESSAGE (no webhook configured)"
    exit 0
fi

PAYLOAD=$(jq -nc \
    --arg host "$HOST" \
    --arg severity "$SEVERITY" \
    --arg message "$MESSAGE" \
    --arg date "$DATE" \
    '{host:$host, severity:$severity, message:$message, date:$date}')

if ! curl -sS -X POST -H 'Content-Type: application/json' \
        --max-time 10 \
        -d "$PAYLOAD" "$ALERT_WEBHOOK" > /dev/null
then
    logger -t alert "Échec envoi alerte vers $ALERT_WEBHOOK"
    exit 1
fi

logger -t alert "[$SEVERITY] $MESSAGE envoyé"
```

```bash
sudo apt install jq -y
sudo install -m 0755 alert.sh /usr/local/bin/alert.sh
sudo chmod 0640 /etc/default/alert
sudo /usr/local/bin/alert.sh warning "Test alerting"
```

### Intégration dans `audit.sh` (étendu)

À ajouter à la fin du `audit.sh` du TP2 :

```bash
USAGE=$(df --output=pcent / | tail -1 | tr -d ' %')
if (( USAGE > 85 )); then
    /usr/local/bin/alert.sh critical "Disque / à ${USAGE}% sur $(hostname)"
fi

if ! systemctl is-active --quiet nginx; then
    /usr/local/bin/alert.sh critical "Nginx inactif sur $(hostname)"
fi
```

---

## Étape 9 — Re-audit

```bash
sudo lynis audit system --quiet > ~/apres-hardening.txt
grep "Hardening index" ~/avant-hardening.txt
grep "Hardening index" ~/apres-hardening.txt
diff <(grep "Suggestion" ~/avant-hardening.txt) <(grep "Suggestion" ~/apres-hardening.txt)
```

Gain typique : **+25 à +35 points** sur l'index.

---

## Étape 10 — `LIVRAISON.md` (modèle)

```markdown
# Livraison — VM Ubuntu durcie

## Architecture
[Schéma à insérer]

## Procédure de provisionnement
1. Snapshot VM de base Ubuntu 24.04 LTS
2. Exécuter `provision.sh` (ou playbook Ansible — bonus)
3. Valider via `sudo lynis audit system`

## Accès
| Compte | Rôle | Accès |
|---|---|---|
| `webops` | Exploitation Nginx | SSH par clé, sudo restreint |
| `root` | Désactivé en SSH | Console uniquement |

## Services critiques
| Service | Port | Healthcheck |
|---|---|---|
| SSH | 2222 | `nc -z 127.0.0.1 2222` |
| Nginx | 80, 443 | `curl http://127.0.0.1` |
| metrics-agent | 9100 (local) | `curl http://127.0.0.1:9100/metrics` |

## Sauvegardes
- Quotidiennes 02:00 vers `backup@<NAS>:/var/backups/<host>`
- Rétention : 7 jours local, illimité côté NAS
- Vérification SHA-256 obligatoire avant restore

## Restauration
```bash
scp -P 2222 -i ~/.ssh/backup_key backup@<NAS>:/var/backups/<host>/backup-AAAA-MM-JJ.tar.gz .
sha256sum -c backup-AAAA-MM-JJ.tar.gz.sha256
sudo tar -xzf backup-AAAA-MM-JJ.tar.gz -C /
```

## Smoke tests post-déploiement
```bash
systemctl is-active ssh nginx metrics-agent fail2ban auditd
ss -tulnp | grep -E '(:2222|:80|:443|:9100)'
sudo ufw status verbose
sudo fail2ban-client status sshd
curl -fsS http://127.0.0.1:9100/metrics | head
```

## Hardening
- Avant : index Lynis = 62
- Après : index Lynis = 89
```

---

## 🎓 Évaluation finale — réponses types

### "Le port 2222 a-t-il vraiment sécurisé le serveur ?"

Non. Il limite le bruit des scans automatisés mais ne protège pas contre un attaquant déterminé. Les vraies protections : clés ed25519, fail2ban, désactivation password + root, mises à jour régulières.

### "Gérer le secret webhook sur 100 serveurs ?"

- **À éviter :** copier `/etc/default/alert` à la main.
- **Bien :** secret manager (Vault, AWS SSM, sealed-secrets) + récupération au démarrage via systemd `LoadCredential=` ou via Ansible Vault.
- **Industriel :** rotation automatique des secrets.

### "cron vs systemd timer ?"

| | cron | systemd timer |
|---|---|---|
| Logs | redirection manuelle | natifs (journald) |
| Dépendances | aucune | `After=`, `Wants=` |
| Rattrapage | non | `Persistent=true` |
| Précision | minute | seconde |
| Randomisation | non | `RandomizedDelaySec` |

→ En 2026, sur Ubuntu : préférer systemd timer dès qu'on a accès à root.

### "Indicateurs prioritaires ?"

1. `nginx_up` (disponibilité service)
2. `node_disk_root_used_percent` (>85% = critique)
3. `node_memory_available_kb` (OOM imminent)
4. Authentifications SSH échouées sur 5 min
5. `node_load1` rapporté au nombre de cœurs

### "Fichier `/tmp/.x` exécutable louche"

1. **Ne pas le supprimer tout de suite** — preuves.
2. Isoler le réseau (UFW deny outbound vers l'IP).
3. `ps -ef | grep .x`, identifier le PID.
4. `lsof -p <PID>` pour voir les fichiers/sockets.
5. `cp /tmp/.x /root/forensics/` puis hash et soumettre à VirusTotal.
6. `auditctl` pour tracer toute exécution.
7. Identifier le **vecteur d'entrée** (logs SSH, web).
8. Plan de reconstruction depuis sauvegarde **antérieure** à la compromission.

---

## ✅ Points pédagogiques importants

1. La **règle d'or** : un script de sécurité doit être **idempotent**. Un même `provision.sh` réexécuté ne casse rien.
2. **`Persistent=true`** dans le timer : si la VM dort à 2h, le backup se lance au réveil — c'est crucial sur laptop / VM nomade.
3. **`MemoryDenyWriteExecute=true`** dans systemd : bloque l'exécution de mémoire allouée en écriture — protection anti-exploit classique.
4. **Toujours** signer les sauvegardes (SHA-256 ou GPG) avant transfert.
5. Hardening ≠ sécurité : il faut aussi **détection**, **réponse** et **récupération** (auditd + alerting + backups).
