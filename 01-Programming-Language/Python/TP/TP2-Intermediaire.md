# TP2 — Python pour DevOps · Niveau Intermédiaire

> **Titre :** Audit multi-serveurs avec config YAML, CLI et logs
> **Durée :** 4 à 6 heures
> **Prérequis :** TP1 validé
> **Niveau :** ⭐⭐ Intermédiaire

---

## 🎯 Objectifs

- Lire un **fichier YAML** de configuration.
- Construire une **interface en ligne de commande** (`argparse`).
- Utiliser `subprocess` pour exécuter des **commandes système**.
- Mettre en place un **système de logs** propre (`logging`).
- Gérer les **erreurs** (`try/except`, codes de retour).
- Produire un **rapport JSON** consommable par un autre outil.
- Structurer son code en **modules** (plusieurs fichiers `.py`).

---

## 📋 Contexte

Vous êtes administrateur DevOps. Votre équipe gère **plusieurs serveurs**. On vous demande d'écrire un outil **`audit.py`** qui :

1. Lit une liste de serveurs depuis `serveurs.yaml`.
2. Exécute pour chacun des **vérifications** (ping, port, espace disque, services).
3. Génère un rapport `rapport-AAAAMMJJ-HHMM.json`.
4. Écrit un fichier `audit.log` avec horodatage.
5. Retourne un **code de sortie** : `0` si tout est OK, `1` si au moins un serveur a échoué.

Pour la simplicité de ce TP, **tout est local** (les commandes s'exécutent sur la machine de l'étudiant, on simule les "serveurs" par des entrées dans le YAML).

---

## 🛠️ Mise en place

```bash
mkdir tp2-audit && cd tp2-audit
python3 -m venv .venv && source .venv/bin/activate
pip install pyyaml
```

Structure attendue du projet :

```
tp2-audit/
├── .venv/
├── serveurs.yaml
├── audit.py            # point d'entrée
├── checks.py           # fonctions de vérification
├── rapport.py          # génération du rapport
└── requirements.txt
```

---

## 📂 Fichier `serveurs.yaml` (fourni)

```yaml
serveurs:
  - nom: web-1
    host: 127.0.0.1
    port: 80
    services: [ssh, cron]
    disque_min_go: 5

  - nom: web-2
    host: 127.0.0.1
    port: 443
    services: [ssh]
    disque_min_go: 5

  - nom: db-1
    host: 127.0.0.1
    port: 5432
    services: [cron]
    disque_min_go: 10
```

---

## 🪜 Étapes

### Étape 1 — CLI minimale (30 min)

`audit.py` doit accepter :

```bash
python audit.py --config serveurs.yaml --output rapport.json
python audit.py --config serveurs.yaml --verbose
python audit.py --help
```

```python
import argparse

def parser_args():
    p = argparse.ArgumentParser(description="Audit multi-serveurs")
    p.add_argument("--config", required=True, help="Fichier YAML")
    p.add_argument("--output", default="rapport.json")
    p.add_argument("--verbose", "-v", action="store_true")
    return p.parse_args()
```

### Étape 2 — Lire le YAML (20 min)

```python
import yaml

def charger_config(chemin: str) -> dict:
    with open(chemin) as f:
        return yaml.safe_load(f)
```

✅ **Vérification :** afficher chaque serveur sous forme `nom -> host:port`.

### Étape 3 — Logging (30 min)

Dans `audit.py` :

```python
import logging

def setup_logging(verbose: bool):
    niveau = logging.DEBUG if verbose else logging.INFO
    logging.basicConfig(
        level=niveau,
        format="%(asctime)s [%(levelname)s] %(message)s",
        handlers=[
            logging.FileHandler("audit.log"),
            logging.StreamHandler(),
        ],
    )
```

✅ **Vérification :** `tail -f audit.log` doit voir les messages.

### Étape 4 — Module `checks.py` (1h30)

Implémenter 4 fonctions :

