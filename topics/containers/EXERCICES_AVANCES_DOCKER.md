# Docker — Exercices avancés (Débutant → Expert)

> 25 exercices progressifs en français pour pratiquer Docker.

---

## Niveau Débutant

### Exercice 1 — Premier conteneur

**Objectif** : lancer Nginx, accéder en HTTP, voir les logs, supprimer.

<details>
<summary>Solution</summary>

```bash
docker run -d --name web -p 8080:80 nginx:1.27-alpine
curl localhost:8080
docker logs web
docker rm -f web
```
</details>

---

### Exercice 2 — Conteneur interactif

**Objectif** : ouvrir un shell dans Ubuntu, créer un fichier, sortir, observer qu'il n'existe plus dans une nouvelle instance.

<details>
<summary>Solution</summary>

```bash
docker run -it --rm ubuntu:22.04 bash
# dans le conteneur :
echo "hello" > /tmp/test
exit

docker run -it --rm ubuntu:22.04 cat /tmp/test   # fichier absent
```
</details>

---

### Exercice 3 — Premier Dockerfile

**Objectif** : conteneuriser un script Python "hello world".

`app.py` :
```python
print("Bonjour depuis Docker !")
```

<details>
<summary>Solution</summary>

`Dockerfile` :
```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY app.py .
CMD ["python", "app.py"]
```

```bash
docker build -t hello-py .
docker run --rm hello-py
```
</details>

---

### Exercice 4 — Variables d'environnement

**Objectif** : conteneuriser une app qui lit `NAME` et affiche `Hello $NAME`.

<details>
<summary>Solution</summary>

```python
import os; print(f"Hello {os.getenv('NAME', 'world')}")
```

```bash
docker build -t greet .
docker run --rm -e NAME=Alice greet
```
</details>

---

### Exercice 5 — Bind mount

**Objectif** : monter un dossier local dans un conteneur Nginx pour servir des fichiers statiques.

<details>
<summary>Solution</summary>

```bash
mkdir site && echo "<h1>Hi</h1>" > site/index.html
docker run -d --name web -p 8080:80 -v $(pwd)/site:/usr/share/nginx/html:ro nginx:alpine
curl localhost:8080
```
</details>

---

## Niveau Intermédiaire

### Exercice 6 — Multi-stage Go

**Objectif** : construire une image Go ≤ 20 MB.

<details>
<summary>Solution</summary>

```dockerfile
FROM golang:1.22 AS b
WORKDIR /src
COPY . .
RUN CGO_ENABLED=0 go build -ldflags="-s -w" -o /app

FROM gcr.io/distroless/static:nonroot
COPY --from=b /app /app
USER nonroot
ENTRYPOINT ["/app"]
```

```bash
docker images mygo
```
</details>

---

### Exercice 7 — Multi-stage Node

**Objectif** : construire une image Node prod avec deps de prod uniquement.

<details>
<summary>Solution</summary>

Voir le guide complet section 8.2.
</details>

---

### Exercice 8 — Healthcheck

**Objectif** : ajouter un HEALTHCHECK à une image Flask et observer `(healthy)` dans `docker ps`.

<details>
<summary>Solution</summary>

```dockerfile
HEALTHCHECK --interval=10s --retries=3 \
  CMD curl -fs http://localhost:8000/health || exit 1
```

```bash
docker run -d --name app -p 8000:8000 myflask
docker ps        # STATUS = Up X seconds (healthy)
```
</details>

---

### Exercice 9 — Network custom

**Objectif** : créer un réseau bridge `appnet`, lancer Redis et un client Python qui se connecte par nom.

<details>
<summary>Solution</summary>

```bash
docker network create appnet
docker run -d --name redis --network appnet redis:7-alpine
docker run --rm --network appnet python:3.12-slim \
  python -c "import socket; print(socket.gethostbyname('redis'))"
```
</details>

---

### Exercice 10 — Volume nommé PostgreSQL

**Objectif** : lancer Postgres avec un volume nommé, recréer le conteneur et vérifier que la donnée persiste.

<details>
<summary>Solution</summary>

