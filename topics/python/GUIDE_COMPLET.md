# Python — Le Guide Complet : de 0 à Hero

> Livre de référence en français pour maîtriser Python, des bases aux usages DevOps/SRE en production.

---

## Table des matières

1. [Introduction](#1-introduction)
2. [Installation et environnements](#2-installation-et-environnements)
3. [Syntaxe de base](#3-syntaxe-de-base)
4. [Structures de données](#4-structures-de-données)
5. [Contrôle de flux](#5-contrôle-de-flux)
6. [Fonctions](#6-fonctions)
7. [Programmation orientée objet](#7-poo)
8. [Modules et packages](#8-modules-et-packages)
9. [Gestion d'erreurs](#9-gestion-derreurs)
10. [I/O fichiers et formats](#10-io-fichiers-et-formats)
11. [Itérateurs et générateurs](#11-itérateurs-et-générateurs)
12. [Standard library essentielle](#12-standard-library)
13. [Programmation fonctionnelle](#13-programmation-fonctionnelle)
14. [Concurrence](#14-concurrence)
15. [Tests](#15-tests)
16. [Logging et debugging](#16-logging-et-debugging)
17. [Bibliothèques DevOps](#17-bibliothèques-devops)
18. [Scripting avancé (CLI)](#18-scripting-avancé)
19. [Web et APIs](#19-web-et-apis)
20. [Packaging et distribution](#20-packaging)
21. [Performance et profiling](#21-performance)
22. [Bonnes pratiques](#22-bonnes-pratiques)

---

## 1. Introduction

### 1.1 Histoire

- Conçu par **Guido van Rossum** en 1989-1991.
- Python 2.0 (2000), Python 3.0 (2008).
- **Python 2 EOL** depuis le 1er janvier 2020. **Toujours utiliser Python 3.x.**
- Versions actuelles : 3.10, 3.11, 3.12, 3.13.

### 1.2 Le Zen de Python

```python
import this
```

Quelques principes :
- Beautiful is better than ugly.
- Explicit is better than implicit.
- Simple is better than complex.
- Readability counts.
- There should be one — and preferably only one — obvious way to do it.

### 1.3 Pourquoi Python en DevOps ?

- Syntaxe lisible, courbe d'apprentissage douce.
- **Standard library** très complète (`os`, `subprocess`, `json`, `re`…).
- Écosystème : `boto3`, `kubernetes`, `paramiko`, `ansible`, `requests`, `pytest`.
- Multi-paradigme (impératif, OO, fonctionnel).
- Cross-platform.

### 1.4 Implémentations

- **CPython** (officielle, écrite en C).
- **PyPy** : JIT, plus rapide pour code pur Python.
- **MicroPython** : embarqué.
- **Jython**, **IronPython** : déprécié.

---

## 2. Installation et environnements

### 2.1 Installer Python

**Linux** : généralement présent ; sinon `apt install python3 python3-pip python3-venv`.

**macOS** : `brew install python@3.12`.

**Windows** : <https://python.org> ou Microsoft Store. Cocher "Add to PATH".

**Tous OS** : utiliser **pyenv** pour gérer plusieurs versions.

```bash
pyenv install 3.12.5
pyenv global 3.12.5
pyenv local 3.11.9        # version par projet
```

### 2.2 Environnements virtuels

**Toujours** créer un venv par projet pour isoler les dépendances.

```bash
python -m venv .venv
source .venv/bin/activate          # Linux/macOS
.\.venv\Scripts\Activate.ps1       # Windows PowerShell
pip install --upgrade pip
pip install requests pytest
pip freeze > requirements.txt
deactivate
```

### 2.3 Outils modernes

| Outil | Rôle |
|---|---|
| **Poetry** | Gestion deps + packaging + venv |
| **Pipenv** | Pipfile + venv |
| **uv** (Astral) | Ultra rapide, drop-in pip + venv |
| **conda / mamba** | Data science, gère aussi binaires non-Python |

Exemple Poetry :

```bash
pip install poetry
poetry new myproject
cd myproject
poetry add requests
poetry add --group dev pytest black ruff
poetry install
poetry run pytest
poetry shell
```

---

## 3. Syntaxe de base

### 3.1 Hello World

```python
# hello.py
print("Bonjour le monde")
name = input("Comment t'appelles-tu ? ")
print(f"Salut, {name} !")
```

```bash
python hello.py
```

### 3.2 Indentation

Python utilise l'**indentation** (4 espaces conventionnel) pour structurer le code. Pas d'accolades.

### 3.3 Variables et types

```python
age = 30                  # int
pi = 3.14159              # float
nom = "Alice"             # str
actif = True              # bool
rien = None               # NoneType
```

Python est **dynamiquement typé** (le type peut changer) mais **fortement typé** (pas de conversion implicite hasardeuse).

### 3.4 Type hints (Python 3.5+)

```python
def saluer(nom: str, age: int = 0) -> str:
    return f"{nom} a {age} ans"
```

À vérifier avec **mypy** :
```bash
pip install mypy
mypy mon_module.py
```

### 3.5 Opérateurs

```python
10 + 3      # 13
10 - 3      # 7
10 * 3      # 30
10 / 3      # 3.333... (float)
10 // 3     # 3       (division entière)
10 % 3      # 1       (modulo)
10 ** 3     # 1000    (puissance)

True and False
True or False
not True

5 == 5
5 != 4
5 < 10
'a' in 'banana'
```

### 3.6 f-strings (3.6+)

```python
name = "Alice"
age = 30
print(f"{name} a {age} ans")
print(f"Pi ≈ {3.14159:.2f}")
print(f"{name=}, {age=}")          # Alice=Alice, age=30 (3.8+)
```

---

## 4. Structures de données

### 4.1 Listes

```python
fruits = ["pomme", "banane", "cerise"]
fruits.append("datte")
fruits.insert(0, "abricot")
fruits.remove("banane")
fruits.pop()
fruits.sort()
fruits.reverse()
len(fruits)
fruits[0]
fruits[-1]
fruits[1:3]                # slicing
fruits[:]                  # copie
"pomme" in fruits
[x.upper() for x in fruits]                # comprehension
[x for x in fruits if "a" in x]            # avec filtre
```

### 4.2 Tuples (immuables)

```python
point = (3, 4)
x, y = point               # unpacking
single = (42,)             # virgule obligatoire
```

### 4.3 Sets

```python
s = {1, 2, 3, 2}           # {1, 2, 3}
s.add(4)
s & {2, 3, 5}              # intersection
s | {5, 6}                 # union
s - {1}                    # différence
s ^ {3, 4}                 # XOR
```

### 4.4 Dictionnaires

```python
user = {"name": "Alice", "age": 30}
user["name"]
user.get("email", "n/a")
user["email"] = "a@b.c"
del user["age"]
user.keys() ; user.values() ; user.items()

# Dict comprehension
squares = {x: x**2 for x in range(5)}

# Merge (3.9+)
combined = {**a, **b}        # ou a | b
```

### 4.5 Sélection rapide

| Besoin | Structure |
|---|---|
| Ordonné, mutable | `list` |
| Ordonné, immuable | `tuple` |
| Unique, non ordonné | `set` |
| Clé-valeur | `dict` |
| Pile (LIFO) | `list` (`append`/`pop`) |
| File (FIFO) | `collections.deque` |
| Tas | `heapq` |

---

## 5. Contrôle de flux

```python
# if / elif / else
if age >= 18:
    print("majeur")
elif age >= 13:
    print("ado")
else:
    print("enfant")

# Opérateur ternaire
status = "OK" if code == 200 else "KO"

# for
for fruit in fruits:
    print(fruit)

for i, fruit in enumerate(fruits, start=1):
    print(f"{i}. {fruit}")

for a, b in zip([1,2,3], ["a","b","c"]):
    print(a, b)

# while
n = 10
while n > 0:
    print(n)
    n -= 1
    if n == 5: break
    if n % 2 == 0: continue

# match (3.10+)
def status_label(code):
    match code:
        case 200 | 201:
            return "OK"
        case 404:
            return "Not Found"
        case code if 500 <= code < 600:
            return "Server error"
        case _:
            return "Other"
```

---

## 6. Fonctions

### 6.1 Définition

```python
def somme(a: int, b: int = 0) -> int:
    """Retourne la somme de a et b."""
    return a + b
```

### 6.2 Args / Kwargs

```python
def afficher(*args, **kwargs):
    for a in args: print(a)
    for k, v in kwargs.items(): print(f"{k}={v}")

afficher(1, 2, 3, nom="Alice", age=30)
```

### 6.3 Lambda

```python
carre = lambda x: x ** 2
sorted([3,1,2], key=lambda x: -x)
```

### 6.4 Scope (LEGB)

`Local → Enclosing → Global → Built-in`

```python
x = "global"
def outer():
    x = "enclosing"
    def inner():
        # x = "local"
        print(x)         # enclosing
    inner()
outer()
```

`global` et `nonlocal` pour modifier les variables des scopes supérieurs.

### 6.5 Closures

```python
def multiplicateur(n):
    def mult(x):
        return x * n
    return mult

x2 = multiplicateur(2)
x2(5)                    # 10
```

### 6.6 Décorateurs

```python
import time, functools

def timer(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        t0 = time.perf_counter()
        result = func(*args, **kwargs)
        print(f"{func.__name__} a pris {time.perf_counter()-t0:.3f}s")
        return result
    return wrapper

@timer
def heavy():
    sum(i**2 for i in range(10_000_000))
```

Décorateur paramétré :

```python
def retry(n=3):
    def deco(func):
        @functools.wraps(func)
        def wrapper(*a, **kw):
            for i in range(n):
                try: return func(*a, **kw)
                except Exception as e:
                    if i == n-1: raise
                    print(f"retry {i+1}: {e}")
        return wrapper
    return deco

@retry(n=5)
def fetch(): ...
```

### 6.7 Piège : argument mutable par défaut

```python
def bad(x, lst=[]):          # ⚠ shared between calls
    lst.append(x)
    return lst

def good(x, lst=None):
    lst = [] if lst is None else lst
    lst.append(x)
    return lst
```

---

## 7. POO

### 7.1 Classes

```python
class Person:
    """Une personne."""

    species = "human"        # variable de classe

    def __init__(self, name: str, age: int = 0):
        self.name = name
        self.age = age

    def greet(self) -> str:
        return f"Bonjour, je suis {self.name}."

    def __repr__(self):
        return f"Person(name={self.name!r}, age={self.age})"

p = Person("Alice", 30)
print(p.greet())
```

### 7.2 Héritage

```python
class Employee(Person):
    def __init__(self, name, age, salary):
        super().__init__(name, age)
        self.salary = salary

    def greet(self):                 # override
        return f"{super().greet()} Je gagne {self.salary}€."
```

### 7.3 Encapsulation

Convention :
- `_private` : usage interne (non strict).
- `__name_mangling` : préfixe `_ClassName__name`.

```python
class Account:
    def __init__(self, balance):
        self._balance = balance

    @property
    def balance(self) -> float:
        return self._balance

    @balance.setter
    def balance(self, value: float):
        if value < 0: raise ValueError("négatif interdit")
        self._balance = value
```

### 7.4 Dunder methods

```python
class Vector:
    def __init__(self, x, y): self.x, self.y = x, y
    def __add__(self, other): return Vector(self.x+other.x, self.y+other.y)
    def __eq__(self, other):  return (self.x, self.y) == (other.x, other.y)
    def __hash__(self):       return hash((self.x, self.y))
    def __len__(self):        return 2
    def __getitem__(self, i): return (self.x, self.y)[i]
    def __repr__(self):       return f"Vector({self.x}, {self.y})"
```

### 7.5 Dataclasses (3.7+)

```python
from dataclasses import dataclass, field

@dataclass(frozen=True, slots=True)
class Point:
    x: float
    y: float
    label: str = "?"
    tags: list[str] = field(default_factory=list)
```

Génère `__init__`, `__repr__`, `__eq__`, `__hash__` (si frozen).

### 7.6 ABC

```python
from abc import ABC, abstractmethod

class Storage(ABC):
    @abstractmethod
    def save(self, key, value): ...
    @abstractmethod
    def load(self, key): ...

class S3(Storage):
    def save(self, k, v): ...
    def load(self, k): ...
```

---

## 8. Modules et packages

### 8.1 Import

```python
import os
from os.path import join, exists
import os.path as op
from collections import defaultdict, Counter
```

### 8.2 Structure d'un package

```
myproject/
├── pyproject.toml
├── src/
│   └── mypkg/
│       ├── __init__.py
│       ├── cli.py
│       ├── core.py
│       └── utils/
│           ├── __init__.py
│           └── strings.py
└── tests/
    └── test_core.py
```

`pyproject.toml` minimal :

```toml
[project]
name = "mypkg"
version = "0.1.0"
requires-python = ">=3.10"
dependencies = ["requests>=2"]

[project.scripts]
mycli = "mypkg.cli:main"

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"
```

### 8.3 `__name__ == "__main__"`

```python
def main():
    ...

if __name__ == "__main__":
    main()
```

Permet d'exécuter le module en script tout en le rendant importable.

---

## 9. Gestion d'erreurs

```python
try:
    response = requests.get(url, timeout=5)
    response.raise_for_status()
    data = response.json()
except requests.Timeout:
    print("timeout")
except requests.HTTPError as e:
    print(f"HTTP {e.response.status_code}")
except requests.RequestException as e:
    print(f"erreur réseau : {e}")
else:
    process(data)        # uniquement si aucune exception
finally:
    cleanup()
```

### 9.1 Exception personnalisée

```python
class NotFoundError(Exception):
    """Ressource non trouvée."""
```

### 9.2 Context manager

```python
with open("file.txt", "r", encoding="utf-8") as f:
    data = f.read()
# fichier fermé automatiquement
```

Créer un context manager :

```python
from contextlib import contextmanager

@contextmanager
def chronometer(name):
    import time
    t0 = time.perf_counter()
    try:
        yield
    finally:
        print(f"{name}: {time.perf_counter()-t0:.3f}s")

with chronometer("calcul"):
    sum(range(10**6))
```

Ou en classe :

```python
class Timer:
    def __enter__(self):
        import time
        self.t0 = time.perf_counter(); return self
    def __exit__(self, exc_type, exc, tb):
        import time
        self.elapsed = time.perf_counter() - self.t0
        return False    # propage l'exception
```

---

## 10. I/O fichiers et formats

### 10.1 Lecture/écriture

```python
with open("data.txt", "r", encoding="utf-8") as f:
    contenu = f.read()
    # ou : for line in f:

with open("out.txt", "w", encoding="utf-8") as f:
    f.write("hello\n")

with open("log.txt", "a") as f:
    f.write("nouvelle ligne\n")
```

### 10.2 Pathlib (recommandé)

```python
from pathlib import Path

p = Path.home() / "data" / "file.txt"
p.parent.mkdir(parents=True, exist_ok=True)
p.write_text("contenu", encoding="utf-8")
print(p.read_text())
print(p.suffix, p.stem, p.name)
for f in Path(".").rglob("*.py"):
    print(f)
```

### 10.3 JSON

```python
import json

data = {"name": "Alice", "tags": [1, 2, 3]}
s = json.dumps(data, indent=2, ensure_ascii=False)
back = json.loads(s)

with open("data.json", "w") as f:
    json.dump(data, f, indent=2)
```

### 10.4 YAML

```python
import yaml                # pip install pyyaml

with open("config.yaml") as f:
    cfg = yaml.safe_load(f)
yaml.safe_dump(cfg, open("out.yaml", "w"))
```

### 10.5 CSV

```python
import csv

with open("data.csv", newline="", encoding="utf-8") as f:
    reader = csv.DictReader(f)
    for row in reader:
        print(row["name"], row["age"])

with open("out.csv", "w", newline="") as f:
    writer = csv.DictWriter(f, fieldnames=["a","b"])
    writer.writeheader()
    writer.writerow({"a": 1, "b": 2})
```

### 10.6 TOML (3.11+ built-in lecture)

```python
import tomllib              # 3.11+
with open("pyproject.toml", "rb") as f:
    cfg = tomllib.load(f)
```

---

## 11. Itérateurs et générateurs

### 11.1 Itérateur custom

```python
class Compteur:
    def __init__(self, n): self.n = n
    def __iter__(self):    self.i = 0; return self
    def __next__(self):
        if self.i >= self.n: raise StopIteration
        self.i += 1; return self.i
```

### 11.2 Générateur

```python
def fibo():
    a, b = 0, 1
    while True:
        yield a
        a, b = b, a + b

import itertools
for n in itertools.islice(fibo(), 10):
    print(n)
```

### 11.3 Generator expression

```python
total = sum(x**2 for x in range(10**6))     # lazy, mémoire O(1)
```

### 11.4 `itertools`

```python
from itertools import chain, product, permutations, combinations, groupby, accumulate, count, cycle, repeat

list(chain([1,2], [3,4]))                # [1,2,3,4]
list(product([0,1], repeat=3))           # cartésien
list(accumulate([1,2,3,4]))              # [1,3,6,10]
```

---

## 12. Standard library

Modules à connaître absolument :

| Module | Usage |
|---|---|
| `os`, `sys` | OS et runtime |
| `pathlib` | Chemins |
| `subprocess` | Lancer des commandes |
| `shutil` | Fichiers haut niveau (copy, rmtree) |
| `datetime`, `time` | Dates et heures |
| `re` | Regex |
| `json`, `csv`, `xml`, `tomllib` | Formats |
| `logging` | Logs |
| `argparse` | CLI |
| `collections` | `Counter`, `defaultdict`, `deque`, `OrderedDict`, `namedtuple` |
| `functools` | `reduce`, `lru_cache`, `partial`, `wraps` |
| `itertools` | Itérateurs |
| `typing` | Type hints |
| `dataclasses` | Classes data |
| `enum` | Énumérations |
| `concurrent.futures` | Thread/Process pools |
| `asyncio` | Async/await |
| `unittest` | Tests |
| `tempfile` | Fichiers temporaires |
| `random` | Aléatoire |
| `secrets` | Sécurité aléatoire (tokens) |
| `hashlib` | Hashs |
| `urllib`, `http` | Réseau |

### Exemples rapides

```python
import subprocess
res = subprocess.run(["ls", "-la"], capture_output=True, text=True, check=True)
print(res.stdout)

import collections
c = collections.Counter("abracadabra")    # Counter({'a': 5, 'b': 2, ...})

from functools import lru_cache
@lru_cache(maxsize=None)
def fib(n): return n if n < 2 else fib(n-1) + fib(n-2)

import datetime as dt
now = dt.datetime.now(dt.timezone.utc)
iso = now.isoformat()
```

---

## 13. Programmation fonctionnelle

```python
from functools import reduce
from operator import add

nums = [1, 2, 3, 4, 5]

list(map(lambda x: x*2, nums))                 # [2,4,6,8,10]
list(filter(lambda x: x % 2 == 0, nums))       # [2, 4]
reduce(add, nums)                              # 15

# Souvent plus pythonique :
[x*2 for x in nums]
[x for x in nums if x % 2 == 0]
sum(nums)
```

### `functools.partial`

```python
from functools import partial
mul = lambda x, y: x * y
double = partial(mul, 2)
double(5)         # 10
```

---

## 14. Concurrence

### 14.1 Le GIL

Le **Global Interpreter Lock** empêche l'exécution concurrente de **bytecode Python** sur plusieurs threads d'un même process. Conséquence :

- **Threads** : utiles pour I/O-bound (réseau, disque), pas pour CPU-bound.
- **Multiprocessing** : nécessaire pour CPU-bound (chaque process a son interpréteur et son GIL).
- **asyncio** : single-thread + boucle d'événements pour énormément d'I/O concurrents.

> Python 3.13 introduit **PEP 703** (GIL optionnel, expérimental).

### 14.2 Threading

```python
from concurrent.futures import ThreadPoolExecutor

def fetch(url):
    import requests
    return requests.get(url, timeout=5).status_code

urls = ["https://example.com"] * 50
with ThreadPoolExecutor(max_workers=10) as exe:
    for url, code in zip(urls, exe.map(fetch, urls)):
        print(url, code)
```

### 14.3 Multiprocessing

```python
from concurrent.futures import ProcessPoolExecutor

def cpu_heavy(n): return sum(i*i for i in range(n))

with ProcessPoolExecutor() as exe:
    results = list(exe.map(cpu_heavy, [10**6]*4))
```

### 14.4 asyncio

```python
import asyncio
import aiohttp

async def fetch(session, url):
    async with session.get(url, timeout=5) as r:
        return r.status

async def main():
    async with aiohttp.ClientSession() as session:
        tasks = [fetch(session, "https://example.com") for _ in range(100)]
        results = await asyncio.gather(*tasks)
        print(sum(1 for s in results if s == 200))

asyncio.run(main())
```

| Quand | Outil |
|---|---|
| I/O-bound, <100 tâches | ThreadPool |
| I/O-bound, milliers | asyncio |
| CPU-bound | ProcessPool ou C extension |

---

## 15. Tests

### 15.1 pytest

```bash
pip install pytest pytest-cov
```

```python
# test_math.py
def add(a, b): return a + b

def test_add():
    assert add(2, 3) == 5

import pytest

@pytest.mark.parametrize("a,b,expected", [(1,1,2),(0,0,0),(-1,1,0)])
def test_param(a, b, expected):
    assert add(a, b) == expected

def test_raises():
    with pytest.raises(ZeroDivisionError):
        1 / 0
```

```bash
pytest -v
pytest --cov=mypkg --cov-report=html
```

### 15.2 Fixtures

```python
@pytest.fixture
def temp_dir(tmp_path):
    (tmp_path / "data.txt").write_text("hello")
    return tmp_path

def test_with_dir(temp_dir):
    assert (temp_dir / "data.txt").read_text() == "hello"
```

### 15.3 Mocking

```python
from unittest.mock import patch, MagicMock

def fetch_users():
    import requests
    return requests.get("https://api/users").json()

def test_fetch_users():
    with patch("requests.get") as mock_get:
        mock_get.return_value = MagicMock(json=lambda: [{"id":1}])
        assert fetch_users() == [{"id":1}]
```

---

## 16. Logging et debugging

### 16.1 Logging

```python
import logging

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(name)s - %(message)s",
)
log = logging.getLogger(__name__)

log.debug("détail")
log.info("OK")
log.warning("attention")
log.error("erreur")
log.exception("avec stacktrace")        # à utiliser dans except
```

Niveaux : DEBUG < INFO < WARNING < ERROR < CRITICAL.

**Logging structuré** : `structlog` ou JSON formatter pour Loki/ELK.

```python
import json, logging
class JsonFormatter(logging.Formatter):
    def format(self, record):
        return json.dumps({
            "ts": self.formatTime(record),
            "level": record.levelname,
            "msg": record.getMessage(),
            "logger": record.name,
        })
```

### 16.2 Debugger

```python
import pdb; pdb.set_trace()       # ancien
breakpoint()                      # 3.7+
```

Commandes pdb : `n` (next), `s` (step), `c` (continue), `l` (list), `p var`, `pp`, `bt`, `q`.

IDE : VS Code, PyCharm.

---

## 17. Bibliothèques DevOps

### 17.1 requests (HTTP)

```python
import requests

r = requests.get("https://api.github.com/users/octocat", timeout=5)
r.raise_for_status()
print(r.json())

session = requests.Session()
session.headers.update({"Authorization": f"Bearer {token}"})
r = session.post(url, json={"a":1})
```

### 17.2 boto3 (AWS)

```python
import boto3

s3 = boto3.client("s3", region_name="eu-west-1")
s3.upload_file("local.txt", "my-bucket", "remote.txt")

for obj in s3.list_objects_v2(Bucket="my-bucket").get("Contents", []):
    print(obj["Key"], obj["Size"])

ec2 = boto3.resource("ec2")
for inst in ec2.instances.filter(Filters=[{"Name":"tag:Env","Values":["prod"]}]):
    print(inst.id, inst.state)
```

### 17.3 kubernetes

```python
from kubernetes import client, config

config.load_kube_config()
v1 = client.CoreV1Api()

for pod in v1.list_pod_for_all_namespaces().items:
    print(pod.metadata.namespace, pod.metadata.name, pod.status.phase)
```

### 17.4 docker

```python
import docker
client = docker.from_env()
for c in client.containers.list():
    print(c.name, c.status)
client.images.pull("nginx:alpine")
client.containers.run("nginx:alpine", detach=True, name="web", ports={"80/tcp": 8080})
```

### 17.5 paramiko (SSH)

```python
import paramiko
ssh = paramiko.SSHClient()
ssh.set_missing_host_key_policy(paramiko.AutoAddPolicy())
ssh.connect("host", username="user", key_filename="~/.ssh/id_rsa")
stdin, stdout, stderr = ssh.exec_command("uptime")
print(stdout.read().decode())
```

### 17.6 psutil (système)

```python
import psutil
print(psutil.cpu_percent(interval=1))
print(psutil.virtual_memory().percent)
for p in psutil.process_iter(["pid","name","cpu_percent"]):
    print(p.info)
```

---

## 18. Scripting avancé

### 18.1 argparse

```python
import argparse

parser = argparse.ArgumentParser(description="Mon outil")
parser.add_argument("input", help="fichier d'entrée")
parser.add_argument("-o", "--output", default="out.txt")
parser.add_argument("--verbose", "-v", action="count", default=0)
parser.add_argument("--mode", choices=["fast","safe"], default="safe")
parser.add_argument("--limit", type=int, default=100)

sub = parser.add_subparsers(dest="cmd")
sub.add_parser("init")
deploy = sub.add_parser("deploy")
deploy.add_argument("--env")

args = parser.parse_args()
print(args)
```

### 18.2 click

```python
import click

@click.group()
@click.option("--debug/--no-debug", default=False)
@click.pass_context
def cli(ctx, debug):
    ctx.ensure_object(dict); ctx.obj["DEBUG"] = debug

@cli.command()
@click.option("--env", type=click.Choice(["dev","prod"]), required=True)
@click.argument("version")
def deploy(env, version):
    """Déploie VERSION sur ENV."""
    click.echo(f"Déploiement de {version} sur {env}")

if __name__ == "__main__": cli()
```

### 18.3 typer (basé sur click + types)

```python
import typer
app = typer.Typer()

@app.command()
def deploy(version: str, env: str = "dev", dry: bool = False):
    typer.echo(f"deploy {version} → {env} (dry={dry})")

if __name__ == "__main__": app()
```

---

## 19. Web et APIs

### 19.1 FastAPI (recommandé)

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel

app = FastAPI()

class User(BaseModel):
    name: str
    age: int = 0

@app.get("/users/{uid}")
def get_user(uid: int):
    if uid == 0: raise HTTPException(404, "introuvable")
    return {"id": uid, "name": "Alice"}

@app.post("/users")
def create_user(u: User):
    return {"created": u}
```

```bash
pip install "fastapi[standard]"
fastapi dev main.py        # serveur dev
```

### 19.2 Flask (minimaliste)

```python
from flask import Flask, jsonify, request
app = Flask(__name__)

@app.get("/health")
def health(): return {"ok": True}

@app.post("/echo")
def echo(): return jsonify(request.get_json())
```

---

## 20. Packaging

### 20.1 pyproject.toml moderne

```toml
[project]
name = "mycli"
version = "1.0.0"
description = "Mon outil CLI"
authors = [{ name = "Alice", email = "a@b.c" }]
requires-python = ">=3.10"
dependencies = ["click>=8", "requests>=2"]
readme = "README.md"
license = { text = "MIT" }

[project.optional-dependencies]
dev = ["pytest", "ruff", "mypy"]

[project.scripts]
mycli = "mycli.cli:main"

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"
```

### 20.2 Build + publish

```bash
pip install build twine
python -m build
twine upload dist/*               # PyPI
twine upload --repository testpypi dist/*
```

### 20.3 Dockerisation

```dockerfile
FROM python:3.12-slim AS b
WORKDIR /app
COPY pyproject.toml .
RUN pip install --user --no-cache-dir .
COPY . .

FROM python:3.12-slim
WORKDIR /app
COPY --from=b /root/.local /root/.local
COPY --from=b /app /app
ENV PATH=/root/.local/bin:$PATH
USER 1000
ENTRYPOINT ["mycli"]
```

---

## 21. Performance

### 21.1 Profilage

```bash
python -X importtime script.py     # temps d'import
python -m cProfile -s cumulative script.py
python -m timeit -n 1000 "sum(range(1000))"
```

```python
import cProfile
cProfile.run("ma_fonction()", "profile.out")
import pstats
pstats.Stats("profile.out").sort_stats("cumulative").print_stats(20)
```

### 21.2 Optimisations courantes

- Préférer comprehensions à `map/filter` + lambda.
- `set` pour les recherches `in`.
- `''.join(list)` au lieu de `s += s2` en boucle.
- `lru_cache` pour memoization.
- Numpy/Pandas pour calculs vectoriels.
- Cython/Numba/Mypyc/PyPy pour CPU intensif.

### 21.3 Mémoire

```bash
pip install memory_profiler
mprof run script.py
mprof plot
```

---

## 22. Bonnes pratiques

1. **PEP 8** (style). Linter : **ruff**, formatter : **black**.
2. **Type hints** + **mypy**.
3. Tests **pytest** + couverture > 80%.
4. **Venv** par projet (jamais en global).
5. `requirements.txt` ou `pyproject.toml` versionné.
6. `pre-commit` : hooks de qualité automatiques.
7. Logs structurés (JSON) avec **logging** ou **structlog**.
8. Pas de secrets dans le code (variables d'env, Vault, AWS SM).
9. Gestion d'erreurs explicite (jamais `except Exception: pass`).
10. **Docstrings** Google/Numpy style + Sphinx pour génération doc.
11. CI : lint + tests + build + deploy.
12. Sécurité : `bandit`, `safety`/`pip-audit`.

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.5.0
    hooks: [{ id: ruff }, { id: ruff-format }]
  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.10.0
    hooks: [{ id: mypy }]
```

---

## Conclusion

Vous avez parcouru Python de la syntaxe aux usages DevOps. Étapes suivantes :

1. Pratiquez les [exercices avancés](./EXERCICES_AVANCES.md).
2. Préparez votre entretien avec le [guide d'interview](./INTERVIEW.md).
3. Lisez **Fluent Python** (Luciano Ramalho).
4. Contribuez à un projet open-source.
5. Spécialisez-vous : asyncio, data, ML, ops, embedded.

