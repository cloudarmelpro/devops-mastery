# TP2 — Bash / Terminal · Niveau Intermédiaire

> **Titre :** Analyseur de logs Nginx avec alerting
> **Durée :** 4 à 6 heures
> **Prérequis :** TP1 validé.
> **Niveau :** ⭐⭐ Intermédiaire

---

## 🎯 Objectifs

- Parser un fichier de logs **au format combined Nginx**.
- Maîtriser `awk` (agrégation, conditions, tableaux associatifs).
- Utiliser `getopts` pour des **options courtes** propres.
- Utiliser **`trap`** pour le nettoyage automatique.
- Produire un **rapport texte** ET un **rapport JSON** (avec `jq`).
- Déclencher des **alertes** sur des seuils.
- Écrire un **fichier de log de l'outil** (`/var/log/...`).

---

## 📋 Contexte

Vous êtes responsable d'un site qui reçoit ~50 000 requêtes par jour. Chaque matin, vous voulez un rapport :

- Top 10 IPs
- Top 10 URLs
- Distribution des codes HTTP (200, 301, 404, 500…)
- Bande passante consommée
- Heures de pointe
- **Alerte** si > 100 erreurs 5xx ou > 1000 erreurs 4xx

Votre script `nginx-stats.sh` doit s'exécuter en cron et envoyer un webhook quand quelque chose cloche.

---

## 🛠️ Mise en place

```bash
mkdir -p ~/tp2-bash && cd ~/tp2-bash
sudo apt install jq -y
touch nginx-stats.sh && chmod +x nginx-stats.sh

# Récupérer un log d'exemple
wget https://raw.githubusercontent.com/elastic/examples/master/Common%20Data%20Formats/nginx_logs/nginx_logs.gz
gunzip nginx_logs.gz
wc -l nginx_logs                # ~51 462 lignes
head -3 nginx_logs
```

Format combined Nginx :

```
93.180.71.3 - - [17/May/2015:08:05:32 +0000] "GET /downloads/product_1 HTTP/1.1" 304 0 "-" "Debian APT-HTTP/1.3 (0.8.16~exp12ubuntu10.21)"
```

| Champ awk | Contenu |
|---|---|
| `$1` | IP source |
| `$4` | `[date` |
| `$7` | URL |
| `$9` | Code HTTP |
| `$10` | Taille de la réponse (octets) |

---

## 🪜 Étapes

### 🔵 Étape 1 — Squelette et options (45 min)

```bash
#!/usr/bin/env bash
set -euo pipefail
IFS=$'\n\t'

LOG_DEFAULT="/var/log/nginx/access.log"
SEUIL_5XX=100
SEUIL_4XX=1000
FORMAT="text"
OUTPUT=""
WEBHOOK="${ALERT_WEBHOOK:-}"

usage() {
    cat <<EOF
Usage: $0 [OPTIONS]
  -f FICHIER     Fichier de log (défaut: $LOG_DEFAULT)
  -o FICHIER     Sauvegarder le rapport (sinon stdout)
  -j             Format JSON
  -w URL         Webhook d'alerte
  -h             Cette aide
EOF
}

while getopts ":f:o:jw:h" opt; do
    case $opt in
        f) LOG=$OPTARG ;;
        o) OUTPUT=$OPTARG ;;
        j) FORMAT="json" ;;
        w) WEBHOOK=$OPTARG ;;
        h) usage ; exit 0 ;;
        :) echo "Option -$OPTARG nécessite un argument" >&2 ; exit 1 ;;
        \?) echo "Option inconnue : -$OPTARG" >&2 ; exit 1 ;;
    esac
done
shift $((OPTIND-1))

LOG="${LOG:-$LOG_DEFAULT}"
[[ -r "$LOG" ]] || { echo "Log illisible : $LOG" >&2 ; exit 2 ; }
```

✅ Tester : `./nginx-stats.sh -h`, `./nginx-stats.sh -f /inexistant` (doit échouer avec code 2).

---

### 🔵 Étape 2 — Trap & fichier temporaire (15 min)

```bash
TMP=$(mktemp -d /tmp/nginxstats.XXXXXX)
trap 'rm -rf "$TMP"' EXIT
```

✅ **Vérification :** lancer le script, l'interrompre avec Ctrl+C → le dossier `$TMP` doit disparaître.

---