```bash
docker volume create pgdata
docker run -d --name pg -e POSTGRES_PASSWORD=x -v pgdata:/var/lib/postgresql/data postgres:16
docker exec -it pg psql -U postgres -c "CREATE TABLE t(id int); INSERT INTO t VALUES (42);"
docker rm -f pg
docker run -d --name pg -e POSTGRES_PASSWORD=x -v pgdata:/var/lib/postgresql/data postgres:16
docker exec -it pg psql -U postgres -c "SELECT * FROM t;"   # 42
```
</details>

---

### Exercice 11 — Docker Compose web+db

**Objectif** : Compose avec FastAPI + PostgreSQL + Adminer, dépendance via healthcheck.

<details>
<summary>Solution</summary>

```yaml
services:
  api:
    build: ./api
    depends_on: { db: { condition: service_healthy } }
    environment: { DATABASE_URL: postgresql://postgres:pwd@db/app }
    ports: ["8000:8000"]
  db:
    image: postgres:16
    environment: { POSTGRES_PASSWORD: pwd, POSTGRES_DB: app }
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "postgres"]
      interval: 3s
    volumes: [pgdata:/var/lib/postgresql/data]
  adminer:
    image: adminer
    ports: ["8081:8080"]
volumes: { pgdata: {} }
```
</details>

---

### Exercice 12 — Compose profiles

**Objectif** : ajouter un service `pgadmin` qui ne démarre que avec `--profile debug`.

<details>
<summary>Solution</summary>

```yaml
services:
  pgadmin:
    image: dpage/pgadmin4
    profiles: [debug]
    environment: { PGADMIN_DEFAULT_EMAIL: a@b.c, PGADMIN_DEFAULT_PASSWORD: x }
    ports: ["5050:80"]
```

```bash
docker compose --profile debug up -d
```
</details>

---

## Niveau Avancé

### Exercice 13 — Optimisation de taille

**Objectif** : prendre une image naïve Python (≥ 1 GB) et la passer sous 150 MB.

<details>
<summary>Solution</summary>

Image naïve :
```dockerfile
FROM python:3.12
COPY . /app
RUN pip install -r /app/requirements.txt
CMD ["python", "/app/main.py"]
```

Optimisée :
```dockerfile
FROM python:3.12-slim AS b
WORKDIR /app
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt
COPY . .

FROM python:3.12-slim
WORKDIR /app
COPY --from=b /root/.local /root/.local
COPY --from=b /app /app
ENV PATH=/root/.local/bin:$PATH
USER 1000
CMD ["python", "main.py"]
```

```bash
docker images myapp
```
</details>

---

### Exercice 14 — Scan d'image

**Objectif** : scanner une image avec Trivy et corriger 1 CVE en mettant à jour la base.

<details>
<summary>Solution</summary>

```bash
trivy image --severity HIGH,CRITICAL myapp:1.0
# Si la CVE est dans openssl par exemple :
# changer FROM debian:11 → debian:12, rebuild, re-scan
```
</details>

---

### Exercice 15 — Build multi-arch

**Objectif** : publier une image pour `linux/amd64` et `linux/arm64`.

<details>
<summary>Solution</summary>

```bash
docker buildx create --use --name multi
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t ghcr.io/acme/myapp:1.0 \
  --push .
docker buildx imagetools inspect ghcr.io/acme/myapp:1.0
```
</details>

---

### Exercice 16 — Registry privé + TLS

**Objectif** : lancer un registry privé, pusher une image, l'utiliser depuis un autre poste.

<details>
<summary>Solution</summary>

```bash
docker run -d --name registry -p 5000:5000 \
  -v $(pwd)/data:/var/lib/registry registry:2
docker tag myapp:1.0 reg.local:5000/myapp:1.0
docker push reg.local:5000/myapp:1.0
```

Pour la prod : ajouter TLS (cert), authentification htpasswd, et placer derrière un reverse proxy (Nginx/Traefik) avec Let's Encrypt.
</details>

---

### Exercice 17 — Secrets BuildKit

**Objectif** : passer un token GitHub pour `npm install` sans qu'il apparaisse dans les layers.

<details>
<summary>Solution</summary>

```dockerfile
# syntax=docker/dockerfile:1.7
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc \
    npm ci --only=production
```

