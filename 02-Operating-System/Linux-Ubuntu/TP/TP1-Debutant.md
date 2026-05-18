# TP1 — Linux Ubuntu · Niveau Débutant

> **Titre :** Prise en main du shell et des commandes essentielles
> **Durée :** 3 à 4 heures
> **Prérequis :** Ubuntu 22.04+ installé (VM, WSL2, ou bare metal)
> **Niveau :** ⭐ Débutant

---

## 🎯 Objectifs

À la fin de ce TP, l'étudiant saura :

- Se repérer dans l'arborescence Linux.
- Manipuler **fichiers et dossiers** en ligne de commande.
- Lire des fichiers de différentes façons (`cat`, `less`, `head`, `tail`).
- Comprendre et modifier les **permissions** (rwx, chmod, chown).
- Lister, filtrer, tuer des **processus**.
- Installer un paquet avec **APT**.
- Trouver de l'aide (`man`, `--help`, `tldr`).

---

## 🛠️ Mise en place

1. Ouvrir un terminal Ubuntu (ou WSL2).
2. Vérifier la version :

```bash
lsb_release -a
```

3. Mettre à jour le système :

```bash
sudo apt update && sudo apt upgrade -y
```

---

## 🪜 Exercices guidés

### 🟢 Partie 1 — Navigation et arborescence (30 min)

**Exo 1.1.** Affichez le chemin courant.

```bash
pwd
```

**Exo 1.2.** Affichez votre dossier personnel et placez-vous-y.

```bash
echo $HOME
cd ~
```

**Exo 1.3.** Listez le contenu de `/etc` :
- normalement
- en format détaillé
- en incluant les fichiers cachés
- avec les tailles lisibles

```bash
ls /etc
ls -l /etc
ls -la /etc
ls -lah /etc
```

**Exo 1.4.** Naviguez :

```bash
cd /var/log     # se rendre dans /var/log
cd ..           # remonter d'un cran
cd -            # revenir au dossier précédent
cd              # retour au home
```

**Question Q1.** À quoi servent les dossiers suivants : `/etc`, `/var/log`, `/usr/bin`, `/tmp`, `/home` ?

---

### 🟢 Partie 2 — Création et manipulation de fichiers (45 min)

**Exo 2.1.** Créez l'arborescence suivante dans votre home :

```
tp1/
├── notes/
│   ├── jour1.txt
│   └── jour2.txt
└── scripts/
    └── hello.sh
```

```bash
mkdir -p ~/tp1/notes ~/tp1/scripts
touch ~/tp1/notes/jour1.txt ~/tp1/notes/jour2.txt
touch ~/tp1/scripts/hello.sh
tree ~/tp1            # installer tree si nécessaire : sudo apt install tree
```

**Exo 2.2.** Écrivez `Bonjour DevOps` dans `jour1.txt` **sans éditeur**.

```bash
echo "Bonjour DevOps" > ~/tp1/notes/jour1.txt
cat ~/tp1/notes/jour1.txt
```

**Exo 2.3.** Ajoutez une deuxième ligne **sans écraser**.

```bash
echo "Première journée OK" >> ~/tp1/notes/jour1.txt
```

❗ **Attention :** `>` écrase, `>>` ajoute.

**Exo 2.4.** Copiez `jour1.txt` en `jour1.bak` :

```bash
cp ~/tp1/notes/jour1.txt ~/tp1/notes/jour1.bak
```

**Exo 2.5.** Renommez `jour2.txt` en `jour2-old.txt` :

```bash
mv ~/tp1/notes/jour2.txt ~/tp1/notes/jour2-old.txt
```

**Exo 2.6.** Supprimez `jour1.bak` puis le dossier `scripts` entier.

```bash
rm ~/tp1/notes/jour1.bak
rm -r ~/tp1/scripts          # -r pour récursif
```

⚠️ `rm -rf /` détruit votre système. Toujours réfléchir avant `rm -rf`.

---

### 🟢 Partie 3 — Lire et chercher dans des fichiers (30 min)

**Exo 3.1.** Téléchargez un fichier log d'exemple :

```bash
mkdir -p ~/tp1/data && cd ~/tp1/data
wget https://raw.githubusercontent.com/elastic/examples/master/Common%20Data%20Formats/nginx_logs/nginx_logs.gz
gunzip nginx_logs.gz
ls -lh
```

**Exo 3.2.** Affichez :
- les 10 premières lignes
- les 20 dernières lignes
- le fichier page par page (quitter avec `q`)
- le nombre total de lignes

```bash
head nginx_logs
tail -n 20 nginx_logs
less nginx_logs
wc -l nginx_logs
```

**Exo 3.3.** Cherchez toutes les lignes contenant `404` :

```bash
grep "404" nginx_logs
grep -c "404" nginx_logs        # juste le nombre
grep -i "GET" nginx_logs | head # insensible à la casse, limité à 10
```

**Exo 3.4.** Combien de requêtes ont retourné un code `200` ?

```bash
grep -c " 200 " nginx_logs
```

---

### 🟢 Partie 4 — Pipes et redirections (30 min)

**Exo 4.1.** Top 10 des IPs les plus actives.

```bash
awk '{print $1}' nginx_logs | sort | uniq -c | sort -rn | head -10
```

