# Jenkins — Préparation à l'entretien

> ~50 questions/réponses en français par niveau pour un entretien CI/CD focalisé Jenkins.

---

## Junior

<details>
<summary>1. Qu'est-ce que Jenkins ?</summary>

Serveur d'automatisation open-source en Java, principalement utilisé pour le CI/CD (build, test, déploiement). Extensible via 1800+ plugins, prend en charge n'importe quel langage et tout type de tâche scriptable.
</details>

<details>
<summary>2. Différence CI vs CD ?</summary>

- **CI** : intégration continue, build + tests automatiques à chaque commit.
- **CD** : *Continuous Delivery* (artefact prêt, déploiement manuel) ou *Continuous Deployment* (déploiement automatique en prod si tests OK).
</details>

<details>
<summary>3. Qu'est-ce qu'un Jenkinsfile ?</summary>

Un fichier (Groovy) versionné dans le repo source décrivant le pipeline CI/CD. Permet le "Pipeline as Code", visible en revue de PR.
</details>

<details>
<summary>4. Pipeline déclarative vs scriptée ?</summary>

- **Déclarative** : syntaxe structurée `pipeline { agent; stages { stage { steps { } } } }`. Recommandée.
- **Scriptée** : Groovy libre `node { stage('x') { ... } }`. Plus flexible, plus risqué.

On peut mélanger avec `script { }` dans une déclarative.
</details>

<details>
<summary>5. À quoi sert un agent ?</summary>

Une machine (ou conteneur) qui exécute les builds. Le controller dispatche les jobs vers les agents, identifiés par des labels. `agent { label 'linux && docker' }`.
</details>

<details>
<summary>6. Différence stage vs step ?</summary>

- **Stage** : étape logique du pipeline (Build, Test, Deploy). Visible dans l'UI.
- **Step** : action concrète (sh, sh, archiveArtifacts).
</details>

<details>
<summary>7. À quoi servent les triggers ?</summary>

Déclencher automatiquement un build : `cron`, `pollSCM`, `githubPush()`, `upstream`. Évite la nécessité de lancer manuellement.
</details>

<details>
<summary>8. Comment passer un paramètre à un job ?</summary>

Section `parameters { string(name: 'X', defaultValue: 'y') }` puis `${params.X}`. À l'UI, le build affiche un formulaire avant exécution.
</details>

<details>
<summary>9. Que fait `post { always { } }` ?</summary>

Bloc exécuté en fin de pipeline (peu importe le résultat). Variantes : `success`, `failure`, `unstable`, `aborted`, `changed`, `cleanup`.
</details>

<details>
<summary>10. Comment archiver les artefacts ?</summary>

```groovy
archiveArtifacts artifacts: 'dist/*.tar.gz', fingerprint: true
```
</details>

---

## Intermédiaire

<details>
<summary>11. Qu'est-ce qu'un Multibranch Pipeline ?</summary>

Type de job qui scanne un repo Git et crée automatiquement un sous-job par **branche** (et par **PR**) ayant un `Jenkinsfile`. Idéal pour le workflow GitFlow / trunk-based.
</details>

<details>
<summary>12. Comment exécuter des stages en parallèle ?</summary>

```groovy
stage('Tests') {
    parallel {
        stage('Unit')  { steps { sh 'mvn test' } }
        stage('Lint')  { steps { sh 'mvn checkstyle:check' } }
    }
}
```
</details>

<details>
<summary>13. Qu'est-ce qu'une Shared Library ?</summary>

Code Groovy partagé entre plusieurs Jenkinsfiles. Structure `vars/` (étapes globales `dockerBuildAndPush`), `src/` (classes), `resources/`. Importée avec `@Library('lib@version') _`.
</details>

<details>
<summary>14. Comment gérer des secrets dans un pipeline ?</summary>

`withCredentials([...])` (`usernamePassword`, `string`, `file`, `sshUserPrivateKey`). Les valeurs sont masquées dans les logs. Pour des secrets externes : plugin **HashiCorp Vault**, **AWS Credentials**.
</details>

<details>
<summary>15. Différence agent any / docker / kubernetes ?</summary>

- `any` : n'importe quel agent disponible.
- `docker { image 'x' }` : agent existant + step exécuté dans un conteneur.
- `kubernetes { yaml '...' }` : provisionne un Pod K8s avec un ou plusieurs conteneurs.
</details>

<details>
<summary>16. À quoi sert `when` ?</summary>

Conditionne l'exécution d'un stage : branche, tag, expression, environnement, changeset. Évite des stages inutiles.
</details>

