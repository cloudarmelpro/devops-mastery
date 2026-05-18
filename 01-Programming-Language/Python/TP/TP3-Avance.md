# TP3 — Python pour DevOps · Niveau Avancé

> **Titre :** `deployctl` — Outil CLI de déploiement packagé
> **Durée :** 8 à 12 heures (étalable sur 2 séances)
> **Prérequis :** TP1 et TP2 validés
> **Niveau :** ⭐⭐⭐ Avancé

---

## 🎯 Objectifs

- Concevoir un **outil CLI professionnel** (`pip install -e .`).
- Utiliser `click` (alternative moderne à `argparse`).
- Interagir avec l'**API GitHub** (REST).
- Exécuter des commandes sur un **serveur distant via SSH** (`paramiko` ou `fabric`).
- Mettre en place le **typage statique** (`type hints`, `mypy`).
- Écrire des **tests unitaires** (`pytest`) avec mocks.
- Gérer la **configuration** (fichier + variables d'environnement + secrets).
- **Packager** correctement (`pyproject.toml`).
- Mettre en place une **CI** GitHub Actions basique pour le projet.

---

## 📋 Contexte

Vous travaillez dans une équipe qui déploie une application sur plusieurs serveurs.
Vous devez écrire un outil CLI nommé **`deployctl`** qui automatise le cycle de déploiement.

### Fonctionnalités attendues

```bash
# Lister les releases GitHub du dépôt configuré
deployctl releases list

# Afficher les serveurs configurés
deployctl servers list

# Déployer une version sur un environnement
deployctl deploy --version v1.4.0 --env staging

# Vérifier la santé d'un environnement
deployctl healthcheck --env prod

# Rollback à la version précédente
deployctl rollback --env staging
```

---

## 🏗️ Architecture cible

```
deployctl/
├── pyproject.toml
├── README.md
├── .env.example
├── deployctl/
│   ├── __init__.py
│   ├── __main__.py          # point d'entrée
│   ├── cli.py               # commandes click
│   ├── config.py            # chargement config + env vars
│   ├── github_client.py     # API GitHub
│   ├── ssh_client.py        # exécution SSH
│   ├── deploy.py            # logique métier déploiement
│   ├── healthcheck.py
│   └── exceptions.py        # exceptions custom
├── tests/
│   ├── __init__.py
│   ├── test_github_client.py
│   ├── test_deploy.py
│   └── conftest.py
└── .github/
    └── workflows/
        └── ci.yml
```

---

## 🛠️ Mise en place

```bash
mkdir deployctl-project && cd deployctl-project
python3 -m venv .venv && source .venv/bin/activate
pip install click requests paramiko python-dotenv pyyaml pytest pytest-mock mypy ruff
```

### `pyproject.toml`

```toml
[project]
name = "deployctl"
version = "0.1.0"
description = "Outil CLI de déploiement multi-serveurs"
requires-python = ">=3.10"
dependencies = [
    "click>=8.1",
    "requests>=2.31",
    "paramiko>=3.4",
    "python-dotenv>=1.0",
    "pyyaml>=6.0",
]

[project.scripts]
deployctl = "deployctl.cli:cli"

[build-system]
requires = ["setuptools>=68"]
build-backend = "setuptools.build_meta"
```

Installation locale en mode édition :

```bash
pip install -e .
deployctl --help
```

---

## 🪜 Étapes (énoncé volontairement ouvert)

### Étape 1 — Configuration (1h)

Créer `config.py` qui :

- Charge `config.yaml` (serveurs, environnements).
- Charge `.env` pour les secrets (`GITHUB_TOKEN`, `SSH_KEY_PATH`).
- Valide la présence des champs obligatoires.
- Lève `ConfigError` si invalide.

`config.yaml` exemple :

```yaml
github:
  repo: org/mon-app

environnements:
  staging:
    serveurs: [staging-1.exemple.com]
    user: deploy
    app_dir: /opt/mon-app

  prod:
    serveurs: [prod-1.exemple.com, prod-2.exemple.com]
    user: deploy
    app_dir: /opt/mon-app
```

### Étape 2 — Client GitHub (1h30)

`github_client.py` doit fournir :

```python
class GitHubClient:
    def __init__(self, token: str, repo: str): ...
    def list_releases(self, limit: int = 10) -> list[Release]: ...
    def get_release(self, tag: str) -> Release: ...
```

- Utiliser **`requests.Session`** pour le pooling.
- Gérer le **rate-limiting** (header `X-RateLimit-Remaining`).
- Définir une **dataclass `Release`** : `tag`, `name`, `url`, `published_at`.
- Lever `GitHubError` en cas d'échec HTTP.

### Étape 3 — Client SSH (2h)

`ssh_client.py` :

```python
class SSHClient:
    def __init__(self, host: str, user: str, key_path: Path): ...
    def __enter__(self) -> "SSHClient": ...
    def __exit__(self, *args) -> None: ...
    def run(self, cmd: str, check: bool = True) -> CommandResult: ...
    def upload(self, local: Path, remote: str) -> None: ...
```

- Utiliser `paramiko.SSHClient`.
- Connexion par **clé privée** uniquement (jamais mot de passe).
- Implémenter comme **context manager**.
- Lever `SSHError` avec un message clair sur échec.

### Étape 4 — Logique de déploiement (2h)

`deploy.py` orchestrant :

1. Vérifier que la version GitHub existe.
2. Pour chaque serveur de l'environnement :
   - Connexion SSH.
   - Télécharger le tarball de la release dans `/tmp`.
   - Décompresser dans `<app_dir>/releases/<version>`.
   - Mettre à jour le **symlink** `<app_dir>/current` → `releases/<version>`.
   - Redémarrer le service systemd (`sudo systemctl restart mon-app`).
   - Healthcheck local (curl localhost:8080/health).
3. Si une étape échoue → **rollback** automatique.

### Étape 5 — Commandes CLI (1h)

`cli.py` avec `click` :

```python
import click

@click.group()
@click.option("--config", default="config.yaml")
@click.pass_context
def cli(ctx, config):
    ctx.obj = charger_config(config)

@cli.command()
@click.option("--version", required=True)
@click.option("--env", required=True, type=click.Choice(["staging", "prod"]))
@click.pass_obj
def deploy(cfg, version, env):
    """Déployer une version sur un environnement."""
    ...
```

### Étape 6 — Tests unitaires (1h30)

Couverture cible : **≥ 70 %**

Exemple `test_github_client.py` :

```python
def test_list_releases_succes(requests_mock):
    requests_mock.get(
        "https://api.github.com/repos/org/app/releases",
        json=[{"tag_name": "v1.0", "name": "First", "html_url": "...", "published_at": "..."}],
    )
    client = GitHubClient("fake-token", "org/app")
    releases = client.list_releases()
    assert len(releases) == 1
    assert releases[0].tag == "v1.0"
```

Pour SSH, utiliser un **mock** complet de `paramiko.SSHClient`.

### Étape 7 — Qualité du code (1h)

```bash
ruff check deployctl/        # linter
ruff format deployctl/       # formatter
mypy deployctl/              # type checking
pytest --cov=deployctl       # tests + couverture
```

Tous ces outils doivent **passer sans erreur**.

### Étape 8 — CI GitHub Actions (45 min)

`.github/workflows/ci.yml` :

```yaml
name: CI
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install -e ".[dev]"
      - run: ruff check .
      - run: mypy deployctl/
      - run: pytest --cov=deployctl --cov-fail-under=70
```

---

## ✅ Critères de réussite

| Critère | Points |
|---|---:|
| Projet `pip install -e .` fonctionnel | 2 |
| CLI `deployctl` avec sous-commandes click | 3 |
| Client GitHub avec gestion rate-limit | 3 |
| Client SSH context-manager, par clé | 3 |
| Déploiement avec symlink + rollback | 4 |
| Exceptions custom (`ConfigError`, `SSHError`, `GitHubError`) | 2 |
| Tests pytest, couverture ≥ 70 % | 4 |
| `mypy` passe sans erreur | 2 |
| `ruff check` passe sans erreur | 2 |
| CI GitHub Actions verte | 3 |
| README clair avec exemples | 2 |
| **Bonus** : notification Slack/Discord sur déploiement | +3 |
| **Bonus** : déploiement parallèle multi-serveurs avec `asyncio` + `asyncssh` | +5 |
| **Bonus** : intégration `sentry-sdk` pour le suivi d'erreurs | +2 |
| **Total** | **/30** |

---

## 🔒 Bonnes pratiques à respecter

1. **Aucun secret en clair** dans le code ou le YAML. `.env` est dans `.gitignore`.
2. Toujours **typer** les signatures de fonctions publiques.
3. Toujours définir un **`timeout`** sur les appels réseau.
4. **Logger** chaque étape avec un niveau approprié.
5. Le rollback **doit** être atomique : si le redémarrage échoue, revenir au symlink précédent.
6. Le code est **idempotent** : redéployer la même version ne casse rien.
7. Toutes les fonctions doivent **soit** retourner un résultat **soit** lever une exception — jamais d'`exit()` enfoui dans la logique métier.

---

## 🎓 Évaluation finale (oral + démo)

L'étudiant doit, en 20 minutes :

1. Faire une démonstration end-to-end (5 min).
2. Expliquer son architecture (5 min).
3. Justifier un choix technique (ex. pourquoi `click` vs `argparse`, pourquoi `paramiko` vs `fabric`) (5 min).
4. Répondre à une question piège (5 min) :
   - "Que se passe-t-il si le réseau coupe pendant un déploiement multi-serveurs ?"
   - "Comment empêcher deux déploiements concurrents sur le même environnement ?"
   - "Où mettre les secrets en production ?"

---

## 📚 Prochaine étape dans la roadmap

➡️ Cet outil sera la base pour aborder **Configuration Management** (Ansible) et **CI/CD** : on remplacera la logique de déploiement par un playbook Ansible déclenché par GitHub Actions.
