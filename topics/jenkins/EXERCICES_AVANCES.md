# Jenkins — Exercices avancés (Débutant → Expert)

> 22 exercices progressifs en français pour pratiquer Jenkins.

---

## Niveau Débutant

### Exercice 1 — Job freestyle "Hello"

**Objectif** : créer un Freestyle job qui affiche `Hello $USER` via shell.

<details>
<summary>Solution</summary>

- New Item → Freestyle project → `hello`
- Build steps → Execute shell : `echo "Hello $USER from $(hostname)"`
- Build Now → consulter Console Output
</details>

---

### Exercice 2 — Premier Jenkinsfile

**Objectif** : créer un Pipeline avec 3 stages (Checkout, Build, Test) qui exécutent juste un `echo`.

<details>
<summary>Solution</summary>

```groovy
pipeline {
    agent any
    stages {
        stage('Checkout') { steps { echo 'Checkout OK' } }
        stage('Build')    { steps { echo 'Build OK'    } }
        stage('Test')     { steps { echo 'Tests OK'    } }
    }
}
```
</details>

---

### Exercice 3 — Paramètres

**Objectif** : ajouter un paramètre `NAME` (string) et un paramètre `ENV` (choice : dev/staging/prod).

<details>
<summary>Solution</summary>

```groovy
pipeline {
    agent any
    parameters {
        string(name: 'NAME', defaultValue: 'world', description: 'Qui saluer ?')
        choice(name: 'ENV', choices: ['dev','staging','prod'])
    }
    stages {
        stage('Hello') {
            steps { echo "Hello ${params.NAME} on ${params.ENV}" }
        }
    }
}
```
</details>

---

### Exercice 4 — Post actions

**Objectif** : ajouter `post { success { ... } failure { ... } always { ... } }`.

<details>
<summary>Solution</summary>

```groovy
post {
    always  { echo "Build #${BUILD_NUMBER} terminé" }
    success { echo "✅ OK" }
    failure { echo "❌ KO : ${BUILD_URL}" }
}
```
</details>

---

### Exercice 5 — Git checkout

**Objectif** : récupérer un repo Git public et lister les fichiers.

<details>
<summary>Solution</summary>

```groovy
pipeline {
    agent any
    stages {
        stage('Checkout') {
            steps {
                git url: 'https://github.com/jenkins-docs/simple-java-maven-app.git', branch: 'master'
            }
        }
        stage('Ls') { steps { sh 'ls -la' } }
    }
}
```
</details>

---

## Niveau Intermédiaire

### Exercice 6 — Parallèle

**Objectif** : exécuter 3 stages en parallèle (Lint, Unit, Integration).

<details>
<summary>Solution</summary>

```groovy
stage('Tests') {
    parallel {
        stage('Lint')        { steps { sh 'echo lint' } }
        stage('Unit')        { steps { sh 'echo unit' } }
        stage('Integration') { steps { sh 'echo it'   } }
    }
}
```
</details>

---

### Exercice 7 — Conditions when

**Objectif** : exécuter le stage Deploy **uniquement** sur la branche `main`.

<details>
<summary>Solution</summary>

```groovy
stage('Deploy') {
    when { branch 'main' }
    steps { echo "Déploiement de la branche main" }
}
```
</details>

---

### Exercice 8 — Credentials

**Objectif** : utiliser un credential `db-creds` (username/password) dans un shell sans le révéler.

<details>
<summary>Solution</summary>

```groovy
stage('DB') {
    steps {
        withCredentials([usernamePassword(credentialsId: 'db-creds', usernameVariable: 'U', passwordVariable: 'P')]) {
            sh 'echo "User=$U" && echo "Pwd=***"'
        }
    }
}
```
</details>

---

### Exercice 9 — Build Docker dans le pipeline

**Objectif** : construire une image Docker à partir d'un Dockerfile du repo et la tagger avec le SHA Git.

<details>
<summary>Solution</summary>

```groovy
stage('Docker') {
    steps {
        script {
            env.GIT_SHA = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
        }
        sh "docker build -t myapp:${env.GIT_SHA} ."
    }
}
```
</details>

---

### Exercice 10 — Multibranch Pipeline

**Objectif** : créer un Multibranch Pipeline qui scanne un repo GitHub et construit chaque branche/PR ayant un Jenkinsfile.

<details>
<summary>Solution</summary>

1. New Item → Multibranch Pipeline.
2. Branch Sources → GitHub → URL repo + credentials.
3. Scan period : 5 minutes.
4. Sauvegarder → Jenkins découvre automatiquement les branches.
</details>

---

### Exercice 11 — Archive artifacts + JUnit

**Objectif** : archiver `target/*.jar` et publier les résultats JUnit.

<details>
<summary>Solution</summary>

```groovy
post {
    always {
        archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
        junit '**/target/surefire-reports/*.xml'
    }
}
```
</details>

---

### Exercice 12 — Retry + timeout

**Objectif** : encadrer une étape "flaky" par 3 retry et un timeout de 5 minutes.

<details>
<summary>Solution</summary>

```groovy
steps {
    retry(3) {
        timeout(time: 5, unit: 'MINUTES') {
            sh './flaky.sh'
        }
    }
}
```
</details>

---

## Niveau Avancé

### Exercice 13 — Matrix build

**Objectif** : construire une matrice OS × JDK (3 OS × 2 JDK).

<details>
<summary>Solution</summary>

```groovy
matrix {
    axes {
        axis { name 'OS';  values 'ubuntu','alpine','debian' }
        axis { name 'JDK'; values '17','21' }
    }
    stages {
        stage('Build') {
            steps { echo "OS=${OS} JDK=${JDK}" }
        }
    }
}
```
</details>

---

