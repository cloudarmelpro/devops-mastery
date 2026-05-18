# Corrigé — TP3 Python (Avancé)

> Projet `deployctl` — outil CLI de déploiement packagé.
> Note : ce corrigé fournit les **fichiers clés** ; certaines options bonus sont laissées à l'étudiant.

---

## 📁 Structure finale

```
deployctl-project/
├── pyproject.toml
├── README.md
├── .env.example
├── config.yaml
├── deployctl/
│   ├── __init__.py
│   ├── __main__.py
│   ├── cli.py
│   ├── config.py
│   ├── github_client.py
│   ├── ssh_client.py
│   ├── deploy.py
│   ├── healthcheck.py
│   └── exceptions.py
├── tests/
│   ├── __init__.py
│   ├── conftest.py
│   ├── test_github_client.py
│   ├── test_config.py
│   └── test_deploy.py
└── .github/workflows/ci.yml
```

---

## `pyproject.toml`

```toml
[project]
name = "deployctl"
version = "0.1.0"
description = "Outil CLI de déploiement multi-serveurs"
authors = [{name = "DevOps Lab"}]
requires-python = ">=3.10"
dependencies = [
    "click>=8.1",
    "requests>=2.31",
    "paramiko>=3.4",
    "python-dotenv>=1.0",
    "pyyaml>=6.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0",
    "pytest-mock>=3.12",
    "pytest-cov>=5.0",
    "requests-mock>=1.11",
    "mypy>=1.10",
    "ruff>=0.5",
    "types-requests",
    "types-paramiko",
    "types-PyYAML",
]

[project.scripts]
deployctl = "deployctl.cli:cli"

[build-system]
requires = ["setuptools>=68"]
build-backend = "setuptools.build_meta"

[tool.ruff]
line-length = 100
target-version = "py310"

[tool.mypy]
strict = true
ignore_missing_imports = true

[tool.pytest.ini_options]
addopts = "-ra --cov=deployctl --cov-report=term-missing"
```

---

## `deployctl/exceptions.py`

```python
class DeployCtlError(Exception):
    """Exception de base."""

class ConfigError(DeployCtlError):
    """Configuration invalide."""

class GitHubError(DeployCtlError):
    """Erreur côté GitHub."""

class SSHError(DeployCtlError):
    """Erreur d'exécution SSH."""

class DeploymentError(DeployCtlError):
    """Échec d'un déploiement."""
```

---

## `deployctl/config.py`

```python
from dataclasses import dataclass
from pathlib import Path

import yaml
from dotenv import load_dotenv
import os

from .exceptions import ConfigError


@dataclass(frozen=True)
class GitHubCfg:
    repo: str
    token: str


@dataclass(frozen=True)
class EnvCfg:
    nom: str
    serveurs: list[str]
    user: str
    app_dir: str


@dataclass(frozen=True)
class Config:
    github: GitHubCfg
    environnements: dict[str, EnvCfg]
    ssh_key_path: Path


def charger_config(chemin: str | Path) -> Config:
    load_dotenv()
    p = Path(chemin)
    if not p.exists():
        raise ConfigError(f"Fichier introuvable : {p}")

    raw = yaml.safe_load(p.read_text(encoding="utf-8"))

    repo = raw.get("github", {}).get("repo")
    if not repo:
        raise ConfigError("github.repo manquant")

    token = os.environ.get("GITHUB_TOKEN")
    if not token:
        raise ConfigError("Variable d'environnement GITHUB_TOKEN absente")

    key_path = Path(os.environ.get("SSH_KEY_PATH", "~/.ssh/id_ed25519")).expanduser()
    if not key_path.exists():
        raise ConfigError(f"Clé SSH introuvable : {key_path}")

    envs: dict[str, EnvCfg] = {}
    for nom, data in (raw.get("environnements") or {}).items():
        envs[nom] = EnvCfg(
            nom=nom,
            serveurs=list(data["serveurs"]),
            user=data["user"],
            app_dir=data["app_dir"],
        )

    if not envs:
        raise ConfigError("Aucun environnement défini")

    return Config(
        github=GitHubCfg(repo=repo, token=token),
        environnements=envs,
        ssh_key_path=key_path,
    )
```

---

