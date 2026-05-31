# Python — Préparation à l'entretien

> ~60 questions/réponses en français par niveau, orientation DevOps/SRE.

---

## Junior

<details>
<summary>1. Différences entre Python 2 et 3 ?</summary>

- `print` est une fonction en 3.
- Strings : `str` Unicode par défaut, `bytes` séparé.
- `int` unifié (plus de `long`).
- Division `/` retourne `float` en 3.
- `range` lazy.
- `dict.keys()` retourne une vue.

**Python 2 est end-of-life depuis 2020. Toujours Python 3.**
</details>

<details>
<summary>2. Mutable vs immutable ?</summary>

- **Immutables** : `int`, `float`, `bool`, `str`, `tuple`, `frozenset`, `bytes`.
- **Mutables** : `list`, `dict`, `set`, objet custom (par défaut).

Conséquences : hashabilité, passage par référence, partage entre fonctions.
</details>

<details>
<summary>3. Différence `is` vs `==` ?</summary>

- `is` : même objet en mémoire (compare `id()`).
- `==` : valeurs égales (appelle `__eq__`).

`a is None` est la bonne idiome ; `a == None` fonctionne mais déconseillé.
</details>

<details>
<summary>4. Que fait `range()` ?</summary>

Retourne un objet itérable lazy générant les entiers à la demande (n'alloue pas la liste). Très efficace mémoire en Python 3.
</details>

<details>
<summary>5. List comprehension ?</summary>

```python
[x*2 for x in range(5) if x % 2 == 0]   # [0, 4, 8]
```

Plus rapide et plus lisible que `for` + `append` ou que `map`/`filter` + `lambda`.
</details>

<details>
<summary>6. Tuple vs list ?</summary>

- `tuple` : immuable, hashable (utilisable comme clé dict), généralement plus rapide.
- `list` : mutable, méthodes (append, remove, sort).
</details>

<details>
<summary>7. Comment lire un fichier ?</summary>

```python
with open("f.txt", encoding="utf-8") as f:
    for line in f: print(line.rstrip())
```

`with` garantit la fermeture, même en cas d'exception.
</details>

<details>
<summary>8. À quoi sert `with` ?</summary>

C'est un context manager : appelle `__enter__` au début, `__exit__` à la sortie (même sur exception). Utilisé pour fichiers, connexions, locks, transactions.
</details>

<details>
<summary>9. Différence `append` vs `extend` ?</summary>

- `lst.append(x)` : ajoute **un élément** (peut être une liste imbriquée).
- `lst.extend(iter)` : ajoute **chaque élément** de l'itérable.
</details>

<details>
<summary>10. À quoi sert `if __name__ == "__main__":` ?</summary>

Permet d'exécuter du code uniquement quand le fichier est lancé en script direct, et pas quand il est importé comme module. Indispensable pour `multiprocessing` sur Windows.
</details>

<details>
<summary>11. Que fait `pip install` ?</summary>

Télécharge et installe un package depuis PyPI (ou autre index) dans l'environnement Python courant. Avec `--user` : dans le home user. Dans un venv : isolé au projet.
</details>

<details>
<summary>12. Pourquoi utiliser un venv ?</summary>

Isoler les dépendances d'un projet, éviter les conflits de versions, ne pas polluer le Python système, faciliter la reproductibilité.
</details>

---

## Intermédiaire

<details>
<summary>13. Qu'est-ce que le GIL ?</summary>

**Global Interpreter Lock** : verrou empêchant l'exécution concurrente de **bytecode Python** sur plusieurs threads d'un même process CPython. Conséquence : threads inefficaces pour CPU-bound (utiliser multiprocessing), mais OK pour I/O-bound (le GIL est relâché pendant les attentes).

Python 3.13 introduit un GIL optionnel (PEP 703).
</details>

<details>
<summary>14. Différence `*args` / `**kwargs` ?</summary>

- `*args` : tuple d'arguments positionnels supplémentaires.
- `**kwargs` : dict d'arguments nommés supplémentaires.

```python
def f(*args, **kwargs): ...
f(1, 2, a=3, b=4)   # args=(1,2), kwargs={'a':3,'b':4}
```
</details>

<details>
<summary>15. Qu'est-ce qu'un décorateur ?</summary>

Fonction qui prend une fonction et en retourne une autre (souvent en ajoutant un comportement). Sucre syntaxique :
```python
@deco
def f(): ...
# équivalent à :
f = deco(f)
```
</details>

<details>
<summary>16. Générateur vs itérateur ?</summary>

- Un **itérateur** : objet avec `__iter__` et `__next__`, lève `StopIteration`.
- Un **générateur** : fonction avec `yield`. Forme la plus simple de créer un itérateur. Lazy, mémoire constante.

Generator expression : `(x*2 for x in range(10))`.
</details>

<details>
<summary>17. Différence shallow copy vs deep copy ?</summary>

- `copy.copy(x)` : copie l'objet, mais pas les sous-objets imbriqués.
- `copy.deepcopy(x)` : copie récursivement tout.

```python
import copy
a = [[1,2],[3,4]]
b = copy.copy(a)        # b[0] is a[0]
c = copy.deepcopy(a)    # c[0] is not a[0]
```
</details>

<details>
<summary>18. Pourquoi l'argument mutable par défaut est-il un piège ?</summary>

```python
def f(x, lst=[]):
    lst.append(x); return lst
print(f(1))   # [1]
print(f(2))   # [1, 2] ← partagé entre appels
```

Le default est évalué **une seule fois** à la définition. Solution : `lst=None` puis `lst = [] if lst is None else lst`.
</details>

<details>
<summary>19. Différence threading vs multiprocessing vs asyncio ?</summary>

| Modèle | Quand | Limite |
|---|---|---|
| **threading** | I/O concurrent < 100 | GIL bloque CPU |
| **multiprocessing** | CPU intensif | Coût création process, pickle |
| **asyncio** | I/O massif (milliers) | Nécessite libs async (aiohttp, asyncpg) |
</details>

<details>
<summary>20. Qu'est-ce qu'une coroutine ?</summary>

Fonction `async def`. Elle ne s'exécute pas directement : `await` la planifie dans la boucle d'événements (asyncio). Permet la concurrence single-threaded.
</details>

<details>
<summary>21. Comment fonctionne `lru_cache` ?</summary>

`@functools.lru_cache(maxsize=128)` mémorise les valeurs de retour selon les arguments hashables. Utile pour mémoïzation. `lru_cache(maxsize=None)` = cache illimité. Reset : `f.cache_clear()`.
</details>

<details>
<summary>22. Différence `@staticmethod` / `@classmethod` ?</summary>

```python
class A:
    def normal(self): ...        # accès à l'instance
    @classmethod
    def from_dict(cls, d): ...   # accès à la classe (factory)
    @staticmethod
    def helper(x): ...           # juste une fonction dans le namespace
```
</details>

<details>
<summary>23. Que sont les dataclasses ?</summary>

Décorateur (3.7+) qui génère automatiquement `__init__`, `__repr__`, `__eq__` à partir d'attributs typés. Optionnel : `frozen`, `slots`, `order`.

Avantage : moins de boilerplate qu'une classe classique. Pour validation : `pydantic` ou `attrs`.
</details>

<details>
<summary>24. À quoi sert `typing` et `mypy` ?</summary>

`typing` fournit les annotations (`List`, `Dict`, `Optional`, `Callable`, `Generic`…). `mypy` est un linter statique qui vérifie la cohérence. N'affecte pas l'exécution mais réduit énormément les bugs.

Depuis 3.9-3.10, on peut utiliser les types built-in directement (`list[int]`, `int | None`).
</details>

<details>
<summary>25. Différence `__str__` vs `__repr__` ?</summary>

- `__repr__` : représentation **non-ambiguë**, idéale pour debug. Cible : développeur.
- `__str__` : représentation **lisible** pour utilisateur. Si absent, fallback sur `__repr__`.
</details>

<details>
<summary>26. Comment Python gère-t-il la mémoire ?</summary>

- **Reference counting** : chaque objet a un compteur. Quand il tombe à 0, l'objet est désalloué immédiatement.
- **Garbage collector** : détecte les cycles de références (gen 0/1/2). `gc.collect()` force un cycle.
</details>

<details>
<summary>27. Qu'est-ce que le pattern singleton ? Comment l'implémenter ?</summary>

Garantir une seule instance d'une classe. Implémentations :
- Module Python (intrinsèquement singleton).
- Variable de classe + `__new__`.
- Metaclass.
- Décorateur.

Exemple via metaclass : voir exercice 20.
</details>

<details>
<summary>28. Qu'est-ce qu'une metaclass ?</summary>

Classe d'une classe. Permet de modifier la création des classes elles-mêmes. `type` est la metaclass par défaut. Utilisations : ORMs (Django, SQLAlchemy), ABC, frameworks de validation.
</details>

---

## Senior

<details>
<summary>29. Comment optimiser un script Python lent ?</summary>

1. **Profiler** d'abord (`cProfile`, `line_profiler`).
2. Optimiser l'algorithme avant les détails.
3. Utiliser `set`/`dict` pour les recherches.
4. Comprehensions plutôt que boucles.
5. `lru_cache` pour mémoïzation.
6. Vectoriser avec **numpy** si applicable.
7. C extensions, Cython, Numba, PyPy.
8. Multiprocessing pour CPU-bound.
9. asyncio pour I/O.
10. Caching externe (Redis).
</details>

<details>
<summary>30. Comment debug un memory leak Python ?</summary>

- `tracemalloc` (snapshot allocations).
- `memory_profiler` (`@profile`).
- `gc.get_objects()` pour inspecter.
- `objgraph` pour visualiser les références.
- Causes fréquentes : caches non bornés, références circulaires avec `__del__`, closures qui capturent trop.
</details>

<details>
<summary>31. Différence générateur vs liste ?</summary>

| | Liste | Générateur |
|---|---|---|
| Mémoire | O(n) | O(1) |
| Vitesse 1ère itération | rapide après | lazy |
| Réutilisable | oui | non (consommé) |
| Cas | dataset entier en RAM | streaming |
</details>

<details>
<summary>32. Comment écrire un test pytest avec fixtures et paramétrage ?</summary>

```python
@pytest.fixture
def db(): conn = ...; yield conn; conn.close()

@pytest.mark.parametrize("n,expected", [(0,0),(1,1),(10,55)])
def test_fib(n, expected, db):
    assert fib(n) == expected
```
</details>

<details>
<summary>33. Quand utiliser `asyncio` vs `threading` ?</summary>

- **asyncio** : nombre élevé d'I/O concurrents (milliers de sockets), libs async disponibles, contrôle fin du scheduling.
- **threading** : intégration avec libs sync (requests, paramiko), petite échelle, plus simple à raisonner.

Souvent : `concurrent.futures.ThreadPoolExecutor` est un bon compromis.
</details>

<details>
<summary>34. Quels patterns pour rendre une classe testable ?</summary>

- **Dependency injection** : passer les dépendances en argument.
- Éviter les singletons cachés et les imports au runtime.
- Interfaces (`Protocol`) pour mocker facilement.
- Fonctions pures quand possible.
</details>

<details>
<summary>35. Différence `Protocol` vs `ABC` ?</summary>

- **ABC** : héritage explicite, `register()` possible.
- **Protocol** (PEP 544, 3.8+) : duck typing structurel statique. Pas besoin d'hériter. Vérifié par mypy.

Recommandé pour APIs modernes.
</details>

<details>
<summary>36. Comment versionner et packager une lib Python ?</summary>

- `pyproject.toml` avec `[project]`, `version`, `dependencies`, `scripts`.
- Build : `python -m build`.
- Publish : `twine upload dist/*` vers PyPI.
- Tag Git correspondant.
- Releases automatisées via CI (GitHub Actions + `pypa/gh-action-pypi-publish`).
</details>

<details>
<summary>37. Qu'est-ce que pydantic ?</summary>

Lib de validation/parsing basée sur les type hints. Génère des erreurs riches. Très utilisée avec FastAPI. v2 (Rust core) est très rapide.

```python
from pydantic import BaseModel, Field
class User(BaseModel):
    name: str
    age: int = Field(ge=0, le=150)
```
</details>

<details>
<summary>38. Quelles librairies utilisez-vous en DevOps Python ?</summary>

`requests`, `httpx`, `aiohttp`, `boto3`, `kubernetes`, `docker`, `paramiko`, `fabric`, `ansible-runner`, `hvac` (Vault), `psutil`, `click`/`typer`, `pyyaml`, `jinja2`, `structlog`, `prometheus_client`, `pytest`, `ruff`, `mypy`.
</details>

<details>
<summary>39. Comment gérer la configuration d'une app ?</summary>

- Variables d'environnement (12-factor) → `os.environ` ou `pydantic-settings`.
- Fichier YAML/TOML + override par env.
- Vault / AWS SM pour les secrets.
- Validation au démarrage (fail fast).
</details>

<details>
<summary>40. À quoi sert `__slots__` ?</summary>

Déclarer explicitement les attributs autorisés ; remplace le `__dict__` de l'instance. Avantages : moins de mémoire (jusqu'à -40 %), accès plus rapide. Inconvénient : pas d'ajout dynamique.

