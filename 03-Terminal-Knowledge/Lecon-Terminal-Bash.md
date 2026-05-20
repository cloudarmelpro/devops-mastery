# Leçon 3 — Terminal Knowledge & Bash avancé

> Module : **Terminal Knowledge** (Scripting, Process Monitoring, Performance Monitoring, Networking Tools, Text Manipulation, Editors)
> Public : étudiants ayant validé les leçons 1 (Python) et 2 (Linux Ubuntu)
> Durée estimée : **16 à 20 heures** (théorie + manipulations + TP)
> Prérequis : connaître les commandes Linux de base (TP1 Ubuntu), savoir naviguer dans le shell

---

## 1. Objectifs pédagogiques

À la fin de cette leçon, l'étudiant sera capable de :

1. Comprendre **comment fonctionne un shell** et le rôle de Bash.
2. Écrire des **scripts Bash robustes** (variables, conditions, boucles, fonctions, tableaux).
3. Manipuler du texte de façon avancée avec **grep / sed / awk / jq**.
4. Surveiller **processus, CPU, mémoire, disque, réseau** avec les bons outils.
5. Diagnostiquer un **problème réseau** avec `ss`, `dig`, `traceroute`, `tcpdump`, `curl`.
6. Utiliser les **patterns DevOps** : `set -euo pipefail`, traps, here-docs, process substitution.
7. Paralléliser des tâches en shell (`xargs -P`, `&`, `wait`, GNU `parallel`).
8. Survivre dans **Vim** (édition, recherche, sauvegarde).
9. Déboguer un script (`set -x`, `shellcheck`, `bashdb`).
10. Connaître les **outils modernes** (`ripgrep`, `fd`, `bat`, `fzf`, `tldr`).

---

## 2. Le shell : la "ligne de commande" expliquée

### 2.1 Qu'est-ce qu'un shell ?

Un **shell** est un programme qui lit des commandes, les exécute et affiche le résultat.
C'est l'interface entre l'utilisateur et le noyau Linux.

```
+-----------+      +--------+      +--------+
|   vous    | ───> |  bash  | ───> | kernel |
+-----------+      +--------+      +--------+
```

### 2.2 Les shells courants

| Shell | Usage |
|---|---|
| `sh` (Bourne shell) | Le plus minimal, POSIX, scripts ultra-portables |
| `bash` (Bourne Again Shell) | **Le standard Linux**, riche, par défaut sur Ubuntu |
| `zsh` | Shell interactif puissant (autocomplétion, thèmes) — défaut macOS |
| `fish` | Très ergonomique pour les humains, **incompatible** avec sh/bash |
| `dash` | Léger, conforme POSIX, utilisé par `/bin/sh` sur Ubuntu (scripts système) |