## `deployctl/github_client.py`

```python
from dataclasses import dataclass
from datetime import datetime

import requests

from .exceptions import GitHubError


@dataclass(frozen=True)
class Release:
    tag: str
    name: str
    url: str
    published_at: datetime


class GitHubClient:
    BASE = "https://api.github.com"

    def __init__(self, token: str, repo: str) -> None:
        self.repo = repo
        self.session = requests.Session()
        self.session.headers.update({
            "Authorization": f"Bearer {token}",
            "Accept": "application/vnd.github+json",
            "X-GitHub-Api-Version": "2022-11-28",
        })

    def _get(self, path: str, **params) -> requests.Response:
        url = f"{self.BASE}{path}"
        r = self.session.get(url, params=params, timeout=10)
        remaining = r.headers.get("X-RateLimit-Remaining")
        if remaining and int(remaining) < 5:
            # logger un warning serait préférable
            pass
        if r.status_code >= 400:
            raise GitHubError(f"{r.status_code} {url} : {r.text[:200]}")
        return r

    def list_releases(self, limit: int = 10) -> list[Release]:
        r = self._get(f"/repos/{self.repo}/releases", per_page=limit)
        return [self._to_release(x) for x in r.json()]

    def get_release(self, tag: str) -> Release:
        r = self._get(f"/repos/{self.repo}/releases/tags/{tag}")
        return self._to_release(r.json())

    @staticmethod
    def _to_release(d: dict) -> Release:
        return Release(
            tag=d["tag_name"],
            name=d.get("name") or d["tag_name"],
            url=d["html_url"],
            published_at=datetime.fromisoformat(d["published_at"].replace("Z", "+00:00")),
        )
```

---

## `deployctl/ssh_client.py`

```python
from dataclasses import dataclass
from pathlib import Path
from types import TracebackType

import paramiko

from .exceptions import SSHError


@dataclass
class CommandResult:
    code: int
    stdout: str
    stderr: str

    @property
    def ok(self) -> bool:
        return self.code == 0


class SSHClient:
    def __init__(self, host: str, user: str, key_path: Path, port: int = 22) -> None:
        self.host = host
        self.user = user
        self.key_path = Path(key_path).expanduser()
        self.port = port
        self._client: paramiko.SSHClient | None = None

    def __enter__(self) -> "SSHClient":
        try:
            client = paramiko.SSHClient()
            client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
            client.connect(
                hostname=self.host,
                port=self.port,
                username=self.user,
                key_filename=str(self.key_path),
                timeout=10,
                allow_agent=False,
                look_for_keys=False,
            )
            self._client = client
            return self
        except (paramiko.SSHException, OSError) as e:
            raise SSHError(f"Connexion impossible à {self.host}: {e}") from e

    def __exit__(
        self,
        exc_type: type[BaseException] | None,
        exc: BaseException | None,
        tb: TracebackType | None,
    ) -> None:
        if self._client is not None:
            self._client.close()
            self._client = None

    def run(self, cmd: str, check: bool = True, timeout: float = 60) -> CommandResult:
        if self._client is None:
            raise SSHError("Utiliser comme context manager")
        stdin, stdout, stderr = self._client.exec_command(cmd, timeout=timeout)
        code = stdout.channel.recv_exit_status()
        result = CommandResult(
            code=code,
            stdout=stdout.read().decode(),
            stderr=stderr.read().decode(),
        )
        if check and not result.ok:
            raise SSHError(f"`{cmd}` a échoué ({code}) : {result.stderr.strip()}")
        return result

    def upload(self, local: Path, remote: str) -> None:
        if self._client is None:
            raise SSHError("Utiliser comme context manager")
        sftp = self._client.open_sftp()
        try:
            sftp.put(str(local), remote)
        finally:
            sftp.close()
```

---

## `deployctl/deploy.py`

