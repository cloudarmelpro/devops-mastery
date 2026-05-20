# TP1 — Bash / Terminal · Niveau Débutant

> **Titre :** Mon premier script de diagnostic système
> **Durée :** 3 à 4 heures
> **Prérequis :** Leçon 3 lue. Ubuntu 22.04+ ouvert.
> **Niveau :** ⭐ Débutant

---

## 🎯 Objectifs

À la fin de ce TP, l'étudiant saura :

- Créer un **script Bash exécutable**.
- Utiliser **variables, conditions, boucles, fonctions**.
- Lire les **arguments** (`$1`, `$@`) et les **options** simples.
- Combiner les commandes avec **pipes et redirections**.
- Adopter le **mode strict** (`set -euo pipefail`).
- Logger avec des **niveaux** et des couleurs.
- Manipuler du texte avec `grep`, `awk`, `sort`, `uniq`.

---

## 📋 Contexte

Vous arrivez en astreinte sur un serveur Linux et vous voulez un **état des lieux en 5 secondes** : OS, charge, mémoire, disque, services critiques, ports ouverts. Vous écrivez **`diagnose.sh`** : un outil de poche que vous garderez toute votre carrière.

---

## 🛠️ Mise en place

```bash
mkdir -p ~/tp1-bash && cd ~/tp1-bash
touch diagnose.sh
chmod +x diagnose.sh
sudo apt install shellcheck -y
```

---

## 🪜 Étapes

### 🟢 Étape 1 — Squelette obligatoire (15 min)

Tout script Bash de production doit commencer par :

```bash
#!/usr/bin/env bash
set -euo pipefail
IFS=$'\n\t'

main() {
    echo "Hello, DevOps !"
}

main "$@"
```

Tester :

```bash
./diagnose.sh
shellcheck diagnose.sh        # doit passer sans erreur
```

✅ Garder cette structure dans **tous** vos scripts.

---

### 🟢 Étape 2 — Couleurs et fonction `log` (20 min)

Ajouter en haut, après `IFS=` :

```bash
# Couleurs (désactivées si pas de TTY)
if [[ -t 1 ]]; then
    C_RED=$'\033[31m'; C_GREEN=$'\033[32m'; C_YELLOW=$'\033[33m'
    C_BLUE=$'\033[34m'; C_RESET=$'\033[0m'
else
    C_RED=''; C_GREEN=''; C_YELLOW=''; C_BLUE=''; C_RESET=''
fi

log_info()  { echo "${C_BLUE}[INFO]${C_RESET}  $*"; }
log_ok()    { echo "${C_GREEN}[OK]${C_RESET}    $*"; }
log_warn()  { echo "${C_YELLOW}[WARN]${C_RESET}  $*"; }
log_error() { echo "${C_RED}[ERROR]${C_RESET} $*" >&2; }
```

Tester depuis `main` :

```bash
log_info "Démarrage du diagnostic"
log_ok "Tout va bien"
log_warn "Attention"
log_error "Échec"
```

---

### 🟢 Étape 3 — Section "Système" (20 min)

Créer une fonction `section_systeme` qui affiche :

- Nom d'hôte (`hostname`)
- OS (lire `/etc/os-release`)
- Noyau (`uname -r`)
- Architecture (`uname -m`)
- Uptime (`uptime -p`)

```bash
section_systeme() {
    log_info "== Système =="
    echo "  Hostname     : $(hostname)"
    echo "  OS           : $(. /etc/os-release && echo "$PRETTY_NAME")"
    echo "  Kernel       : $(uname -r)"
    echo "  Architecture : $(uname -m)"
    echo "  Uptime       : $(uptime -p)"
}
```

L'appeler depuis `main()`.

---

### 🟢 Étape 4 — Section "Charge & mémoire" (25 min)

```bash
section_charge() {
    log_info "== Charge & mémoire =="

    # Charge moyenne
    read -r load1 load5 load15 _ < /proc/loadavg
    local nproc; nproc=$(nproc)
    echo "  Load avg     : $load1 (1m) / $load5 (5m) / $load15 (15m)  [${nproc} cœurs]"

    # Mémoire
    local mem_total mem_used mem_pct
    mem_total=$(awk '/MemTotal/ {print $2}' /proc/meminfo)
    mem_used=$(( mem_total - $(awk '/MemAvailable/ {print $2}' /proc/meminfo) ))
    mem_pct=$(( mem_used * 100 / mem_total ))
    echo "  Mémoire      : $(( mem_used / 1024 )) Mo / $(( mem_total / 1024 )) Mo  (${mem_pct}%)"
}
```

**Défi :** ajouter un `log_warn` si le load1 dépasse le nombre de cœurs.

---

### 🟢 Étape 5 — Section "Disque" (20 min)

```bash
section_disque() {
    log_info "== Disque =="
    df -hT -x tmpfs -x devtmpfs --output=source,fstype,size,used,avail,pcent,target | tail -n +1
}
```

**Défi :** boucler sur les partitions et marquer en rouge celles utilisées > 85 %.

Indice :

```bash
df -h --output=pcent,target | tail -n +2 | while read -r pct mount; do
    pct_val="${pct%%%}"          # retire le %
    if (( pct_val > 85 )); then
        log_warn "$mount à $pct"
    fi
done
```

---