### 🔵 Étape 3 — Statistiques globales (45 min)

Écrire des fonctions qui calculent et écrivent dans `$TMP/*.txt` :

```bash
total_requetes() {
    wc -l < "$LOG"
}

top_ips() {
    awk '{print $1}' "$LOG" | sort | uniq -c | sort -rn | head -10
}

top_urls() {
    awk '{print $7}' "$LOG" | sort | uniq -c | sort -rn | head -10
}

distribution_codes() {
    awk '{print $9}' "$LOG" | sort | uniq -c | sort -rn
}

bande_passante() {
    awk '{sum += $10} END {printf "%.2f Mo\n", sum/1024/1024}' "$LOG"
}

heures_de_pointe() {
    # Extraire l'heure depuis [17/May/2015:08:05:32]
    awk -F'[' '{print $2}' "$LOG" \
        | awk -F: '{print $2}' \
        | sort | uniq -c | sort -rn | head -5
}

erreurs_5xx() {
    awk '$9 ~ /^5/' "$LOG" | wc -l
}

erreurs_4xx() {
    awk '$9 ~ /^4/' "$LOG" | wc -l
}
```

---

### 🔵 Étape 4 — Rapport texte (30 min)

```bash
rapport_texte() {
    cat <<EOF
==========================================
  RAPPORT NGINX — $(date '+%Y-%m-%d %H:%M')
==========================================
Fichier analysé : $LOG
Total requêtes  : $(total_requetes)
Bande passante  : $(bande_passante)
Erreurs 4xx     : $(erreurs_4xx)
Erreurs 5xx     : $(erreurs_5xx)

-- Top 10 IPs --
$(top_ips)

-- Top 10 URLs --
$(top_urls)

-- Distribution codes HTTP --
$(distribution_codes)

-- Heures de pointe --
$(heures_de_pointe)
==========================================
EOF
}
```

✅ Tester : `./nginx-stats.sh -f nginx_logs`

---

### 🔵 Étape 5 — Rapport JSON (45 min)

Construire un JSON avec `jq` pour rester propre.

```bash
rapport_json() {
    local total bp e4 e5
    total=$(total_requetes)
    bp=$(bande_passante)
    e4=$(erreurs_4xx)
    e5=$(erreurs_5xx)

    # Top IPs en JSON
    local top_ips_json
    top_ips_json=$(awk '{print $1}' "$LOG" | sort | uniq -c | sort -rn | head -10 \
        | awk '{printf "{\"count\":%d,\"ip\":\"%s\"}\n", $1, $2}' | jq -s .)

    local top_urls_json
    top_urls_json=$(awk '{print $7}' "$LOG" | sort | uniq -c | sort -rn | head -10 \
        | awk '{printf "{\"count\":%d,\"url\":\"%s\"}\n", $1, $2}' | jq -s .)

    local codes_json
    codes_json=$(awk '{print $9}' "$LOG" | sort | uniq -c | sort -rn \
        | awk '{printf "{\"count\":%d,\"code\":\"%s\"}\n", $1, $2}' | jq -s .)

    jq -n \
        --arg date "$(date -Is)" \
        --arg log "$LOG" \
        --arg bp "$bp" \
        --argjson total "$total" \
        --argjson e4 "$e4" \
        --argjson e5 "$e5" \
        --argjson top_ips "$top_ips_json" \
        --argjson top_urls "$top_urls_json" \
        --argjson codes "$codes_json" \
        '{
            date: $date,
            log_file: $log,
            total_requetes: $total,
            bande_passante: $bp,
            erreurs_4xx: $e4,
            erreurs_5xx: $e5,
            top_ips: $top_ips,
            top_urls: $top_urls,
            codes_http: $codes
        }'
}
```

✅ Tester : `./nginx-stats.sh -f nginx_logs -j | jq .total_requetes`

---

### 🔵 Étape 6 — Alerting (45 min)