```python
import logging
from pathlib import Path

from .config import Config, EnvCfg
from .exceptions import DeploymentError
from .github_client import GitHubClient
from .ssh_client import SSHClient

log = logging.getLogger(__name__)


def deployer(cfg: Config, env_nom: str, version: str) -> None:
    env = cfg.environnements.get(env_nom)
    if env is None:
        raise DeploymentError(f"Environnement inconnu : {env_nom}")

    gh = GitHubClient(cfg.github.token, cfg.github.repo)
    release = gh.get_release(version)
    log.info(f"Release {release.tag} confirmée")

    tarball_url = f"https://github.com/{cfg.github.repo}/archive/refs/tags/{version}.tar.gz"

    for host in env.serveurs:
        log.info(f"→ déploiement {version} sur {host}")
        try:
            _deployer_sur_serveur(host, env, version, tarball_url, cfg.ssh_key_path)
        except Exception as e:
            log.error(f"Échec sur {host} : {e}. Rollback…")
            _rollback(host, env, cfg.ssh_key_path)
            raise DeploymentError(f"Déploiement {version} échoué sur {host}") from e

    log.info(f"✅ Déploiement {version} terminé sur {env_nom}")


def _deployer_sur_serveur(
    host: str,
    env: EnvCfg,
    version: str,
    tarball_url: str,
    key: Path,
) -> None:
    release_dir = f"{env.app_dir}/releases/{version}"
    current = f"{env.app_dir}/current"

    with SSHClient(host, env.user, key) as ssh:
        ssh.run(f"mkdir -p {release_dir}")
        ssh.run(f"curl -sSL {tarball_url} | tar -xz -C {release_dir} --strip-components=1")
        # Symlink atomique : nouveau symlink puis mv (mv -T = atomique sur Linux)
        ssh.run(f"ln -sfn {release_dir} {current}.new && mv -Tf {current}.new {current}")
        ssh.run("sudo systemctl restart mon-app")
        # Healthcheck simple
        ssh.run("curl -fsS http://127.0.0.1:8080/health > /dev/null", timeout=15)


def _rollback(host: str, env: EnvCfg, key: Path) -> None:
    try:
        with SSHClient(host, env.user, key) as ssh:
            # Détecter la release précédente
            r = ssh.run(
                f"ls -1t {env.app_dir}/releases | sed -n '2p'",
                check=False,
            )
            previous = r.stdout.strip()
            if previous:
                ssh.run(f"ln -sfn {env.app_dir}/releases/{previous} {env.app_dir}/current.new && "
                        f"mv -Tf {env.app_dir}/current.new {env.app_dir}/current")
                ssh.run("sudo systemctl restart mon-app", check=False)
    except Exception as e:
        log.error(f"Rollback impossible sur {host} : {e}")
```

---

## `deployctl/cli.py`

```python
import logging
import sys

import click

from .config import charger_config
from .deploy import deployer
from .exceptions import DeployCtlError
from .github_client import GitHubClient

logging.basicConfig(level=logging.INFO, format="%(asctime)s [%(levelname)s] %(message)s")


@click.group()
@click.option("--config", "config_path", default="config.yaml", show_default=True)
@click.pass_context
def cli(ctx: click.Context, config_path: str) -> None:
    """deployctl — outil CLI de déploiement."""
    try:
        ctx.obj = charger_config(config_path)
    except DeployCtlError as e:
        click.echo(f"❌ {e}", err=True)
        sys.exit(2)


@cli.group()
def releases() -> None:
    """Gestion des releases GitHub."""


@releases.command("list")
@click.option("--limit", default=10, show_default=True)
@click.pass_obj
def releases_list(cfg, limit: int) -> None:
    gh = GitHubClient(cfg.github.token, cfg.github.repo)
    for r in gh.list_releases(limit=limit):
        click.echo(f"{r.tag:15} {r.published_at.date()}  {r.name}")


@cli.group()
def servers() -> None:
    """Gestion des serveurs."""


@servers.command("list")
@click.pass_obj
def servers_list(cfg) -> None:
    for nom, env in cfg.environnements.items():
        click.echo(f"== {nom} ==")
        for s in env.serveurs:
            click.echo(f"  - {env.user}@{s}  ({env.app_dir})")


@cli.command()
@click.option("--version", required=True)
@click.option("--env", "env_nom", required=True)
@click.pass_obj
def deploy(cfg, version: str, env_nom: str) -> None:
    try:
        deployer(cfg, env_nom, version)
    except DeployCtlError as e:
        click.echo(f"❌ {e}", err=True)
        sys.exit(1)


if __name__ == "__main__":
    cli()
```

