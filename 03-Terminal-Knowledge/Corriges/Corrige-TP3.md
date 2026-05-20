# Corrigé — TP3 Bash (Avancé)

> Projet `bashops` — squelette complet, fichiers clés livrés ici.
> Note : certaines portions bonus (HMAC, rate-limit, rapport HTML) sont laissées à l'étudiant.

---

## 📁 Structure finale

```
bashops/
├── bashops
├── Makefile
├── README.md
├── envs/{staging,prod}.yaml
├── lib/{log,config,retry,lock,ssh,report}.sh
├── commands/{exec,healthcheck,deploy,report}.sh
├── tests/{test_retry,test_log,test_config}.bats
└── .github/workflows/ci.yml
```

---

## `envs/staging.yaml`

```yaml
serveurs:
  - host: staging-web-1.exemple.com
    user: deploy
  - host: staging-web-2.exemple.com
    user: deploy

options:
  ssh_port: 22
  ssh_key: ~/.ssh/id_ed25519
  timeout: 30
  retries: 3
  parallelism: 4
```

---

## `lib/log.sh`

```bash
# shellcheck shell=bash

: "${LOG_FILE:=/tmp/bashops.log}"
: "${LOG_LEVEL:=INFO}"

declare -A _LOG_LEVELS=([DEBUG]=0 [INFO]=1 [WARN]=2 [ERROR]=3)

if [[ -t 2 ]]; then
    _C_RED=$'\033[31m'; _C_GREEN=$'\033[32m'
    _C_YELLOW=$'\033[33m'; _C_BLUE=$'\033[34m'; _C_RESET=$'\033[0m'
else
    _C_RED=''; _C_GREEN=''; _C_YELLOW=''; _C_BLUE=''; _C_RESET=''
fi

_log() {
    local niveau="$1"; shift
    local cible="${_LOG_LEVELS[$LOG_LEVEL]:-1}"
    local courant="${_LOG_LEVELS[$niveau]:-1}"
    (( courant < cible )) && return 0
    local ts color
    ts=$(date -Is)
    case $niveau in
        DEBUG) color="$_C_BLUE" ;;
        INFO)  color="$_C_GREEN" ;;
        WARN)  color="$_C_YELLOW" ;;
        ERROR) color="$_C_RED" ;;
    esac
    printf '%s %s[%s]%s %s\n' "$ts" "$color" "$niveau" "$_C_RESET" "$*" >&2
    printf '%s [%s] %s\n' "$ts" "$niveau" "$*" >> "$LOG_FILE" 2>/dev/null || true
}

log::debug() { _log DEBUG "$@"; }
log::info()  { _log INFO  "$@"; }
log::warn()  { _log WARN  "$@"; }
log::error() { _log ERROR "$@"; }
```

---

## `lib/config.sh`

```bash
# shellcheck shell=bash

CFG_HOSTS=()
CFG_USER=""
CFG_SSH_PORT=22
CFG_SSH_KEY=""
CFG_TIMEOUT=30
CFG_RETRIES=3
CFG_PARALLEL=4

config::load() {
    local env="$1"
    local fichier="${ROOT}/envs/${env}.yaml"

    [[ -r "$fichier" ]] || { log::error "Env introuvable : $env ($fichier)" ; return 2 ; }

    mapfile -t CFG_HOSTS < <(yq -r '.serveurs[].host' "$fichier")
    [[ ${#CFG_HOSTS[@]} -gt 0 ]] || { log::error "Aucun serveur dans $fichier" ; return 2 ; }

    CFG_USER=$(yq -r '.serveurs[0].user' "$fichier")
    CFG_SSH_PORT=$(yq -r '.options.ssh_port // 22' "$fichier")
    CFG_SSH_KEY=$(yq -r '.options.ssh_key' "$fichier" | sed "s|^~|$HOME|")
    CFG_TIMEOUT=$(yq -r '.options.timeout // 30' "$fichier")
    CFG_RETRIES=$(yq -r '.options.retries // 3' "$fichier")
    CFG_PARALLEL=$(yq -r '.options.parallelism // 4' "$fichier")

    log::debug "Env=$env hosts=${#CFG_HOSTS[@]} user=$CFG_USER parallel=$CFG_PARALLEL"
}
```

---

## `lib/retry.sh`