### 🟢 Étape 6 — Section "Services" (25 min)

Vérifier 3 services attendus en paramètre du script.

```bash
section_services() {
    log_info "== Services =="
    local services=("$@")
    if [[ ${#services[@]} -eq 0 ]]; then
        services=(ssh cron)        # défaut
    fi
    for svc in "${services[@]}"; do
        if systemctl is-active --quiet "$svc"; then
            log_ok "$svc actif"
        else
            log_error "$svc INACTIF"
        fi
    done
}
```

Appeler :

```bash
section_services nginx ssh cron
```

---

### 🟢 Étape 7 — Section "Réseau" (20 min)

```bash
section_reseau() {
    log_info "== Réseau =="
    echo "  IP locale    : $(hostname -I | awk '{print $1}')"
    echo "  Ports en écoute :"
    ss -tulnp 2>/dev/null | awk 'NR>1 {printf "    %-6s %-22s %s\n", $1, $5, $7}'
}
```

---

### 🟢 Étape 8 — Section "Top processus" (15 min)

```bash
section_top() {
    log_info "== Top 5 processus (CPU & mémoire) =="
    echo "  -- CPU --"
    ps -eo pid,user,%cpu,%mem,comm --sort=-%cpu | head -6 | sed 's/^/    /'
    echo "  -- MEM --"
    ps -eo pid,user,%cpu,%mem,comm --sort=-%mem | head -6 | sed 's/^/    /'
}
```

---

### 🟢 Étape 9 — Options en ligne de commande (30 min)

L'utilisateur doit pouvoir choisir quelles sections afficher :

```bash
./diagnose.sh                          # tout
./diagnose.sh --systeme --disque       # juste deux sections
./diagnose.sh --services nginx ssh     # services personnalisés
./diagnose.sh --help
```

Implémentation simple avec `case` :

```bash
usage() {
    cat <<EOF
Usage: $0 [OPTIONS]
  --all (défaut)
  --systeme
  --charge
  --disque
  --services [s1 s2 …]
  --reseau
  --top
  -h, --help
EOF
}

main() {
    [[ $# -eq 0 ]] && { tout ; exit 0 ; }
    local services=()
    while [[ $# -gt 0 ]]; do
        case $1 in
            --all)      tout ;;
            --systeme)  section_systeme ;;
            --charge)   section_charge ;;
            --disque)   section_disque ;;
            --reseau)   section_reseau ;;
            --top)      section_top ;;
            --services)
                shift
                while [[ $# -gt 0 && $1 != --* ]]; do
                    services+=("$1"); shift
                done
                section_services "${services[@]}"
                continue
                ;;
            -h|--help) usage ; exit 0 ;;
            *) log_error "Option inconnue : $1" ; usage ; exit 1 ;;
        esac
        shift
    done
}

tout() {
    section_systeme
    section_charge
    section_disque
    section_services
    section_reseau
    section_top
}
```

---

### 🟢 Étape 10 — Validation finale (15 min)

```bash
shellcheck diagnose.sh        # ne doit afficher AUCUN warning
./diagnose.sh --help
./diagnose.sh                 # rapport complet
./diagnose.sh --disque --services nginx ssh
./diagnose.sh --commandeBidon # doit afficher un usage et exit 1
echo $?
```

---

## ✅ Critères de réussite

| Critère | Points |
|---|---:|
| Script exécutable avec `set -euo pipefail` | 2 |
| Fonction `log_*` colorée et fallback sans TTY | 2 |
| 6 sections fonctionnelles | 6 |
| Options CLI (`--systeme`, etc.) | 3 |
| `--help` clair | 1 |
| `shellcheck` passe sans warning | 3 |
| Gestion d'erreur (option inconnue → exit 1) | 1 |
| Idempotent : peut être ré-exécuté sans effet de bord | 1 |
| Variables `local` dans toutes les fonctions | 1 |
| **Bonus** : seuils colorés (load/disque/mémoire) | +2 |
| **Bonus** : sortie au format JSON avec option `--json` | +3 |
| **Bonus** : page d'aide générée depuis un dictionnaire | +1 |
| **Total** | **/20** |

---

## 🐛 Pièges classiques

| Symptôme | Cause | Correction |
|---|---|---|
| `unbound variable` à l'exécution | `set -u` + variable jamais initialisée | Initialiser `local services=()` |
| Couleurs visibles dans un pipe | `[[ -t 1 ]]` non testé | Tester si stdout est un TTY |
| `df` se plante sur tmpfs | Inclus par défaut | `df -x tmpfs -x devtmpfs` |
| Erreur "command not found" en cron | `PATH` minimal | Mettre `/usr/local/bin:/usr/bin:/bin` au début |
| Script qui s'arrête à `grep` qui ne trouve rien | `set -e` | `grep … || true` |

---

## 🎓 Évaluation orale (5-10 min)

1. Pourquoi `set -euo pipefail` ?
2. Pourquoi `local` dans les fonctions ?
3. Différence entre `$@` et `$*` ?
4. Pourquoi `[[ -t 1 ]]` avant d'utiliser des couleurs ?
5. Que retourne `systemctl is-active --quiet nginx` ?

---

## 📚 Pour la prochaine fois

➡️ **TP2 (Intermédiaire)** : analyser des logs Nginx, agréger des stats avec `awk`, déclencher des alertes.
