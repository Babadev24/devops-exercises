# Kubernetes — Exercices avancés (Débutant → Expert)

> 25 exercices progressifs en français pour pratiquer Kubernetes (niveau CKAD/CKA/CKS).
> Chaque exercice : **contexte**, **objectifs**, **prérequis**, **solution** (repliée).

> Prérequis général : un cluster (Kind, Minikube, k3d, ou managé) et `kubectl`.

---

## Niveau Débutant

### Exercice 1 — Premier pod

**Objectif** : créer un pod Nginx, le lister, voir ses logs, s'y connecter.

<details>
<summary>Solution</summary>

```bash
kubectl run nginx --image=nginx:1.27-alpine
kubectl get pod nginx
kubectl logs nginx
kubectl exec -it nginx -- sh
kubectl delete pod nginx
```
</details>

---

### Exercice 2 — Deployment + Service ClusterIP

**Objectif** : déployer 3 réplicas Nginx et y accéder via un Service ClusterIP. Tester avec `kubectl port-forward`.

<details>
<summary>Solution</summary>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: web }
spec:
  replicas: 3
  selector: { matchLabels: { app: web } }
  template:
    metadata: { labels: { app: web } }
    spec:
      containers:
        - name: nginx
          image: nginx:1.27-alpine
          ports: [{ containerPort: 80 }]
---
apiVersion: v1
kind: Service
metadata: { name: web }
spec:
  selector: { app: web }
  ports: [{ port: 80, targetPort: 80 }]
```

```bash
kubectl apply -f web.yaml
kubectl port-forward svc/web 8080:80
curl localhost:8080
```
</details>

---

### Exercice 3 — Rolling update

**Objectif** : passer Nginx de `1.27-alpine` à `1.28-alpine` et observer le rollout.

<details>
<summary>Solution</summary>

```bash
kubectl set image deploy/web nginx=nginx:1.28-alpine
kubectl rollout status deploy/web
kubectl rollout history deploy/web
kubectl rollout undo deploy/web
```
</details>

---

### Exercice 4 — ConfigMap

**Objectif** : injecter une variable `LOG_LEVEL=debug` via ConfigMap dans un pod.

<details>
<summary>Solution</summary>

```yaml
apiVersion: v1
kind: ConfigMap
metadata: { name: app-cfg }
data: { LOG_LEVEL: debug }
---
apiVersion: v1
kind: Pod
metadata: { name: app }
spec:
  containers:
    - name: app
      image: busybox
      command: ["sh", "-c", "env | grep LOG_LEVEL; sleep 3600"]
      envFrom: [{ configMapRef: { name: app-cfg } }]
```

```bash
kubectl apply -f app.yaml
kubectl logs app
```
</details>

---

### Exercice 5 — Secret

**Objectif** : créer un Secret `db-creds` (`user=admin`, `password=S3cret!`) et le monter en variables d'environnement.

<details>
<summary>Solution</summary>

```bash
kubectl create secret generic db-creds \
  --from-literal=user=admin --from-literal=password='S3cret!'
```

```yaml
spec:
  containers:
    - name: app
      image: busybox
      command: ["sh", "-c", "env | grep -E 'user|password'; sleep 3600"]
      envFrom: [{ secretRef: { name: db-creds } }]
```
</details>

---

## Niveau Intermédiaire

### Exercice 6 — Probes

**Objectif** : ajouter liveness et readiness probes HTTP `/healthz` à un Deployment, avec `initialDelaySeconds: 5`, `periodSeconds: 10`.

<details>
<summary>Solution</summary>

```yaml
livenessProbe:
  httpGet: { path: /healthz, port: 8080 }
  initialDelaySeconds: 5
  periodSeconds: 10
readinessProbe:
  httpGet: { path: /ready, port: 8080 }
  initialDelaySeconds: 5
  periodSeconds: 5
