# Docker — Préparation à l'entretien

> ~55 questions/réponses en français par niveau pour un entretien DevOps avec un focus Docker / conteneurs.

---

## Junior

<details>
<summary>1. Qu'est-ce qu'un conteneur ?</summary>

Un processus isolé du reste du système grâce aux **namespaces** Linux (PID, NET, MNT…) et limité en ressources par les **cgroups**. Il partage le noyau de l'hôte, contrairement à une VM. Léger, démarre en millisecondes.
</details>

<details>
<summary>2. Différence conteneur vs VM ?</summary>

| Critère | VM | Conteneur |
|---|---|---|
| Couche | Hyperviseur | Noyau partagé |
| Taille | GB | MB |
| Démarrage | minutes | secondes |
| Isolation | forte | namespaces |
| Densité | faible | élevée |
</details>

<details>
<summary>3. Qu'est-ce que Docker ?</summary>

Une plateforme et un ensemble d'outils pour construire, distribuer et exécuter des conteneurs. Composée du daemon `dockerd`, du CLI `docker`, du build system (BuildKit), du registry, etc.
</details>

<details>
<summary>4. Différence image vs conteneur ?</summary>

- **Image** : modèle immuable, composé de layers (read-only).
- **Conteneur** : instance en exécution d'une image, avec un layer writable au-dessus.
</details>

<details>
<summary>5. À quoi sert un Dockerfile ?</summary>

C'est la **recette** de build d'une image. Une suite d'instructions (`FROM`, `RUN`, `COPY`, `CMD`…) qui décrivent comment construire les layers de l'image.
</details>

<details>
<summary>6. Que fait `docker run` ?</summary>

= `docker create` + `docker start`. Crée un conteneur à partir d'une image et le démarre. Options principales : `-d` (détaché), `-p` (ports), `-v` (volumes), `-e` (env), `--rm` (auto-suppression).
</details>

<details>
<summary>7. Différence `COPY` vs `ADD` ?</summary>

- `COPY` : copie de fichiers/dossiers depuis le contexte.
- `ADD` : `COPY` + supporte URLs et extraction auto des tar.gz.

Préférer `COPY` (explicite) ; utiliser `ADD` uniquement si on a vraiment besoin de l'extraction.
</details>

<details>
<summary>8. Différence `CMD` vs `ENTRYPOINT` ?</summary>

- `CMD` : commande/arguments par défaut, overridable en ligne de commande.
- `ENTRYPOINT` : commande figée, à laquelle s'ajoutent les arguments du `docker run`.

Pattern combiné :
```dockerfile
ENTRYPOINT ["python","-m","app"]
CMD ["--port","8080"]
```
</details>

<details>
<summary>9. Comment exposer un port ?</summary>

- `EXPOSE 80` dans le Dockerfile : déclaratif (documentation), aucun effet réel.
- `-p 8080:80` au `docker run` : map effectivement le port hôte → conteneur.
</details>

<details>
<summary>10. Comment voir les logs d'un conteneur ?</summary>

```bash
docker logs -f --tail 100 <name>
```
</details>

---

## Intermédiaire

<details>
<summary>11. Qu'est-ce qu'un layer ?</summary>

Une "couche" filesystem produite par une instruction `RUN`, `COPY` ou `ADD`. Les layers sont **immuables**, identifiés par un hash, **partagés** entre images. Le conteneur ajoute un layer writable au-dessus de l'image.
</details>

<details>
<summary>12. Comment optimiser le cache de build ?</summary>

- Ordonner du moins changeant (deps OS, lib) au plus changeant (code source).
- Copier `package.json`/`requirements.txt` puis installer **avant** de copier le code.
- Combiner les `RUN` pour limiter les layers.
- Utiliser BuildKit `--mount=type=cache` pour les caches de paquets.
</details>

<details>
<summary>13. Qu'est-ce qu'un multi-stage build ?</summary>

Plusieurs `FROM` dans un même Dockerfile. La première étape (souvent "builder") compile, la seconde copie uniquement les artefacts nécessaires dans une image minimale. Réduit drastiquement la taille finale et la surface d'attaque.
</details>

<details>
<summary>14. Quand utiliser un bind mount vs un named volume ?</summary>

- **Bind mount** : développement local (synchro code, hot-reload).
- **Named volume** : production (données persistantes gérées par Docker, portables, backupables, drivers).
</details>

<details>
<summary>15. Drivers réseau Docker ?</summary>

`bridge` (défaut), `host`, `none`, `overlay` (Swarm), `macvlan`, `ipvlan`, `container:<name>`.
</details>

<details>
<summary>16. Comment deux conteneurs communiquent-ils ?</summary>

