# Corrigé — TP2 Bash (Intermédiaire)

> Script `nginx-stats.sh` complet : parsing + JSON + alerting.

---

## `nginx-stats.sh`

```bash
#!/usr/bin/env bash
#
# nginx-stats.sh — Analyse de logs Nginx (format combined) + rapport + alerting.
#
set -euo pipefail
IFS=$'\n\t'

# --- Configuration par défaut ---
LOG_DEFAULT="/var/log/nginx/access.log"
SEUIL_5XX="${SEUIL_5XX:-100}"
SEUIL_4XX="${SEUIL_4XX:-1000}"
WEBHOOK="${ALERT_WEBHOOK:-}"
FORMAT="text"
OUTPUT=""
LOG=""

# --- Logging interne ---
log()      { echo "$(date '+%F %T') [bash-stats] $*" >&2; }
log::warn() { log "WARN  $*"; }
log::err()  { log "ERROR $*"; }

# --- Usage ---
usage() {
    cat <<EOF
Usage: $(basename "$0") [OPTIONS]

  -f FICHIER    Fichier de log Nginx (défaut: $LOG_DEFAULT)
  -o FICHIER    Sauvegarder le rapport (défaut: stdout)
  -j            Format JSON (défaut: text)
  -w URL        Webhook d'alerte (sinon ALERT_WEBHOOK env)
  -h            Cette aide

Variables d'env :
  SEUIL_5XX     Nombre d'erreurs 5xx déclenchant une alerte (défaut: 100)
  SEUIL_4XX     Idem pour les 4xx (défaut: 1000)
  ALERT_WEBHOOK URL du webhook
EOF
}

# --- Parser les options ---
while getopts ":f:o:jw:h" opt; do
    case $opt in
        f) LOG=$OPTARG ;;
        o) OUTPUT=$OPTARG ;;
        j) FORMAT="json" ;;
        w) WEBHOOK=$OPTARG ;;
        h) usage ; exit 0 ;;
        :) log::err "Option -$OPTARG nécessite un argument" ; exit 1 ;;
        \?) log::err "Option inconnue : -$OPTARG" ; usage ; exit 1 ;;
    esac
done
shift $((OPTIND-1))

LOG="${LOG:-$LOG_DEFAULT}"
[[ -r "$LOG" ]] || { log::err "Log illisible : $LOG" ; exit 2 ; }
[[ -s "$LOG" ]] || { log::err "Log vide : $LOG" ; exit 3 ; }

# --- Trap nettoyage ---
TMP=$(mktemp -d /tmp/nginxstats.XXXXXX)
trap 'rm -rf "$TMP"' EXIT

# --- Statistiques ---

total_requetes() { wc -l < "$LOG"; }

top_ips() {
    awk '{print $1}' "$LOG" | sort | uniq -c | sort -rn | head -10
}

top_urls() {
    awk '{print $7}' "$LOG" | sort | uniq -c | sort -rn | head -10
}

distribution_codes() {
    awk '{print $9}' "$LOG" | sort | uniq -c | sort -rn
}

bande_passante_mo() {
    awk '{s+=$10} END {printf "%.2f", s/1024/1024}' "$LOG"
}

heures_de_pointe() {
    awk -F'[' '{print $2}' "$LOG" \
        | awk -F: '{print $2}' \
        | sort | uniq -c | sort -rn | head -5
}

erreurs_5xx() { awk '$9 ~ /^5[0-9][0-9]$/' "$LOG" | wc -l; }
erreurs_4xx() { awk '$9 ~ /^4[0-9][0-9]$/' "$LOG" | wc -l; }

# --- Rapport texte ---
rapport_texte() {
    local total bp e4 e5
    total=$(total_requetes)
    bp=$(bande_passante_mo)
    e4=$(erreurs_4xx)
    e5=$(erreurs_5xx)

    cat <<EOF
================================================================
  RAPPORT NGINX — $(date '+%Y-%m-%d %H:%M')
================================================================
Fichier         : $LOG
Total requêtes  : $total
Bande passante  : ${bp} Mo
Erreurs 4xx     : $e4 (seuil: $SEUIL_4XX)
Erreurs 5xx     : $e5 (seuil: $SEUIL_5XX)

-- Top 10 IPs --
$(top_ips)

-- Top 10 URLs --
$(top_urls)

-- Distribution codes HTTP --
$(distribution_codes)

-- Heures de pointe --
$(heures_de_pointe)
================================================================
EOF
}

# --- Rapport JSON ---
rapport_json() {
    local total bp e4 e5
    total=$(total_requetes)
    bp=$(bande_passante_mo)
    e4=$(erreurs_4xx)
    e5=$(erreurs_5xx)

    local top_ips_json top_urls_json codes_json hours_json

    top_ips_json=$(awk '{print $1}' "$LOG" | sort | uniq -c | sort -rn | head -10 \
        | awk '{printf "{\"count\":%d,\"ip\":\"%s\"}\n", $1, $2}' \
        | jq -s .)

    top_urls_json=$(awk '{print $7}' "$LOG" | sort | uniq -c | sort -rn | head -10 \
        | awk '{printf "{\"count\":%d,\"url\":\"%s\"}\n", $1, $2}' \
        | jq -s .)

    codes_json=$(awk '{print $9}' "$LOG" | sort | uniq -c | sort -rn \
        | awk '{printf "{\"count\":%d,\"code\":\"%s\"}\n", $1, $2}' \
        | jq -s .)

    hours_json=$(awk -F'[' '{print $2}' "$LOG" \
        | awk -F: '{print $2}' | sort | uniq -c | sort -rn | head -5 \
        | awk '{printf "{\"count\":%d,\"hour\":\"%s\"}\n", $1, $2}' \
        | jq -s .)

    jq -n \
        --arg date "$(date -Is)" \
        --arg log "$LOG" \
        --arg bp "$bp" \
        --argjson total "$total" \
        --argjson e4 "$e4" \
        --argjson e5 "$e5" \
        --argjson seuil_4xx "$SEUIL_4XX" \
        --argjson seuil_5xx "$SEUIL_5XX" \
        --argjson top_ips "$top_ips_json" \
        --argjson top_urls "$top_urls_json" \
        --argjson codes "$codes_json" \
        --argjson hours "$hours_json" \
        '{
            date: $date,
            log_file: $log,
            total_requetes: $total,
            bande_passante_mo: ($bp | tonumber),
            erreurs_4xx: $e4,
            erreurs_5xx: $e5,
            seuils: {x4: $seuil_4xx, x5: $seuil_5xx},
            top_ips: $top_ips,
            top_urls: $top_urls,
            codes_http: $codes,
            heures_de_pointe: $hours
        }'
}

# --- Alerting ---
envoyer_alerte() {
    local message="$1"
    if [[ -z "$WEBHOOK" ]]; then
        logger -t nginx-stats "ALERTE (no webhook): $message"
        log::warn "ALERTE (no webhook): $message"
        return 0
    fi
    local payload
    payload=$(jq -nc \
        --arg host "$(hostname)" \
        --arg severity "warning" \
        --arg message "$message" \
        '{host:$host, severity:$severity, message:$message}')

    if ! curl -sS --max-time 10 \
            -X POST -H 'Content-Type: application/json' \
            -d "$payload" "$WEBHOOK" >/dev/null
    then
        log::err "Échec envoi webhook"
        return 1
    fi
    log::warn "ALERTE envoyée : $message"
}

verifier_seuils() {
    local e4 e5
    e4=$(erreurs_4xx)
    e5=$(erreurs_5xx)
    (( e5 > SEUIL_5XX )) && envoyer_alerte "$(hostname): $e5 erreurs 5xx (seuil $SEUIL_5XX)"
    (( e4 > SEUIL_4XX )) && envoyer_alerte "$(hostname): $e4 erreurs 4xx (seuil $SEUIL_4XX)"
    return 0
}

# --- Orchestration ---
main() {
    case $FORMAT in
        text) rapport_texte ;;
        json) rapport_json ;;
        *)    log::err "Format inconnu : $FORMAT" ; exit 1 ;;
    esac > "${OUTPUT:-/dev/stdout}"

    verifier_seuils
}

main
```

