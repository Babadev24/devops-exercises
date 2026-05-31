# Python — Exercices avancés (Débutant → Expert)

> 25 exercices progressifs en français, orientés DevOps / SRE.

---

## Niveau Débutant

### Exercice 1 — FizzBuzz

**Objectif** : afficher 1 à 100, mais "Fizz" si multiple de 3, "Buzz" si multiple de 5, "FizzBuzz" si les deux.

<details>
<summary>Solution</summary>

```python
for i in range(1, 101):
    out = ("Fizz" if i % 3 == 0 else "") + ("Buzz" if i % 5 == 0 else "")
    print(out or i)
```
</details>

---

### Exercice 2 — Compter les mots d'un fichier

**Objectif** : retourner les 5 mots les plus fréquents d'un fichier texte.

<details>
<summary>Solution</summary>

```python
from collections import Counter
from pathlib import Path
import re

text = Path("article.txt").read_text(encoding="utf-8").lower()
words = re.findall(r"\w+", text)
print(Counter(words).most_common(5))
```
</details>

---

### Exercice 3 — Renommer en masse

**Objectif** : renommer tous les `.jpeg` en `.jpg` dans un dossier (récursivement).

<details>
<summary>Solution</summary>

```python
from pathlib import Path
for p in Path(".").rglob("*.jpeg"):
    p.rename(p.with_suffix(".jpg"))
```
</details>

---

### Exercice 4 — JSON → CSV

**Objectif** : convertir une liste de dicts JSON en CSV.

<details>
<summary>Solution</summary>

```python
import json, csv
data = json.load(open("data.json"))
with open("data.csv", "w", newline="") as f:
    w = csv.DictWriter(f, fieldnames=data[0].keys())
    w.writeheader(); w.writerows(data)
```
</details>

---

### Exercice 5 — Décorateur de timing

**Objectif** : écrire `@timer` qui affiche la durée d'une fonction.

<details>
<summary>Solution</summary>

```python
import time, functools
def timer(f):
    @functools.wraps(f)
    def w(*a, **k):
        t0 = time.perf_counter(); r = f(*a, **k)
        print(f"{f.__name__}: {time.perf_counter()-t0:.3f}s"); return r
    return w
```
</details>

---

## Niveau Intermédiaire

### Exercice 6 — CLI argparse

**Objectif** : CLI `mycli` avec sous-commandes `init` et `deploy --env`.

<details>
<summary>Solution</summary>

Voir le guide complet section 18.1.
</details>

---

### Exercice 7 — Parser de logs Apache

**Objectif** : extraire le top 10 des IPs ayant fait le plus de requêtes.

Format : `IP - - [date] "GET /path" status size`.

<details>
<summary>Solution</summary>

```python
import re
from collections import Counter
from pathlib import Path

ip_re = re.compile(r"^(\d+\.\d+\.\d+\.\d+)")
c = Counter()
for line in Path("access.log").read_text().splitlines():
    m = ip_re.match(line)
    if m: c[m.group(1)] += 1
for ip, n in c.most_common(10):
    print(f"{n:>6}  {ip}")
```
</details>

---

### Exercice 8 — Client REST

**Objectif** : interroger l'API publique GitHub, afficher les 5 derniers commits d'un repo.

<details>
<summary>Solution</summary>

```python
import requests
r = requests.get("https://api.github.com/repos/python/cpython/commits",
                 params={"per_page": 5}, timeout=10)
r.raise_for_status()
for c in r.json():
    print(c["sha"][:7], c["commit"]["author"]["date"], c["commit"]["message"].splitlines()[0])
```
</details>

---

### Exercice 9 — Monitoring système

**Objectif** : afficher CPU, RAM, disque toutes les 2 secondes avec `psutil`.

<details>
<summary>Solution</summary>

```python
import psutil, time
while True:
    cpu = psutil.cpu_percent(interval=1)
    mem = psutil.virtual_memory().percent
    disk = psutil.disk_usage("/").percent
    print(f"CPU={cpu}%  MEM={mem}%  DISK={disk}%")
    time.sleep(1)
```
</details>

---

### Exercice 10 — Backup S3

**Objectif** : uploader tous les `.tar.gz` d'un dossier vers un bucket S3.

<details>
<summary>Solution</summary>

```python
import boto3
from pathlib import Path

s3 = boto3.client("s3")
for p in Path("./backups").glob("*.tar.gz"):
    s3.upload_file(str(p), "my-backups", p.name)
    print(f"uploaded {p.name}")
```
</details>

---

### Exercice 11 — Context manager perso

**Objectif** : implémenter un context manager `cd(path)` qui change de répertoire temporairement.

<details>
<summary>Solution</summary>

```python
import os
from contextlib import contextmanager

@contextmanager
def cd(path):
    old = os.getcwd()
    os.chdir(path)
    try: yield
    finally: os.chdir(old)

with cd("/tmp"):
    print(os.getcwd())
print(os.getcwd())
```
</details>

---

### Exercice 12 — Refactor en POO

