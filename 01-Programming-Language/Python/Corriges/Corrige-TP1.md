# Corrigé — TP1 Python (Débutant)

> Script complet `info_systeme.py` avec bonus JSON et couleurs.

---

## 📁 Arborescence finale attendue

```
tp1-info-systeme/
├── .venv/
├── info_systeme.py
├── requirements.txt
└── README.md
```

## `requirements.txt`

```
psutil>=5.9
colorama>=0.4
```

---

## `info_systeme.py` — version complète et commentée

```python
"""Rapport système — TP1 Python pour DevOps.

Usage :
    python info_systeme.py
    python info_systeme.py --json
    python info_systeme.py --json > rapport.json
"""

import argparse
import json
import platform
import socket
import shutil
import sys
import time
from datetime import datetime

import psutil
from colorama import Fore, Style, init as colorama_init

colorama_init(autoreset=True)


# ---------- Utilitaires ----------

def to_go(octets: int) -> float:
    """Convertit des octets en gigaoctets (base binaire)."""
    return octets / (1024 ** 3)


def format_uptime(seconds: float) -> str:
    """Formate l'uptime en 'X jours, Y heures, Z minutes'."""
    seconds = int(seconds)
    jours, reste = divmod(seconds, 86400)
    heures, reste = divmod(reste, 3600)
    minutes = reste // 60
    parties = []
    if jours:
        parties.append(f"{jours} jour{'s' if jours > 1 else ''}")
    if heures:
        parties.append(f"{heures} heure{'s' if heures > 1 else ''}")
    parties.append(f"{minutes} minute{'s' if minutes > 1 else ''}")
    return ", ".join(parties)


# ---------- Collecte ----------

def infos_os() -> dict:
    return {
        "systeme": f"{platform.system()} {platform.release()}",
        "noyau": platform.version(),
        "architecture": platform.machine(),
        "hostname": socket.gethostname(),
        "ip": _ip_locale(),
    }


def _ip_locale() -> str:
    """Retourne l'IP locale principale, ou 127.0.0.1 en cas d'échec."""
    try:
        with socket.socket(socket.AF_INET, socket.SOCK_DGRAM) as s:
            s.connect(("8.8.8.8", 80))
            return s.getsockname()[0]
    except OSError:
        return "127.0.0.1"


def infos_cpu() -> dict:
    return {
        "modele": platform.processor() or "Inconnu",
        "coeurs_physiques": psutil.cpu_count(logical=False),
        "coeurs_logiques": psutil.cpu_count(logical=True),
        "utilisation_pct": psutil.cpu_percent(interval=1),
    }


def infos_memoire() -> dict:
    m = psutil.virtual_memory()
    return {
        "total_go": round(to_go(m.total), 2),
        "utilise_go": round(to_go(m.used), 2),
        "libre_go": round(to_go(m.available), 2),
        "utilisation_pct": m.percent,
    }


def infos_disque(point: str = "/") -> dict:
    if platform.system() == "Windows":
        point = "C:\\"
    d = shutil.disk_usage(point)
    return {
        "point_de_montage": point,
        "total_go": round(to_go(d.total), 2),
        "utilise_go": round(to_go(d.used), 2),
        "libre_go": round(to_go(d.free), 2),
        "utilisation_pct": round(d.used / d.total * 100, 1),
    }


def infos_uptime() -> dict:
    secondes = time.time() - psutil.boot_time()
    return {
        "secondes": int(secondes),
        "lisible": format_uptime(secondes),
    }


def collecter() -> dict:
    return {
        "date": datetime.now().isoformat(timespec="seconds"),
        "os": infos_os(),
        "cpu": infos_cpu(),
        "memoire": infos_memoire(),
        "disque": infos_disque(),
        "uptime": infos_uptime(),
    }


# ---------- Affichage ----------

def afficher_texte(d: dict) -> None:
    sep = "=" * 60
    print(f"{Fore.CYAN}{sep}")
    print(f"{Fore.CYAN}       RAPPORT SYSTÈME - {d['os']['hostname']} - {d['date']}")
    print(f"{Fore.CYAN}{sep}{Style.RESET_ALL}")

    print(f"🖥️  Système          : {d['os']['systeme']}")
    print(f"🧬  Noyau            : {d['os']['noyau']}")
    print(f"🏗️  Architecture     : {d['os']['architecture']}")
    print(f"🌐  Nom d'hôte       : {d['os']['hostname']}")
    print(f"📡  Adresse IP       : {d['os']['ip']}\n")

    cpu = d["cpu"]
    print(f"🧠  CPU")
    print(f"    Modèle           : {cpu['modele']}")
    print(f"    Cœurs physiques  : {cpu['coeurs_physiques']}")
    print(f"    Cœurs logiques   : {cpu['coeurs_logiques']}")
    couleur = Fore.RED if cpu["utilisation_pct"] > 80 else Fore.GREEN
    print(f"    Utilisation      : {couleur}{cpu['utilisation_pct']} %{Style.RESET_ALL}\n")

    m = d["memoire"]
    print(f"💾  Mémoire")
    print(f"    Totale           : {m['total_go']} Go")
    print(f"    Utilisée         : {m['utilise_go']} Go ({m['utilisation_pct']} %)")
    print(f"    Libre            : {m['libre_go']} Go\n")

    dk = d["disque"]
    print(f"💿  Disque {dk['point_de_montage']}")
    print(f"    Total            : {dk['total_go']} Go")
    print(f"    Utilisé          : {dk['utilise_go']} Go ({dk['utilisation_pct']} %)")
    print(f"    Libre            : {dk['libre_go']} Go\n")

    print(f"⏱️  Uptime           : {d['uptime']['lisible']}")
    print(f"{Fore.CYAN}{sep}{Style.RESET_ALL}")


# ---------- Point d'entrée ----------

def main() -> int:
    parser = argparse.ArgumentParser(description="Rapport système DevOps")
    parser.add_argument("--json", action="store_true", help="Sortie JSON")
    args = parser.parse_args()

    data = collecter()
    if args.json:
        print(json.dumps(data, indent=2, ensure_ascii=False))
    else:
        afficher_texte(data)
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

---

## ✅ Points pédagogiques importants à expliquer aux étudiants

1. **`if __name__ == "__main__":`** : permet d'importer le module sans déclencher `main()`. À expliquer absolument.
2. **`psutil.cpu_percent(interval=1)`** : sans `interval`, retourne `0.0` au premier appel.
3. **Conversion d'unités** : préférer une fonction `to_go` réutilisable plutôt que dupliquer `x / (1024**3)`.
4. **Gestion multi-OS** : `shutil.disk_usage("/")` plante sous Windows → toujours conditionner.
5. **`with socket.socket(...) as s`** : context manager → libère la socket même en cas d'erreur.
6. **`sys.exit(main())`** : permet d'avoir un code de retour exploitable (0 = OK).

---

## 🐛 Erreurs fréquentes vues en TP

| Symptôme | Cause | Correction |
|---|---|---|
| Affichage sans interval pour CPU = 0 % | `cpu_percent()` sans `interval` | `psutil.cpu_percent(interval=1)` |
| `socket.gaierror` au démarrage | DNS local cassé | Fallback `127.0.0.1` (voir `_ip_locale`) |
| `KeyError` sous Windows | Disque `/` inexistant | Conditionner par `platform.system()` |
| Couleurs cassées sous Windows | Pas de `colorama_init()` | Toujours initialiser `colorama` |