```bash
DOCKER_BUILDKIT=1 docker build --secret id=npmrc,src=$HOME/.npmrc -t app .
docker history app   # le secret n'apparaît pas
```
</details>

---

### Exercice 18 — Cache mount

**Objectif** : accélérer un build Python en cachant le dossier pip.

<details>
<summary>Solution</summary>

```dockerfile
# syntax=docker/dockerfile:1.7
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt
```
</details>

---

### Exercice 19 — Conteneur read-only sécurisé

**Objectif** : lancer Nginx en read-only avec tmpfs et user non-root.

<details>
<summary>Solution</summary>

```bash
docker run -d --name web \
  --read-only \
  --tmpfs /var/cache/nginx --tmpfs /var/run \
  --user 101:101 \
  --cap-drop ALL --cap-add NET_BIND_SERVICE \
  --security-opt no-new-privileges:true \
  -p 8080:80 nginx:alpine
```
</details>

---

### Exercice 20 — Signature cosign

**Objectif** : signer une image, vérifier la signature.

<details>
<summary>Solution</summary>

```bash
cosign generate-key-pair
cosign sign --key cosign.key ghcr.io/acme/myapp:1.0
cosign verify --key cosign.pub ghcr.io/acme/myapp:1.0
```
</details>

---

## Niveau Expert

### Exercice 21 — Debugging "container exits immediately"

**Contexte** : `docker run myimg` retourne immédiatement, sans logs visibles.

<details>
<summary>Solution</summary>

```bash
docker logs --details <id>
docker inspect <id> | jq '.[0].State'   # OOMKilled ? ExitCode ?
docker run -it --entrypoint sh myimg    # entrer manuellement
```

Causes typiques :
- CMD pointe vers un binaire absent.
- Permissions (USER sans accès au filesystem).
- Variable d'env manquante.
- Process en arrière-plan qui sort immédiatement (toujours **foreground** dans un container).
</details>

---

### Exercice 22 — Rootless Docker

**Objectif** : installer et utiliser Docker rootless.

<details>
<summary>Solution</summary>

```bash
curl -fsSL https://get.docker.com/rootless | sh
export PATH=$HOME/bin:$PATH
export DOCKER_HOST=unix:///run/user/$UID/docker.sock
systemctl --user start docker
docker run --rm hello-world
```
</details>

---

### Exercice 23 — Compose en CI

**Objectif** : utiliser `docker compose` dans GitHub Actions pour des tests d'intégration.

<details>
<summary>Solution</summary>

```yaml
name: CI
on: [pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: docker compose -f compose.test.yaml up -d --wait
      - run: docker compose -f compose.test.yaml run --rm tests pytest
      - if: always()
        run: docker compose -f compose.test.yaml down -v
```
</details>

---

### Exercice 24 — Dive : analyser les layers

**Objectif** : utiliser `dive` pour identifier les layers qui gaspillent de l'espace.

<details>
<summary>Solution</summary>

```bash
docker run --rm -it \
  -v /var/run/docker.sock:/var/run/docker.sock \
  wagoodman/dive:latest myapp:1.0
```

Repérer les layers avec un faible "efficiency score" (< 95 %) et factoriser les `RUN`.
</details>

---

### Exercice 25 — Conversion vers Kubernetes

**Objectif** : convertir le `compose.yaml` de l'exercice 11 en manifests K8s avec `kompose`, puis raffiner.

<details>
<summary>Solution</summary>

```bash
kompose convert -f compose.yaml -o k8s/
kubectl apply -f k8s/
```

Raffinements à faire manuellement :
- Ajouter probes liveness/readiness.
- Remplacer Secret stringData par un Secret externe.
- Créer un Ingress.
- Ajouter requests/limits + HPA.
- Anti-affinity et PDB pour la résilience.
</details>

---

## Pour aller plus loin

- Lisez la spécification **OCI**.
- Maîtrisez **BuildKit** (mounts, cache, secrets).
- Apprenez **Podman + Buildah** comme alternatives.
- Construisez votre propre **registry Harbor** avec scan automatique.
- Mettez en place une chaîne **SBOM + signature + verify** dans votre CI.