**Décomposez** chaque étape :
- `awk '{print $1}'` → extrait la 1ère colonne (IP)
- `sort` → trie
- `uniq -c` → compte les doublons (besoin d'être trié avant !)
- `sort -rn` → tri numérique inverse
- `head -10` → 10 premiers

**Exo 4.2.** Sauvegardez ce top dans `top-ips.txt`.

**Exo 4.3.** Quels sont les 5 chemins (URLs) les plus demandés ?

Indice : la 7ème colonne contient l'URL.

---

### 🟢 Partie 5 — Permissions (45 min)

**Exo 5.1.** Créez un script `~/tp1/hello.sh` :

```bash
cat > ~/tp1/hello.sh <<'EOF'
#!/usr/bin/env bash
echo "Bonjour, $(whoami) !"
EOF
```

**Exo 5.2.** Essayez de l'exécuter :

```bash
~/tp1/hello.sh
```

❌ Erreur "Permission denied". Pourquoi ?

```bash
ls -l ~/tp1/hello.sh
```

**Exo 5.3.** Ajoutez le droit d'exécution au propriétaire :

```bash
chmod u+x ~/tp1/hello.sh
~/tp1/hello.sh
```

**Exo 5.4.** Comprendre les permissions octales.

| Notation | Signification |
|---|---|
| `chmod 755` | rwxr-xr-x — exécutable par tous |
| `chmod 644` | rw-r--r-- — fichier classique |
| `chmod 600` | rw------- — fichier privé (clés SSH) |
| `chmod 700` | rwx------ — dossier privé |

```bash
chmod 755 ~/tp1/hello.sh
ls -l ~/tp1/hello.sh
```

**Exo 5.5.** Créez un fichier "secret" lisible seulement par vous :

```bash
echo "mot de passe : 1234" > ~/tp1/secret.txt
chmod 600 ~/tp1/secret.txt
ls -l ~/tp1/secret.txt
```

**Exo 5.6.** (Optionnel) Créez un utilisateur `bob` et essayez d'accéder à votre fichier secret depuis son compte.

```bash
sudo adduser bob
sudo su - bob
cat /home/<votre-user>/tp1/secret.txt   # → Permission denied attendue
exit
```

---

### 🟢 Partie 6 — Processus (30 min)

**Exo 6.1.** Affichez tous les processus :

```bash
ps aux
ps aux | grep bash
```

**Exo 6.2.** Lancez `htop` (installez-le si nécessaire) :

```bash
sudo apt install htop -y
htop
# F10 ou q pour quitter
```

**Exo 6.3.** Lancez un processus qui "bloque" et tuez-le.

Dans **un terminal** :
```bash
sleep 600
```

Dans **un autre terminal** :
```bash
ps aux | grep sleep
kill <PID>            # SIGTERM (poli)
kill -9 <PID>         # SIGKILL (brutal)
pkill sleep           # tuer par nom
```

**Question Q6.** Différence entre `kill -15` et `kill -9` ?

---

### 🟢 Partie 7 — Paquets APT (20 min)

**Exo 7.1.** Cherchez et installez `tree` et `htop`.

```bash
apt search ^tree$
sudo apt install tree htop -y
```

**Exo 7.2.** Listez les paquets installés contenant "python" :

```bash
apt list --installed | grep python
```

**Exo 7.3.** Désinstallez `tree`, puis réinstallez-le. Quelle différence entre `apt remove` et `apt purge` ?

---

### 🟢 Partie 8 — Trouver de l'aide (15 min)

```bash
man ls
man grep
ls --help | head -30
```

Installez `tldr` (très utile) :

```bash
sudo apt install tldr -y
tldr tar
tldr find
```

---

## 📝 Livrables attendus

L'étudiant doit produire un fichier `rendu-tp1.md` contenant :

1. Les réponses aux questions **Q1**, **Q6** et autres.
2. Le contenu de `top-ips.txt` (top 10 IPs).
3. Le résultat de `ls -la ~/tp1`.
4. Le contenu du script `hello.sh` et la preuve qu'il s'exécute.
5. Les commandes utilisées pour la partie 5 (avec captures d'écran si possible).

---

## ✅ Critères de réussite

| Critère | Points |
|---|---:|
| Navigation maîtrisée (pwd, cd, ls) | 2 |
| Création/copie/déplacement/suppression OK | 2 |
| Utilisation correcte de `>`, `>>` et pipes | 3 |
| `grep`, `awk`, `sort`, `uniq` combinés | 4 |
| Permissions : chmod, chown maîtrisés | 3 |
| Processus : ps, kill, htop | 2 |
| APT : installation, recherche | 2 |
| Réponses aux questions | 2 |
| **Bonus** : alias et fichier `.bashrc` personnalisé | +2 |
| **Total** | **/20** |

---

## 🐛 Erreurs classiques

| Erreur | Cause | Solution |
|---|---|---|
| `Permission denied` à l'exécution | Pas de bit `x` | `chmod +x fichier` |
| `command not found` | Paquet pas installé | `apt search` puis `apt install` |
| `rm: cannot remove ...: Is a directory` | Manque `-r` | `rm -r dossier/` |
| `>` qui écrase un fichier important | Confondu avec `>>` | Toujours vérifier avant |

---

## 📚 Pour la prochaine fois

➡️ **TP2 (Intermédiaire)** : mettre en service un serveur web sécurisé (Nginx + SSH + UFW + systemd + cron).