### Exercice 14 — Input manuel (approbation)

**Objectif** : demander une confirmation avant un déploiement prod, avec timeout de 10 min.

<details>
<summary>Solution</summary>

```groovy
stage('Approval') {
    when { branch 'main' }
    steps {
        timeout(time: 10, unit: 'MINUTES') {
            input message: "Déployer en PROD ?", ok: 'Yes',
                  submitter: 'lead-devs,ops'
        }
    }
}
```
</details>

---

### Exercice 15 — Shared Library

**Objectif** : créer une shared library avec une fonction `say(msg)` et l'utiliser dans un Jenkinsfile.

<details>
<summary>Solution</summary>

Repo `shared-lib/vars/say.groovy` :
```groovy
def call(String msg) {
    echo "📣 ${msg}"
}
```

Manage Jenkins → Global Pipeline Libraries → Add `shared-lib` (Git URL, default version `main`).

Jenkinsfile :
```groovy
@Library('shared-lib@main') _
pipeline {
    agent any
    stages {
        stage('Hello') { steps { say 'Hello depuis la shared lib !' } }
    }
}
```
</details>

---

### Exercice 16 — Agent Docker

**Objectif** : exécuter un stage dans un conteneur `python:3.12-slim`.

<details>
<summary>Solution</summary>

```groovy
pipeline {
    agent none
    stages {
        stage('Python') {
            agent { docker { image 'python:3.12-slim' } }
            steps { sh 'python --version && pip --version' }
        }
    }
}
```
</details>

---

### Exercice 17 — Agent Kubernetes dynamique

**Objectif** : exécuter un build Maven + Docker dans un pod K8s avec 2 conteneurs.

<details>
<summary>Solution</summary>

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
                - name: kaniko
                  image: gcr.io/kaniko-project/executor:debug
                  command: ['/busybox/cat']
                  tty: true
            '''
        }
    }
    stages {
        stage('Build') {
            steps {
                container('maven') { sh 'mvn -B package' }
                container('kaniko') {
                    sh '/kaniko/executor --dockerfile Dockerfile --context $(pwd) --destination ghcr.io/acme/app:$BUILD_NUMBER'
                }
            }
        }
    }
}
```
</details>

---

### Exercice 18 — Webhook GitHub

**Objectif** : déclencher un build à chaque push GitHub.

<details>
<summary>Solution</summary>

1. Plugin **GitHub** installé.
2. Repo GitHub → Settings → Webhooks → URL `https://jenkins.example.com/github-webhook/`, content type `application/json`, secret optionnel.
3. Dans Jenkinsfile : `triggers { githubPush() }` (ou cochée dans la config classique).
</details>

---

### Exercice 19 — Notifications Slack

**Objectif** : envoyer un message Slack sur succès et échec.

<details>
<summary>Solution</summary>

- Plugin **Slack Notification**, configurer le workspace + token.
- Pipeline :

```groovy
post {
    success { slackSend channel: '#ci', color: 'good',   message: "✅ #${BUILD_NUMBER} OK : ${JOB_NAME}" }
    failure { slackSend channel: '#ci', color: 'danger', message: "❌ #${BUILD_NUMBER} KO : ${BUILD_URL}" }
}
```
</details>

---

## Niveau Expert

### Exercice 20 — JCasC

**Objectif** : décrire la config Jenkins (URL, SSO, library globale, credentials) en JCasC YAML.

<details>
<summary>Solution</summary>

Voir le guide complet section 16.1. Activer le plugin **Configuration as Code**, placer le fichier dans `$JENKINS_HOME/casc_configs/` et redémarrer.
</details>

---

### Exercice 21 — Déploiement Kubernetes + Helm

**Objectif** : pipeline qui build une image, la pousse sur GHCR, et déploie via Helm sur un cluster K8s.

<details>
<summary>Solution</summary>

```groovy
pipeline {
    agent { kubernetes { /* pod template */ } }
    environment {
        IMAGE = "ghcr.io/acme/app"
        TAG   = "${BUILD_NUMBER}"
    }
    stages {
        stage('Build') { steps { container('docker') { sh "docker build -t ${IMAGE}:${TAG} ." } } }
        stage('Push') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'ghcr', usernameVariable: 'U', passwordVariable: 'P')]) {
                    container('docker') {
                        sh '''
                          echo $P | docker login ghcr.io -u $U --password-stdin
                          docker push ${IMAGE}:${TAG}
                        '''
                    }
                }
            }
        }
        stage('Deploy') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig-prod', variable: 'KUBECONFIG')]) {
                    container('helm') {
                        sh """
                          helm upgrade --install app ./chart \
                            --namespace prod --create-namespace \
                            --set image.repository=${IMAGE} \
                            --set image.tag=${TAG} \
                            --atomic --timeout 5m
                        """
                    }
                }
            }
        }
    }
}
```
</details>

---

### Exercice 22 — Gestion d'erreurs avec catchError

**Objectif** : marquer le build comme `UNSTABLE` si les tests d'intégration échouent, sans avorter.

<details>
<summary>Solution</summary>

```groovy
stage('Integration tests') {
    steps {
        catchError(buildResult: 'UNSTABLE', stageResult: 'UNSTABLE') {
            sh 'pytest tests/integration'
        }
    }
}
```
</details>

---

## Pour aller plus loin

- Mettez en place **JCasC + Job DSL** pour un Jenkins 100% reproductible.
- Migrez vos agents permanents vers le **Kubernetes plugin**.
- Construisez une **Shared Library** pour standardiser les pipelines de votre organisation.
- Intégrez **SonarQube + Vault + Slack** dans un pipeline réel.
- Préparez votre **plan de migration** vers GitHub Actions / Tekton si nécessaire.