```python
class Point:
    __slots__ = ("x", "y")
    def __init__(self, x, y): self.x, self.y = x, y
```
</details>

<details>
<summary>41. Quel est le MRO (Method Resolution Order) ?</summary>

L'ordre dans lequel Python cherche un attribut/méthode dans les classes parentes. Calculé par l'algorithme **C3** (linearisation). Visible via `Cls.__mro__` ou `Cls.mro()`.

Utile en héritage multiple pour comprendre quelle méthode parent est appelée par `super()`.
</details>

<details>
<summary>42. Comment Python implémente `dict` en interne ?</summary>

Table de hachage open-addressing. Depuis 3.7, **ordre d'insertion garanti**. Depuis 3.6, optimisé en mémoire (compact dict). Lookup O(1) moyen.
</details>

<details>
<summary>43. Différence `concurrent.futures` vs `asyncio` ?</summary>

- `concurrent.futures` : abstraction unifiée pour threads/process, sync API.
- `asyncio` : modèle async, basé sur coroutines, single-thread.

On peut combiner : `loop.run_in_executor()` pour appeler du code bloquant depuis asyncio.
</details>

<details>
<summary>44. Quel est l'intérêt de `functools.partial` ?</summary>

Fixer certains arguments d'une fonction pour en créer une nouvelle. Plus propre que `lambda`.