**Objectif** : transformer un script procédural de gestion de compte bancaire en classe `Account` avec dépôt, retrait, transfert.

<details>
<summary>Solution</summary>

```python
from dataclasses import dataclass

@dataclass
class Account:
    owner: str
    balance: float = 0.0

    def deposit(self, n: float):
        if n <= 0: raise ValueError("montant > 0")
        self.balance += n

    def withdraw(self, n: float):
        if n > self.balance: raise ValueError("solde insuffisant")
        self.balance -= n

    def transfer(self, other: "Account", n: float):
        self.withdraw(n); other.deposit(n)
```
</details>

---

### Exercice 13 — Tests pytest

**Objectif** : écrire 5 tests pytest pour la classe `Account` ci-dessus, dont 1 paramétré et 1 avec `pytest.raises`.

<details>
<summary>Solution</summary>

```python
import pytest
from account import Account

@pytest.fixture
def acc(): return Account("Alice", 100)

def test_deposit(acc):
    acc.deposit(50); assert acc.balance == 150

def test_withdraw(acc):
    acc.withdraw(30); assert acc.balance == 70

def test_overdraw(acc):
    with pytest.raises(ValueError): acc.withdraw(1000)

@pytest.mark.parametrize("n", [-1, 0])
def test_invalid_deposit(acc, n):
    with pytest.raises(ValueError): acc.deposit(n)

def test_transfer(acc):
    b = Account("Bob"); acc.transfer(b, 40)
    assert (acc.balance, b.balance) == (60, 40)
```
</details>

---

## Niveau Avancé

### Exercice 14 — Décorateur retry paramétré

**Objectif** : `@retry(times=3, backoff=2)` qui retente une fonction avec délai exponentiel.

<details>
<summary>Solution</summary>

```python
import time, functools, random
def retry(times=3, backoff=1, exceptions=(Exception,)):
    def deco(f):
        @functools.wraps(f)
        def w(*a, **k):
            for i in range(times):
                try: return f(*a, **k)
                except exceptions as e:
                    if i == times - 1: raise
                    delay = backoff * (2 ** i) + random.random()
                    print(f"retry {i+1}/{times} dans {delay:.1f}s : {e}")
                    time.sleep(delay)
        return w
    return deco
```
</details>

---

### Exercice 15 — Générateur de pagination API

**Objectif** : générateur qui pagine une API REST `?page=N` jusqu'à épuisement, yield chaque item.

<details>
<summary>Solution</summary>

```python
import requests

def paginate(url):
    page = 1
    while True:
        r = requests.get(url, params={"page": page, "per_page": 100}, timeout=10)
        r.raise_for_status()
        items = r.json()
        if not items: return
        yield from items
        page += 1

for repo in paginate("https://api.github.com/users/python/repos"):
    print(repo["name"])
```
</details>

---

### Exercice 16 — Async crawler

**Objectif** : avec aiohttp, télécharger en parallèle 50 URLs et compter les codes HTTP.

<details>
<summary>Solution</summary>

```python
import asyncio, aiohttp
from collections import Counter

async def fetch(session, url):
    try:
        async with session.get(url, timeout=10) as r:
            return r.status
    except Exception:
        return -1

async def main(urls):
    async with aiohttp.ClientSession() as s:
        results = await asyncio.gather(*(fetch(s, u) for u in urls))
    print(Counter(results))

urls = ["https://example.com"] * 50
asyncio.run(main(urls))
```
</details>

---

### Exercice 17 — Multiprocessing CPU-bound

**Objectif** : calculer en parallèle la somme des carrés sur 4 segments de range(10^7).

<details>
<summary>Solution</summary>

```python
from concurrent.futures import ProcessPoolExecutor

def square_sum(start, stop):
    return sum(i*i for i in range(start, stop))

def main():
    N = 10**7
    chunks = [(i*N//4, (i+1)*N//4) for i in range(4)]
    with ProcessPoolExecutor() as exe:
        results = exe.map(lambda c: square_sum(*c), chunks)
    print(sum(results))

if __name__ == "__main__": main()
```

> Note : `lambda` non picklable → utiliser `functools.partial` ou `starmap`.
</details>

---

### Exercice 18 — Mock d'API en tests

**Objectif** : tester `fetch_users()` sans appeler l'API réelle, avec `unittest.mock`.

<details>
<summary>Solution</summary>

```python
from unittest.mock import patch, Mock

def fetch_users():
    import requests
    return requests.get("https://api/users", timeout=5).json()

def test_fetch():
    fake = Mock()
    fake.json.return_value = [{"id":1,"name":"a"}]
    with patch("requests.get", return_value=fake):
        assert fetch_users() == [{"id":1,"name":"a"}]
```
</details>

---

### Exercice 19 — CLI Click avec config YAML

**Objectif** : CLI qui charge un YAML, valide avec pydantic et lance une action.

<details>
<summary>Solution</summary>