```
</details>

---

### Exercice 7 — Resources et QoS

**Objectif** : créer un pod **Guaranteed** et un pod **Burstable**. Vérifier la classe.

<details>
<summary>Solution</summary>

Guaranteed (requests == limits) :
```yaml
resources:
  requests: { cpu: "200m", memory: "256Mi" }
  limits:   { cpu: "200m", memory: "256Mi" }
```

Burstable :
```yaml
resources:
  requests: { cpu: "100m", memory: "128Mi" }
  limits:   { cpu: "500m", memory: "256Mi" }
```

```bash
kubectl get pod <name> -o jsonpath='{.status.qosClass}'
```
</details>

---

### Exercice 8 — HPA

**Objectif** : autoscaler un Deployment entre 2 et 10 replicas sur 70% de CPU. Générer du load avec `hey` ou `siege`.

<details>
<summary>Solution</summary>

```bash
kubectl autoscale deploy web --cpu-percent=70 --min=2 --max=10
# ou avec un manifest HPA v2 (voir guide)

# Générer la charge
kubectl run -it --rm load --image=busybox -- sh -c \
  "while true; do wget -q -O- http://web; done"

kubectl get hpa -w
```
</details>

---

### Exercice 9 — Ingress avec TLS

**Objectif** : exposer une app via Ingress Nginx sur `app.local`, avec TLS auto-signé.

<details>
<summary>Solution</summary>

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt -subj "/CN=app.local"
kubectl create secret tls app-tls --cert=tls.crt --key=tls.key
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata: { name: app }
spec:
  ingressClassName: nginx
  tls: [{ hosts: [app.local], secretName: app-tls }]
  rules:
    - host: app.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend: { service: { name: web, port: { number: 80 } } }
```

Ajouter `127.0.0.1 app.local` dans `/etc/hosts`.
</details>

---

### Exercice 10 — StatefulSet PostgreSQL

**Objectif** : déployer un StatefulSet PostgreSQL 1 replica avec un PVC de 5Gi.

<details>
<summary>Solution</summary>

```yaml
apiVersion: v1
kind: Service
metadata: { name: postgres }
spec:
  clusterIP: None
  selector: { app: postgres }
  ports: [{ port: 5432 }]
---
apiVersion: apps/v1
kind: StatefulSet
metadata: { name: postgres }
spec:
  serviceName: postgres
  replicas: 1
  selector: { matchLabels: { app: postgres } }
  template:
    metadata: { labels: { app: postgres } }
    spec:
      containers:
        - name: pg
          image: postgres:16
          env:
            - name: POSTGRES_PASSWORD
              value: changeme
          ports: [{ containerPort: 5432 }]
          volumeMounts: [{ name: data, mountPath: /var/lib/postgresql/data }]
  volumeClaimTemplates:
    - metadata: { name: data }
      spec:
        accessModes: [ReadWriteOnce]
        resources: { requests: { storage: 5Gi } }
```
</details>

---

### Exercice 11 — CronJob de backup

**Objectif** : créer un CronJob qui s'exécute toutes les nuits à 2h et écrit la date dans un fichier sur un PVC partagé.

<details>
<summary>Solution</summary>

```yaml
apiVersion: batch/v1
kind: CronJob
metadata: { name: stamp }
spec:
  schedule: "0 2 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: stamp
              image: busybox
              command: ["sh", "-c", "date >> /data/log.txt"]
              volumeMounts: [{ name: data, mountPath: /data }]
          volumes:
            - name: data
              persistentVolumeClaim: { claimName: shared-pvc }
```
</details>

---

### Exercice 12 — RBAC

**Objectif** : créer un ServiceAccount `viewer` autorisé à `get/list` les pods uniquement dans le namespace `dev`.

<details>
<summary>Solution</summary>

```yaml
apiVersion: v1
kind: ServiceAccount
metadata: { name: viewer, namespace: dev }
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata: { name: pod-viewer, namespace: dev }
rules:
  - apiGroups: [""]
    resources: [pods]
    verbs: [get, list, watch]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata: { name: viewer-binding, namespace: dev }
subjects:
  - kind: ServiceAccount
    name: viewer
    namespace: dev
roleRef:
  kind: Role
  name: pod-viewer
  apiGroup: rbac.authorization.k8s.io
```

