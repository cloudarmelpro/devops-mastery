# Corrigé — TP1 Bash (Débutant)

> Script `diagnose.sh` complet, prêt à `shellcheck`.

---

## `diagnose.sh`

```bash
#!/usr/bin/env bash
#
# diagnose.sh — Diagnostic système rapide (sections optionnelles).
#
# Usage : diagnose.sh [--all|--systeme|--charge|--disque|--reseau|--top|--services s1 s2 …] [-h]
#
set -euo pipefail
IFS=$'\n\t'

# --- Couleurs (auto si TTY) ---
if [[ -t 1 ]]; then
    C_RED=$'\033[31m'; C_GREEN=$'\033[32m'; C_YELLOW=$'\033[33m'
    C_BLUE=$'\033[34m'; C_BOLD=$'\033[1m'; C_RESET=$'\033[0m'
else
    C_RED=''; C_GREEN=''; C_YELLOW=''; C_BLUE=''; C_BOLD=''; C_RESET=''
fi

log_info()  { echo "${C_BLUE}${C_BOLD}[INFO]${C_RESET}  $*"; }
log_ok()    { echo "${C_GREEN}[OK]${C_RESET}    $*"; }
log_warn()  { echo "${C_YELLOW}[WARN]${C_RESET}  $*"; }
log_error() { echo "${C_RED}[ERROR]${C_RESET} $*" >&2; }

# --- Sections ---

section_systeme() {
    log_info "== Système =="
    local pretty
    pretty=$(. /etc/os-release && echo "${PRETTY_NAME:-Inconnu}")
    echo "  Hostname     : $(hostname)"
    echo "  OS           : $pretty"
    echo "  Kernel       : $(uname -r)"
    echo "  Architecture : $(uname -m)"
    echo "  Uptime       : $(uptime -p)"
    echo "  Utilisateur  : $(whoami)"
}

section_charge() {
    log_info "== Charge & mémoire =="
    local load1 load5 load15 _ignore nproc_count load_pct
    read -r load1 load5 load15 _ignore < /proc/loadavg
    nproc_count=$(nproc)
    echo "  Load avg     : $load1 (1m) / $load5 (5m) / $load15 (15m)  [${nproc_count} cœurs]"

    # Alerte si load1 > nb cœurs
    load_pct=$(awk -v l="$load1" -v c="$nproc_count" 'BEGIN {printf "%.0f", (l / c) * 100}')
    if (( load_pct > 80 )); then
        log_warn "  → charge à ${load_pct}% — système chargé"
    fi

    local mem_total mem_avail mem_used mem_pct
    mem_total=$(awk '/MemTotal/ {print $2}' /proc/meminfo)
    mem_avail=$(awk '/MemAvailable/ {print $2}' /proc/meminfo)
    mem_used=$(( mem_total - mem_avail ))
    mem_pct=$(( mem_used * 100 / mem_total ))
    echo "  Mémoire      : $(( mem_used / 1024 )) Mo / $(( mem_total / 1024 )) Mo  (${mem_pct}%)"
    (( mem_pct > 85 )) && log_warn "  → mémoire à ${mem_pct}%"

    local swap_total swap_used
    swap_total=$(awk '/SwapTotal/ {print $2}' /proc/meminfo)
    if (( swap_total > 0 )); then
        swap_used=$(awk '/SwapTotal/ {t=$2} /SwapFree/ {f=$2} END {print t-f}' /proc/meminfo)
        echo "  Swap         : $(( swap_used / 1024 )) Mo / $(( swap_total / 1024 )) Mo"
    else
        echo "  Swap         : non activé"
    fi
}

section_disque() {
    log_info "== Disque =="
    df -hT -x tmpfs -x devtmpfs -x squashfs --output=source,fstype,size,used,avail,pcent,target \
        | awk 'NR==1 {print "  " $0} NR>1 {
            pct=$6; gsub("%","",pct);
            if (pct+0 > 85) printf "  \033[31m%s\033[0m\n", $0;
            else if (pct+0 > 70) printf "  \033[33m%s\033[0m\n", $0;
            else printf "  %s\n", $0;
          }'
}

section_services() {
    log_info "== Services =="
    local services=("$@")
    if [[ ${#services[@]} -eq 0 ]]; then
        services=(ssh cron)
    fi
    for svc in "${services[@]}"; do
        if systemctl is-active --quiet "$svc" 2>/dev/null; then
            log_ok "$svc actif"
        else
            log_error "$svc INACTIF"
        fi
    done
}

section_reseau() {
    log_info "== Réseau =="
    local ip
    ip=$(hostname -I 2>/dev/null | awk '{print $1}')
    echo "  IP locale    : ${ip:-N/A}"
    echo "  Interface(s) :"
    ip -brief -4 addr show 2>/dev/null | awk '{printf "    %-12s %s\n", $1, $3}'
    echo "  Ports en écoute :"
    ss -tulnp 2>/dev/null | awk 'NR>1 {printf "    %-6s %-22s %s\n", $1, $5, $7}' | sort -u | head -20
}

section_top() {
    log_info "== Top 5 processus =="
    echo "  -- CPU --"
    ps -eo pid,user,%cpu,%mem,comm --sort=-%cpu --no-headers | head -5 \
        | awk '{printf "    %-6s %-10s %5s%% %5s%% %s\n", $1, $2, $3, $4, $5}'
    echo "  -- MEM --"
    ps -eo pid,user,%cpu,%mem,comm --sort=-%mem --no-headers | head -5 \
        | awk '{printf "    %-6s %-10s %5s%% %5s%% %s\n", $1, $2, $3, $4, $5}'
}

tout() {
    section_systeme
    section_charge
    section_disque
    section_services
    section_reseau
    section_top
}

usage() {
    cat <<EOF
Usage: $(basename "$0") [OPTIONS]

  --all                 Toutes les sections (défaut si pas d'argument)
  --systeme             Infos système
  --charge              CPU + mémoire
  --disque              Partitions
  --services s1 s2 …    Vérifier des services systemd
  --reseau              Interfaces et ports
  --top                 Top 5 processus
  -h, --help            Cette aide

Exemples :
  $(basename "$0")
  $(basename "$0") --disque --reseau
  $(basename "$0") --services nginx ssh fail2ban
EOF
}

main() {
    if [[ $# -eq 0 ]]; then
        tout
        exit 0
    fi

    local services=()
    while [[ $# -gt 0 ]]; do
        case $1 in
            --all)      tout ; shift ;;
            --systeme)  section_systeme ; shift ;;
            --charge)   section_charge ; shift ;;
            --disque)   section_disque ; shift ;;
            --reseau)   section_reseau ; shift ;;
            --top)      section_top ; shift ;;
            --services)
                shift
                while [[ $# -gt 0 && $1 != --* ]]; do
                    services+=("$1")
                    shift
                done
                section_services "${services[@]}"
                services=()
                ;;
            -h|--help)  usage ; exit 0 ;;
            *)
                log_error "Option inconnue : $1"
                usage
                exit 1
                ;;
        esac
    done
}

main "$@"
```