```bash
# shellcheck shell=bash

retry::with_backoff() {
    local max="${1:-3}"; shift
    local delai=1 i=0
    until "$@"; do
        ((i++))
        if (( i >= max )); then
            log::error "Abandon après $max essais : $*"
            return 1
        fi
        log::warn "Échec, retry dans ${delai}s ($i/$max)"
        sleep "$delai"
        delai=$(( delai * 2 ))
        (( delai > 30 )) && delai=30
    done
}
```

---

## `lib/lock.sh`

```bash
# shellcheck shell=bash

: "${LOCK_DIR:=/tmp/bashops-locks}"

lock::acquire() {
    local env="$1"
    mkdir -p "$LOCK_DIR"
    local lock_file="${LOCK_DIR}/${env}.lock"
    exec 9>"$lock_file"
    if ! flock -n 9; then
        log::error "Verrou pris pour env=$env (autre exécution en cours)"
        return 1
    fi
    log::debug "Lock acquis : $lock_file"
}
```

---

## `lib/ssh.sh`

```bash
# shellcheck shell=bash

ssh::run() {
    local host="$1"; shift
    timeout "${CFG_TIMEOUT}" \
        ssh -n \
            -o BatchMode=yes \
            -o ConnectTimeout=5 \
            -o StrictHostKeyChecking=accept-new \
            -o LogLevel=ERROR \
            -p "${CFG_SSH_PORT}" \
            -i "${CFG_SSH_KEY}" \
            "${CFG_USER}@${host}" \
            -- "$@"
}

ssh::run_all() {
    local cmd="$*"
    local dir
    dir=$(mktemp -d /tmp/bashops-run.XXXXXX)
    log::info "Exécution sur ${#CFG_HOSTS[@]} serveur(s), parallélisme=$CFG_PARALLEL"

    export CFG_TIMEOUT CFG_SSH_PORT CFG_SSH_KEY CFG_USER CFG_RETRIES LOG_FILE LOG_LEVEL

    printf '%s\n' "${CFG_HOSTS[@]}" \
        | xargs -I{} -P "${CFG_PARALLEL}" \
            bash -c 'ssh::_run_one "$0" "$1" "$2"' {} "$dir" "$cmd"

    echo "$dir"
}

ssh::_run_one() {
    local host="$1" dir="$2"; shift 2
    local cmd="$*"
    local out="${dir}/${host}.out"
    local err="${dir}/${host}.err"
    local code_file="${dir}/${host}.code"

    if retry::with_backoff "$CFG_RETRIES" ssh::run "$host" "$cmd" >"$out" 2>"$err"; then
        echo 0 > "$code_file"
        log::info "$host : OK"
    else
        local code=$?
        echo "$code" > "$code_file"
        log::error "$host : ÉCHEC ($(tail -1 "$err" 2>/dev/null || echo 'no stderr'))"
    fi
}

export -f ssh::run ssh::_run_one retry::with_backoff \
          log::info log::warn log::error log::debug _log
```

---

## `lib/report.sh`

