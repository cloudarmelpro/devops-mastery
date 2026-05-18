# Leçon 1 — Python pour le DevOps

> Module : Learn a Programming Language → Python
> Public : étudiants débutants en DevOps
> Durée estimée : **8 à 12 heures** (théorie + TP)
> Prérequis : aucun, sauf savoir utiliser un terminal

---

## 1. Objectifs pédagogiques

À la fin de cette leçon, l'étudiant sera capable de :

1. Comprendre **pourquoi** Python est le langage favori des ingénieurs DevOps.
2. Installer Python proprement (versions multiples, environnements virtuels).
3. Maîtriser la **syntaxe essentielle** : variables, types, conditions, boucles, fonctions.
4. Manipuler **fichiers, dossiers et variables d'environnement** depuis Python.
5. Exécuter des **commandes système** depuis un script Python (`subprocess`).
6. Lire/écrire des fichiers de configuration courants en DevOps : `JSON`, `YAML`, `.env`.
7. Appeler une **API HTTP** (REST) avec `requests`.
8. Écrire un **script CLI** avec `argparse`.
9. Mettre en place des **logs** et la **gestion d'erreurs**.
10. Réaliser un mini-projet d'automatisation (ex. : audit d'un serveur).

---

## 2. Pourquoi Python en DevOps ?

| Critère | Python | Bash | Go |
|---|---|---|---|
| Lisibilité | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐ |
| Bibliothèques (cloud, API, parsing) | ⭐⭐⭐⭐⭐ | ⭐ | ⭐⭐⭐ |
| Préinstallé sur Linux | ✅ | ✅ | ❌ |
| Outils écrits en Python | Ansible, AWS CLI, SaltStack, Azure CLI, Fabric | — | Terraform, Kubernetes, Docker |

**À retenir :** Python est l'outil de "colle" du DevOps. Quand un outil ne fait pas exactement ce qu'on veut, on écrit un script Python.

---

## 3. Installation et environnement

### 3.1 Vérifier la version

```bash
python3 --version
# Python 3.11.x ou supérieur recommandé
```

### 3.2 Gérer plusieurs versions avec `pyenv` (Linux/macOS)

```bash
curl https://pyenv.run | bash
pyenv install 3.12.0
pyenv global 3.12.0
```

### 3.3 Environnements virtuels (`venv`)

**Règle d'or :** jamais installer de paquets directement avec `sudo pip install`. Toujours dans un environnement virtuel.

```bash
python3 -m venv .venv
source .venv/bin/activate      # Linux / macOS
.\.venv\Scripts\Activate.ps1   # Windows PowerShell
pip install requests pyyaml
deactivate
```

### 3.4 Gestionnaire moderne : `uv` ou `poetry` (à découvrir plus tard)

---

## 4. Syntaxe essentielle (rappels rapides)

### 4.1 Variables et types

```python
nom = "serveur-prod-01"   # str
cpu_count = 4              # int
load_avg = 0.75            # float
is_online = True           # bool
tags = ["web", "prod"]     # list
config = {"port": 8080}    # dict
```

### 4.2 Conditions

```python
if load_avg > 0.9:
    print("ALERTE : charge élevée")
elif load_avg > 0.7:
    print("Attention")
else:
    print("OK")
```

### 4.3 Boucles

```python
for serveur in ["web-1", "web-2", "db-1"]:
    print(f"Ping {serveur}")

# Boucle while pour attendre qu'un service soit prêt
while not service_ready():
    time.sleep(5)
```

### 4.4 Fonctions

```python
def health_check(host: str, port: int = 80) -> bool:
    """Retourne True si le port répond."""
    # ... logique ici
    return True
```

---

## 5. Manipuler fichiers et OS

### 5.1 Lire / écrire un fichier

```python
from pathlib import Path

log = Path("/var/log/app.log")
contenu = log.read_text()

Path("rapport.txt").write_text("Audit terminé\n")
```

### 5.2 Parcourir un dossier

```python
for fichier in Path("/etc/nginx").rglob("*.conf"):
    print(fichier)
```

### 5.3 Variables d'environnement