Tester :
```bash
kubectl auth can-i get pods -n dev --as=system:serviceaccount:dev:viewer
kubectl auth can-i delete pods -n dev --as=system:serviceaccount:dev:viewer
```
</details>

---

### Exercice 13 — NetworkPolicy

**Objectif** : dans `prod`, autoriser uniquement les pods avec label `tier=frontend` à parler aux pods `tier=backend` sur le port 8080.

<details>
<summary>Solution</summary>

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: frontend-to-backend, namespace: prod }
spec:
  podSelector: { matchLabels: { tier: backend } }
  policyTypes: [Ingress]
  ingress:
    - from:
        - podSelector: { matchLabels: { tier: frontend } }
      ports:
        - protocol: TCP
          port: 8080
```
</details>

---

### Exercice 14 — Helm chart custom

**Objectif** : créer un chart `mychart`, paramétrer le nombre de replicas et l'image, et installer 2 releases (dev/prod) avec des valeurs différentes.

<details>
<summary>Solution</summary>

```bash
helm create mychart
# Modifier templates/deployment.yaml et values.yaml
helm install dev  ./mychart -f values-dev.yaml -n dev --create-namespace
helm install prod ./mychart -f values-prod.yaml -n prod --create-namespace
helm list -A
```
</details>

---

## Niveau Avancé

### Exercice 15 — PodDisruptionBudget + anti-affinity

**Objectif** : déployer 3 replicas web avec PDB `minAvailable: 2` et **anti-affinity** pour les répartir sur 3 nœuds différents.

<details>
<summary>Solution</summary>

```yaml
spec:
  template:
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - topologyKey: kubernetes.io/hostname
              labelSelector:
                matchLabels: { app: web }
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata: { name: web-pdb }
spec:
  minAvailable: 2
  selector: { matchLabels: { app: web } }
```
</details>

---

### Exercice 16 — TopologySpreadConstraints

**Objectif** : répartir 6 replicas sur 3 zones AWS de façon équilibrée.

<details>
<summary>Solution</summary>

```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector: { matchLabels: { app: web } }
```
</details>

---

### Exercice 17 — Init container "wait-for-db"

**Objectif** : empêcher le démarrage d'un app pod tant que `postgres:5432` n'est pas joignable.

<details>
<summary>Solution</summary>

```yaml
initContainers:
  - name: wait-db
    image: busybox
    command: ['sh', '-c', 'until nc -z postgres 5432; do echo waiting; sleep 2; done']
```
</details>

---

### Exercice 18 — Pod Security Standards

**Objectif** : appliquer le profil `restricted` au namespace `prod` et corriger un Deployment qui exécute `runAsUser: 0`.

<details>
<summary>Solution</summary>

```bash
kubectl label namespace prod \
  pod-security.kubernetes.io/enforce=restricted --overwrite
```

Corriger le Deployment :
```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  allowPrivilegeEscalation: false
  capabilities: { drop: [ALL] }
  seccompProfile: { type: RuntimeDefault }
```
</details>

---

### Exercice 19 — Image scan + admission policy

**Objectif** : avec **Kyverno**, interdire le déploiement d'images taguées `latest`.

<details>
<summary>Solution</summary>

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata: { name: disallow-latest }
spec:
  validationFailureAction: enforce
  rules:
    - name: validate-image-tag
      match: { any: [{ resources: { kinds: [Pod] } }] }
      validate:
        message: "L'image ne doit pas être taguée 'latest'."
        pattern:
          spec:
            containers:
              - image: "!*:latest"
```
</details>

---

### Exercice 20 — Custom Resource + opérateur Kopf

**Objectif** : créer un CRD `Greeting` (avec `spec.name`) et un controller Python (Kopf) qui logue `Hello <name>`.

<details>
<summary>Solution</summary>

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata: { name: greetings.acme.com }
spec:
  group: acme.com
  scope: Namespaced
  names: { kind: Greeting, plural: greetings, singular: greeting }
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                name: { type: string }
```

`operator.py` :
```python
import kopf