<details>
<summary>17. À quoi sert `tools` ?</summary>

Installation auto d'outils déclarés dans Manage Jenkins → Tools (JDK, Maven, NodeJS). Pratique sans Docker mais peu reproductible. Préférer un agent containerisé.
</details>

<details>
<summary>18. Qu'est-ce que la sandbox Groovy ?</summary>

Restriction d'exécution Groovy/Java pour les pipelines : empêche l'accès à des méthodes sensibles (filesystem, threads, reflection). Les méthodes non-whitelistées doivent être approuvées par un admin. **Désactiver = donner root sur le controller**.
</details>

<details>
<summary>19. Comment déclencher un autre job ?</summary>

```groovy
build job: 'other-job', parameters: [string(name: 'VERSION', value: '1.2')], wait: true
```
</details>

<details>
<summary>20. Comment publier des résultats de tests ?</summary>

```groovy
post { always { junit '**/target/surefire-reports/*.xml' } }
```

Plus plugins pour coverage (`recordCoverage`), HTML reports (`publishHTML`), SonarQube.
</details>

<details>
<summary>21. Comment intégrer un webhook GitHub ?</summary>

1. Plugin GitHub.
2. GitHub repo → Settings → Webhooks → URL `https://<jenkins>/github-webhook/`.
3. `triggers { githubPush() }` (ou natif Multibranch).
</details>

<details>
<summary>22. À quoi sert Blue Ocean ?</summary>

UI moderne visualisant les pipelines (étapes parallèles, navigation par stage). Bien pour onboarding. Développement ralenti, ne pas en faire l'unique interface.
</details>

<details>
<summary>23. Comment gérer la rétention des builds ?</summary>

```groovy
options { buildDiscarder(logRotator(numToKeepStr: '20', daysToKeepStr: '60')) }
```
</details>

<details>
<summary>24. Qu'est-ce que `input` ?</summary>

Met en pause le pipeline et demande une confirmation humaine. Combiner avec `timeout`. Permet aussi de demander des paramètres au moment du run.
</details>

<details>
<summary>25. Différence freestyle vs pipeline ?</summary>

| Critère | Freestyle | Pipeline |
|---|---|---|
| Config | UI uniquement | Jenkinsfile (Git) |
| Versionnable | Non | Oui |
| Complexité supportée | Faible | Élevée (boucles, parallel, fonctions) |
| Multibranch | Non | Oui |

Freestyle = pour prototyper ; Pipeline = pour le sérieux.
</details>

---

## Senior / Lead

<details>
<summary>26. Comment scaler Jenkins ?</summary>

