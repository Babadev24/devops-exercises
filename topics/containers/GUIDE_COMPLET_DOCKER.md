# Docker — Le Guide Complet : de 0 à Hero

> Livre de référence en français pour maîtriser Docker et les conteneurs, de la philosophie aux usages production.

---

## Table des matières

1. [Introduction aux conteneurs](#1-introduction-aux-conteneurs)
2. [Installation](#2-installation)
3. [Architecture Docker](#3-architecture-docker)
4. [Images](#4-images)
5. [Conteneurs](#5-conteneurs)
6. [Dockerfile](#6-dockerfile)
7. [Bonnes pratiques Dockerfile](#7-bonnes-pratiques-dockerfile)
8. [Multi-stage builds](#8-multi-stage-builds)
9. [Networking](#9-networking)
10. [Volumes et persistance](#10-volumes-et-persistance)
11. [Docker Compose](#11-docker-compose)
12. [Registry](#12-registry)
13. [Sécurité](#13-sécurité)
14. [Optimisation des images](#14-optimisation-des-images)
15. [Debugging](#15-debugging)
16. [Docker en production](#16-docker-en-production)
17. [Alternatives modernes](#17-alternatives-modernes)
18. [Migration vers Kubernetes](#18-migration-vers-kubernetes)

---

## 1. Introduction aux conteneurs

### 1.1 Conteneur vs VM

| Aspect | VM | Conteneur |
|---|---|---|
| Couche | Hyperviseur (matériel virtuel) | Noyau partagé |
| Taille | GB | MB |
| Démarrage | Minutes | Secondes / ms |
| Isolation | Forte (kernel séparé) | Moins forte (namespaces) |
| Densité | Faible | Très élevée |

### 1.2 Comment ça marche ?

Un conteneur Linux repose sur 3 primitives du noyau :

- **Namespaces** : isolation (PID, NET, MNT, UTS, IPC, USER, CGROUP, TIME).
- **Cgroups** : limitation des ressources (CPU, RAM, IO).
- **Capabilities** + **Seccomp** + **AppArmor/SELinux** : restrictions de droits.

### 1.3 Histoire

- 1979 — `chroot` (Unix V7).
- 2000 — FreeBSD `jails`.
- 2008 — **LXC** sur Linux.
- 2013 — **Docker** popularise les conteneurs (au-dessus de LXC puis libcontainer).
- 2015 — **OCI** (Open Container Initiative) standardise images + runtime.
- 2016+ — **containerd**, **CRI-O**, **Podman**.
- 2020+ — Docker Desktop devient payant pour les grandes entreprises, ascension de Podman.

### 1.4 OCI

Deux spécifications standards :

- **Image Spec** : format d'une image (layers, manifest, config).
- **Runtime Spec** : comment exécuter une bundle (chemin filesystem + config JSON).

Toute image Docker est compatible avec tout runtime OCI (containerd, CRI-O, Podman…).

---

## 2. Installation

### 2.1 Linux

**Ubuntu** :
```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker $USER   # se reconnecter ensuite
docker version
```

### 2.2 Windows / macOS

**Docker Desktop** (UI + WSL2 sur Windows, VM légère sur Mac).

Alternatives : **Rancher Desktop**, **Podman Desktop**, **Colima** (macOS).

### 2.3 Test

```bash
docker run --rm hello-world
docker info
```

---

## 3. Architecture Docker

```
+----------------+              +------------------+
|  docker (CLI)  |  REST/sock   |  dockerd         |
|                +─────────────►│  (daemon)        |
+----------------+              +────────┬─────────+
                                          │ gRPC
                                          ▼
                                  +────────────────+
                                  |  containerd    |
                                  +────────┬───────+
                                           │
                                           ▼
                                  +────────────────+
                                  |   runc (OCI)   |
                                  +────────────────+
                                           │
                                           ▼
                                  +────────────────+
                                  |  conteneur     |
                                  +────────────────+
```

- **CLI** : commande `docker` envoie des requêtes REST au démon.
- **dockerd** : gère images, réseaux, volumes ; délègue l'exécution.
- **containerd** : runtime de plus haut niveau (pulled images, snapshots).
- **runc** : exécute réellement les conteneurs (OCI runtime).

---

## 4. Images

### 4.1 Concepts

Une image = ensemble de **layers** read-only empilés, plus un **manifest** et une **config** JSON. Identifiée par :

- **nom + tag** : `nginx:1.27-alpine`
- **digest** (immuable) : `nginx@sha256:abc...`

Stockage local : `/var/lib/docker/overlay2/`.

### 4.2 Manipuler

```bash
docker pull nginx:1.27-alpine
docker images
docker image ls --digests
docker inspect nginx:1.27-alpine
docker history nginx:1.27-alpine
docker tag nginx:1.27-alpine myregistry.local/nginx:1.27
docker push myregistry.local/nginx:1.27
docker rmi nginx:1.27-alpine
docker image prune -a               # supprime les unused
```

### 4.3 Layers et cache

Chaque instruction `RUN`, `COPY`, `ADD` crée un layer. Les layers sont **partagés** entre images : `python:3.12` et `node:20` ont le même layer Debian base.

> Astuce : ordonner le Dockerfile du **moins changeant** (deps) au **plus changeant** (code) pour maximiser le cache.

### 4.4 Bases d'images

| Base | Taille | Sécurité | Usage |
|---|---|---|---|
| `scratch` | 0 | Min surface | Go static, Rust |
| `distroless` (Google) | ~20 MB | Pas de shell | Apps compilées |
| `alpine` | ~5 MB | Petit, musl libc | Léger général |
| `debian:slim` / `ubuntu` | ~30-70 MB | Familier | Cas généraux |

---

## 5. Conteneurs

### 5.1 Cycle de vie

`created → running → (paused) → stopped → removed`

```bash
docker run -d --name web -p 8080:80 nginx:1.27-alpine
docker ps
docker ps -a
docker stop web
docker start web
docker restart web
docker pause web ; docker unpause web
docker rm -f web
```

### 5.2 Run : options essentielles

```bash
docker run \
  --name app \
  -d \                              # détaché
  --rm \                            # supprimer à la sortie
  -it \                             # tty interactif
  -p 8080:80 \                      # port hôte:conteneur
  -p 127.0.0.1:9090:9090 \          # bind sur IP spécifique
  -v $(pwd)/data:/data \            # bind mount
  -v mydata:/var/lib/mysql \        # named volume
  -e DB_HOST=db \                   # env var
  --env-file .env \
  --network mynet \
  --restart unless-stopped \
  --memory 512m \
  --cpus 1.5 \
  --read-only \
  --user 1000:1000 \
  --cap-drop ALL \
  --security-opt no-new-privileges:true \
  myimage:1.0
```

### 5.3 Exec, logs, stats

```bash
docker exec -it web sh
docker logs -f --tail 100 web
docker stats
docker top web
docker inspect web | jq '.[0].State'
docker cp web:/etc/nginx/nginx.conf ./nginx.conf
```

### 5.4 Restart policies

| Policy | Quand |
|---|---|
| `no` | Jamais |
| `on-failure[:max]` | Si exit code ≠ 0 |
| `always` | Toujours (même après reboot daemon) |
| `unless-stopped` | Toujours sauf si stoppé manuellement |

---

## 6. Dockerfile

### 6.1 Toutes les instructions

```dockerfile
FROM debian:12-slim AS base       # image de base (+ alias multi-stage)
ARG VERSION=1.0                    # variable de build
LABEL org.opencontainers.image.source="https://github.com/acme/app"
ENV APP_HOME=/opt/app \            # var d'env (persistante)
    PATH=$APP_HOME/bin:$PATH
WORKDIR $APP_HOME                  # cd
RUN apt-get update && apt-get install -y --no-install-recommends curl \
    && rm -rf /var/lib/apt/lists/*
COPY --chown=app:app src/ ./src/   # depuis le contexte
ADD https://x/y.tar.gz /tmp/       # supporte URL + auto-extract tar
USER 1000:1000
EXPOSE 8080
VOLUME ["/data"]
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1
STOPSIGNAL SIGTERM
ENTRYPOINT ["python", "-m"]        # exécutable
CMD ["app"]                        # args par défaut
SHELL ["/bin/bash", "-c"]
ONBUILD COPY . /app                # appliqué quand cette image est FROM
```

### 6.2 ENTRYPOINT vs CMD

| Forme | Comportement |
|---|---|
| `CMD ["app", "--flag"]` (exec) | Commande complète (PID 1 = app) |
| `CMD app --flag` (shell) | `/bin/sh -c "app --flag"` (PID 1 = sh) |
| `ENTRYPOINT` | Commande fixe, args par CMD |

Pattern recommandé :

```dockerfile
ENTRYPOINT ["python", "-m", "app"]
CMD ["--port", "8080"]
```

```bash
docker run myimg                       # python -m app --port 8080
docker run myimg --port 9000           # python -m app --port 9000
docker run --entrypoint sh myimg       # override
```

### 6.3 COPY vs ADD

- `COPY` : copie simple (préférée).
- `ADD` : COPY + supporte URL + extraction auto des `.tar.gz`. Plus de surprises → éviter sauf besoin spécifique.

### 6.4 ARG vs ENV

- `ARG` : disponible **uniquement au build**.
- `ENV` : disponible au build **et** au runtime.

```dockerfile
ARG VERSION
ENV APP_VERSION=$VERSION
```

```bash
docker build --build-arg VERSION=1.2.3 -t app:1.2.3 .
```

### 6.5 Build

```bash
docker build -t myapp:1.0 .
docker build -f Dockerfile.prod -t myapp:prod .
docker build --no-cache .
docker build --target builder .         # multi-stage : arrêt à une étape
docker build --progress=plain --no-cache .
```

---

## 7. Bonnes pratiques Dockerfile

1. **`.dockerignore`** systématique :
   ```
   .git
   node_modules
   __pycache__
   *.log
   .env
   ```
2. **Une seule responsabilité par image**.
3. **Ordonner** : deps stables → deps applicatives → code.
4. **Combiner** les `RUN` pour limiter les layers.
5. **Pinner** les versions (`nginx:1.27.0` plutôt que `latest`).
6. **`--no-install-recommends`** + cleanup `/var/lib/apt/lists/*`.
7. **User non-root** (`USER 1000`).
8. **HEALTHCHECK** explicite.
9. **Image minimale** (alpine, distroless, scratch).
10. **Multi-stage** pour séparer build et runtime.
11. **Signal handling** : utiliser la forme `exec` pour que SIGTERM atteigne le process.
12. **Pas de secrets** dans l'image (utiliser BuildKit secrets ou env runtime).

---

## 8. Multi-stage builds

### 8.1 Go

```dockerfile
# syntax=docker/dockerfile:1.7
FROM golang:1.22 AS builder
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /out/app ./cmd/app

FROM gcr.io/distroless/static:nonroot
COPY --from=builder /out/app /app
USER nonroot
ENTRYPOINT ["/app"]
```

Résultat : image ~10 MB.

### 8.2 Node.js

```dockerfile
FROM node:20-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:20-alpine AS runtime
WORKDIR /app
ENV NODE_ENV=production
COPY --from=deps  /app/node_modules ./node_modules
COPY --from=build /app/dist ./dist
USER node
EXPOSE 3000
CMD ["node", "dist/server.js"]
```

### 8.3 Python

```dockerfile
FROM python:3.12-slim AS builder
WORKDIR /app
ENV PIP_NO_CACHE_DIR=1 PIP_DISABLE_PIP_VERSION_CHECK=1
COPY requirements.txt .
RUN pip install --user -r requirements.txt
COPY . .

FROM python:3.12-slim
WORKDIR /app
COPY --from=builder /root/.local /root/.local
COPY --from=builder /app /app
ENV PATH=/root/.local/bin:$PATH
USER 1000
CMD ["gunicorn", "-b", "0.0.0.0:8000", "app:app"]
```

### 8.4 Java

```dockerfile
FROM eclipse-temurin:21-jdk AS build
WORKDIR /app
COPY . .
RUN ./mvnw -B package -DskipTests

FROM eclipse-temurin:21-jre-alpine
COPY --from=build /app/target/*.jar /app.jar
ENTRYPOINT ["java","-jar","/app.jar"]
```

---

## 9. Networking

### 9.1 Drivers

| Driver | Description |
|---|---|
| `bridge` (défaut) | Réseau virtuel + NAT |
| `host` | Partage la stack réseau de l'hôte |
| `none` | Aucun réseau |
| `overlay` | Multi-hôtes (Swarm) |
| `macvlan` | Conteneur a sa propre MAC sur le LAN |
| `ipvlan` | Variante L2/L3 |

### 9.2 Bridge custom (recommandé)

```bash
docker network create app-net
docker run -d --name db  --network app-net postgres:16
docker run -d --name web --network app-net -e DB_HOST=db myweb
```

Les conteneurs sur le même bridge custom se voient **par nom DNS** (DNS interne de Docker).

### 9.3 Ports

```bash
docker run -p 8080:80              # 0.0.0.0:8080 → :80
docker run -p 127.0.0.1:8080:80    # localhost only
docker run -P                       # ports EXPOSE mappés aléatoirement
docker port web
```

### 9.4 Inspection

```bash
docker network ls
docker network inspect app-net
```

---

## 10. Volumes et persistance

### 10.1 Types

| Type | Cmd | Cas |
|---|---|---|
| **Named volume** | `-v mydata:/var/lib/mysql` | Production, géré par Docker |
| **Bind mount** | `-v $(pwd)/src:/src` | Dev local |
| **tmpfs** | `--tmpfs /tmp` | RAM, éphémère |

### 10.2 Commandes

```bash
docker volume create mydata
docker volume ls
docker volume inspect mydata
docker volume rm mydata
docker volume prune
```

### 10.3 Backup d'un volume

```bash
docker run --rm \
  -v mydata:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/mydata.tgz -C /data .
```

### 10.4 Drivers

NFS, CIFS, AWS EBS, Azure Disk… via plugins. Cas typique : volumes partagés multi-hôtes (Swarm/K8s).

---

## 11. Docker Compose

### 11.1 Concept

Définir une stack multi-conteneurs en YAML, gérer son cycle de vie.

### 11.2 Exemple complet

`compose.yaml` (Compose v2) :

```yaml
services:
  web:
    image: nginx:1.27-alpine
    ports: ["8080:80"]
    depends_on:
      api:
        condition: service_healthy
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
    networks: [app]

  api:
    build:
      context: ./api
      target: runtime
      args: { VERSION: 1.2.3 }
    env_file: .env
    environment:
      DB_HOST: db
    depends_on:
      db:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 10s
      retries: 5
    networks: [app]

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes: [pgdata:/var/lib/postgresql/data]
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "postgres"]
      interval: 5s
    networks: [app]
    secrets: [db_password]

volumes:
  pgdata:

networks:
  app:
    driver: bridge

secrets:
  db_password:
    file: ./secrets/db_password.txt
```

### 11.3 Commandes essentielles

```bash
docker compose up -d
docker compose up --build
docker compose ps
docker compose logs -f api
docker compose exec api sh
docker compose restart web
docker compose down              # stop + remove
docker compose down -v           # + volumes
docker compose config            # affiche le YAML final résolu
docker compose --profile dev up  # profils
```

### 11.4 Override

`compose.override.yaml` est appliqué automatiquement par-dessus `compose.yaml`. Permet d'avoir un comportement dev différent (ports, bind mount du code, etc.).

```bash
docker compose -f compose.yaml -f compose.prod.yaml up -d
```

### 11.5 Profils

```yaml
services:
  debugger:
    image: debug
    profiles: [debug]
```

```bash
docker compose --profile debug up
```

---

## 12. Registry

### 12.1 Registries publics et privés

- **Docker Hub** (par défaut)
- **GHCR** (GitHub Container Registry)
- **ECR** (AWS), **GCR/Artifact Registry** (GCP), **ACR** (Azure)
- **Harbor**, **Nexus**, **Artifactory** (on-prem)

### 12.2 Login / push / pull

```bash
docker login ghcr.io -u USER -p $GHCR_TOKEN
docker tag myapp:1.0 ghcr.io/acme/myapp:1.0
docker push ghcr.io/acme/myapp:1.0
docker pull ghcr.io/acme/myapp:1.0
```

### 12.3 Registry local

```bash
docker run -d --name registry -p 5000:5000 \
  -v $(pwd)/registry-data:/var/lib/registry \
  registry:2
docker tag myapp:1.0 localhost:5000/myapp:1.0
docker push localhost:5000/myapp:1.0
```

### 12.4 Rate limit Docker Hub

Hub anonymes : 100 pulls / 6h par IP. Authentifié : 200. À éviter en CI : utiliser un mirror ou registry interne.

---

## 13. Sécurité

### 13.1 Principes

- **Image minimale** + scan régulier.
- **User non-root** dans le conteneur.
- **Read-only filesystem**.
- **Drop capabilities** + seccomp/AppArmor.
- **Pas de secrets** dans l'image.
- **Signature** des images (cosign).
- **SBOM** (Software Bill of Materials).

### 13.2 Scan d'image

```bash
trivy image myapp:1.0
grype myapp:1.0
docker scout cves myapp:1.0
```

### 13.3 Run sécurisé

```bash
docker run \
  --read-only \
  --tmpfs /tmp \
  --cap-drop ALL \
  --cap-add NET_BIND_SERVICE \
  --security-opt no-new-privileges:true \
  --security-opt seccomp=default.json \
  --user 1000:1000 \
  --pids-limit 100 \
  --memory 256m \
  myapp:1.0
```

### 13.4 Signature avec cosign

```bash
cosign generate-key-pair
cosign sign --key cosign.key ghcr.io/acme/myapp:1.0
cosign verify --key cosign.pub ghcr.io/acme/myapp:1.0
```

### 13.5 SBOM

```bash
syft myapp:1.0 -o spdx-json > sbom.json
docker sbom myapp:1.0
```

### 13.6 Rootless Docker

```bash
dockerd-rootless-setuptool.sh install
```

Mode où le démon tourne en user (pas root). Réduit la surface d'attaque. Limitations : ports < 1024 nécessitent capability, perfs réseau légèrement réduites.

### 13.7 Secrets BuildKit

```dockerfile
# syntax=docker/dockerfile:1.7
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc \
    npm install
```

```bash
DOCKER_BUILDKIT=1 docker build --secret id=npmrc,src=$HOME/.npmrc -t app .
```

Le secret n'apparaît dans **aucun layer** ni `history`.

---

## 14. Optimisation des images

### 14.1 Mesurer

```bash
docker images myapp
docker history myapp:1.0
dive myapp:1.0           # outil tiers fantastique
```

### 14.2 Stratégies

- **Multi-stage** systématique.
- **Distroless** ou **scratch** pour les binaires Go/Rust.
- **Combiner `RUN`** :
  ```dockerfile
  RUN apt-get update && apt-get install -y curl ca-certificates \
      && rm -rf /var/lib/apt/lists/*
  ```
- **Cache mounts** (BuildKit) :
  ```dockerfile
  RUN --mount=type=cache,target=/root/.cache/pip \
      pip install -r requirements.txt
  ```
- **Bind mounts au build** :
  ```dockerfile
  RUN --mount=type=bind,source=requirements.txt,target=/tmp/req.txt \
      pip install -r /tmp/req.txt
  ```
- **Multi-arch** :
  ```bash
  docker buildx create --use
  docker buildx build --platform linux/amd64,linux/arm64 -t myapp:1.0 --push .
  ```

### 14.3 Avant / Après typique

| Stack | Naïf | Optimisé |
|---|---|---|
| Go | 900 MB (golang full) | 12 MB (distroless) |
| Node | 1.2 GB | 180 MB (alpine + prune) |
| Python | 1 GB | 120 MB (slim + multi-stage) |
| Java | 600 MB (jdk) | 200 MB (jre alpine + jlink) |

---

## 15. Debugging

### 15.1 Outils intégrés

```bash
docker logs -f --tail 100 web
docker exec -it web sh
docker inspect web | jq '.[0].State'
docker top web
docker stats
docker events
docker diff web              # changements filesystem vs image
docker cp web:/tmp/data ./   # extraire des fichiers
```

### 15.2 Conteneur de debug

```bash
docker run --rm -it --net container:web nicolaka/netshoot   # diagnostic réseau
docker run --rm -it --pid container:web alpine ps aux
```

### 15.3 Build qui foire

```bash
docker build --progress=plain --no-cache .
docker run --rm -it $(docker build -q --target builder .) sh   # entrer dans une étape
```

### 15.4 Conteneur qui crashe immédiatement

```bash
docker run -it --entrypoint sh myapp:1.0
docker logs --details web
```

---

## 16. Docker en production

### 16.1 Logging

Le default `json-file` peut saturer le disque. Configurer :

```json
// /etc/docker/daemon.json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "5"
  }
}
```

Drivers : `json-file`, `journald`, `syslog`, `gelf`, `fluentd`, `awslogs`, `gcplogs`.

### 16.2 Monitoring

- **cAdvisor** pour métriques par conteneur.
- **Prometheus** scrape cAdvisor + node-exporter.
- **ctop** (TUI léger).
- Endpoint metrics natif daemon : `/metrics`.

### 16.3 Healthcheck

Indispensable. Sans ça, le runtime n'a aucune visibilité sur l'état réel du process.

### 16.4 Update / rollback

Si vous gérez manuellement (sans K8s/Swarm) :

```bash
docker pull myapp:1.1
docker stop web && docker rm web
docker run -d --name web myapp:1.1
```

Mieux : **Docker Swarm**, **Nomad**, ou **Kubernetes**.

### 16.5 Limites de ressources

**Toujours** définir `--memory`, `--cpus` (ou OOM aléatoire).

---

## 17. Alternatives modernes

| Outil | Description |
|---|---|
| **Podman** | Daemonless, compatible CLI Docker, rootless natif, pods façon K8s |
| **Buildah** | Build d'images sans daemon (souvent avec Podman) |
| **Skopeo** | Copie/inspection d'images entre registries |
| **Kaniko** | Build d'images dans un conteneur K8s (CI) |
| **img** | Build OCI sans privilège |
| **nerdctl** | CLI Docker-like pour containerd |
| **containerd direct** | Pour environnements minimaux (K8s) |
| **Buildpacks (CNB)** | Build sans Dockerfile (heuristique) |

### Podman exemple

```bash
podman run -d --name web -p 8080:80 nginx
podman generate kube web > web.yaml      # exporter en manifest K8s
podman play kube web.yaml                # exécuter un manifest K8s localement
```

---

## 18. Migration vers Kubernetes

### 18.1 De Compose vers K8s

**Kompose** :

```bash
kompose convert -f compose.yaml -o k8s/
```

Génère Deployments, Services, PVCs. Bon point de départ, mais à raffiner manuellement (probes, secrets, ressources, Ingress).

### 18.2 Différences clés

| Concept Compose | Équivalent K8s |
|---|---|
| `services:` | Deployment + Service |
| `depends_on` | Init container + probes |
| `volumes:` named | PVC + StorageClass |
| `networks:` | Namespaces + NetworkPolicies |
| `secrets:` | Secret object |
| `ports:` | Service + Ingress |
| `restart:` | restartPolicy |

### 18.3 Modèle hybride

- **Dev** : Compose pour rapidité locale.
- **CI** : Compose pour tests d'intégration.
- **Staging/Prod** : Kubernetes.
- Utiliser **Tilt** ou **Skaffold** pour boucler dev → K8s local rapidement.

---

## Conclusion

Vous avez désormais une maîtrise complète de Docker. Étapes suivantes :

1. Pratiquez les [exercices avancés](./EXERCICES_AVANCES_DOCKER.md).
2. Préparez votre entretien avec le [guide d'interview](./INTERVIEW_DOCKER.md).
3. Maîtrisez **BuildKit**, **buildx**, **cosign**, **trivy**.
4. Passez à Kubernetes en lisant le [guide K8s](../kubernetes/GUIDE_COMPLET.md).