**Règle DevOps :** scripts de production en **bash** (#!/usr/bin/env bash), interactif au choix.

### 2.3 Voir et changer son shell

```bash
echo $SHELL              # shell de login
cat /etc/shells          # shells installés
chsh -s /usr/bin/zsh     # changer son shell par défaut
```

---

## 3. Variables et expansion

### 3.1 Déclarer et utiliser

```bash
NOM="alice"
echo "Bonjour $NOM"          # interpolation
echo 'Bonjour $NOM'          # littéral (apostrophes)
echo "Date: $(date)"         # sous-shell, syntaxe moderne
echo "Date: `date`"          # ancienne syntaxe, à éviter
```

### 3.2 Portée et exports

```bash
VAR="local"
export VAR_GLOBALE="visible par les sous-processus"
```

⚠️ **Sans `export`**, la variable n'est pas vue par les programmes lancés depuis le shell.

### 3.3 Variables spéciales

| Variable | Signification |
|---|---|
| `$0` | Nom du script |
| `$1`, `$2`, … | Arguments positionnels |
| `$@` | Tous les arguments (préserve les guillemets) |
| `$*` | Tous les arguments (collés) |
| `$#` | Nombre d'arguments |
| `$?` | Code de retour de la dernière commande |
| `$$` | PID du shell courant |
| `$!` | PID du dernier processus lancé en arrière-plan |
| `$_` | Dernier argument de la commande précédente |

### 3.4 Expansion (substitutions puissantes)

```bash
NOM="serveur-web-01"

echo ${NOM}                  # serveur-web-01
echo ${NOM:0:7}              # serveur (extraction sous-chaîne)
echo ${NOM%-*}               # serveur-web (retire à droite à partir du dernier -)
echo ${NOM##*-}              # 01 (ne garde que la partie après le dernier -)
echo ${NOM/web/api}          # serveur-api-01 (remplacement)
echo ${VAR_INEXISTANTE:-defaut}   # 'defaut' si vide
echo ${VAR:?message}         # erreur fatale si vide
echo ${#NOM}                 # longueur

# Expansion d'accolades
echo {1..5}                  # 1 2 3 4 5
echo backup-{2024,2025,2026}.tar.gz
echo serveur-{web,db,api}-{01..03}.local
```

---

## 4. Conditions

### 4.1 Codes de retour

Toute commande retourne un **code de sortie** (0 = succès, ≠0 = échec).

```bash
ls /etc && echo "OK"
ls /inexistant || echo "Échec"
ping -c1 google.com >/dev/null && echo "internet OK"
```

### 4.2 `if / elif / else`

```bash
if [[ $USER == "root" ]]; then
    echo "Vous êtes root"
elif [[ $USER == "alice" ]]; then
    echo "Bonjour Alice"
else
    echo "Utilisateur : $USER"
fi
```

### 4.3 Tests utiles (`[[ ... ]]`)

```bash
[[ -f /etc/passwd ]]         # fichier existe
[[ -d /var/log ]]            # dossier existe
[[ -x /usr/bin/python3 ]]    # exécutable
[[ -z "$VAR" ]]              # vide
[[ -n "$VAR" ]]              # non vide
[[ "$a" == "$b" ]]           # égalité (chaîne)
[[ "$a" != "$b" ]]
[[ "$a" =~ ^[0-9]+$ ]]       # regex (entier positif)
[[ $n -gt 10 ]]              # > (entier)
[[ $n -lt 10 ]]              # <
[[ -f a && -f b ]]           # ET
[[ -f a || -f b ]]           # OU
```

⚠️ Toujours utiliser **`[[ ... ]]`** (bash) et non `[ ... ]` (POSIX). Plus puissant et plus sûr.

### 4.4 `case`

```bash
case "$1" in
    start)   echo "Démarrage" ;;
    stop)    echo "Arrêt" ;;
    restart) echo "Redémarrage" ;;
    *)       echo "Usage: $0 {start|stop|restart}" ; exit 1 ;;
esac
```

---

## 5. Boucles

```bash
# for sur une liste
for srv in web-1 web-2 db-1; do
    echo "Vérification $srv"
done

# for sur une plage
for i in {1..10}; do echo $i; done

# for "à la C"
for ((i=0; i<10; i++)); do echo $i; done

# while
i=0
while [[ $i -lt 5 ]]; do
    echo $i
    ((i++))
done

# until (jusqu'à ce que vrai)
until curl -sf http://localhost:8080/health; do
    echo "Pas encore prêt…"
    sleep 2
done

# break / continue
for n in {1..10}; do
    [[ $n -eq 3 ]] && continue
    [[ $n -eq 7 ]] && break
    echo $n
done
```

### 5.1 Lire un fichier ligne par ligne

```bash
while IFS= read -r ligne; do
    echo "→ $ligne"
done < /etc/hosts
```

⚠️ `IFS=` empêche le découpage par espaces ; `-r` empêche l'interprétation des backslashes.
**Toujours** utiliser ce pattern pour lire un fichier.

---

## 6. Fonctions

```bash
audit_disque() {
    local seuil=${1:-80}
    local utilise
    utilise=$(df --output=pcent / | tail -1 | tr -d ' %')
    if (( utilise > seuil )); then
        echo "⚠️  Disque à ${utilise}% (>${seuil})"
        return 1
    fi
    echo "✅ Disque à ${utilise}%"
    return 0
}

if ! audit_disque 85; then
    /usr/local/bin/alert.sh critical "disque plein"
fi
```

**Règles :**

- Toujours utiliser **`local`** pour les variables internes.
- Retourner un **code** (`return 0` / `return 1`), pas une chaîne.
- Pour "retourner" une chaîne, l'imprimer sur stdout et capturer avec `$(...)`.

---

## 7. Tableaux

```bash
# Tableau classique
serveurs=(web-1 web-2 db-1)
echo "${serveurs[0]}"
echo "${serveurs[@]}"            # tous
echo "${#serveurs[@]}"           # longueur
serveurs+=("cache-1")            # append

for s in "${serveurs[@]}"; do
    echo "→ $s"
done

# Tableau associatif (bash 4+)
declare -A ports
ports[ssh]=22
ports[http]=80
ports[https]=443

echo "${ports[ssh]}"
for nom in "${!ports[@]}"; do
    echo "$nom -> ${ports[$nom]}"
done
```

---

## 8. Le mode strict (obligatoire en production)

```bash
#!/usr/bin/env bash
set -euo pipefail
IFS=$'\n\t'
```

| Option | Effet |
|---|---|
| `-e` | Quitte au premier échec |
| `-u` | Variable non définie = erreur (évite les fautes de frappe) |
| `-o pipefail` | Le pipe échoue si **n'importe quelle** étape échoue |
| `IFS=$'\n\t'` | Sépare sur newline/tab uniquement (sécurité contre les espaces) |

### 8.1 Désactiver localement

```bash
set +e
commande_qui_peut_echouer
code=$?
set -e
```

---

## 9. `trap` — nettoyage et signaux

```bash
TMP=$(mktemp -d)
trap 'rm -rf "$TMP"' EXIT             # toujours nettoyer
trap 'echo "Interrompu" ; exit 130' INT TERM
```

Signaux courants : `EXIT`, `INT` (Ctrl+C), `TERM` (`kill`), `HUP` (terminal fermé), `ERR`.

```bash
trap 'echo "Erreur ligne $LINENO"' ERR
```

---

## 10. Here-docs et here-strings

```bash
# Here-doc
cat > config.yaml <<EOF
server: $(hostname)
date: $(date -Is)
EOF

# Empêcher l'interpolation (variables littérales)
cat > script.py <<'EOF'
print("$USER n'est pas interpolé")
EOF

# Here-string
grep root <<< "$(cat /etc/passwd)"
```

---

## 11. Process substitution

```bash
# Comparer deux sorties sans fichier intermédiaire
diff <(ls /etc) <(ls /etc.bak)

# Lire deux fichiers en parallèle
paste <(cut -d: -f1 /etc/passwd) <(cut -d: -f3 /etc/passwd)
```

---

## 12. Manipulation de texte (la boîte à outils DevOps)

### 12.1 `grep` (avec extended regex)

```bash
grep -E "^(ERROR|WARN)" app.log              # début de ligne
grep -E "[0-9]{3}\.[0-9]{3}\.[0-9]{3}\.[0-9]{3}" log   # IPs
grep -v "DEBUG" app.log                       # exclure
grep -A 3 -B 2 "Exception" app.log            # 3 lignes après, 2 avant
grep -r "TODO" --include='*.py' .             # récursif filtré
grep -c "404" access.log                      # juste le nombre
```

### 12.2 `sed` — flux de transformation

```bash
sed 's/foo/bar/' file                # 1ère occurrence par ligne
sed 's/foo/bar/g' file               # toutes
sed -i 's/8080/9090/g' config.yaml   # en place
sed -n '10,20p' file                 # extraire lignes 10 à 20
sed '/^$/d' file                     # supprimer lignes vides
sed -i '/^#/d' file                  # supprimer commentaires
sed -E 's/[0-9]+/N/g' file           # regex étendue
```

### 12.3 `awk` — le couteau suisse

```bash
# Colonne 1
awk '{print $1}' access.log

# Filtrage + transformation
awk '$9 == 404 {print $7}' access.log         # URLs avec code 404

# Avec champs séparés par :
awk -F: '{print $1, $7}' /etc/passwd

# Agrégation
awk '{count[$1]++} END {for (ip in count) print count[ip], ip}' access.log | sort -rn

# Somme d'une colonne
awk '{sum+=$10} END {print sum}' access.log   # taille totale transférée

# Filtres conditionnels
awk '$10 > 1000000' access.log                # requêtes > 1 Mo

# Header puis lignes
awk 'NR==1 {next} {print}' file               # ignore la 1ère ligne
```

### 12.4 `jq` — JSON en CLI

```bash
echo '{"name":"alice","age":30}' | jq .
curl -s api.github.com/users/torvalds | jq '.login, .public_repos'
cat data.json | jq '.users[] | select(.active) | .email'
kubectl get pods -o json | jq '.items[].metadata.name'
```

### 12.5 Outils d'appoint

```bash
cut -d: -f1,3 /etc/passwd       # colonnes
tr ',' '\n' < liste.csv         # remplacer des caractères
tr 'a-z' 'A-Z'                  # majuscules
sort -u                          # tri unique
sort -k2 -t:                     # tri par 2e colonne, séparateur :
uniq -c                          # compter doublons (besoin de tri avant)
wc -l, wc -c, wc -w              # lignes, octets, mots
xargs                            # transforme stdin en arguments
```

---

## 13. Process & Performance Monitoring

### 13.1 Processus

```bash
ps aux                    # tous les processus, format BSD
ps -ef                    # format SysV
ps -ef --forest           # arbre
ps -p <PID>               # un seul
pgrep nginx               # PIDs par nom
pidof sshd                # idem
pstree                    # arbre complet
```

### 13.2 Vue dynamique

```bash
top                       # historique, peu pratique
htop                      # version moderne (à installer)
btop                      # encore plus moderne, graphique
glances                   # multi-écrans, web
atop                      # historique avec stockage
```

### 13.3 CPU et charge

```bash
uptime                    # load average sur 1/5/15 minutes
mpstat 1                  # CPU par cœur (paquet sysstat)
vmstat 1                  # vue système globale
nproc                     # nombre de cœurs
```

**Lecture du load average :**
- Sur un serveur **4 cœurs**, load = 4.0 → 100 % chargé, OK.
- Load > nombre de cœurs → file d'attente, latence.

### 13.4 Mémoire

```bash
free -h
cat /proc/meminfo
smem -t                   # par processus, mémoire partagée
```

### 13.5 Disque

```bash
df -h                     # espace par filesystem
du -sh /var/log/*         # taille par dossier
iostat -x 1               # IOPS et latence
iotop                     # top mais pour les I/O
ncdu /                    # explorateur de tailles
```

### 13.6 Réseau (côté stats)

```bash
ss -s                     # résumé sockets
ss -tulnp                 # ports en écoute (TCP/UDP)
iftop                     # top par connexion
nethogs                   # top par processus
vnstat                    # historique trafic
```

### 13.7 Tout-en-un

```bash
sar -u 1 10               # CPU
sar -r 1 10               # mémoire
sar -n DEV 1 10           # réseau
sar -d 1 10               # disque
sar -A                    # tout
```

---

## 14. Networking Tools

### 14.1 Configuration

```bash
ip a                      # interfaces (remplace ifconfig)
ip r                      # table de routage
ip neigh                  # cache ARP
nmcli                     # NetworkManager
hostname -I               # IP locale
```

### 14.2 Tester un hôte

```bash
ping -c4 google.com
ping -c4 -W1 8.8.8.8      # timeout 1s
mtr google.com            # ping + traceroute en temps réel
traceroute google.com
```

### 14.3 DNS

```bash
dig google.com
dig +short google.com A
dig +short google.com MX
dig @8.8.8.8 google.com   # serveur DNS spécifique
host google.com
nslookup google.com
```

### 14.4 Tester un port / une socket

```bash
nc -zv host 80                   # netcat : port ouvert ?
nc -lvp 1234                     # écouter sur 1234
nc -w 2 host 80 < requete.http   # envoyer un payload

# socat (plus puissant)
socat - TCP:host:80

# ss (remplace netstat)
ss -tulnp
ss -tan state established
```

### 14.5 HTTP

```bash
curl -I https://exemple.com               # juste les headers
curl -v https://exemple.com               # verbeux (TLS, headers)
curl -fsSL https://exemple.com            # silencieux, suit redirects, fail si erreur HTTP
curl -X POST -H 'Content-Type: application/json' -d '{"k":"v"}' https://api
curl -o file.tar.gz https://...           # sauvegarder

wget -c https://...                       # reprise possible
httpie / http (alternative ergonomique)
```

### 14.6 Capture de paquets

```bash
sudo tcpdump -i eth0 port 80              # filtrer par port
sudo tcpdump -i any -w capture.pcap       # écrire dans un fichier
sudo tcpdump -nn -A 'port 80'             # afficher payload
wireshark capture.pcap                    # interface graphique
```

---

## 15. Job control & parallélisme

```bash
long_script.sh &              # arrière-plan
jobs                          # voir les jobs
fg %1                         # ramener au premier plan
bg %1
disown -h %1                  # détacher (survit à la fermeture du terminal)
nohup long_script.sh &        # immune au SIGHUP

# Attendre tous les enfants
for srv in web-1 web-2 web-3; do
    deploy_to "$srv" &
done
wait
echo "Tous les déploiements terminés"

# xargs en parallèle
cat serveurs.txt | xargs -I{} -P 4 ssh {} 'uptime'

# GNU parallel (à installer)
parallel -j 8 ssh {} 'uptime' :::: serveurs.txt
```

---

## 16. Éditeurs en console

### 16.1 Nano (le plus simple)

```bash
nano fichier.txt
# Ctrl+O sauvegarder, Ctrl+X quitter, Ctrl+W rechercher
```

### 16.2 Vim — survie minimale (à enseigner ABSOLUMENT)

Vim est partout sur les serveurs. Savoir au moins **entrer, éditer, sauvegarder, sortir**.

| Touche | Action |
|---|---|
| `i` | Mode insertion (taper du texte) |
| `Esc` | Retour au mode normal |
| `:w` | Sauvegarder |
| `:q` | Quitter |
| `:wq` ou `ZZ` | Sauvegarder et quitter |
| `:q!` | Quitter sans sauvegarder |
| `dd` | Supprimer la ligne |
| `yy` | Copier la ligne |
| `p` | Coller |
| `u` | Undo |
| `Ctrl+r` | Redo |
| `/motif` | Rechercher |
| `n` / `N` | Occurrence suivante / précédente |
| `:%s/foo/bar/g` | Remplacer dans tout le fichier |
| `gg` / `G` | Début / fin de fichier |
| `:set number` | Afficher les numéros de ligne |

**Astuce pédagogique :** lancer `vimtutor` en cours — tutoriel interactif de 30 minutes inclus avec Vim.

### 16.3 Configuration minimale `~/.vimrc`

```vim
syntax on
set number
set relativenumber
set expandtab
set tabstop=4
set shiftwidth=4
set autoindent
set hlsearch
set incsearch
set ignorecase smartcase
set mouse=a
```

---

## 17. Patterns DevOps en Bash

### 17.1 Logger proprement

```bash
log() {
    local niveau="$1" ; shift
    printf '%s [%s] %s\n' "$(date '+%F %T')" "$niveau" "$*" >&2
}

log INFO "Démarrage"
log ERROR "Échec connexion à $HOST"
```

### 17.2 Script idempotent

```bash
# Crée le user seulement s'il n'existe pas
if ! id -u webops >/dev/null 2>&1; then
    sudo adduser --disabled-password --gecos "" webops
fi

# Crée le dossier s'il n'existe pas, sinon ne fait rien
sudo install -d -o webops -g webops -m 0755 /var/www/site
```

### 17.3 Retry avec back-off

```bash
retry() {
    local max=$1 ; shift
    local delai=1 i=0
    until "$@"; do
        ((i++))
        if (( i >= max )); then
            log ERROR "Abandon après $max essais : $*"
            return 1
        fi
        log WARN "Échec, retry dans ${delai}s ($i/$max)"
        sleep "$delai"
        delai=$(( delai * 2 ))
    done
}

retry 5 curl -fsS https://api.exemple.com/health
```

### 17.4 Verrou pour empêcher exécution concurrente

```bash
exec 9>/var/lock/monscript.lock
if ! flock -n 9; then
    echo "Déjà en cours, abandon."
    exit 1
fi
# … code protégé …
```

### 17.5 Usage / help auto-généré

```bash
usage() {
    cat <<EOF
Usage: $0 [OPTIONS] HOST
  -p PORT     Port (défaut: 22)
  -v          Verbeux
  -h          Cette aide
EOF
}

while getopts ":p:vh" opt; do
    case $opt in
        p) PORT=$OPTARG ;;
        v) VERBOSE=1 ;;
        h) usage ; exit 0 ;;
        :) echo "Option -$OPTARG nécessite un argument" >&2 ; exit 1 ;;
        \?) echo "Option inconnue : -$OPTARG" >&2 ; usage ; exit 1 ;;
    esac
done
shift $((OPTIND-1))
```

---

## 18. Débogage

### 18.1 `set -x` — trace d'exécution

```bash
set -x                # active
deploiement
set +x                # désactive
```

Ou directement : `bash -x mon_script.sh`.

### 18.2 `shellcheck` — linter Bash

```bash
sudo apt install shellcheck -y
shellcheck mon_script.sh
```

**À configurer dans la CI**. Détecte 80 % des bugs avant exécution.

### 18.3 Bonnes habitudes

```bash
#!/usr/bin/env bash
set -euo pipefail
IFS=$'\n\t'

# Toujours :
# - vérifier les arguments en haut
# - utiliser des fonctions
# - logger avec un niveau
# - nettoyer avec trap EXIT
# - retourner un code de sortie clair
```

---

## 19. Outils modernes (productivité 2026)

| Ancien | Moderne | Apport |
|---|---|---|
| `grep` | `ripgrep` (`rg`) | 10× plus rapide, respect de `.gitignore` |
| `find` | `fd` | Syntaxe humaine, parallèle |
| `cat` | `bat` | Syntaxe colorée, numéros |
| `ls` | `eza` (ex `exa`) | Couleurs, icônes, git |
| `top` | `btop` | Belles barres, graphiques |
| `man` | `tldr` | Exemples pratiques |
| `cd` | `zoxide` (`z`) | Mémorise les dossiers fréquents |
| `Ctrl+R` | `fzf` | Fuzzy finder universel |
| `time` | `hyperfine` | Benchmarks reproductibles |
| `df` | `duf` | Vue tabulaire colorée |
| `du` | `dust` | Visualisation des gros dossiers |

Installation rapide sur Ubuntu :

```bash
sudo apt install ripgrep fd-find bat fzf tldr -y
# Note : sur Ubuntu, fd s'appelle `fdfind`, bat s'appelle `batcat`
```

---

## 20. Récapitulatif — "Cheat sheet" DevOps

```bash
# Diagnostic rapide
uptime ; free -h ; df -h /

# Top 10 IPs dans un access log
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head

# Top 5 processus mémoire
ps aux --sort=-%mem | head -6

# Ports en écoute
ss -tulnp

# Test API et code HTTP
curl -o /dev/null -s -w "%{http_code}\n" https://exemple.com

# Suivi log en filtrant
tail -F /var/log/nginx/error.log | grep -v favicon

# Trouver gros fichiers
find /var -type f -size +100M -exec ls -lh {} \;
du -h /var | sort -rh | head

# Tuer tous les processus d'un user
sudo pkill -u alice

# Nombre de cœurs et RAM
nproc ; free -g | awk '/Mem:/ {print $2 " Go"}'
```

---

## 21. Évaluation — questions de contrôle

1. Différence entre `[[ ... ]]` et `[ ... ]` ?
2. Que signifie `set -euo pipefail` ?
3. Pourquoi `while IFS= read -r ligne; do … done < file` plutôt que `for ligne in $(cat file)` ?
4. Donnez 3 différences entre `sed` et `awk`.
5. Quelle commande remplace `netstat -tulnp` aujourd'hui ?
6. Comment exécuter 4 tâches en parallèle ?
7. À quoi sert `trap '… ' EXIT` ?
8. Comment écrire une fonction qui retourne une chaîne ?
9. Pourquoi `shellcheck` est-il indispensable ?
10. Quelle est la différence entre `nohup`, `&` et `disown` ?

---

## 22. Ressources

- 📘 *The Linux Command Line* — William Shotts ([gratuit](https://linuxcommand.org/tlcl.php))
- 📘 *Pro Bash Programming* — Chris F.A. Johnson (Apress)
- 📘 *bash Cookbook* — Carl Albing, JP Vossen (O'Reilly)
- 🌐 [Google Shell Style Guide](https://google.github.io/styleguide/shellguide.html)
- 🌐 [explainshell.com](https://explainshell.com) — décortique n'importe quelle commande
- 🌐 [tldr.sh](https://tldr.sh) — exemples concrets en ligne
- 🧪 [overthewire.org/wargames/bandit](https://overthewire.org/wargames/bandit/) — jeu pour pratiquer le shell

---

## 23. Étapes suivantes dans la roadmap

➡️ **TP1 (Débutant)** : forge ton premier "script DevOps" — diagnostic système complet.
➡️ **TP2 (Intermédiaire)** : analyse de logs Nginx + métriques exportées + alerting.
➡️ **TP3 (Avancé)** : framework d'orchestration multi-serveurs en pur Bash (SSH parallèle, retry, lock, logging).
➡️ **Leçon 4** : Git & Version Control (suite logique : versionner ses scripts).
