# TP3 — Bash / Terminal · Niveau Avancé

> **Titre :** `bashops` — Mini-framework d'orchestration multi-serveurs en pur Bash
> **Durée :** 8 à 12 heures (étalable sur 2 séances)
> **Prérequis :** TP1 et TP2 validés.
> **Niveau :** ⭐⭐⭐ Avancé

---

## 🎯 Objectifs

- Concevoir un **CLI Bash modulaire** (`bashops <commande> [options]`).
- Maîtriser le **SSH non interactif** avec **exécution parallèle** (`xargs -P`).
- Implémenter **retry exponentiel**, **timeout**, **lock** (`flock`), **rate limiting**.
- Structurer un projet Bash en **modules** (`lib/*.sh`).
- Mettre en place une **CI GitHub Actions** qui lance `shellcheck` + tests Bats.
- Écrire des **tests** avec [Bats](https://github.com/bats-core/bats-core).
- Gérer une **configuration multi-environnements** (YAML lu via `yq`).
- Produire un **rapport HTML** ou Markdown des résultats.

---

## 📋 Contexte

Vous gérez une flotte de **serveurs Ubuntu**. Vous voulez un outil léger, sans dépendance lourde (pas d'Ansible / Salt pour ce besoin), qui permet de :

```bash
bashops --env staging exec 'uptime'
bashops --env prod    healthcheck
bashops --env prod    deploy --version v1.4.0
bashops --env staging report --format html > rapport.html
```

L'outil doit :

- Se connecter en SSH sur tous les serveurs de l'environnement (en parallèle).
- Respecter un **timeout** par commande et un **retry** automatique.
- Ne pas exécuter deux fois en parallèle sur le même environnement (lock).
- Logger chaque opération.
- Produire un rapport.

---

## 🏗️ Architecture cible

```
bashops/
├── bashops                     # script principal (exécutable)
├── lib/
│   ├── log.sh
│   ├── config.sh
│   ├── ssh.sh
│   ├── retry.sh
│   ├── lock.sh
│   └── report.sh
├── commands/
│   ├── exec.sh
│   ├── healthcheck.sh
│   ├── deploy.sh
│   └── report.sh
├── envs/
│   ├── staging.yaml
│   └── prod.yaml
├── tests/
│   ├── test_retry.bats
│   ├── test_config.bats
│   └── test_log.bats
├── .github/workflows/ci.yml
├── Makefile
└── README.md
```

---

## 🛠️ Mise en place

```bash
mkdir -p bashops/{lib,commands,envs,tests,.github/workflows}
cd bashops
sudo apt install yq shellcheck bats jq -y
```

> ⚠️ Sur Ubuntu, le `yq` standard est celui de **mikefarah** (Go). Vérifier : `yq --version`.

---

## 🪜 Étapes (énoncé volontairement ouvert)

### 🟣 Étape 1 — `lib/log.sh` (45 min)

```bash
# lib/log.sh — système de logs avec niveaux et couleurs

: "${LOG_FILE:=/var/log/bashops.log}"
: "${LOG_LEVEL:=INFO}"

declare -A _LOG_LEVELS=([DEBUG]=0 [INFO]=1 [WARN]=2 [ERROR]=3)

_log() {
    local niveau="$1"; shift
    local cible_int="${_LOG_LEVELS[$LOG_LEVEL]}"
    local courant_int="${_LOG_LEVELS[$niveau]}"
    (( courant_int < cible_int )) && return 0
    local ts; ts=$(date -Is)
    echo "$ts [$niveau] $*" | tee -a "$LOG_FILE" >&2
}

log::debug() { _log DEBUG "$@"; }
log::info()  { _log INFO  "$@"; }
log::warn()  { _log WARN  "$@"; }
log::error() { _log ERROR "$@"; }
```

**Test Bats :** `tests/test_log.bats` doit vérifier que `log::debug` ne sort rien si `LOG_LEVEL=INFO`.

---

### 🟣 Étape 2 — `lib/config.sh` (1h)

Lire un YAML d'environnement :

```yaml
# envs/prod.yaml
serveurs:
  - host: prod-web-1.exemple.com
    user: deploy
  - host: prod-web-2.exemple.com
    user: deploy
options:
  ssh_port: 22
  ssh_key: ~/.ssh/id_ed25519
  timeout: 30
  retries: 3
  parallelism: 4
```

```bash
# lib/config.sh
config::load() {
    local env="$1"
    local fichier="envs/${env}.yaml"
    [[ -r "$fichier" ]] || { log::error "Env introuvable : $env"; return 2; }

    CFG_HOSTS=()
    while IFS= read -r host; do
        CFG_HOSTS+=("$host")
    done < <(yq '.serveurs[].host' "$fichier")

    CFG_USER=$(yq '.serveurs[0].user' "$fichier")
    CFG_SSH_PORT=$(yq '.options.ssh_port // 22' "$fichier")
    CFG_SSH_KEY=$(yq '.options.ssh_key' "$fichier" | sed "s|^~|$HOME|")
    CFG_TIMEOUT=$(yq '.options.timeout // 30' "$fichier")
    CFG_RETRIES=$(yq '.options.retries // 3' "$fichier")
    CFG_PARALLEL=$(yq '.options.parallelism // 4' "$fichier")
}
```

---

### 🟣 Étape 3 — `lib/retry.sh` (45 min)

```bash
retry::with_backoff() {
    local max="${1:-3}" ; shift
    local delai=1 i=0
    until "$@"; do
        ((i++))
        if (( i >= max )); then
            log::error "Échec définitif après $max essais : $*"
            return 1
        fi
        log::warn "Échec, retry dans ${delai}s ($i/$max)"
        sleep "$delai"
        delai=$(( delai * 2 ))
    done
}
```

Tests Bats : succès au 2e essai, échec après N essais.

---

### 🟣 Étape 4 — `lib/lock.sh` (30 min)

Empêcher deux exécutions concurrentes sur le même environnement :

```bash
LOCK_DIR="/var/lock/bashops"

lock::acquire() {
    local env="$1"
    mkdir -p "$LOCK_DIR"
    local lock_file="${LOCK_DIR}/${env}.lock"
    exec 9>"$lock_file"
    if ! flock -n 9; then
        log::error "Déjà en cours pour env=$env (lock: $lock_file)"
        return 1
    fi
    log::debug "Lock acquis pour $env"
}

# Le lock est libéré automatiquement à la fermeture du fd 9 (fin du script)
```

---

### 🟣 Étape 5 — `lib/ssh.sh` (1h30)

```bash
ssh::run() {
    local host="$1" ; shift
    local cmd="$*"
    timeout "${CFG_TIMEOUT}" \
        ssh -n \
            -o BatchMode=yes \
            -o ConnectTimeout=5 \
            -o StrictHostKeyChecking=accept-new \
            -o LogLevel=ERROR \
            -p "${CFG_SSH_PORT}" \
            -i "${CFG_SSH_KEY}" \
            "${CFG_USER}@${host}" \
            "$cmd"
}

ssh::run_all() {
    local cmd="$*"
    local resultats_dir; resultats_dir=$(mktemp -d)
    log::info "Exécution sur ${#CFG_HOSTS[@]} serveurs (parallélisme: $CFG_PARALLEL)…"

    printf '%s\n' "${CFG_HOSTS[@]}" \
        | xargs -I{} -P "${CFG_PARALLEL}" \
            bash -c '_run_one "$@"' _ {} "$resultats_dir" "$cmd"

    echo "$resultats_dir"
}

_run_one() {
    local host="$1" dir="$2" ; shift 2
    local cmd="$*"
    local out="${dir}/${host}.out"
    local err="${dir}/${host}.err"
    local code_file="${dir}/${host}.code"

    if retry::with_backoff "$CFG_RETRIES" ssh::run "$host" "$cmd" >"$out" 2>"$err"; then
        echo 0 > "$code_file"
        log::info "$host : OK"
    else
        echo 1 > "$code_file"
        log::error "$host : ÉCHEC ($(tail -1 "$err"))"
    fi
}

export -f _run_one ssh::run retry::with_backoff log::info log::warn log::error log::debug _log
```

⚠️ **Subtilité :** `xargs -P` lance des sous-shells qui ne connaissent pas les fonctions parent → **`export -f`** est obligatoire.

---

### 🟣 Étape 6 — `lib/report.sh` (1h)

Lire le dossier de résultats produits par `ssh::run_all` et générer un rapport :

```bash
report::generate() {
    local dir="$1" format="${2:-text}"
    local hosts ok=0 fail=0
    hosts=("${CFG_HOSTS[@]}")

    case "$format" in
        text) _report_text "$dir" ;;
        json) _report_json "$dir" ;;
        html) _report_html "$dir" ;;
        md)   _report_md   "$dir" ;;
        *)    log::error "Format inconnu: $format" ; return 1 ;;
    esac
}

_report_md() {
    local dir="$1"
    echo "# Rapport bashops — $(date -Is)"
    echo
    echo "| Hôte | Code | Sortie |"
    echo "|---|---:|---|"
    for h in "${CFG_HOSTS[@]}"; do
        local code; code=$(cat "${dir}/${h}.code" 2>/dev/null || echo "?")
        local out;  out=$(head -1 "${dir}/${h}.out" 2>/dev/null | tr '|' ' ')
        echo "| $h | $code | $out |"
    done
}
```

Implémenter `_report_json`, `_report_html` (table HTML simple).

---

### 🟣 Étape 7 — `bashops` (script principal, 1h)

```bash
#!/usr/bin/env bash
set -euo pipefail
IFS=$'\n\t'

ROOT="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
# shellcheck source=lib/log.sh
source "$ROOT/lib/log.sh"
source "$ROOT/lib/config.sh"
source "$ROOT/lib/retry.sh"
source "$ROOT/lib/lock.sh"
source "$ROOT/lib/ssh.sh"
source "$ROOT/lib/report.sh"

ENV=""
COMMANDE=""

usage() {
    cat <<EOF
bashops — orchestrateur SSH multi-serveurs

USAGE:
    bashops --env ENV COMMANDE [args]

COMMANDES:
    exec CMD              Exécute CMD sur tous les serveurs
    healthcheck           Vérifie ssh + nginx + disque
    deploy --version V    Déploie la version V
    report [--format f]   Génère un rapport (formats: text|json|md|html)

OPTIONS:
    --env ENV       (obligatoire) Nom de l'environnement (staging|prod)
    -v / --verbose  Mode debug
    -h / --help     Cette aide
EOF
}

while [[ $# -gt 0 ]]; do
    case $1 in
        --env)        ENV="$2"; shift 2 ;;
        -v|--verbose) LOG_LEVEL=DEBUG; shift ;;
        -h|--help)    usage ; exit 0 ;;
        exec|healthcheck|deploy|report)
            COMMANDE="$1"; shift; break ;;
        *)            echo "Option inconnue : $1" >&2 ; usage ; exit 1 ;;
    esac
done

[[ -z "$ENV" || -z "$COMMANDE" ]] && { usage ; exit 1 ; }

config::load "$ENV"
lock::acquire "$ENV" || exit 1

# shellcheck source=commands/exec.sh
source "$ROOT/commands/${COMMANDE}.sh"
"cmd::${COMMANDE}" "$@"
```

Chaque fichier `commands/<nom>.sh` doit définir une fonction `cmd::<nom>`.

Exemple `commands/exec.sh` :

```bash
cmd::exec() {
    local commande="$*"
    [[ -z "$commande" ]] && { log::error "Aucune commande" ; return 1 ; }
    local dir; dir=$(ssh::run_all "$commande")
    report::generate "$dir" text
    trap 'rm -rf "$dir"' EXIT
}
```

---

### 🟣 Étape 8 — Tests Bats (1h)

`tests/test_retry.bats` :

```bash
#!/usr/bin/env bats

load '../lib/log.sh'
load '../lib/retry.sh'

@test "retry succès au 1er essai" {
    run retry::with_backoff 3 true
    [ "$status" -eq 0 ]
}

@test "retry échec après 3 essais" {
    run retry::with_backoff 3 false
    [ "$status" -eq 1 ]
}

@test "retry succès au 2e essai" {
    counter=0
    fonction_qui_reussit_au_deux() {
        counter=$((counter+1))
        [[ $counter -ge 2 ]]
    }
    export -f fonction_qui_reussit_au_deux
    run retry::with_backoff 5 fonction_qui_reussit_au_deux
    [ "$status" -eq 0 ]
}
```

```bash
bats tests/
```

---

### 🟣 Étape 9 — Makefile (15 min)

```makefile
.PHONY: test lint install clean

lint:
	shellcheck bashops lib/*.sh commands/*.sh

test:
	bats tests/

install:
	install -m 0755 bashops /usr/local/bin/
	install -d /usr/local/lib/bashops
	cp -r lib commands envs /usr/local/lib/bashops/

clean:
	rm -rf /tmp/bashops.* /var/log/bashops.log
```

---

### 🟣 Étape 10 — CI GitHub Actions (30 min)

`.github/workflows/ci.yml` :

```yaml
name: CI
on: [push, pull_request]

jobs:
  lint-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install dependencies
        run: |
          sudo apt update
          sudo apt install -y shellcheck bats yq jq

      - name: Lint
        run: make lint

      - name: Tests
        run: make test
```

---

## ✅ Critères de réussite

| Critère | Points |
|---|---:|
| Architecture modulaire (lib/, commands/) | 3 |
| `bashops --help` clair | 1 |
| `config::load` valide le YAML et donne des valeurs par défaut | 3 |
| `retry::with_backoff` testable et exponentiel | 3 |
| `lock::acquire` empêche les exécutions concurrentes | 3 |
| `ssh::run_all` parallèle avec `xargs -P` | 4 |
| `report::generate` 4 formats (text/json/md/html) | 4 |
| `shellcheck` passe sur tous les fichiers | 3 |
| Tests Bats fonctionnels (≥ 5 tests) | 3 |
| CI verte | 3 |
| `Makefile` install/test/lint/clean | 2 |
| Documentation README.md du projet | 2 |
| Gestion d'erreurs : un serveur down ne casse pas les autres | 3 |
| Code de sortie global = 0 si tout OK, 1 sinon | 1 |
| **Bonus** : commande `deploy` avec rollback automatique | +5 |
| **Bonus** : rate limit (max X opérations / minute) | +3 |
| **Bonus** : intégration `--dry-run` | +2 |
| **Bonus** : signature de logs (HMAC) pour audit | +3 |
| **Total** | **/38** |

---

## 🎓 Évaluation finale (30 min — démo + grand oral)

**Démo (10 min) :**
1. `bashops --env staging exec 'hostname'` → tous les serveurs répondent.
2. Couper un serveur, relancer → message d'erreur mais les autres OK.
3. Lancer deux fois en parallèle → la deuxième est bloquée par le lock.
4. Exécuter le rapport HTML, ouvrir dans un navigateur.

**Questions (20 min) :**

1. Pourquoi `export -f` avant `xargs` ?
2. Comment `flock` survit-il au fait que le script crash brutalement ?
3. Quel risque y a-t-il avec `StrictHostKeyChecking=accept-new` ?
4. Comment géreriez-vous 500 serveurs au lieu de 10 ?
5. Quel est le **prix** (perf, complexité) du retry + parallélisme ?
6. Pourquoi écrire ça en Bash plutôt qu'en Python ou Go ?
7. Comment versionnerez-vous les configurations d'environnement (`envs/*.yaml`) ?
8. À quel moment cet outil devient-il insuffisant et il faut passer à Ansible ?

---

## 🔒 Bonnes pratiques cruciales

1. **`set -euo pipefail` partout** — y compris dans les modules sourcés.
2. **`StrictHostKeyChecking=accept-new`** est un compromis : on accepte la 1ère fois, on refuse si la clé change.
3. **`BatchMode=yes`** empêche SSH de prompter pour un mot de passe : si la clé ne marche pas, il échoue immédiatement.
4. **Le timeout** est obligatoire — un SSH bloqué peut faire mourir tout le pipeline.
5. **Le lock** doit utiliser `flock` (pas un fichier `pidfile` à la main) — sinon une exécution crashée laisse un état corrompu.
6. **`export -f`** : indispensable pour utiliser des fonctions Bash dans `xargs`/`parallel`.
7. **Tests Bats** : sans tests, un script de production qui modifie du shared state est un risque inacceptable.
8. **Versionner les configs** dans un dépôt **séparé** ou dans le même dépôt mais avec **revue de code**.

---

## 📚 Pour la suite

➡️ Cet outil est volontairement minimal. À partir d'environ **20 serveurs** ou pour des opérations complexes, **Ansible** devient plus approprié (idempotence native, dry-run, modules, inventaire dynamique).
➡️ **Leçon 4** : Git & Version Control → versionner proprement ce projet et collaborer dessus.