- Sur le **bridge par défaut** : uniquement par IP (DNS non géré). Déconseillé.
- Sur un **bridge custom** (`docker network create`) : DNS automatique par nom de conteneur. **Pratique recommandée**.
- Via **Compose** : tous les services d'un projet partagent un réseau créé automatiquement.
</details>

<details>
<summary>17. Différence entre `docker exec` et `docker attach` ?</summary>

- `docker exec` : démarre **un nouveau process** dans le conteneur.
- `docker attach` : se branche sur le **process PID 1** (stdin/stdout). Quitter peut tuer le conteneur si pas en mode détaché.
</details>

<details>
<summary>18. Pourquoi mes données disparaissent après suppression d'un conteneur ?</summary>

Le layer writable du conteneur est supprimé avec lui. Pour conserver des données, monter un **volume** ou un **bind mount** sur le chemin concerné (ex. `/var/lib/postgresql/data`).
</details>

<details>
<summary>19. Que fait `docker compose up` ?</summary>

Lit `compose.yaml`, construit les images si nécessaire (`build:`), crée le(s) réseau(x), les volumes, puis démarre les services dans l'ordre des `depends_on`. `-d` pour détaché.
</details>

<details>
<summary>20. Comment gérer les variables d'environnement en Compose ?</summary>