```python
from functools import partial
def request(method, url): ...
get = partial(request, "GET")
```
</details>

---

## Lead / Architecte

<details>
<summary>45. Comment structurer un gros projet Python ?</summary>

```
project/
├── pyproject.toml
├── README.md
├── src/
│   └── mypkg/
│       ├── __init__.py
│       ├── api/
│       ├── core/
│       ├── infrastructure/
│       └── cli.py
├── tests/
└── docs/
```

Layout `src/` évite les imports accidentels du repo. Modules par **domaine** (DDD), pas par type technique.
</details>

<details>
<summary>46. Que pensez-vous de Python en production ?</summary>

Excellent pour scripts, automation, web (FastAPI, Django), data/ML. Limites : performances CPU brut (palliées par numpy/C ext), démarrage lent (vs Go), runtime relativement lourd. Pour services à très haute perf : Go/Rust souvent préférés.
</details>

<details>
<summary>47. Stratégie de migration Python 2 → 3 ?</summary>

- Audit : `caniusepython3`, `2to3`.
- Compatibilité progressive : `from __future__ import ...`, `six`, `python-future`.
- Couverture de tests d'abord.
- Module par module en parallèle.
- CI testant les deux jusqu'à coupure.
</details>

<details>
<summary>48. Quel framework web choisir en 2025 ?</summary>