```bash
# shellcheck shell=bash

report::generate() {
    local dir="$1" format="${2:-text}"
    case "$format" in
        text) _report_text "$dir" ;;
        json) _report_json "$dir" ;;
        md)   _report_md   "$dir" ;;
        html) _report_html "$dir" ;;
        *)    log::error "Format inconnu : $format" ; return 1 ;;
    esac
}

_report_text() {
    local dir="$1"
    echo "===== Rapport bashops — $(date -Is) ====="
    for h in "${CFG_HOSTS[@]}"; do
        local code; code=$(cat "${dir}/${h}.code" 2>/dev/null || echo "?")
        local statut="ÉCHEC"
        [[ $code == "0" ]] && statut="OK"
        echo "-- $h ($statut, code=$code) --"
        cat "${dir}/${h}.out" 2>/dev/null || echo "(no output)"
        echo
    done
}

_report_md() {
    local dir="$1"
    echo "# Rapport bashops"
    echo
    echo "Date : $(date -Is)"
    echo
    echo "| Hôte | Code | Sortie |"
    echo "|---|---:|---|"
    for h in "${CFG_HOSTS[@]}"; do
        local code out
        code=$(cat "${dir}/${h}.code" 2>/dev/null || echo "?")
        out=$(head -1 "${dir}/${h}.out" 2>/dev/null | tr '|' '/' | head -c 80)
        echo "| $h | $code | $out |"
    done
}

_report_json() {
    local dir="$1"
    local items=()
    for h in "${CFG_HOSTS[@]}"; do
        local code; code=$(cat "${dir}/${h}.code" 2>/dev/null || echo 1)
        local out;  out=$(cat "${dir}/${h}.out" 2>/dev/null || echo "")
        items+=("$(jq -nc \
            --arg host "$h" \
            --argjson code "$code" \
            --arg out "$out" \
            '{host:$host, code:$code, output:$out}')")
    done
    printf '%s\n' "${items[@]}" | jq -s \
        --arg date "$(date -Is)" \
        '{date:$date, results:., total:length, failed:(map(select(.code != 0)) | length)}'
}

_report_html() {
    local dir="$1"
    cat <<HEADER
<!doctype html>
<html lang="fr"><head><meta charset="utf-8">
<title>Rapport bashops</title>
<style>
body{font-family:system-ui;margin:2em} table{border-collapse:collapse}
td,th{padding:.4em .8em;border:1px solid #ccc} .ok{background:#dfd} .ko{background:#fdd}
</style></head><body>
<h1>Rapport bashops — $(date -Is)</h1>
<table><tr><th>Hôte</th><th>Code</th><th>Sortie</th></tr>
HEADER
    for h in "${CFG_HOSTS[@]}"; do
        local code; code=$(cat "${dir}/${h}.code" 2>/dev/null || echo "?")
        local out;  out=$(head -3 "${dir}/${h}.out" 2>/dev/null | sed 's/</\&lt;/g')
        local cls="ko"; [[ $code == "0" ]] && cls="ok"
        cat <<EOF
<tr class="$cls"><td>$h</td><td>$code</td><td><pre>$out</pre></td></tr>
EOF
    done
    echo "</table></body></html>"
}
```

---

## `commands/exec.sh`

```bash
# shellcheck shell=bash
cmd::exec() {
    local commande="$*"
    [[ -z "$commande" ]] && { log::error "Usage: bashops --env ENV exec 'commande'" ; return 1 ; }

    local dir; dir=$(ssh::run_all "$commande")
    report::generate "$dir" text

    local failed=0
    for h in "${CFG_HOSTS[@]}"; do
        local code; code=$(cat "${dir}/${h}.code" 2>/dev/null || echo 1)
        [[ "$code" != "0" ]] && failed=$((failed+1))
    done

    rm -rf "$dir"
    return $(( failed > 0 ? 1 : 0 ))
}
```

## `commands/healthcheck.sh`

```bash
# shellcheck shell=bash
cmd::healthcheck() {
    local cmd='echo "== uptime ==" && uptime && echo "== disque ==" && df -h / && echo "== nginx ==" && systemctl is-active nginx 2>/dev/null || true'
    cmd::exec "$cmd"
}
```

## `commands/deploy.sh`

```bash
# shellcheck shell=bash
cmd::deploy() {
    local version=""
    while [[ $# -gt 0 ]]; do
        case $1 in
            --version) version="$2" ; shift 2 ;;
            *)         log::error "Option inconnue : $1" ; return 1 ;;
        esac
    done
    [[ -z "$version" ]] && { log::error "--version requis" ; return 1 ; }

    log::info "Déploiement version $version"
    local cmd
    cmd="cd /opt/app && git fetch --tags && git checkout $version && sudo systemctl restart mon-app"
    cmd::exec "$cmd"
}
```

## `commands/report.sh`

```bash
# shellcheck shell=bash
cmd::report() {
    local format="text"
    while [[ $# -gt 0 ]]; do
        case $1 in
            --format) format="$2" ; shift 2 ;;
            *)        log::error "Option inconnue : $1" ; return 1 ;;
        esac
    done
    local dir; dir=$(ssh::run_all "uptime")
    report::generate "$dir" "$format"
    rm -rf "$dir"
}
```

---

## `bashops` (script principal)