```bash
envoyer_alerte() {
    local message="$1"
    if [[ -z "$WEBHOOK" ]]; then
        logger -t nginx-stats "ALERTE (no webhook): $message"
        return 0
    fi
    curl -sS -X POST -H 'Content-Type: application/json' \
        --max-time 10 \
        -d "$(jq -nc --arg msg "$message" --arg host "$(hostname)" \
                    '{host:$host, severity:"warning", message:$msg}')" \
        "$WEBHOOK" >/dev/null || logger -t nginx-stats "Échec envoi webhook"
}

verifier_seuils() {
    local e4 e5
    e4=$(erreurs_4xx)
    e5=$(erreurs_5xx)
    if (( e5 > SEUIL_5XX )); then
        envoyer_alerte "⚠️ $e5 erreurs 5xx (seuil: $SEUIL_5XX) sur $LOG"
    fi
    if (( e4 > SEUIL_4XX )); then
        envoyer_alerte "⚠️ $e4 erreurs 4xx (seuil: $SEUIL_4XX) sur $LOG"
    fi
}
```

À appeler depuis `main`.

---

### 🔵 Étape 7 — Orchestration (15 min)

```bash
main() {
    case $FORMAT in
        text) rapport_texte ;;
        json) rapport_json ;;
    esac > "${OUTPUT:-/dev/stdout}"

    verifier_seuils
}

main
```

---

### 🔵 Étape 8 — Validation finale (30 min)

```bash
shellcheck nginx-stats.sh         # 0 warnings
./nginx-stats.sh -f nginx_logs
./nginx-stats.sh -f nginx_logs -j | jq .
./nginx-stats.sh -f nginx_logs -o rapport.txt && cat rapport.txt

# Test alerting (webhook de test)
./nginx-stats.sh -f nginx_logs -w https://webhook.site/<votre-uuid>
```

---

### 🔵 Étape 9 — Cron quotidien (15 min)

```bash
sudo cp nginx-stats.sh /usr/local/bin/
sudo chmod 755 /usr/local/bin/nginx-stats.sh
sudo crontab -e
```

Ligne à ajouter :

```
0 6 * * * /usr/local/bin/nginx-stats.sh -f /var/log/nginx/access.log -o /var/log/nginx-stats-$(date +\%F).txt -w https://webhook.site/...
```

---

## ✅ Critères de réussite

| Critère | Points |
|---|---:|
| `getopts` propre + `--help` | 2 |
| `trap EXIT` pour nettoyage | 2 |
| 8 fonctions de stat avec `awk` correctes | 8 |
| Rapport texte lisible | 3 |
| Rapport JSON valide (`jq .` passe) | 3 |
| Alerting webhook avec fallback `logger` | 3 |
| Seuils configurables | 1 |
| `shellcheck` sans warnings | 2 |
| Cron installable (chemin absolu, sortie redirigée) | 1 |
| **Bonus** : option `--since 24h` (filtrer par date) | +3 |
| **Bonus** : graphique ASCII des codes HTTP | +2 |
| **Bonus** : support des logs compressés (`zcat`) | +2 |
| **Total** | **/25** |

---

## 🎁 Bonus — Filtrer par date

```bash
# Filtre les lignes des dernières 24h
filter_24h() {
    local seuil; seuil=$(date -d '24 hours ago' '+%d/%b/%Y:%H:%M:%S')
    awk -v seuil="$seuil" '
        {
            split($4, t, "[\\[ ]")
            if (t[2] >= seuil) print
        }
    ' "$LOG"
}
```

---

## 🐛 Pièges classiques

| Symptôme | Cause | Correction |
|---|---|---|
| `awk` ne capture pas l'IP | Mauvais séparateur | Format Combined → espace par défaut, OK |
| `(( e5 > SEUIL ))` échoue | Variable vide → erreur arithmétique | `e5=${e5:-0}` |
| Webhook avec `&` dans URL | Shell interprète `&` | Toujours quoter `"$WEBHOOK"` |
| Cron : "command not found" | PATH minimal | Chemin absolu dans le crontab |
| `jq` plante sur entrée vide | Pas de données | Tester `[[ -s "$LOG" ]]` avant |

---

## 🎓 Évaluation orale (10 min)

1. Pourquoi `getopts` plutôt que parser `$@` à la main ?
2. Quel est l'intérêt de `trap EXIT` ?
3. Pourquoi utiliser `jq -n --argjson` plutôt que construire le JSON à la main ?
4. Comment géreriez-vous des logs sur 30 jours sans surcharger la RAM ?
5. Que faire si le webhook est lent et bloque le script ?

---

## 📚 Pour la prochaine fois

➡️ **TP3 (Avancé)** : framework d'orchestration multi-serveurs en pur Bash (SSH parallèle, retry, lock, modules).