- **FastAPI** : APIs REST/gRPC modernes, type hints, async, OpenAPI auto.
- **Django** : application complète (admin, ORM, auth), monolithe puissant.
- **Flask** : minimaliste, idéal microservices simples.
- **Starlette** / **Litestar** : alternatives async.

Pour DevOps tooling interne : FastAPI gagne presque toujours.
</details>

<details>
<summary>49. Comment garantir la qualité d'un code Python en équipe ?</summary>

- **ruff** (lint + format, ultra rapide, remplace black/isort/flake8).
- **mypy** ou **pyright** (types).
- **pytest** + coverage > 80%.
- **pre-commit** hooks.
- **bandit** (sécurité), **pip-audit** (CVE deps).
- **CI** bloquante sur PR.
- Revue de code obligatoire.
- Documentation (docstrings + Sphinx/MkDocs).
</details>

<details>
<summary>50. Quel futur pour Python ?</summary>

- **GIL optionnel** (PEP 703, expérimental 3.13).
- **JIT** intégré (PEP 744, en cours).
- **mypyc / Cython** pour compilation à la performance C.
- **Pydantic v2** / **uv** : montée des outils Rust pour Python.
- **WebAssembly** (Pyodide, PyScript).
- Adoption massive en IA/ML (continue).
- Renforcement du typage progressif.
</details>