```bash
#!/usr/bin/env bash
set -euo pipefail
IFS=$'\n\t'

ROOT="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"

# shellcheck source=lib/log.sh
source "$ROOT/lib/log.sh"
# shellcheck source=lib/config.sh
source "$ROOT/lib/config.sh"
# shellcheck source=lib/retry.sh
source "$ROOT/lib/retry.sh"
# shellcheck source=lib/lock.sh
source "$ROOT/lib/lock.sh"
# shellcheck source=lib/ssh.sh
source "$ROOT/lib/ssh.sh"
# shellcheck source=lib/report.sh
source "$ROOT/lib/report.sh"

ENV=""
COMMANDE=""

usage() {
    cat <<EOF
bashops — orchestrateur SSH multi-serveurs

USAGE:
    bashops --env ENV COMMANDE [args...]

COMMANDES:
    exec CMD                Exécute CMD sur tous les serveurs
    healthcheck             uptime + disque + nginx
    deploy --version V      Déploie la version V
    report [--format f]     Rapport (text|json|md|html)

OPTIONS:
    --env ENV               (obligatoire) staging | prod | …
    -v, --verbose           DEBUG
    -h, --help              cette aide
EOF
}

while [[ $# -gt 0 ]]; do
    case $1 in
        --env)         ENV="$2" ; shift 2 ;;
        -v|--verbose)  LOG_LEVEL=DEBUG ; shift ;;
        -h|--help)     usage ; exit 0 ;;
        exec|healthcheck|deploy|report)
                       COMMANDE="$1" ; shift ; break ;;
        *)             log::error "Argument inconnu : $1" ; usage ; exit 1 ;;
    esac
done

[[ -z "$ENV" ]]       && { log::error "--env est obligatoire" ; exit 1 ; }
[[ -z "$COMMANDE" ]]  && { log::error "Commande manquante"     ; usage ; exit 1 ; }

config::load "$ENV"   || exit 2
lock::acquire "$ENV"  || exit 1

# shellcheck source=commands/exec.sh
source "$ROOT/commands/${COMMANDE}.sh"

"cmd::${COMMANDE}" "$@"
```

```bash
chmod +x bashops
```

---

## `tests/test_retry.bats`

```bash
#!/usr/bin/env bats

setup() {
    ROOT="$(cd "$(dirname "$BATS_TEST_FILENAME")/.." && pwd)"
    LOG_FILE=/dev/null
    LOG_LEVEL=ERROR
    # shellcheck source=../lib/log.sh
    source "$ROOT/lib/log.sh"
    # shellcheck source=../lib/retry.sh
    source "$ROOT/lib/retry.sh"
}

@test "retry: réussit au 1er essai" {
    run retry::with_backoff 3 true
    [ "$status" -eq 0 ]
}

@test "retry: échoue après 3 essais" {
    run retry::with_backoff 3 false
    [ "$status" -eq 1 ]
}

@test "retry: réussit au 2e essai" {
    local script="$BATS_TMPDIR/r.sh"
    cat > "$script" <<'EOF'
#!/usr/bin/env bash
COUNT_FILE="$1"
n=$(cat "$COUNT_FILE" 2>/dev/null || echo 0)
echo $((n+1)) > "$COUNT_FILE"
test "$n" -ge 1
EOF
    chmod +x "$script"
    local counter; counter=$(mktemp)
    run retry::with_backoff 5 "$script" "$counter"
    [ "$status" -eq 0 ]
}
```

## `tests/test_log.bats`

```bash
#!/usr/bin/env bats

setup() {
    ROOT="$(cd "$(dirname "$BATS_TEST_FILENAME")/.." && pwd)"
    LOG_FILE="$BATS_TMPDIR/log.txt"
    : > "$LOG_FILE"
    LOG_LEVEL=INFO
    # shellcheck source=../lib/log.sh
    source "$ROOT/lib/log.sh"
}

@test "log: debug ne sort pas en INFO" {
    run log::debug "message debug"
    [ "$status" -eq 0 ]
    ! grep -q "debug" "$LOG_FILE"
}

@test "log: info sort en INFO" {
    run log::info "message info"
    [ "$status" -eq 0 ]
    grep -q "message info" "$LOG_FILE"
}

@test "log: error sort toujours" {
    LOG_LEVEL=ERROR
    source "$ROOT/lib/log.sh"
    run log::error "message error"
    [ "$status" -eq 0 ]
    grep -q "message error" "$LOG_FILE"
}
```

---

## `Makefile`

```makefile
.PHONY: all lint test install clean help

all: lint test

lint:
	@echo ">> shellcheck"
	@shellcheck -x bashops lib/*.sh commands/*.sh

test:
	@echo ">> bats"
	@bats tests/

install:
	install -m 0755 -d /usr/local/lib/bashops
	cp -r lib commands envs /usr/local/lib/bashops/
	install -m 0755 bashops /usr/local/bin/bashops

clean:
	rm -rf /tmp/bashops* /tmp/nginxstats.*

help:
	@echo "Cibles : lint, test, install, clean"
```

