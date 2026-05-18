# Corrigé — TP2 Python (Intermédiaire)

> Projet complet `audit-multi-serveurs` : 4 fichiers + YAML + requirements.

---

## 📁 Structure finale

```
tp2-audit/
├── .venv/
├── serveurs.yaml
├── audit.py
├── checks.py
├── rapport.py
├── requirements.txt
└── audit.log               (généré)
```

## `requirements.txt`

```
pyyaml>=6.0
```

## `serveurs.yaml`

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

## `checks.py`

```python
"""Fonctions de vérification d'un serveur."""

import platform
import shutil
import socket
import subprocess


def ping(host: str, timeout: float = 1.0) -> bool:
    """Retourne True si l'hôte répond à un ping ICMP."""
    flag = "-n" if platform.system() == "Windows" else "-c"
    wait = "-w" if platform.system() == "Windows" else "-W"
    try:
        r = subprocess.run(
            ["ping", flag, "1", wait, str(int(timeout)), host],
            capture_output=True,
            timeout=timeout + 2,
        )
        return r.returncode == 0
    except (subprocess.TimeoutExpired, FileNotFoundError):
        return False


def port_ouvert(host: str, port: int, timeout: float = 2.0) -> bool:
    """Retourne True si le port TCP est ouvert."""
    try:
        with socket.create_connection((host, port), timeout=timeout):
            return True
    except (OSError, socket.timeout):
        return False


def disque_libre_go(point_de_montage: str = "/") -> float:
    """Retourne l'espace libre en Go (base 1024)."""
    if platform.system() == "Windows":
        point_de_montage = "C:\\"
    u = shutil.disk_usage(point_de_montage)
    return u.free / (1024 ** 3)


def service_actif(nom: str) -> bool:
    """Retourne True si le service systemd est actif (Linux uniquement)."""
    if platform.system() != "Linux":
        return False
    try:
        r = subprocess.run(
            ["systemctl", "is-active", nom],
            capture_output=True,
            text=True,
            timeout=5,
        )
        return r.stdout.strip() == "active"
    except (subprocess.TimeoutExpired, FileNotFoundError):
        return False
```

---

## `rapport.py`

```python
"""Génération du rapport JSON."""

import json
from datetime import datetime
from pathlib import Path


def ecrire_rapport(resultats: list[dict], chemin: str) -> Path:
    rapport = {
        "date": datetime.now().isoformat(timespec="seconds"),
        "nb_serveurs": len(resultats),
        "nb_echecs": sum(1 for r in resultats if not r["ok"]),
        "tout_ok": all(r["ok"] for r in resultats),
        "resultats": resultats,
    }
    p = Path(chemin)
    p.write_text(json.dumps(rapport, indent=2, ensure_ascii=False), encoding="utf-8")
    return p
```

---

## `audit.py`