- Section `environment:` (statique).
- `env_file:` (depuis un `.env`).
- Substitution `${VAR}` (lit depuis l'environnement shell ou `.env` à la racine).
- Pour les secrets : section `secrets:` (fichier monté en `/run/secrets/`).
</details>

<details>
<summary>21. Comment construire pour ARM (Raspberry Pi) ?</summary>

```bash
docker buildx create --use
docker buildx build --platform linux/arm64 -t myapp:arm64 --load .
```

Pour publier multi-arch :
```bash
docker buildx build --platform linux/amd64,linux/arm64 -t reg/myapp:1.0 --push .
```
</details>

<details>
<summary>22. À quoi sert un HEALTHCHECK ?</summary>

Permet au daemon de connaître l'état applicatif du conteneur (`healthy`/`unhealthy`). Utile pour orchestrateurs (`depends_on: condition: service_healthy`, Swarm replace auto).
</details>

<details>
<summary>23. Que stocke `.dockerignore` ?</summary>

Les chemins **exclus du contexte** envoyé au daemon. Réduit la taille du contexte, accélère les builds, évite d'inclure secrets (`.env`, `.git`, `node_modules`…).
</details>

<details>
<summary>24. Comment limiter les ressources d'un conteneur ?</summary>

```bash
docker run --memory 512m --cpus 1.5 --pids-limit 100 myapp
```

En Compose : `deploy.resources.limits` (Swarm) ou directement `mem_limit`, `cpus` (compose v3.x non swarm).
</details>

<details>
<summary>25. Restart policies disponibles ?</summary>

`no`, `on-failure[:max-retries]`, `always`, `unless-stopped`. Choisir `unless-stopped` pour services prod.
</details>

<details>
<summary>26. Comment passer un secret à un conteneur en build ?</summary>

**BuildKit secrets** :
```dockerfile
# syntax=docker/dockerfile:1.7
RUN --mount=type=secret,id=tok,target=/run/secrets/tok cat /run/secrets/tok
```
```bash
DOCKER_BUILDKIT=1 docker build --secret id=tok,src=$HOME/token.txt .
```

Le secret n'apparaît dans **aucun layer**.
</details>

---

## Senior

<details>
<summary>27. Comment fonctionne l'isolation d'un conteneur Linux ?</summary>

- **Namespaces** : isolation (PID, NET, MNT, UTS, IPC, USER, CGROUP).
- **cgroups v2** : limitation CPU, RAM, IO, PIDs.
- **Capabilities** : sous-ensemble de root (DROP ALL recommandé).
- **Seccomp** : filtre des syscalls.
- **AppArmor/SELinux** : MAC.

Le **kernel reste partagé** : une faille kernel = sortie de conteneur potentielle.
</details>

<details>
<summary>28. Pourquoi un conteneur n'est pas une "sandbox" sécurisée ?</summary>

Il partage le kernel de l'hôte. Une faille kernel ou une mauvaise config (privileged, capabilities trop larges, hostPath…) peut conduire à une évasion. Pour isolation forte : **gVisor**, **Kata Containers** (microVMs), **Firecracker**.
</details>

<details>
<summary>29. Différence entre Docker, containerd et runc ?</summary>

- **runc** : runtime OCI bas niveau, exécute un conteneur depuis un bundle.
- **containerd** : runtime de haut niveau (images, snapshots, networking, CRI pour K8s). Utilise runc.
- **Docker** : suite englobant CLI + dockerd + containerd + build + compose.

Depuis K8s 1.24, Docker n'est plus utilisé directement par K8s (containerd ou CRI-O).
</details>

<details>
<summary>30. Comment fonctionne le réseau bridge par défaut ?</summary>

Docker crée un bridge Linux `docker0`. Chaque conteneur a une veth pair (une dans le namespace réseau, une dans l'hôte attachée au bridge). Le NAT (iptables MASQUERADE) gère la sortie internet. Le port mapping ajoute des règles DNAT.
</details>

<details>
<summary>31. Quelles bonnes pratiques pour réduire la taille d'image ?</summary>

- Image base minimale (alpine, distroless, scratch).
- Multi-stage.
- Combiner les `RUN` et nettoyer (`apt-get clean`, `rm -rf /var/lib/apt/lists/*`).
- `.dockerignore` strict.
- Cache mounts BuildKit pour ne pas garder de cache pip/npm.
- Outils : `dive`, `slim`.
</details>

<details>
<summary>32. Comment gérer les secrets en production ?</summary>

- **Jamais** dans l'image (ils restent dans les layers).
- BuildKit secrets pour le build.
- Variables d'env runtime (orchestrateur).
- Secret stores externes : Vault, AWS SM, GCP SM ; injection via init container ou agent (Vault Agent, External Secrets).
- Pour Compose/Swarm : section `secrets:` (montée en `/run/secrets/`).
</details>

<details>
<summary>33. Comment debugger un conteneur qui crashe au démarrage ?</summary>

```bash
docker logs --details <id>
docker inspect <id> | jq '.[0].State'    # ExitCode, OOMKilled
docker run -it --entrypoint sh <image>   # entrer manuellement
docker events
```

Causes typiques : binaire absent, USER sans permission, env var manquante, port déjà pris, OOM, process en arrière-plan qui sort, mauvais shebang.
</details>

<details>
<summary>34. Comment vérifier la sécurité d'une image ?</summary>

- Scan vulnérabilités : `trivy`, `grype`, `docker scout`.
- SBOM : `syft`, `docker sbom`.
- Lint Dockerfile : `hadolint`.
- Signature : `cosign sign/verify`.
- Politiques admission (Kyverno, OPA) côté cluster.
- Tests de runtime : `falco`, `tracee`.
</details>

<details>
<summary>35. Que vaut Docker Swarm aujourd'hui ?</summary>

Mode natif du daemon Docker pour orchestration multi-hôtes. Simple à mettre en place, mais sa **maintenance est ralentie** depuis le succès de Kubernetes. Adapté pour petites infras, edge, équipes sans besoin K8s. Pour production sérieuse multi-app : K8s/Nomad recommandés.
</details>

<details>
<summary>36. Différence Docker vs Podman ?</summary>

| Critère | Docker | Podman |
|---|---|---|
| Daemon | Oui (dockerd) | Non (fork/exec direct) |
| Rootless | Possible | Natif |
| Compatibilité CLI | référence | `alias docker=podman` |
| Pods | Non (Compose seul) | Oui (concept inspiré K8s) |
| Build | BuildKit | Buildah |
| Compose | Compose v2 | `podman compose` ou Compose v2 via socket |
</details>

<details>
<summary>37. Comment fonctionne BuildKit ?</summary>

Backend de build moderne (par défaut depuis Docker 23). Apports :
- Build parallèle des étapes indépendantes.
- Cache distant (registry).
- Mount cache, mount secret, mount bind.
- Frontends pluggables (`# syntax=`).
- Multi-arch via `buildx`.
- Sortie structurée (`--progress=plain`).
</details>

<details>
<summary>38. Comment fonctionne `docker pull` ?</summary>

1. Resolve nom → tag/digest (registry par défaut Docker Hub).
2. GET manifest (liste des layers et leurs digests).
3. Téléchargement parallèle des layers manquants (chacun vérifié par hash).
4. Décompression et stockage dans le snapshotter (overlay2).
5. Création des références locales.
</details>

<details>
<summary>39. Quelles métriques surveiller pour des conteneurs ?</summary>

- CPU usage (throttling rate +++).
- Memory usage / working set / limit %.
- Network bytes/errors.
- Disk IO.
- Restarts count (OOM ?).
- Health check status.
- Image age et CVE count.

Outils : cAdvisor, Prometheus + node-exporter, ctop, Datadog/NewRelic.
</details>

<details>
<summary>40. Comment scaler Docker sans Kubernetes ?</summary>

- **Docker Swarm** (services, replicas, rolling updates).
- **Nomad** (orchestrateur HashiCorp, multi-workload).
- Reverse proxy + plusieurs hôtes Docker (HAProxy/Traefik).
- Une fois les besoins complexes (autoscaling, multi-tenant, observabilité avancée) → migrer K8s.
</details>

---

## Lead / Architecte

<details>
<summary>41. Quelle stratégie pour gouverner les images en entreprise ?</summary>

- Registry privé (Harbor, Artifactory).
- Politique d'images de base "golden" maintenues par l'équipe plateforme.
- Scan obligatoire en CI ; merge bloqué si CRITICAL.
- Signature cosign + admission policy (`vérifier la signature`).
- SBOM stocké à côté de chaque image.
- Rotation/expiration des tags + immutabilité par digest.
- Quarantaine des images non utilisées.
</details>

<details>
<summary>42. Quels patterns pour la CI/CD Docker ?</summary>

- Build dans CI avec **buildx** et cache distant (registry).
- Tag par SHA commit + branche + version sémantique.
- Tests d'image (`container-structure-test`, `goss`).
- Scan + SBOM avant push.
- Signature cosign.
- Promotion d'image (`dev → staging → prod`) **sans rebuild** (re-tag/re-sign).
- Déploiement via ArgoCD/Flux (GitOps) qui watch les nouvelles digests.
</details>

<details>
<summary>43. Comment éviter "ça marche chez moi" lié à Docker ?</summary>

- Versions pinées (image + deps).
- BuildKit pour reproductibilité.
- Volumes documentés.
- Healthchecks.
- Logs structurés vers stdout.
- Tests d'image (smoke) en CI.
- 12-factor app : config externe, statelessness.
</details>

<details>
<summary>44. Quels sont les inconvénients de "latest" ?</summary>

- Non reproductible.
- Cache rebuild faussé.
- Pas de rollback fiable.
- Sécurité : difficile d'auditer.
- Comportement non déterministe entre clusters/jobs.

Toujours préférer un tag versionné ou un digest.
</details>

<details>
<summary>45. Quelle différence entre la forme exec et la forme shell de CMD ?</summary>

- **Exec** `CMD ["app","--flag"]` : exec direct, PID 1 = app, reçoit SIGTERM. Recommandé.
- **Shell** `CMD app --flag` : exécuté via `/bin/sh -c`, PID 1 = sh, signaux non transmis correctement (zombie processes, shutdowns lents).
</details>

<details>
<summary>46. Pourquoi tini, dumb-init ?</summary>

Init minimaliste à utiliser comme PID 1 pour reaper les processus zombies et propager les signaux quand l'app n'est pas conçue pour. Docker propose `--init` qui injecte `tini` automatiquement.
</details>

<details>
<summary>47. Avantages d'un registry pull-through cache ?</summary>

Mirror local qui cache Docker Hub. Avantages : éviter le rate-limit Docker Hub, accélérer les pulls en CI, fonctionner offline.
</details>

<details>
<summary>48. Que sont les "Buildpacks" ?</summary>

Cloud Native Buildpacks (CNB) construisent des images **sans Dockerfile**, en détectant le langage et appliquant des buildpacks préfabriqués. Avantages : sécurité (images maintenues), reproductibilité, mise à jour de la base sans rebuild des layers app (rebase). CLI : `pack`.
</details>

<details>
<summary>49. Pourquoi distroless est-il populaire ?</summary>

Image Google sans shell, sans gestionnaire de paquets, sans utils OS. Contient juste les libs nécessaires pour exécuter l'app. Surface d'attaque minimale, taille petite (~20 MB). Inconvénient : pas de shell pour debug (utiliser `kubectl debug` ou ephemeral container).
</details>

<details>
<summary>50. Quel futur pour Docker ?</summary>

- Docker reste dominant côté **dev local** (Desktop, Compose).
- En production : containerd/CRI-O ont pris le relais via K8s.
- **WASM** émerge comme runtime alternatif (Docker Desktop + WasmEdge).
- Renforcement chaîne logicielle (cosign, SBOM, attestations SLSA).
- Convergence dev → cloud avec **Dev Containers**, **Docker Build Cloud**.
- Compose v2 + Compose Specification continue d'évoluer comme standard ouvert.
</details>

---

## Conseils

- Mémorisez 10-15 commandes Docker couramment utilisées.
- Ayez un Dockerfile multi-stage optimisé prêt à l'esprit (Go, Node, Python).
- Sachez expliquer **comment** Docker fonctionne (namespaces, cgroups), pas seulement l'utiliser.
- Connaissez Podman, BuildKit, cosign : ce sont des sujets actuels.
- Préparez 1 scénario "j'ai réduit une image de X à Y MB" — toujours apprécié.