---

## 🎁 Bonus — Filtrer par date (`--since 24h`)

Ajouter dans `getopts` (avec `s:` ou option longue) :

```bash
# Convertit "24h", "7d" en timestamp epoch
parse_since() {
    case "$1" in
        *h) echo $(( $(date +%s) - ${1%h} * 3600 )) ;;
        *d) echo $(( $(date +%s) - ${1%d} * 86400 )) ;;
        *)  echo $(date -d "$1" +%s 2>/dev/null) ;;
    esac
}

filter_log() {
    local seuil_epoch="$1"
    awk -v seuil="$seuil_epoch" '
        function nginx_to_epoch(s,    tab, mois, parts) {
            # [17/May/2015:08:05:32 +0000]
            mois["Jan"]=1; mois["Feb"]=2; mois["Mar"]=3; mois["Apr"]=4
            mois["May"]=5; mois["Jun"]=6; mois["Jul"]=7; mois["Aug"]=8
            mois["Sep"]=9; mois["Oct"]=10; mois["Nov"]=11; mois["Dec"]=12
            gsub(/[\[\]:\/]/, " ", s)
            split(s, tab, " ")
            # tab = [day, month, year, h, m, s, tz]
            t = sprintf("%04d %02d %02d %02d %02d %02d",
                        tab[3], mois[tab[2]], tab[1], tab[4], tab[5], tab[6])
            return mktime(t)
        }
        {
            if (nginx_to_epoch($4) >= seuil) print
        }
    ' "$LOG" > "$TMP/filtered.log"
    LOG="$TMP/filtered.log"
}
```

