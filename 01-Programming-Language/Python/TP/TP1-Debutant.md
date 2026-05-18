# TP1 — Python pour DevOps · Niveau Débutant

> **Titre :** Mon premier script DevOps — Infos système
> **Durée :** 2 à 3 heures
> **Prérequis :** Python 3.10+ installé, terminal ouvert
> **Niveau :** ⭐ Débutant

---

## 🎯 Objectifs

À la fin de ce TP, l'étudiant saura :

- Créer et exécuter un script Python.
- Utiliser des modules de la **bibliothèque standard** (`os`, `platform`, `socket`, `shutil`).
- Manipuler **variables, conditions, boucles, fonctions**.
- Formater un affichage propre dans le terminal.
- Installer et utiliser un paquet externe avec `pip` (`psutil`).

---

## 🛠️ Mise en place (15 min)

```bash
mkdir tp1-info-systeme && cd tp1-info-systeme
python3 -m venv .venv
source .venv/bin/activate     # Linux/Mac
# .\.venv\Scripts\Activate.ps1   # Windows
pip install psutil
```

Créez un fichier `info_systeme.py`.

---

## 📋 Énoncé

Écrire un script `info_systeme.py` qui affiche un **rapport système** au format suivant :

```
============================================================
       RAPPORT SYSTÈME - mon-pc - 2026-05-18 14:30
============================================================
🖥️  Système          : Linux Ubuntu 24.04
🧬  Noyau            : 6.8.0-31-generic
🏗️  Architecture     : x86_64
🌐  Nom d'hôte       : mon-pc
📡  Adresse IP       : 192.168.1.42

🧠  CPU
    Modèle           : Intel Core i7-12700
    Cœurs physiques  : 8
    Cœurs logiques   : 16
    Utilisation      : 23.5 %

💾  Mémoire
    Totale           : 16.00 Go
    Utilisée         : 5.42 Go (33.9 %)
    Libre            : 10.58 Go

💿  Disque /
    Total            : 500.00 Go
    Utilisé          : 210.00 Go (42.0 %)
    Libre            : 290.00 Go

⏱️  Uptime           : 2 jours, 4 heures
============================================================
```

---

## 🪜 Étapes guidées

### Étape 1 — Imports et squelette (15 min)

```python
import platform
import socket
import shutil
import psutil
from datetime import datetime

def main():
    print("Hello, DevOps!")

if __name__ == "__main__":
    main()
```

Exécutez : `python info_systeme.py`. Vous devez voir `Hello, DevOps!`.

### Étape 2 — Récupérer les infos OS (20 min)

Documentation : `platform.system()`, `platform.release()`, `platform.machine()`, `socket.gethostname()`.

**À faire :**
1. Créer une fonction `infos_os()` qui retourne un dictionnaire avec : `systeme`, `noyau`, `architecture`, `hostname`.
2. L'appeler depuis `main()` et afficher.

### Étape 3 — Récupérer l'adresse IP (10 min)

Indice :
```python
ip = socket.gethostbyname(socket.gethostname())
```

### Étape 4 — Infos CPU (30 min)

Avec `psutil` :
- `psutil.cpu_count(logical=False)` → cœurs physiques
- `psutil.cpu_count(logical=True)` → cœurs logiques
- `psutil.cpu_percent(interval=1)` → utilisation en %

**À faire :** Créer une fonction `infos_cpu()`.

### Étape 5 — Infos mémoire (20 min)

```python
mem = psutil.virtual_memory()
# mem.total, mem.used, mem.available, mem.percent
```

Les valeurs sont en **octets**. Vous devez les convertir en **Go**.

➡️ Créez une fonction utilitaire :
```python
def to_go(octets: int) -> float:
    return octets / (1024 ** 3)
```

### Étape 6 — Infos disque (15 min)

```python
disk = shutil.disk_usage("/")     # Linux/Mac
# disk = shutil.disk_usage("C:\\")   # Windows
```

### Étape 7 — Uptime (15 min)

```python
import time
uptime_seconds = time.time() - psutil.boot_time()
```

**Défi :** convertir en `X jours, Y heures, Z minutes`.

### Étape 8 — Mise en forme finale (20 min)

Créer une fonction `afficher_rapport(data: dict)` qui produit la sortie demandée.
Utiliser des **f-strings** avec alignement :

```python
print(f"💾  Mémoire")
print(f"    Totale           : {to_go(mem.total):.2f} Go")
```

---

## ✅ Critères de réussite

| Critère | Points |
|---|---:|
| Le script s'exécute sans erreur | 2 |
| Toutes les infos demandées sont affichées | 4 |
| Le code est organisé en fonctions (≥ 4 fonctions) | 3 |
| Les unités sont lisibles (Go, %, jours) | 2 |
| Le script est commenté et `main()` utilise `if __name__ == "__main__"` | 2 |
| **Bonus** : affichage couleur (module `colorama`) | +2 |
| **Bonus** : option `--json` qui retourne du JSON | +3 |
| **Total** | **/15** |

---

## 🎁 Pour aller plus loin

1. Ajouter les **infos réseau** par interface (`psutil.net_if_addrs()`).
2. Ajouter le **top 5 des processus** consommant le plus de CPU.
3. Sauvegarder le rapport dans un fichier `rapport-<date>.txt`.
4. Comparer la sortie sous Linux vs Windows (différences à noter).

---

## 🐛 Erreurs fréquentes

| Erreur | Cause | Solution |
|---|---|---|
| `ModuleNotFoundError: psutil` | Pas dans le venv | `pip install psutil` (venv activé) |
| `[WinError ...] disk_usage` | Chemin Linux sous Windows | Utiliser `"C:\\"` au lieu de `"/"` |
| `socket.gaierror` | Hostname mal résolu | Hardcoder `"127.0.0.1"` en fallback |

---

## 📚 Pour la prochaine fois

➡️ **TP2 (Intermédiaire)** : auditer plusieurs serveurs distants avec un fichier YAML et une CLI.