---

## 🎁 Bonus — Option `--json`

À ajouter (nécessite `jq`) :

```bash
section_json() {
    local mem_total mem_avail load1 _
    read -r load1 _ _ _ < /proc/loadavg
    mem_total=$(awk '/MemTotal/ {print $2}' /proc/meminfo)
    mem_avail=$(awk '/MemAvailable/ {print $2}' /proc/meminfo)

    jq -n \
      --arg host "$(hostname)" \
      --arg os "$(. /etc/os-release && echo "$PRETTY_NAME")" \
      --arg kernel "$(uname -r)" \
      --arg arch "$(uname -m)" \
      --arg uptime "$(uptime -p)" \
      --arg load1 "$load1" \
      --argjson mem_total_kb "$mem_total" \
      --argjson mem_avail_kb "$mem_avail" \
      '{
        host: $host,
        os: $os,
        kernel: $kernel,
        arch: $arch,
        uptime: $uptime,
        load_1m: $load1,
        memory: {total_kb: $mem_total_kb, available_kb: $mem_avail_kb}
      }'
}
```

Ajouter `--json` dans le `case` de `main`.

---

## ✅ Points pédagogiques importants

1. **`if [[ -t 1 ]]`** : test si stdout est un TTY → désactive les couleurs si `./diagnose.sh > rapport.txt`.
2. **`local`** dans toutes les fonctions : isole les variables → pas de pollution.
3. **`shift` après lecture d'option** : évite des boucles infinies sur les options inconnues.
4. **`/etc/os-release`** lu via sourcing (`. /etc/...`) dans un sous-shell `$()` → ne pollue pas l'environnement.
5. **`set -e` + `grep || true`** : si une commande peut retourner non-zero légitimement, l'expliciter.
6. **Couleurs ANSI** : `$'\033[31m'` (syntaxe `$'…'`) interprète correctement `\033`.

---

## 🐛 Pièges classiques

| Symptôme | Cause | Solution |
|---|---|---|
| `unbound variable PRETTY_NAME` | `set -u` + variable non définie | `${PRETTY_NAME:-Inconnu}` |
| Couleurs visibles dans un fichier | Pas de test TTY | `[[ -t 1 ]]` avant |
| `services` reset après chaque arg | Re-déclaration locale dans le case | Réinitialiser après usage |
| `df` plante sur certains FS | Inclusion par défaut | `-x tmpfs -x devtmpfs` |
| `read -r load1 load5 load15 _` warning shellcheck | `_` est une variable | `_ignore` ou `# shellcheck disable=SC2034` |