---

## 🎁 Bonus — Graphique ASCII des codes HTTP

```bash
graphique_codes() {
    local total
    total=$(total_requetes)
    awk '{print $9}' "$LOG" | sort | uniq -c | sort -rn | head -10 | \
        awk -v total="$total" '{
            n = int(($1 / total) * 50)
            bar = ""
            for (i = 0; i < n; i++) bar = bar "█"
            printf "  %-5s %8d  %s\n", $2, $1, bar
        }'
}
```

---

## ✅ Points pédagogiques importants

1. **`getopts ":f:o:jw:h"`** : les `:` après les options qui prennent un argument (`-f file`).
2. **`shift $((OPTIND-1))`** : après `getopts`, consommer les options pour que `$@` ne contienne que les arguments positionnels restants.
3. **`mktemp -d /tmp/nginxstats.XXXXXX`** : suffixe `XXXXXX` obligatoire pour la randomisation.
4. **`jq -n --argjson`** : construire un JSON sans risque d'injection. **Jamais** concaténer à la main.
5. **`awk '$9 ~ /^5[0-9][0-9]$/'`** : regex précise > `^5` (qui matche 5, 500 et 50).
6. **`logger -t`** : fallback vers syslog quand pas de webhook configuré → toujours trace.
7. **`--max-time 10`** sur `curl` : protège contre un webhook lent qui bloquerait le script.

---

## 🐛 Pièges classiques

| Symptôme | Cause | Solution |
|---|---|---|
| `(( e5 > SEUIL_5XX ))` plante | Variable vide | Forcer un défaut : `e5=${e5:-0}` ou via `getopts` |
| JSON invalide | Caractères spéciaux dans URLs | Utiliser `jq -n --arg` |
| `webhook` ralentit le cron | Pas de timeout | `--max-time 10` |
| `awk` ne compte rien | Champ mal numéroté | `awk -F' '` (par défaut OK pour combined) |
| `wc -l < empty.log` retourne 0 mais `(( ))` plante | (cas limite) | Ajouter test `[[ -s "$LOG" ]]` au début |
| Sort `sort -rn` ne trie pas | Caractères blancs | `sort -k1 -rn` pour préciser la colonne |

---

## 🎓 Réponses aux questions d'évaluation

1. **`getopts` vs parsing manuel** : `getopts` gère les regroupements (`-jv`), les arguments manquants, les options inconnues, automatiquement. Standard et portable.

2. **`trap EXIT`** : garantit le nettoyage même en cas d'erreur (`set -e` qui sort), de Ctrl+C, ou de signal `TERM`. Sans ça, `/tmp` se remplit.

3. **`jq -n --argjson`** : (a) sécurise contre l'injection JSON, (b) gère l'échappement automatique, (c) lisible. Concaténer du JSON à la main est une erreur en 2026.

4. **Logs sur 30 jours sans surcharger la RAM** : éviter `sort | uniq` qui charge tout. Utiliser `awk` avec tableaux associatifs (un seul passage) + `gzip`/`zcat` pour les archives. Ou diviser pour mieux régner : un script par jour, agrégation finale.

5. **Webhook lent** : `curl --max-time 10`, plus `&` + `wait` avec un timeout global pour ne pas dépasser X secondes au total.