---

## Questions code "à compléter / déboguer"

<details>
<summary>51. Qu'affiche ce code et pourquoi ?</summary>

```python
def f(x=[]):
    x.append(1); return x
print(f()); print(f()); print(f())
```

Réponse : `[1] [1, 1] [1, 1, 1]`. Le default mutable est partagé.
</details>

<details>
<summary>52. Comment écrire une boucle async qui télécharge 10 URLs en parallèle ?</summary>

```python
import asyncio, aiohttp
async def fetch(s, u): 
    async with s.get(u) as r: return await r.text()
async def main(urls):
    async with aiohttp.ClientSession() as s:
        return await asyncio.gather(*(fetch(s, u) for u in urls))
asyncio.run(main([...]))
```
</details>

<details>
<summary>53. Comment lire un fichier de 100 GB sans le charger en mémoire ?</summary>

Itération ligne par ligne ou par chunks :
```python
with open(p, "rb") as f:
    while chunk := f.read(1024*1024):
        process(chunk)
```
</details>

<details>
<summary>54. Différence entre :
```python
x = [i*i for i in range(10**7)]
y = (i*i for i in range(10**7))
```
?</summary>

`x` est une liste matérialisée (~80 Mo RAM). `y` est un générateur (qq octets). Itérer sur `y` est plus économe ; mais on ne peut le parcourir qu'une fois.
</details>

<details>
<summary>55. Comment ferez-vous un timeout sur une requête HTTP ?</summary>

```python
requests.get(url, timeout=(connect_timeout, read_timeout))
# ou timeout=5 (les deux)
```

**Toujours** définir un timeout : par défaut, requests attend indéfiniment.
</details>

<details>
<summary>56. Comment paralléliser ce code CPU-bound ?</summary>

```python
nums = [...]
results = [heavy(n) for n in nums]
```

→
```python
from concurrent.futures import ProcessPoolExecutor
with ProcessPoolExecutor() as exe:
    results = list(exe.map(heavy, nums))
```
</details>

<details>
<summary>57. Pourquoi `len([])`, `len("a")`, `len({})` fonctionnent-ils tous ?</summary>

Tous ces types implémentent `__len__`. `len()` est une fonction built-in qui appelle ce dunder. C'est du **duck typing** + protocoles.
</details>

<details>
<summary>58. Comment gérer un script qui doit faire 10 000 requêtes API avec rate limit ?</summary>

- `asyncio` + `aiohttp` + `asyncio.Semaphore(n)` pour limiter la concurrence.
- Backoff exponentiel sur 429.
- Optionnellement `aiolimiter` pour le rate-limit précis.
- Reprendre depuis un état (idempotence) en cas de crash.
</details>

<details>
<summary>59. Comment exposer un script Python en CLI installable ?</summary>

Dans `pyproject.toml` :
```toml
[project.scripts]
mycli = "mypkg.cli:main"
```
Puis `pip install .` → la commande `mycli` est disponible.
</details>

<details>
<summary>60. Comment minimiser la taille d'une image Docker Python ?</summary>

- `python:3.12-slim` ou Alpine.
- Multi-stage : compiler les wheels dans builder, copier `~/.local`.
- Cache mount BuildKit pour pip.
- `--no-cache-dir`.
- `user` non-root.
- Distroless si le binaire est compilable.
</details>

---

## Conseils

- Connaissez les pièges classiques (mutable defaults, late binding closures, GIL).
- Maîtrisez **type hints**, **dataclasses**, **pytest**.
- Ayez un projet personnel à présenter (CLI, microservice, automation).
- Code propre > code malin : "Readability counts".
- Préparez la lecture en direct d'un bout de code (pair-coding fréquent en entretien).