- Aucun build sur le controller (`numExecutors: 0`).
- **Agents éphémères Kubernetes** : un Pod par build.
- Plusieurs controllers indépendants par équipe si nécessaire (Jenkins n'est pas multi-master HA).
- Optimiser JVM (`-Xmx`, G1GC).
- Stocker `JENKINS_HOME` sur SSD rapide.
- Cache build (registry Docker, Maven repo Nexus).
</details>

<details>
<summary>27. Comment sécuriser un Jenkins ?</summary>

- Auth via OIDC/LDAP (jamais d'auth locale en prod).
- Role-Based Authorization (moindre privilège).
- TLS sur le front, reverse proxy.
- Sandbox Groovy activée.
- Aucun job sur le controller.
- Plugins maintenus uniquement, MAJ régulière (CVE fréquentes).
- Audit Trail.
- Approbation de méthodes Groovy en revue.
- Backups + tests de restauration de `JENKINS_HOME`.
</details>

<details>
<summary>28. Qu'est-ce que JCasC ?</summary>

**Jenkins Configuration as Code** : décrit toute la config Jenkins (sécurité, agents, plugins, credentials, libraries…) en YAML. Permet un Jenkins **reproductible**, versionné, redéployable. Indispensable en prod sérieuse.
</details>

<details>
<summary>29. Qu'est-ce que le Job DSL ?</summary>

Plugin pour décrire des jobs (Freestyle, Pipeline, Multibranch…) en Groovy. Couplé à JCasC, on a une infra Jenkins 100% as code.
</details>

<details>
<summary>30. Différence Declarative vs Scripted ?</summary>

- **Declarative** : structure stricte, validable, lisible, blocs `pipeline/agent/stages/post`. Recommandée.
- **Scripted** : Groovy libre, plus puissant mais plus risqué (besoin de tests, sandbox plus permissive parfois nécessaire).
- On peut "ouvrir" du scripted dans une declarative via `script { }`.
</details>

<details>
<summary>31. Comment versionner une Shared Library ?</summary>

Tagger en SemVer (`v1.2.0`) et **épingler** dans le Jenkinsfile : `@Library('shared-lib@v1.2.0') _`. Jamais `@main` en prod (risque de breaking change involontaire).
</details>

<details>
<summary>32. Comment debugger un pipeline qui freeze ?</summary>

- Vérifier les agents : labels disponibles, capacity, queue.
- `Manage Jenkins → System Information`.
- `Manage Jenkins → Script Console` pour Groovy ad-hoc.
- Thread dump (`/threadDump`).
- Logs : `jenkins.log` + logs par build.
- Si `input` : sur la page du build, cliquer Resume/Abort.
</details>

<details>
<summary>33. Comment intégrer Jenkins dans une approche GitOps ?</summary>

- Jenkins build + pousse l'image.
- ArgoCD/Flux watch un repo manifest qui contient l'image tag.
- Jenkins met à jour le manifest (PR ou commit auto) → ArgoCD synchronise.
- Évite que Jenkins fasse du `kubectl apply` direct.
</details>

<details>
<summary>34. Jenkins vs GitHub Actions ?</summary>

| Critère | Jenkins | GitHub Actions |
|---|---|---|
| Hosting | Self-hosted (souvent) | SaaS (ou self runners) |
| Langage pipeline | Groovy | YAML |
| Maintenance | Élevée (plugins) | Faible |
| Flexibilité | Très élevée | Élevée (marketplace) |
| Coût | Infra + ops | Per-minute / inclus |
| Multi-cloud | Oui | Oui (runners self-hosted) |
| Communauté | Mature | En forte croissance |

Choix : héritage Jenkins + besoin de flexibilité → garder. Greenfield + repo GitHub → Actions.
</details>

<details>
<summary>35. Plugins essentiels à connaître ?</summary>

- Pipeline, Pipeline: Multibranch
- Configuration as Code
- Job DSL
- Kubernetes
- Docker Pipeline
- Credentials Binding
- HashiCorp Vault
- Slack Notification
- SonarQube Scanner
- GitHub / GitLab / Bitbucket Branch Source
- Blue Ocean
- Role-Based Authorization Strategy
- Prometheus metrics
- Audit Trail
</details>

<details>
<summary>36. Pourquoi éviter d'exécuter des builds sur le controller ?</summary>

- Sécurité : un build malveillant peut compromettre tout Jenkins.
- Performance : concurrence sur la JVM, IO, mémoire.
- Stabilité : un build qui plante peut tuer Jenkins.

Configurer `numExecutors: 0` sur le controller.
</details>

<details>
<summary>37. Comment migrer de freestyle à pipeline ?</summary>

1. Auditer les jobs existants.
2. Repérer les patterns récurrents → shared library.
3. Convertir job par job, valider en parallèle.
4. Décommissionner les freestyles une fois les pipelines stables.
5. Pour gros volumes : générateur Job DSL ou script ad-hoc lisant `config.xml`.
</details>

<details>
<summary>38. Comment gérer plusieurs environnements (dev/staging/prod) dans un pipeline ?</summary>

- Stages conditionnels via `when { branch ... }` ou via paramètre `ENV`.
- Credentials différents par environnement (folders).
- Kubeconfigs/Vault namespaces séparés.
- Étape `input` d'approbation avant prod.
- Stratégie GitOps : Jenkins met à jour les overlays Kustomize par env.
</details>

<details>
<summary>39. Quelles métriques superviser pour Jenkins ?</summary>

- **Queue length** (waiting jobs).
- **Executors busy / total**.
- **Build duration** par job.
- **Failure rate**.
- **Disk usage** de `JENKINS_HOME`.
- **JVM** (heap, GC pauses).
- **Plugin failures** (Manage Jenkins → System Logs).

Via plugin Prometheus + Grafana + alerting.
</details>

<details>
<summary>40. Comment auditer les actions sur Jenkins ?</summary>

Plugin **Audit Trail** + export vers SIEM. Logge auth, modifs config, lancement de jobs. Couplé à un reverse proxy avec logs détaillés.
</details>

---

## Lead / Architecte

<details>
<summary>41. Quelle stratégie de backup recommandez-vous ?</summary>

- Snapshot complet `JENKINS_HOME` (avec exclusion de `workspaces/`, `caches/`).
- Plugin **ThinBackup** ou cron + rsync.
- Tests de restauration mensuels.
- Versionner la **config** via JCasC (déjà en Git).
- Stockage hors-cluster (S3, NAS distant).
</details>

<details>
<summary>42. Comment monter en charge des centaines de builds simultanés ?</summary>

- Plugin **Kubernetes** : autoscaling des Pods builders.
- Caches partagés (Nexus/Artifactory pour deps, registry Docker mirror).
- Découpage en plusieurs controllers (par BU).
- Build-cache distant (BuildKit `--cache-to/--cache-from`).
- Optimiser les Jenkinsfile (parallel, skip si pas de changements).
</details>

<details>
<summary>43. Comment garantir la reproductibilité d'un build ?</summary>

- Agents containerisés (image versionnée).
- Versions outils pinées (`tools` ou Docker).
- Dependency lock files (package-lock.json, Pipfile.lock, go.sum).
- Cache reproductible (Bazel, Nix).
- Pas d'`installation` à la volée non versionnée.
</details>

<details>
<summary>44. Quel est l'avenir de Jenkins ?</summary>

- Toujours dominant en self-hosted enterprise.
- Concurrence forte de GitHub Actions / GitLab CI / Tekton.
- Stratégie de la fondation : moderniser (Java moderne, plugin BOM, sécurité).
- Pour greenfield, on choisit rarement Jenkins. Pour legacy, il reste indispensable.
- Forks et alternatives : **Jenkins X** (cloud-native, GitOps-first) a perdu en traction.
</details>

<details>
<summary>45. Scénario : un Jenkinsfile met 45 min, vous devez l'accélérer. Approche ?</summary>

1. Profiler chaque stage (`timestamps`, `time` shell).
2. Paralléliser tests (unit, lint, IT).
3. Caches build (Maven `~/.m2`, Gradle, pip, npm) via volumes ou cache manager.
4. Image Docker build cache (`--cache-to/--cache-from registry`).
5. `when changeset` pour ne builder que ce qui change.
6. Réduire la phase Docker (multi-stage optimisé).
7. Agents plus puissants ou plus nombreux.
8. Skip des étapes coûteuses sur PR (lancer en nightly).
</details>

<details>
<summary>46. Comment tester un Jenkinsfile sans tout casser ?</summary>

- `Replay` : modifier le pipeline d'un build précédent et le rejouer.
- Plugin **Pipeline Linter** (`curl -X POST -F "jenkinsfile=<Jenkinsfile" /pipeline-model-converter/validate`).
- `jenkins-pipeline-unit` : framework de tests unitaires Groovy.
- Branches de test + multibranch.
- Lab/staging Jenkins dédié.
</details>

<details>
<summary>47. Quels patterns pour des pipelines DRY ?</summary>

- Shared Library (étapes globales `vars/` + classes `src/`).
- Templates de Jenkinsfile minimaux qui n'invoquent que des steps de la library.
- JCasC pour la config commune.
- Job DSL générant des jobs à partir d'un catalogue.
</details>

<details>
<summary>48. Pourquoi un agent Kubernetes éphémère ?</summary>

- Pas de "snowflake" agent.
- Zéro maintenance OS / dépendances.
- Scale élastique.
- Coût optimal (paye à la seconde).
- Isolation totale entre builds.
- Reproductibilité (image versionnée).
</details>

<details>
<summary>49. Que faire en cas de plugin défaillant après upgrade ?</summary>

1. Vérifier les logs (Manage Jenkins → System Logs).
2. Rollback du plugin (Manage Plugins → Installed → uninstall + installer la version précédente depuis le .hpi).
3. Restaurer un backup de `JENKINS_HOME/plugins/`.
4. Si plugin critique : escalader sur la mailing list Jenkins / GitHub issues.
5. Long terme : maintenir un **plugin BOM** + tests d'upgrade en staging.
</details>

<details>
<summary>50. Comment formerez-vous une équipe à Jenkins ?</summary>

1. Workshop Jenkinsfile déclaratif (atelier hands-on).
2. Catalogue de Shared Library documenté.
3. Templates de Jenkinsfile par stack (Java, Node, Python).
4. Pair-programming pour les premiers pipelines.
5. Revue obligatoire des Jenkinsfile en PR.
6. Documentation interne (Confluence, Backstage).
7. Suivi des métriques (durée, taux d'échec) pour amélioration continue.
</details>

---

## Conseils

- Connaissez **par cœur** la structure d'un Jenkinsfile déclaratif.
- Ayez un exemple de **Shared Library** à présenter.
- Sachez parler **agents Kubernetes** + **JCasC**.
- Préparez 1 anecdote de migration / d'optimisation de pipeline.
- Comprenez les **trade-offs** Jenkins vs alternatives modernes.