@kopf.on.create('acme.com', 'v1', 'greetings')
def hello(spec, **_):
    print(f"Hello {spec.get('name', 'world')}")
```

```bash
pip install kopf kubernetes
kopf run operator.py
```
</details>

---

### Exercice 21 — ArgoCD application

**Objectif** : installer ArgoCD, créer une App pointant sur un repo Git et déployer un Helm chart.

<details>
<summary>Solution</summary>

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

`app.yaml` :
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata: { name: demo, namespace: argocd }
spec:
  project: default
  source:
    repoURL: https://github.com/example/k8s-manifests
    path: helm/demo
    targetRevision: main
    helm:
      valueFiles: [values-prod.yaml]
  destination:
    server: https://kubernetes.default.svc
    namespace: demo
  syncPolicy: { automated: { prune: true, selfHeal: true }, syncOptions: [CreateNamespace=true] }
```
</details>

---

## Niveau Expert

### Exercice 22 — Backup/restore etcd

**Objectif** : sauvegarder etcd, supprimer un objet de test, restaurer.

<details>
<summary>Solution</summary>

```bash
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Restauration : arrêter le static pod kube-apiserver et etcd, puis :
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd.db \
  --data-dir=/var/lib/etcd-new

# Mettre à jour le manifest etcd pour pointer sur /var/lib/etcd-new
```
</details>

---

### Exercice 23 — Mise à jour de cluster avec kubeadm

**Objectif** : upgrader un cluster kubeadm de 1.30 à 1.31.

<details>
<summary>Solution</summary>

Sur le control plane :
```bash
apt update && apt-cache madison kubeadm
apt install -y kubeadm=1.31.0-*
kubeadm upgrade plan
kubeadm upgrade apply v1.31.0
kubectl drain <node> --ignore-daemonsets
apt install -y kubelet=1.31.0-* kubectl=1.31.0-*
systemctl daemon-reload && systemctl restart kubelet
kubectl uncordon <node>
```

Répéter sur chaque worker (avec `kubeadm upgrade node` à la place de `apply`).
</details>

---

### Exercice 24 — Debug d'un Pod Pending

**Contexte** : un Pod reste Pending. Décrivez la procédure d'investigation.

<details>
<summary>Solution</summary>

```bash
kubectl describe pod <p>            # section Events
kubectl get events -A --sort-by=.lastTimestamp
kubectl get nodes -o wide           # capacité, taints
kubectl describe node <node>
```

Causes typiques :
- Aucune node n'a assez de CPU/RAM disponibles → réduire requests, ajouter un nœud (autoscaler).
- PVC en `Pending` → vérifier StorageClass, provisioner.
- Taints sur tous les nœuds → ajouter `tolerations`.
- NodeSelector ne matche rien → corriger label.
- Image inaccessible → vérifier pullSecret.
</details>

---

### Exercice 25 — Migration de zero-downtime

**Objectif** : migrer un Deployment v1 vers v2 avec un test canary à 10% du trafic via **Argo Rollouts** ou **Istio**.

<details>
<summary>Solution (Argo Rollouts)</summary>

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata: { name: web }
spec:
  replicas: 10
  selector: { matchLabels: { app: web } }
  template:
    metadata: { labels: { app: web } }
    spec:
      containers:
        - name: web
          image: web:v2
  strategy:
    canary:
      steps:
        - setWeight: 10
        - pause: { duration: 5m }
        - setWeight: 50
        - pause: { duration: 5m }
        - setWeight: 100
```
</details>

---

## Pour aller plus loin

- Passez la **CKAD** (focus dev) puis la **CKA** (admin).
- Construisez un cluster bare-metal avec kubeadm + Calico.
- Écrivez un opérateur en Go avec **kubebuilder**.
- Mettez en place un pipeline GitOps avec **ArgoCD + Kustomize**.
- Implémentez du **service mesh** avec Linkerd ou Istio.

