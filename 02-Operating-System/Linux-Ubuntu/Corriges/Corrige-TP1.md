# Corrigé — TP1 Ubuntu (Débutant)

> Toutes les commandes attendues + réponses aux questions.

---

## Partie 1 — Navigation

```bash
pwd                          # /home/etudiant
echo $HOME                   # /home/etudiant
cd ~
ls /etc
ls -l /etc
ls -la /etc
ls -lah /etc
cd /var/log
cd ..                        # remonte à /var
cd -                         # retour à /var/log
cd                           # retour à $HOME
```

### Réponse Q1

| Dossier | Rôle |
|---|---|
| `/etc` | Tous les fichiers de **configuration** système et services (texte uniquement) |
| `/var/log` | **Logs** système et applicatifs (syslog, auth.log, nginx, etc.) |
| `/usr/bin` | **Binaires** installés par le gestionnaire de paquets |
| `/tmp` | Fichiers **temporaires** ; vidé au reboot ; lisible par tous |
| `/home` | Dossiers personnels des utilisateurs |

---

## Partie 2 — Création et manipulation

```bash
mkdir -p ~/tp1/notes ~/tp1/scripts
touch ~/tp1/notes/jour1.txt ~/tp1/notes/jour2.txt ~/tp1/scripts/hello.sh
tree ~/tp1

echo "Bonjour DevOps" > ~/tp1/notes/jour1.txt
echo "Première journée OK" >> ~/tp1/notes/jour1.txt
cat ~/tp1/notes/jour1.txt
#   Bonjour DevOps
#   Première journée OK

cp ~/tp1/notes/jour1.txt ~/tp1/notes/jour1.bak
mv ~/tp1/notes/jour2.txt ~/tp1/notes/jour2-old.txt
rm ~/tp1/notes/jour1.bak
rm -r ~/tp1/scripts
```

---

## Partie 3 — Lire et chercher

```bash
mkdir -p ~/tp1/data && cd ~/tp1/data
wget https://raw.githubusercontent.com/elastic/examples/master/Common%20Data%20Formats/nginx_logs/nginx_logs.gz
gunzip nginx_logs.gz

head nginx_logs
tail -n 20 nginx_logs
less nginx_logs                   # q pour quitter
wc -l nginx_logs                  # → 51462 (exemple)

grep "404" nginx_logs
grep -c "404" nginx_logs          # → nombre de lignes contenant 404
grep -ic "GET" nginx_logs
grep -c " 200 " nginx_logs        # → nombre de réponses 200
```

---

## Partie 4 — Pipes et redirections

### Top 10 IPs

```bash
awk '{print $1}' nginx_logs | sort | uniq -c | sort -rn | head -10 > top-ips.txt
cat top-ips.txt
```

### Top 5 URLs

```bash
awk '{print $7}' nginx_logs | sort | uniq -c | sort -rn | head -5
```

### Décomposition pédagogique

```bash
awk '{print $1}' nginx_logs | head -3     # extraction
... | sort | head -3                       # tri (préalable obligatoire à uniq -c)
... | sort | uniq -c | head -3             # comptage
... | sort -rn | head -3                   # tri numérique inverse
```

---

## Partie 5 — Permissions

```bash
cat > ~/tp1/hello.sh <<'EOF'
#!/usr/bin/env bash
echo "Bonjour, $(whoami) !"
EOF

~/tp1/hello.sh                            # Permission denied
ls -l ~/tp1/hello.sh                      # -rw-r--r--
chmod u+x ~/tp1/hello.sh
~/tp1/hello.sh                            # Bonjour, etudiant !

chmod 755 ~/tp1/hello.sh
ls -l ~/tp1/hello.sh                      # -rwxr-xr-x

echo "mot de passe : 1234" > ~/tp1/secret.txt
chmod 600 ~/tp1/secret.txt
ls -l ~/tp1/secret.txt                    # -rw-------

# Test multi-utilisateur (optionnel)
sudo adduser bob
sudo su - bob
cat /home/etudiant/tp1/secret.txt         # Permission denied (attendu)
exit
```

### Tableau récapitulatif `chmod`

| Octal | Symbolique | Cas d'usage |
|---|---|---|
| 644 | rw-r--r-- | Fichier de données |
| 600 | rw------- | Clé privée, .env, secret |
| 700 | rwx------ | Dossier personnel sensible |
| 755 | rwxr-xr-x | Script ou binaire exécutable par tous |
| 750 | rwxr-x--- | Script exécutable par un groupe précis |

---

## Partie 6 — Processus

```bash
ps aux | head
ps aux | grep bash | grep -v grep
sudo apt install htop -y
htop                                       # F10 pour quitter

# Terminal 1
sleep 600 &
jobs

# Terminal 2
ps aux | grep sleep
kill <PID>                                 # SIGTERM
# si rien :
kill -9 <PID>                              # SIGKILL
pkill sleep
```

### Réponse Q6 — `kill -15` vs `kill -9`

| Signal | Numéro | Comportement |
|---|---|---|
| `SIGTERM` | 15 | **Demande poliment** au processus de s'arrêter. Il peut intercepter, sauvegarder, fermer ses connexions, puis quitter. C'est le signal par défaut de `kill`. |
| `SIGKILL` | 9 | **Tue immédiatement** au niveau du noyau. Le processus n'a aucune chance de réagir. Risque de **corruption** (fichiers ouverts, transactions en cours). |

**Règle :** toujours essayer `SIGTERM` d'abord, attendre 5-10s, puis `SIGKILL` en dernier recours.

---

## Partie 7 — APT

```bash
apt search ^tree$
sudo apt install tree htop -y

apt list --installed 2>/dev/null | grep python

sudo apt remove tree                       # garde les configs
sudo apt purge tree                        # supprime aussi /etc/...
sudo apt autoremove                        # nettoie les dépendances orphelines
```

**`remove` vs `purge` :**

- `remove` : désinstalle le binaire, garde la configuration dans `/etc`.
- `purge` : désinstalle ET supprime la configuration.

À utiliser quand on veut "vraiment" repartir de zéro sur un service.

---

## Partie 8 — Aide

```bash
man ls                                     # q pour quitter
ls --help
sudo apt install tldr -y
tldr tar
tldr find
```

---

## ✅ Bonus : `.bashrc` personnalisé

Ajouter à la fin de `~/.bashrc` :

```bash
# Alias DevOps
alias ll='ls -lah'
alias gs='git status'
alias ports='ss -tulnp'
alias myip='curl -s ifconfig.me'

# Prompt avec la branche git
parse_git_branch() {
    git branch 2>/dev/null | sed -e '/^[^*]/d' -e 's/* \(.*\)/ (\1)/'
}
export PS1='\u@\h:\w$(parse_git_branch)\$ '

# Historique partagé entre terminaux
shopt -s histappend
HISTSIZE=10000
HISTFILESIZE=20000
```

Recharger : `source ~/.bashrc`.

---

## 🎯 Synthèse pour l'enseignant

**Points à insister :**

1. La différence `>` vs `>>` est **la** source d'erreurs débutant.
2. Toujours faire **comprendre** chaque chiffre de `chmod 755` plutôt que mémoriser.
3. `rm -rf` doit déclencher chez l'étudiant un **réflexe de relecture** systématique.
4. Le combo `awk | sort | uniq -c | sort -rn` est à graver — il revient partout en DevOps.
5. `man <commande>` est la première source — avant Google et avant ChatGPT.