---

## `.github/workflows/ci.yml`

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
          sudo apt-get update
          sudo apt-get install -y shellcheck bats jq
          sudo wget -qO /usr/local/bin/yq https://github.com/mikefarah/yq/releases/latest/download/yq_linux_amd64
          sudo chmod +x /usr/local/bin/yq

      - name: Lint
        run: make lint

      - name: Tests
        run: make test
```

---

## ✅ Points pédagogiques importants

1. **Sourcing avec `# shellcheck source=...`** : permet à shellcheck de suivre les imports.
2. **`export -f`** : indispensable pour appeler des fonctions depuis `xargs -P`. Oubli classique.
3. **`flock -n 9`** : `-n` = non bloquant. Sans, le 2e processus attend indéfiniment.
4. **`-o BatchMode=yes`** : SSH échoue immédiatement plutôt que prompter, vital pour scripter.
5. **`StrictHostKeyChecking=accept-new`** : accepte la 1ère connexion, refuse si la clé change → bon compromis.
6. **`timeout 30 ssh …`** : double protection avec `ConnectTimeout=5` côté SSH.
7. **Tests Bats** : `run` capture stdout/stderr/$status sans planter le test.
8. **`mapfile -t array < <(...)`** : lit un flux dans un tableau. Plus propre que `while read`.

---

## 🐛 Pièges classiques

| Symptôme | Cause | Solution |
|---|---|---|
| `xargs` ne trouve pas la fonction | Pas d'`export -f` | Toujours exporter les fonctions appelées |
| Lock jamais libéré | `pidfile` à la main | Utiliser `flock` (libéré au close fd auto) |
| SSH prompte pour mot de passe | Pas de `BatchMode` | `-o BatchMode=yes` |
| Tests Bats plantent en CI | Variables d'environnement | `setup()` propre dans chaque fichier |
| `yq` ne parse pas | Mauvais yq (Python vs Go) | Forcer le binaire Mikefarah dans la CI |
| Sourcing relatif `lib/` | Cwd différent | `$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)` |

---

## 🎓 Réponses aux questions du grand oral

1. **`export -f` avant `xargs`** : `xargs` lance un nouveau processus Bash. Les fonctions ne sont héritées qu'avec `export -f`. Sans, on a `command not found`.

2. **`flock` survit au crash** : `exec 9>file ; flock -n 9` — le verrou est tenu par le **file descriptor 9**. Quand le shell meurt (même brutalement), le kernel ferme tous les fd → libération automatique. C'est plus robuste qu'un pidfile.

3. **`StrictHostKeyChecking=accept-new` risque** : si un attaquant fait du MITM **lors de la première connexion**, on accepte sa clé. Mitigations : pré-provisionner `known_hosts` via la configuration management ; ou utiliser des certificats SSH signés par une CA.

4. **500 serveurs** : (a) augmenter `parallelism`, (b) batcher (10 vagues de 50), (c) inventaire dynamique (lire depuis un CMDB ou cloud API), (d) à ce volume, Ansible/Salt sont plus appropriés.

5. **Prix du retry + parallélisme** : (a) **latence** dégradée : si 1 sur 10 échoue, le rapport attend les retries (donc `timeout × retries` worst case), (b) **logs noyés** par les retries, (c) **risque d'effet domino** si la cause est commune (DNS down).

6. **Bash vs Python/Go** : Bash gagne en (a) zéro dépendance (présent partout), (b) glue layer naturelle avec les outils Unix, (c) lecture rapide pour un sysadmin. Bash perd en (d) test/refactoring, (e) types, (f) parallélisme propre. → **Bash pour ≤ 500 lignes, ≤ 20 serveurs**. Au-delà, passer à Python (Fabric, Ansible-as-library) ou Go.

7. **Versionner les configs `envs/*.yaml`** : dans Git avec **revue obligatoire**, idéalement dans un **dépôt séparé** (boundary clair), secrets sortis dans un Vault (jamais en clair).

8. **Quand passer à Ansible** : (a) > 20 serveurs, (b) besoin d'idempotence forte (rejouer 10 fois doit produire le même état), (c) besoin de modules (gestion de packages, users, fichiers), (d) inventaire dynamique multi-cloud, (e) équipe > 3 ops qui modifient les playbooks.
