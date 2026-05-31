# Jenkins — Le Guide Complet : de 0 à Hero

> Livre de référence en français pour maîtriser Jenkins et le CI/CD, du job freestyle aux pipelines as code en production.

---

## Table des matières

1. [Introduction au CI/CD et place de Jenkins](#1-introduction)
2. [Architecture Jenkins](#2-architecture)
3. [Installation](#3-installation)
4. [Configuration globale](#4-configuration)
5. [Jobs Freestyle](#5-jobs-freestyle)
6. [Pipelines déclaratives](#6-pipelines-déclaratives)
7. [Pipelines scriptées](#7-pipelines-scriptées)
8. [Jenkinsfile en pratique](#8-jenkinsfile-en-pratique)
9. [Shared Libraries](#9-shared-libraries)
10. [Agents et nœuds](#10-agents-et-nœuds)
11. [Credentials et secrets](#11-credentials-et-secrets)
12. [Intégrations](#12-intégrations)
13. [Tests et qualité](#13-tests-et-qualité)
14. [Déploiements](#14-déploiements)
15. [Blue Ocean](#15-blue-ocean)
16. [Jenkins as Code (JCasC, Job DSL)](#16-jenkins-as-code)
17. [Sécurité Jenkins](#17-sécurité)
18. [Scaling et HA](#18-scaling-et-ha)
19. [Monitoring](#19-monitoring)
20. [Migration et alternatives modernes](#20-migration-et-alternatives)

---

## 1. Introduction

### 1.1 CI / CD

- **CI** (Continuous Integration) : à chaque commit, build + tests automatisés.
- **CD** (Continuous Delivery) : pipeline jusqu'à un artefact déployable, déploiement en prod sur action humaine.
- **CD** (Continuous Deployment) : déploiement en prod automatique si les tests passent.

### 1.2 Place de Jenkins

Jenkins est un serveur d'automatisation **open-source** (sous Apache 2.0), né en 2011 (fork de Hudson). Il est :

- **Le pionnier** du CI/CD self-hosted, encore très répandu en entreprise (banques, télécoms, industriel).
- **Plugin-driven** : 1800+ plugins.
- **Langage agnostique** : tout ce qui s'exécute en shell/PowerShell peut être orchestré.

### 1.3 Comparaison

| Outil | Hébergement | Pipeline as code | Force | Faiblesse |
|---|---|---|---|---|
| **Jenkins** | Self-hosted (ou managé) | Groovy (Jenkinsfile) | Flexibilité, plugins, communauté | Maintenance plugin, JVM |
| **GitHub Actions** | SaaS GitHub | YAML | Intégration GitHub, écosystème actions | Limité à GitHub |
| **GitLab CI** | SaaS ou self-hosted | YAML (`.gitlab-ci.yml`) | Tout-en-un GitLab | Couplé à GitLab |
| **CircleCI** | SaaS principalement | YAML | UX moderne | Coût |
| **Argo Workflows** | K8s | YAML | Cloud-native | Complexité |
| **Tekton** | K8s | YAML CRD | Cloud-native, modulaire | Maturité UX |
| **Drone** | Self-hosted/SaaS | YAML | Léger | Communauté plus restreinte |

---

## 2. Architecture

```
                        ┌───────────────────────┐
                        │  Controller (master)  │
                        │   - UI / API REST     │
                        │   - Scheduler         │
                        │   - JENKINS_HOME      │
                        └─────┬─────────┬───────┘
                              │ JNLP/SSH│
                              │         │
                  ┌───────────▼─┐    ┌──▼───────────┐
                  │   Agent A   │    │   Agent B    │
                  │   (Linux)   │    │   (Windows)  │
                  │ Workspaces, │    │              │
                  │ Tools, ...  │    │              │
                  └─────────────┘    └──────────────┘
```

- **Controller** : UI, configuration, scheduling. **Ne doit pas** exécuter de jobs en prod (sécurité).
- **Agents** : exécutent les builds. Connexion via JNLP, SSH, ou injectés par des clouds (EC2, Kubernetes…).
- **JENKINS_HOME** : config, plugins, jobs, builds. À sauvegarder !

---

## 3. Installation

### 3.1 Docker (le plus simple)

```bash
docker run -d --name jenkins \
  -p 8080:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  jenkins/jenkins:lts
docker logs jenkins | grep "Please use the following password"
```

### 3.2 Kubernetes (Helm)

```bash
helm repo add jenkins https://charts.jenkins.io
helm install jenkins jenkins/jenkins -n jenkins --create-namespace \
  --set controller.adminPassword=ChangeMe \
  --set persistence.size=20Gi
kubectl port-forward svc/jenkins -n jenkins 8080:8080
```

### 3.3 Linux package

```bash
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | \
  sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/" | \
  sudo tee /etc/apt/sources.list.d/jenkins.list
sudo apt update && sudo apt install -y jenkins
sudo systemctl enable --now jenkins
```

### 3.4 Configuration initiale

1. `http://localhost:8080`
2. Coller le mot de passe initial.
3. **Install suggested plugins**.
4. Créer un admin.
5. Configurer l'URL externe.

---

## 4. Configuration

### 4.1 Manage Jenkins

| Section | Rôle |
|---|---|
| **System** | URL, executors, env vars globales, agents cloud |
| **Tools** | JDK, Maven, Gradle, Node, Git (auto-install ou path) |
| **Plugins** | Installer/MAJ/désinstaller |
| **Credentials** | Stocker secrets (user/pass, SSH, tokens, secret text, certificate) |
| **Security** | Realm (LDAP, OIDC), authorization (Matrix, Role-based) |
| **Nodes** | Déclarer des agents |
| **Configuration as Code** | YAML (JCasC) |

### 4.2 Credentials

Domaines et scopes :
- **Global** : visible par tous les jobs.
- **System** : visible uniquement par Jenkins (pas les jobs).
- **Folder** : limité à un dossier.

Types : `Username/Password`, `SSH Username with private key`, `Secret text`, `Secret file`, `Certificate`, `Docker host certs`, etc.

### 4.3 Sécurité

- Activer **CSRF** protection (par défaut).
- Auth via **OIDC** ou **LDAP** (jamais d'auth locale en prod).
- Autorisation **Role-Based Strategy** (plugin).
- Désactiver l'**inscription des utilisateurs**.

---

## 5. Jobs Freestyle

Interface web pour créer un job sans Jenkinsfile.

**Quand l'utiliser** : prototypage, tâches one-shot, équipes non-dev. Sinon, préférer **Pipeline**.

Anatomie :
- **SCM** : Git (URL + credentials + branche).
- **Build triggers** : poll SCM, webhook, cron, downstream/upstream.
- **Build environment** : delete workspace, set env vars, withCredentials.
- **Build steps** : Execute shell, Invoke Maven, Run Docker…
- **Post-build** : Publish JUnit, archive artifacts, send mail, trigger another job.

---

## 6. Pipelines déclaratives

Le **format moderne**, à privilégier.

### 6.1 Structure complète

```groovy
pipeline {
    agent { label 'linux && docker' }

    options {
        timeout(time: 30, unit: 'MINUTES')
        timestamps()
        ansiColor('xterm')
        buildDiscarder(logRotator(numToKeepStr: '20'))
        disableConcurrentBuilds()
    }

    parameters {
        string(name: 'VERSION', defaultValue: '1.0.0', description: 'Version à déployer')
        choice(name: 'ENV', choices: ['dev','staging','prod'], description: 'Environnement')
        booleanParam(name: 'DRY_RUN', defaultValue: false)
    }

    environment {
        APP_NAME = 'monapp'
        REGISTRY = 'ghcr.io/acme'
        DOCKER_CREDS = credentials('ghcr-token')   // injecte _USR et _PSW
    }

    triggers {
        cron('H 2 * * *')                  // nightly
        pollSCM('H/15 * * * *')            // toutes 15 min
        // githubPush()                    // via plugin
    }

    tools {
        jdk 'temurin-21'
        maven 'maven-3.9'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.GIT_SHA = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
                }
            }
        }

        stage('Build') {
            steps {
                sh 'mvn -B -DskipTests package'
            }
            post {
                always {
                    archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
                }
            }
        }

        stage('Test') {
            parallel {
                stage('Unit') {
                    steps { sh 'mvn -B test' }
                    post { always { junit '**/target/surefire-reports/*.xml' } }
                }
                stage('Lint') {
                    steps { sh 'mvn -B checkstyle:check' }
                }
            }
        }

        stage('Docker build & push') {
            when { branch 'main' }
            steps {
                sh """
                    docker build -t ${REGISTRY}/${APP_NAME}:${GIT_SHA} .
                    echo \$DOCKER_CREDS_PSW | docker login ${REGISTRY} -u \$DOCKER_CREDS_USR --password-stdin
                    docker push ${REGISTRY}/${APP_NAME}:${GIT_SHA}
                """
            }
        }

        stage('Deploy') {
            when {
                allOf {
                    branch 'main'
                    expression { params.ENV != 'prod' || currentBuild.changeSets.size() > 0 }
                }
            }
            steps {
                input message: "Déployer en ${params.ENV} ?", ok: 'Go'
                sh "kubectl set image deploy/${APP_NAME} app=${REGISTRY}/${APP_NAME}:${GIT_SHA} -n ${params.ENV}"
            }
        }
    }

    post {
        success { slackSend channel: '#deploys', message: "✅ Build #${BUILD_NUMBER} OK" }
        failure { slackSend channel: '#deploys', message: "❌ Build #${BUILD_NUMBER} KO : ${BUILD_URL}" }
        always { cleanWs() }
    }
}
```

### 6.2 Directives clés

| Directive | Rôle |
|---|---|
| `agent` | Où exécuter (`any`, `none`, `label '...'`, `docker { image 'x' }`, `kubernetes { yaml ... }`) |
| `stages` / `stage` / `steps` | Découpage logique |
| `environment` | Variables d'env (au niveau pipeline ou stage) |
| `options` | Timeout, ansi, retention, etc. |
| `parameters` | Paramètres demandés au lancement |
| `triggers` | Cron, polling, webhooks |
| `tools` | Outils auto-installés (JDK, Maven…) |
| `when` | Condition d'exécution d'un stage |
| `parallel` | Stages parallèles |
| `post` | `always`, `success`, `failure`, `unstable`, `cleanup` |

### 6.3 `when` conditions

```groovy
when {
    anyOf {
        branch 'main'
        tag 'v*'
    }
    not { changeRequest() }
    environment name: 'DEPLOY', value: 'true'
    expression { fileExists('Dockerfile') }
}
```

---

## 7. Pipelines scriptées

Syntaxe Groovy plus libre, sans `pipeline { }`.

```groovy
node('linux') {
    def img
    stage('Checkout') { checkout scm }
    stage('Build') {
        img = docker.build("acme/app:${env.BUILD_NUMBER}")
    }
    stage('Test') {
        img.inside { sh 'pytest' }
    }
    stage('Push') {
        docker.withRegistry('https://ghcr.io', 'ghcr-token') {
            img.push()
            img.push('latest')
        }
    }
}
```

À utiliser quand on a besoin de **logique avancée** non exprimable en déclaratif (boucles, fonctions, exceptions complexes). Dans un Jenkinsfile déclaratif, on peut "ouvrir une fenêtre" sur le scripted via `script { ... }`.

---

## 8. Jenkinsfile en pratique

### 8.1 Multibranch Pipeline

Jenkins scanne un repo Git, détecte automatiquement les branches (et PRs avec plugin GitHub/GitLab) ayant un `Jenkinsfile`, crée un job par branche/PR.

Configuration :
- New Item → Multibranch Pipeline
- Branch Sources : GitHub / Git / Bitbucket
- Périodicité de scan
- Build strategies (toutes branches / PR seulement / tags)

### 8.2 Patterns utiles

```groovy
// Numéro de version sémantique
script {
    def commits = sh(returnStdout: true, script: 'git rev-list --count HEAD').trim()
    env.VERSION = "1.0.${commits}+${env.GIT_SHA}"
}

// Retry + timeout
retry(3) {
    timeout(time: 5, unit: 'MINUTES') {
        sh './flaky.sh'
    }
}

// Matrix builds
matrix {
    axes {
        axis { name 'OS'; values 'ubuntu', 'alpine' }
        axis { name 'JDK'; values '17', '21' }
    }
    stages {
        stage('Build') { steps { sh "echo Building on ${OS} with JDK ${JDK}" } }
    }
}
```

---

## 9. Shared Libraries

Factorisez la logique pipeline pour la **partager entre repos**.

### 9.1 Structure

```
shared-lib/
├── vars/
│   ├── dockerBuildAndPush.groovy
│   └── slackNotify.groovy
├── src/
│   └── com/acme/
│       └── Helpers.groovy
└── resources/
    └── templates/
```

`vars/dockerBuildAndPush.groovy` :

```groovy
def call(Map config = [:]) {
    def image = config.image ?: error('image manquante')
    def tag   = config.tag   ?: env.BUILD_NUMBER
    def creds = config.creds ?: 'ghcr-token'

    docker.withRegistry("https://${config.registry ?: 'ghcr.io'}", creds) {
        def img = docker.build("${image}:${tag}")
        img.push()
        if (config.alsoLatest) { img.push('latest') }
    }
}
```

### 9.2 Déclaration

**Globalement** : Manage Jenkins → Configure → Global Pipeline Libraries.

**Au niveau Jenkinsfile** :

```groovy
@Library('shared-lib@main') _   // _ requis !

pipeline {
    agent any
    stages {
        stage('Build & push') {
            steps {
                dockerBuildAndPush(image: 'acme/app', tag: env.GIT_SHA, alsoLatest: env.BRANCH_NAME == 'main')
            }
        }
    }
}
```

### 9.3 Versionnage

Toujours **épingler une version** (tag/SHA) en production, jamais `@main`.

---

## 10. Agents et nœuds

### 10.1 Types d'agents

| Type | Cas |
|---|---|
| **SSH** | Linux/Unix permanent |
| **JNLP / inbound** | Agent qui appelle le controller (pratique derrière NAT) |
| **EC2 cloud** | Provisioning à la demande sur AWS |
| **Kubernetes cloud** | Pod par build, scalable, économique |
| **Docker cloud** | Conteneur par build |

### 10.2 Labels

Chaque agent porte des labels (`linux`, `docker`, `gpu`). Les jobs ciblent : `agent { label 'linux && docker' }`.

### 10.3 Kubernetes plugin (recommandé)

Définit des **pod templates** :

```yaml
# Configuration JCasC simplifiée
clouds:
  - kubernetes:
      name: k8s
      jenkinsUrl: http://jenkins:8080
      templates:
        - name: builder
          label: builder
          containers:
            - name: jnlp
              image: jenkins/inbound-agent:latest
            - name: docker
              image: docker:24
              command: cat
              ttyEnabled: true
              privileged: true
```

Dans Jenkinsfile :

```groovy
pipeline {
    agent {
        kubernetes {
            yaml '''
              apiVersion: v1
              kind: Pod
              spec:
                containers:
                - name: maven
                  image: maven:3.9-eclipse-temurin-21
                  command: ['cat']
                  tty: true
                - name: docker
                  image: docker:24
                  command: ['cat']
                  tty: true
                  volumeMounts:
                  - name: docker-sock
                    mountPath: /var/run/docker.sock
                volumes:
                - name: docker-sock
                  hostPath:
                    path: /var/run/docker.sock
            '''
        }
    }
    stages {
        stage('Build') {
            steps {
                container('maven') { sh 'mvn -B package' }
                container('docker') { sh 'docker build -t app:1 .' }
            }
        }
    }
}
```

**Avantages** : agents éphémères, isolation totale, coût optimisé, pas de "snowflake nodes".

---

## 11. Credentials et secrets

### 11.1 `withCredentials`

```groovy
withCredentials([
    usernamePassword(credentialsId: 'ghcr', usernameVariable: 'U', passwordVariable: 'P'),
    string(credentialsId: 'slack-webhook', variable: 'SLACK_URL'),
    file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG'),
    sshUserPrivateKey(credentialsId: 'deploy-key', keyFileVariable: 'KEY', usernameVariable: 'USER')
]) {
    sh '''
      echo "$P" | docker login ghcr.io -u "$U" --password-stdin
      curl -X POST -d 'OK' "$SLACK_URL"
      kubectl apply -f k8s/
    '''
}
```

Les variables sont **masquées dans les logs** (***).

### 11.2 HashiCorp Vault

Plugin **HashiCorp Vault** :

```groovy
withVault(configuration: [vaultUrl: 'https://vault.example.com',
                          vaultCredentialId: 'vault-approle'],
          vaultSecrets: [[path: 'secret/data/app/prod',
                          secretValues: [[envVar: 'DB_PASSWORD', vaultKey: 'db_pwd']]]]) {
    sh 'deploy.sh'
}
```

### 11.3 AWS

Plugin **AWS Credentials** + IAM Role :

```groovy
withAWS(credentials: 'aws-creds', region: 'eu-west-1') {
    sh 'aws s3 sync ./build s3://bucket/'
}
```

---

## 12. Intégrations

| Outil | Plugin / méthode |
|---|---|
| **GitHub** | GitHub Integration + webhook |
| **GitLab** | GitLab Plugin + webhook |
| **Bitbucket** | Bitbucket Plugin |
| **SonarQube** | SonarQube Scanner |
| **Nexus / Artifactory** | Maven/Gradle natif + plugins |
| **Slack** | Slack Notification |
| **Email** | Email Extension |
| **JIRA** | JIRA Plugin (mise à jour de tickets) |
| **Datadog** | Datadog Plugin (metrics, traces) |
| **Prometheus** | Prometheus metrics plugin (`/prometheus`) |

### Webhook GitHub

1. Plugin GitHub installé.
2. Repo GitHub → Settings → Webhooks → `http://jenkins/github-webhook/`.
3. Dans le job : `triggers { githubPush() }`.

---

## 13. Tests et qualité

### 13.1 Publier les résultats JUnit

```groovy
post {
    always {
        junit testResults: '**/target/surefire-reports/*.xml', allowEmptyResults: false
    }
}
```

### 13.2 Coverage

```groovy
post {
    always {
        publishHTML([reportDir: 'coverage', reportFiles: 'index.html', reportName: 'Coverage'])
        recordCoverage(tools: [[parser: 'JACOCO']])   // plugin Coverage
    }
}
```

### 13.3 SonarQube

```groovy
stage('SonarQube') {
    steps {
        withSonarQubeEnv('sonar-prod') {
            sh 'mvn -B sonar:sonar'
        }
        timeout(time: 10, unit: 'MINUTES') {
            waitForQualityGate abortPipeline: true
        }
    }
}
```

### 13.4 Archiver les artefacts

```groovy
archiveArtifacts artifacts: 'dist/*.tar.gz', fingerprint: true
```

---

## 14. Déploiements

### 14.1 SSH

```groovy
sshagent(['deploy-key']) {
    sh 'rsync -avz dist/ deploy@server:/opt/app/'
    sh 'ssh deploy@server systemctl restart app'
}
```

### 14.2 Kubernetes

```groovy
withCredentials([file(credentialsId: 'kubeconfig-prod', variable: 'KUBECONFIG')]) {
    sh '''
      kubectl set image deploy/app app=${IMAGE}:${TAG} -n prod
      kubectl rollout status deploy/app -n prod --timeout=5m
    '''
}
```

### 14.3 Helm

```groovy
sh """
  helm upgrade --install app ./chart \
    --namespace prod --create-namespace \
    --set image.tag=${TAG} \
    --atomic --timeout 5m
"""
```

### 14.4 Ansible

```groovy
ansiblePlaybook(
    playbook: 'deploy.yml',
    inventory: 'inventories/prod',
    extras: "-e app_version=${TAG}",
    credentialsId: 'ansible-ssh'
)
```

### 14.5 Blue/Green & Canary

Avec ArgoCD Rollouts ou un service mesh, Jenkins déclenche `kubectl argo rollouts set image …` puis vérifie le statut.

---

## 15. Blue Ocean

Interface moderne (plus visuelle) pour les pipelines : visualisation des étapes parallèles, debugging, création de pipelines via formulaire.

```bash
# Installer le plugin Blue Ocean
# Puis : http://jenkins/blue
```

À noter : développement ralenti, plus de versions majeures depuis quelques années — toujours utile mais ne pas en dépendre lourdement.

---

## 16. Jenkins as Code

### 16.1 JCasC (Configuration as Code)

Décrire toute la config Jenkins en YAML :

```yaml
jenkins:
  systemMessage: "Jenkins de l'équipe Platform"
  numExecutors: 0           # forcer l'usage d'agents
  mode: EXCLUSIVE
  securityRealm:
    oic:
      clientId: ${OIDC_CLIENT_ID}
      clientSecret: ${OIDC_CLIENT_SECRET}
      wellKnownOpenIDConfigurationUrl: https://sso/.well-known/openid-configuration
  authorizationStrategy:
    roleBased:
      roles:
        global:
          - name: admin
            permissions: [Overall/Administer]
            assignments: [admin-group]

unclassified:
  location:
    url: https://jenkins.example.com
  globalLibraries:
    libraries:
      - name: shared-lib
        defaultVersion: v1.4.2
        retriever:
          modernSCM:
            scm:
              git:
                remote: https://github.com/acme/jenkins-shared-lib.git

credentials:
  system:
    domainCredentials:
      - credentials:
          - usernamePassword:
              id: ghcr-token
              username: ${GHCR_USER}
              password: ${GHCR_PAT}
              scope: GLOBAL
```

```bash
# Placer le fichier dans $JENKINS_HOME/casc_configs/jenkins.yaml
# Plugin "Configuration as Code" l'applique au démarrage
```

### 16.2 Job DSL

Plugin **Job DSL** : décrire des jobs en Groovy.

```groovy
pipelineJob('myapp-pipeline') {
  definition {
    cpsScm {
      scm { git { remote { url 'https://github.com/acme/myapp.git' }; branch '*/main' } }
      scriptPath 'Jenkinsfile'
    }
  }
  triggers { githubPush() }
}
```

Couplé à JCasC, on obtient un Jenkins **100% reproductible** depuis Git.

---

## 17. Sécurité

### 17.1 Principes

1. **Auth via SSO/OIDC/LDAP** ; jamais d'auth locale en prod.
2. **Authorization Matrix** ou **Role-Based** (principe du moindre privilège).
3. **CSRF protection** activée.
4. **Aucun build sur le controller** : `numExecutors: 0`, agents partout.
5. **Sandbox Groovy** activée pour Pipelines.
6. **Approuver les méthodes** non sandboxées avec parcimonie.
7. **Plugins maintenus uniquement**, MAJ régulières (CVE fréquentes).
8. **TLS** sur le frontend.
9. **Audit log** (Plugin Audit Trail).
10. **Backups** réguliers de `JENKINS_HOME`.

### 17.2 Sandbox Groovy

Toute Pipeline tourne dans une sandbox qui restreint l'API Java/Groovy disponible. Méthodes non whitelistées doivent être approuvées par un admin. **Désactiver la sandbox = donner root sur le controller**.

### 17.3 Agent-to-Controller security

Activer l'option de sécurité empêchant les agents d'exécuter du code privilégié sur le controller.

---

## 18. Scaling et HA

### 18.1 Performance

- Controller dédié, **pas de jobs** dessus.
- JVM bien dimensionnée : `-Xmx4g -Xms4g -XX:+UseG1GC`.
- `JENKINS_HOME` sur SSD rapide.
- Limiter le nombre de jobs actifs (build history rotation).
- Plugin **Pipeline Stage View** consomme bcp → décharger via Blue Ocean ou désactiver.

### 18.2 Agents éphémères

Kubernetes plugin → un Pod par build → scale automatique → pas de "snowflake".

### 18.3 HA controller

- Stockage `JENKINS_HOME` partagé (NFS, EFS).
- Actif/passif (un seul controller actif à la fois — Jenkins n'est pas multi-master).
- **CloudBees CI** offre du multi-controller managed.
- Stratégie alternative : plusieurs Jenkins indépendants par équipe.

---

## 19. Monitoring

### 19.1 Exporter Prometheus

Plugin **Prometheus metrics** expose `/prometheus` :

- jobs queue length
- executors busy / idle
- builds duration
- node disponibilité

### 19.2 Logs

`jenkins.log` côté process + logs par build. Centraliser dans Loki/ELK avec un agent.

### 19.3 Suivre les builds longs

Plugin **Build Failure Analyzer**, **Build Time Trend**.

---

## 20. Migration et alternatives

### 20.1 Quand quitter Jenkins ?

- Coût d'opération élevé (maintenance plugins, montée de version, sécurité).
- Besoin de SaaS managé (GitHub Actions, GitLab CI, CircleCI).
- Volonté de tout migrer en YAML déclaratif minimal.
- Cluster K8s déjà en place → Argo Workflows / Tekton.

### 20.2 Stratégies de migration

- **Strangler fig** : nouveau projet en GitHub Actions, anciens restent sur Jenkins.
- **Pipeline-à-pipeline** : convertir Jenkinsfile → workflows en YAML.
- **Tooling** : `jenkins-to-github-actions` (script communautaire), mais souvent réécriture manuelle.

### 20.3 Alternatives modernes

| Outil | Forces |
|---|---|
| **GitHub Actions** | Marketplace énorme, intégration GitHub, SaaS gratuit pour open-source |
| **GitLab CI** | Tout-en-un GitLab, Auto DevOps, runners flexibles |
| **CircleCI** | UX moderne, parallélisme avancé |
| **Tekton** | Cloud-native, CRD, intégration GitOps |
| **Argo Workflows** | Workflows K8s, data pipelines |
| **Drone** | Léger, simple à opérer |
| **Buildkite** | Hybride agent/SaaS, contrôle infra |

---

## Conclusion

Jenkins reste un pilier du CI/CD self-hosted. Maîtrisez :

1. Les **Pipelines déclaratives** et **Jenkinsfile multibranch**.
2. Les **Shared Libraries** pour la factorisation.
3. Les **agents Kubernetes** pour l'élasticité.
4. **JCasC + Job DSL** pour le 100% as code.
5. La **sécurité** (sandbox, RBAC, OIDC).

Pratiquez les [exercices avancés](./EXERCICES_AVANCES.md) puis préparez votre entretien avec le [guide d'interview](./INTERVIEW.md).