`deployctl/__main__.py` :

```python
from .cli import cli

if __name__ == "__main__":
    cli()
```

---

## `tests/conftest.py`

```python
import pytest


@pytest.fixture
def github_token() -> str:
    return "fake-token"


@pytest.fixture
def repo() -> str:
    return "org/app"
```

## `tests/test_github_client.py`

```python
from datetime import datetime, timezone

from deployctl.github_client import GitHubClient


def test_list_releases_ok(requests_mock, github_token, repo):
    requests_mock.get(
        f"https://api.github.com/repos/{repo}/releases",
        json=[
            {
                "tag_name": "v1.0.0",
                "name": "Premier",
                "html_url": "https://example.com/v1",
                "published_at": "2026-01-01T10:00:00Z",
            }
        ],
        headers={"X-RateLimit-Remaining": "100"},
    )
    client = GitHubClient(github_token, repo)
    releases = client.list_releases()

    assert len(releases) == 1
    assert releases[0].tag == "v1.0.0"
    assert releases[0].published_at == datetime(2026, 1, 1, 10, 0, tzinfo=timezone.utc)


def test_get_release_404_leve_erreur(requests_mock, github_token, repo):
    import pytest
    from deployctl.exceptions import GitHubError

    requests_mock.get(
        f"https://api.github.com/repos/{repo}/releases/tags/v9.9.9",
        status_code=404,
        text="Not Found",
    )
    client = GitHubClient(github_token, repo)
    with pytest.raises(GitHubError):
        client.get_release("v9.9.9")
```

## `tests/test_config.py`

```python
import os

import pytest

from deployctl.config import charger_config
from deployctl.exceptions import ConfigError


def test_config_ok(tmp_path, monkeypatch):
    key = tmp_path / "id_rsa"
    key.touch()
    monkeypatch.setenv("GITHUB_TOKEN", "abc")
    monkeypatch.setenv("SSH_KEY_PATH", str(key))

    cfg_file = tmp_path / "config.yaml"
    cfg_file.write_text("""
github:
  repo: org/app
environnements:
  staging:
    serveurs: [host1]
    user: deploy
    app_dir: /opt/app
""")
    cfg = charger_config(cfg_file)
    assert cfg.github.repo == "org/app"
    assert "staging" in cfg.environnements


def test_config_sans_token(tmp_path, monkeypatch):
    monkeypatch.delenv("GITHUB_TOKEN", raising=False)
    cfg_file = tmp_path / "config.yaml"
    cfg_file.write_text("github:\n  repo: org/app\n")
    with pytest.raises(ConfigError, match="GITHUB_TOKEN"):
        charger_config(cfg_file)
```

---

## `.github/workflows/ci.yml`

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
      - run: pytest --cov-fail-under=70
```

---

## ✅ Points pédagogiques importants

1. **Dataclasses immuables (`frozen=True`)** pour `Config`, `Release`, `CommandResult` — sécurise contre les mutations accidentelles.
2. **Context manager SSH** (`__enter__` / `__exit__`) — garantit la fermeture même en cas d'erreur.
3. **`paramiko.AutoAddPolicy`** est acceptable en formation mais à remplacer par `RejectPolicy` + known_hosts en prod.
4. **Symlink atomique** : `ln -sfn newdir current.new && mv -Tf current.new current` (sans cela, fenêtre de race).
5. **Rollback explicite** : ne doit jamais lever d'exception non gérée, sinon on perd l'erreur originale.
6. **Tests sans dépendre du réseau** : `requests_mock` et `monkeypatch` partout, jamais d'appel réel.
7. **Exceptions hiérarchisées** : `DeployCtlError` parent permet de catcher tout dans la CLI.
8. **`click.pass_obj`** : injection propre de la config dans toutes les sous-commandes.

---

## 🐛 Pièges classiques

- Oublier `allow_agent=False, look_for_keys=False` → paramiko tente l'agent ssh, parfois imprévisible.
- Symlink non atomique → entre `rm current` et `ln -s ... current`, le service tombe.
- Ne pas catcher dans `_rollback` → on perd l'erreur initiale.
- Token GitHub dans le YAML → fuite garantie dans Git. Toujours en variable d'environnement.