```python
"""Point d'entrée — audit multi-serveurs."""

import argparse
import logging
import sys
from pathlib import Path

import yaml

from checks import disque_libre_go, ping, port_ouvert, service_actif
from rapport import ecrire_rapport


def parser_args() -> argparse.Namespace:
    p = argparse.ArgumentParser(description="Audit multi-serveurs")
    p.add_argument("--config", required=True, help="Fichier YAML de configuration")
    p.add_argument("--output", default="rapport.json", help="Chemin du rapport JSON")
    p.add_argument("--verbose", "-v", action="store_true")
    return p.parse_args()


def setup_logging(verbose: bool) -> None:
    niveau = logging.DEBUG if verbose else logging.INFO
    logging.basicConfig(
        level=niveau,
        format="%(asctime)s [%(levelname)s] %(message)s",
        handlers=[
            logging.FileHandler("audit.log", encoding="utf-8"),
            logging.StreamHandler(),
        ],
    )


def charger_config(chemin: str) -> dict:
    p = Path(chemin)
    if not p.exists():
        raise FileNotFoundError(f"Configuration introuvable : {chemin}")
    with p.open(encoding="utf-8") as f:
        return yaml.safe_load(f)


def auditer_serveur(srv: dict) -> dict:
    logging.debug(f"→ audit de {srv['nom']}")

    services_status = {s: service_actif(s) for s in srv.get("services", [])}

    resultat = {
        "nom": srv["nom"],
        "host": srv["host"],
        "ping_ok": ping(srv["host"]),
        "port_ok": port_ouvert(srv["host"], srv["port"]),
        "disque_libre_go": round(disque_libre_go(), 2),
        "disque_ok": disque_libre_go() >= srv["disque_min_go"],
        "services": services_status,
    }
    resultat["ok"] = (
        resultat["ping_ok"]
        and resultat["port_ok"]
        and resultat["disque_ok"]
        and all(services_status.values())
    )

    if resultat["ok"]:
        logging.info(f"✅ {srv['nom']} : OK")
    else:
        logging.warning(f"❌ {srv['nom']} : ÉCHEC ({resultat})")
    return resultat


def main() -> int:
    args = parser_args()
    setup_logging(args.verbose)

    try:
        cfg = charger_config(args.config)
    except (FileNotFoundError, yaml.YAMLError) as e:
        logging.error(f"Erreur de configuration : {e}")
        return 2

    resultats = [auditer_serveur(s) for s in cfg.get("serveurs", [])]
    chemin = ecrire_rapport(resultats, args.output)
    logging.info(f"📝 Rapport écrit dans {chemin}")

    tout_ok = all(r["ok"] for r in resultats)
    return 0 if tout_ok else 1


if __name__ == "__main__":
    sys.exit(main())
```

---

## 🎁 Bonus — Exécution parallèle (option `--parallel`)

Remplacer la liste comprehension par :

```python
from concurrent.futures import ThreadPoolExecutor

if args.parallel:
    with ThreadPoolExecutor(max_workers=10) as exe:
        resultats = list(exe.map(auditer_serveur, cfg["serveurs"]))
else:
    resultats = [auditer_serveur(s) for s in cfg["serveurs"]]
```

---

## 🎁 Bonus — Webhook d'alerte

Dans `audit.py`, après écriture du rapport :

```python
import os
import urllib.request
import json as _json

WEBHOOK = os.environ.get("ALERT_WEBHOOK")
if WEBHOOK and not tout_ok:
    payload = _json.dumps({
        "text": f"⚠️ {sum(1 for r in resultats if not r['ok'])} serveur(s) en échec",
    }).encode()
    req = urllib.request.Request(
        WEBHOOK,
        data=payload,
        headers={"Content-Type": "application/json"},
    )
    try:
        urllib.request.urlopen(req, timeout=5)
    except Exception as e:
        logging.warning(f"Échec envoi webhook : {e}")
```

---

## 🧪 Tests manuels (vérification rapide)

```bash
# Cas nominal
python audit.py --config serveurs.yaml --verbose
echo $?              # 0 ou 1

# Config absente
python audit.py --config inexistant.yaml ; echo $?   # 2

# JSON bien formé
python -c "import json; json.load(open('rapport.json'))"
```

---

## ✅ Points pédagogiques importants

1. **`yaml.safe_load`** (jamais `yaml.load`) : `load` exécute du code Python arbitraire.
2. **Timeouts partout** : aucune commande système ou socket sans `timeout=`.
3. **Codes de retour différenciés** : 0 = OK, 1 = audit échoué, 2 = erreur config.
4. **Séparation des responsabilités** : `checks.py` pur (testable), `rapport.py` I/O, `audit.py` orchestration.
5. **Logging à deux destinations** : fichier ET console grâce à `handlers=`.
6. **Type hints** : `list[dict]` rend le code auto-documenté.

---

## 🐛 Pièges typiques

| Symptôme | Cause | Correction |
|---|---|---|
| Script bloque indéfiniment | `subprocess.run` sans `timeout=` | Toujours ajouter `timeout=` |
| `KeyError: 'services'` | Champ absent dans YAML | `srv.get("services", [])` |
| Encoding cassé sous Windows | Pas d'`encoding="utf-8"` | Préciser systématiquement |
| Le rapport n'est pas écrit en cas d'erreur | `try` qui catch tout | Catcher exception **précise** uniquement |