```python
import click, yaml
from pydantic import BaseModel, Field

class Config(BaseModel):
    env: str
    replicas: int = Field(ge=1, le=100)

@click.command()
@click.argument("config", type=click.Path(exists=True))
def main(config):
    raw = yaml.safe_load(open(config))
    cfg = Config(**raw)
    click.echo(f"Déploiement de {cfg.replicas} replicas en {cfg.env}")

if __name__ == "__main__": main()
```
</details>

---

### Exercice 20 — Singleton via metaclass

**Objectif** : implémenter un singleton thread-safe avec une metaclass.

<details>
<summary>Solution</summary>

```python
import threading

class Singleton(type):
    _inst = {}
    _lock = threading.Lock()
    def __call__(cls, *a, **kw):
        with cls._lock:
            if cls not in cls._inst:
                cls._inst[cls] = super().__call__(*a, **kw)
        return cls._inst[cls]

class Config(metaclass=Singleton):
    def __init__(self): self.value = 42

a, b = Config(), Config()
assert a is b
```
</details>

---

## Niveau Expert

### Exercice 21 — Scan de ports asynchrone

**Objectif** : scanner les ports 1-1024 d'un host en async.

<details>
<summary>Solution</summary>

```python
import asyncio

async def check(host, port, sem):
    async with sem:
        try:
            _, w = await asyncio.wait_for(asyncio.open_connection(host, port), 0.5)
            w.close(); await w.wait_closed()
            return port
        except (asyncio.TimeoutError, OSError):
            return None

async def main(host):
    sem = asyncio.Semaphore(500)
    results = await asyncio.gather(*(check(host, p, sem) for p in range(1, 1025)))
    print(sorted(p for p in results if p))

asyncio.run(main("127.0.0.1"))
```
</details>

---

### Exercice 22 — Parser de gros fichier en streaming

**Objectif** : compter les lignes contenant "ERROR" dans un fichier de 10 Go sans le charger en mémoire.

<details>
<summary>Solution</summary>

```python
def count_errors(path):
    n = 0
    with open(path, "r", encoding="utf-8", errors="ignore") as f:
        for line in f:                  # itération lazy ligne par ligne
            if "ERROR" in line: n += 1
    return n

print(count_errors("big.log"))
```
</details>

---

### Exercice 23 — Profiler une fonction lente

**Objectif** : profiler la fonction la plus lente du programme et optimiser une boucle naïve avec set/dict.

<details>
<summary>Solution</summary>

```python
import cProfile, pstats
def slow():
    big = list(range(10_000_000))
    target = 9_999_999
    return target in big          # O(n)

def fast():
    big = set(range(10_000_000))
    return 9_999_999 in big       # O(1) après construction

cProfile.run("slow()", "p.out")
pstats.Stats("p.out").sort_stats("cumulative").print_stats(5)
```
</details>

---

### Exercice 24 — Métaprogrammation avec dataclass automatique

**Objectif** : décorateur `@auto_repr` qui ajoute un `__repr__` listant tous les attributs publics.

<details>
<summary>Solution</summary>

```python
def auto_repr(cls):
    def __repr__(self):
        attrs = ", ".join(f"{k}={v!r}" for k, v in vars(self).items() if not k.startswith("_"))
        return f"{cls.__name__}({attrs})"
    cls.__repr__ = __repr__
    return cls

@auto_repr
class User:
    def __init__(self, name, age): self.name, self.age = name, age

print(User("Alice", 30))
```
</details>

---

### Exercice 25 — Mini-orchestrateur

**Objectif** : écrire un script qui :
1. Liste les pods K8s d'un namespace.
2. Sélectionne ceux non `Running`.
3. Décrit l'événement le plus récent de chacun.
4. Envoie un résumé sur Slack.

<details>
<summary>Solution</summary>

```python
import os, requests
from kubernetes import client, config

config.load_kube_config()
v1 = client.CoreV1Api()
ns = "prod"
pods = v1.list_namespaced_pod(ns).items
bad = [p for p in pods if p.status.phase != "Running"]

lines = []
for p in bad:
    events = v1.list_namespaced_event(ns, field_selector=f"involvedObject.name={p.metadata.name}").items
    events.sort(key=lambda e: e.last_timestamp or "")
    last = events[-1].message if events else "n/a"
    lines.append(f"• {p.metadata.name} ({p.status.phase}) — {last}")

if lines:
    msg = f"⚠️ Pods en erreur dans {ns} :\n" + "\n".join(lines)
    requests.post(os.environ["SLACK_WEBHOOK"], json={"text": msg}, timeout=5)
    print(msg)
else:
    print("Tout va bien ✅")
```
</details>

---

## Pour aller plus loin

- Apprenez **asyncio** en profondeur (event loop, tasks, semaphores).
- Maîtrisez les **type hints avancés** (`Protocol`, `TypeVar`, `Generic`).
- Étudiez **Cython** ou **PyO3 (Rust)** pour les extensions performantes.
- Explorez **FastAPI** + **SQLAlchemy** + **Alembic** pour une stack web complète.
- Contribuez à un projet open-source DevOps en Python (ansible, kubernetes, salt).