```python
def ping(host: str) -> bool:
    """Retourne True si le host répond au ping."""

def port_ouvert(host: str, port: int, timeout: float = 2.0) -> bool:
    """Retourne True si le port TCP est ouvert."""

def disque_libre_go(point_de_montage: str = "/") -> float:
    """Retourne l'espace libre en Go."""

def service_actif(nom: str) -> bool:
    """Retourne True si le service systemd est actif (Linux uniquement)."""
```

**Indices :**

- `ping` : `subprocess.run(["ping", "-c", "1", "-W", "1", host])` (Linux/Mac) ou `["ping", "-n", "1", host]` (Windows).
- `port_ouvert` : module `socket`, méthode `connect_ex((host, port))`.
- `disque_libre_go` : `shutil.disk_usage(...)`.
- `service_actif` : `subprocess.run(["systemctl", "is-active", nom])`, retourne 0 si actif.

⚠️ **Toujours** capturer `subprocess.TimeoutExpired` et `FileNotFoundError`.

### Étape 5 — Boucle d'audit (1h)

Pour chaque serveur du YAML :

```python
resultat = {
    "nom": srv["nom"],
    "ping_ok": ping(srv["host"]),
    "port_ok": port_ouvert(srv["host"], srv["port"]),
    "disque_ok": disque_libre_go() > srv["disque_min_go"],
    "services": {s: service_actif(s) for s in srv["services"]},
}
```

Si **un seul** critère est `False`, le serveur est en échec → logger un `WARNING`.

### Étape 6 — Rapport JSON (30 min)

`rapport.py` :

```python
import json
from datetime import datetime
from pathlib import Path

def ecrire_rapport(resultats: list[dict], chemin: str) -> Path:
    rapport = {
        "date": datetime.now().isoformat(timespec="seconds"),
        "nb_serveurs": len(resultats),
        "nb_echecs": sum(1 for r in resultats if not r["ok"]),
        "resultats": resultats,
    }
    chemin = Path(chemin)
    chemin.write_text(json.dumps(rapport, indent=2, ensure_ascii=False))
    return chemin
```

### Étape 7 — Code de sortie (10 min)

```python
import sys
sys.exit(0 if tout_ok else 1)
```

✅ **Vérification :** `echo $?` après l'exécution doit retourner 0 ou 1.

---

## ✅ Critères de réussite

| Critère | Points |
|---|---:|
| CLI complète avec `--help` | 2 |
| YAML lu correctement | 2 |
| Logs dans `audit.log` ET dans la console | 2 |
| 4 fonctions de check fonctionnelles | 4 |
| Code structuré en plusieurs modules | 2 |
| Rapport JSON bien formé et lisible | 2 |
| Code de sortie correct (0 ou 1) | 2 |
| Gestion des erreurs (try/except, timeouts) | 2 |
| `requirements.txt` présent | 1 |
| Le code passe `python -m py_compile *.py` | 1 |
| **Bonus** : option `--parallel` (threads) | +3 |
| **Bonus** : envoi d'une alerte (webhook) si échec | +3 |
| **Total** | **/20** |

---

## 🎁 Pour aller plus loin

1. **Parallélisme** : auditer 50 serveurs en parallèle avec `concurrent.futures.ThreadPoolExecutor`.
2. **Historique** : garder les 30 derniers rapports, calculer un taux de disponibilité.
3. **Format Prometheus** : exposer les métriques au format `# HELP / # TYPE / metric value`.
4. **Tests unitaires** : tester `port_ouvert` avec un mock de socket.
5. **Conteneuriser** le script avec un `Dockerfile`.

---

## 🐛 Pièges classiques

- Ne **pas** mettre les credentials dans le YAML — utiliser une variable d'environnement.
- `subprocess.run` sans `timeout=` peut **bloquer** indéfiniment.
- `yaml.load()` est dangereux → toujours `yaml.safe_load()`.
- Sur Windows, beaucoup de commandes Linux n'existent pas → conditionner avec `platform.system()`.

---

## 📚 Pour la prochaine fois

➡️ **TP3 (Avancé)** : outil CLI packagé, intégration API GitHub, déploiement SSH, tests pytest.