```python
import os
db_password = os.environ.get("DB_PASSWORD", "valeur_par_defaut")
```

---

## 6. Exécuter des commandes système — `subprocess`

```python
import subprocess

result = subprocess.run(
    ["df", "-h"],
    capture_output=True,
    text=True,
    check=True,
)
print(result.stdout)
```

**⚠️ Sécurité :** ne jamais utiliser `shell=True` avec une entrée utilisateur (risque d'injection).

---

## 7. Fichiers de configuration

### 7.1 JSON (intégré)

```python
import json
with open("config.json") as f:
    config = json.load(f)
print(config["database"]["host"])
```

### 7.2 YAML (très utilisé : Ansible, Kubernetes, Docker Compose)

```bash
pip install pyyaml
```

```python
import yaml
with open("deployment.yaml") as f:
    manifest = yaml.safe_load(f)
print(manifest["spec"]["replicas"])
```

### 7.3 `.env` avec `python-dotenv`

```python
from dotenv import load_dotenv
load_dotenv()
api_key = os.environ["API_KEY"]
```

---

## 8. Appeler une API HTTP — `requests`

```python
import requests

r = requests.get(
    "https://api.github.com/repos/torvalds/linux",
    timeout=10,
)
r.raise_for_status()
data = r.json()
print(f"⭐ Étoiles : {data['stargazers_count']}")
```

---

## 9. Logging et gestion d'erreurs

```python
import logging

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(message)s",
)

try:
    deploy_app()
except subprocess.CalledProcessError as e:
    logging.error(f"Déploiement échoué : {e}")
    raise
else:
    logging.info("Déploiement réussi")
```

**Règle :** dans un script DevOps, **toujours** logger plutôt que `print()`. Les logs partent vers Loki, ELK, CloudWatch…

---

## 10. Créer un outil CLI — `argparse`

```python
import argparse

parser = argparse.ArgumentParser(description="Audit d'un serveur")
parser.add_argument("--host", required=True, help="Hôte à auditer")
parser.add_argument("--verbose", action="store_true")
args = parser.parse_args()

print(f"Audit de {args.host}")
```

Usage :

```bash
python audit.py --host serveur-prod-01 --verbose
```

---

## 11. TP final — Script d'audit système

**Objectif :** écrire un script `audit.py` qui :

1. Récupère le nom de la machine, l'OS, la version du kernel.
2. Liste l'usage disque (`df -h`).
3. Liste les 5 processus consommant le plus de mémoire.
4. Vérifie que les services `ssh` et `nginx` sont actifs.
5. Génère un rapport JSON `rapport-AAAAMMJJ.json`.
6. Envoie une alerte (print rouge) si un service est down.

**Bonus :**
- Ajouter une option CLI `--output json|text`.
- Logger dans `/var/log/audit.log`.
- Empaqueter avec un `requirements.txt`.

---

## 12. Évaluation — questions de contrôle

1. Quelle est la différence entre `python` et `python3` sur Ubuntu ?
2. Pourquoi utilise-t-on `venv` ?
3. Quel module standard permet de lancer une commande shell ?
4. Quel format de configuration est utilisé par Ansible et Kubernetes ?
5. Pourquoi `shell=True` est-il déconseillé dans `subprocess.run` ?
6. Donnez 3 outils DevOps écrits en Python.
7. Comment lire une variable d'environnement de façon sûre ?
8. Quelle bibliothèque pour appeler une API REST ?

---

## 13. Ressources

- 📘 *Automate the Boring Stuff with Python* — Al Sweigart (gratuit en ligne)
- 📘 *Python for DevOps* — Noah Gift, Kennedy Behrman (O'Reilly)
- 🌐 Documentation officielle : https://docs.python.org/3/
- 🧪 Exercices : https://www.hackerrank.com/domains/python

---

## 14. Étapes suivantes dans la roadmap

➡️ **Leçon 2** : Linux Ubuntu (voir `02-Operating-System/Linux-Ubuntu/Lecon-Linux-Ubuntu.md`)
➡️ Plus tard : combiner Python + Linux → scripts d'automatisation, puis Ansible.
