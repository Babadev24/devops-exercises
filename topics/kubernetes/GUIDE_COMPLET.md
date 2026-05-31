# Kubernetes — Le Guide Complet : de 0 à Hero

> Livre de référence en français pour maîtriser Kubernetes, de l'architecture à la production.

---

## Table des matières

1. [Introduction à Kubernetes](#1-introduction-à-kubernetes)
2. [Architecture détaillée](#2-architecture-détaillée)
3. [Installation et clusters](#3-installation-et-clusters)
4. [kubectl en profondeur](#4-kubectl-en-profondeur)
5. [Pods](#5-pods)
6. [Workloads](#6-workloads)
7. [Services et networking](#7-services-et-networking)
8. [Ingress](#8-ingress)
9. [ConfigMaps et Secrets](#9-configmaps-et-secrets)
10. [Volumes et stockage](#10-volumes-et-stockage)
11. [Namespaces, labels, sélecteurs](#11-namespaces-labels-sélecteurs)
12. [RBAC et sécurité](#12-rbac-et-sécurité)
13. [Resource management et autoscaling](#13-resource-management-et-autoscaling)
14. [Helm](#14-helm)
15. [Observabilité](#15-observabilité)
16. [Operators et CRDs](#16-operators-et-crds)
17. [Service Mesh](#17-service-mesh)
18. [GitOps](#18-gitops)
19. [Troubleshooting](#19-troubleshooting)
20. [Bonnes pratiques en production](#20-bonnes-pratiques-en-production)

---

## 1. Introduction à Kubernetes

### 1.1 Définition

**Kubernetes** (K8s, "kube") est un **orchestrateur de conteneurs** open-source, créé par Google et donné à la **CNCF** en 2015. Il automatise :

- Le déploiement
- La mise à l'échelle (horizontale et verticale)
- La gestion (rolling updates, rollbacks)
- La résilience (auto-healing) des applications conteneurisées

### 1.2 Histoire

- **2003-2014** : Google développe **Borg** puis **Omega**, ses orchestrateurs internes.
- **2014** : annonce publique de Kubernetes (v0.x).
- **2015** : v1.0 ; transfert à la **Cloud Native Computing Foundation (CNCF)**.
- **2017** : Kubernetes devient le standard de facto (Docker abandonne Swarm comme priorité, AWS lance EKS, etc.).
- Aujourd'hui : v1.31+, releases tous les 4 mois.

### 1.3 Pourquoi Kubernetes ?

| Problème | Solution K8s |
|---|---|
| Mon conteneur crash | `restartPolicy` + readiness/liveness probes |
| Charge variable | HPA (Horizontal Pod Autoscaler) |
| Déploiement zero-downtime | Rolling updates des Deployments |
| Découverte de services | DNS interne + Services |
| Configuration / secrets | ConfigMap / Secret |
| Stockage persistant | PV / PVC / StorageClass |
| Sécurité | RBAC, NetworkPolicies, PSS |
| Multi-cloud | Couche d'abstraction |

### 1.4 Alternatives

| Outil | Modèle | Maturité |
|---|---|---|
| **Docker Swarm** | Simple, Docker-natif | Déclin |
| **Nomad** (HashiCorp) | Léger, supporte non-conteneurs | Niche |
| **OpenShift** (Red Hat) | K8s + extensions enterprise | Mature, payant |
| **Rancher** | Gestion multi-clusters K8s | Mature |
| **AWS ECS** | Spécifique AWS | Très intégré AWS |

---

## 2. Architecture détaillée

### 2.1 Vue d'ensemble

```
┌─────────────────────── Control Plane ───────────────────────┐
│                                                              │
│   ┌──────────────┐   ┌──────────────┐   ┌────────────────┐   │
│   │  kube-       │   │  Scheduler   │   │  Controller    │   │
│   │  apiserver   │←─→│              │   │  Manager       │   │
│   └──────┬───────┘   └──────────────┘   └────────────────┘   │
│          │                                                    │
│          ▼                                                    │
│      ┌───────┐                                                │
│      │ etcd  │  (KV store : état désiré + observé)            │
│      └───────┘                                                │
└───────────┬──────────────────────────────────────────────────┘
            │
            ▼  (kubelet appelle l'API)
┌──────── Worker Node 1 ──────┐   ┌──────── Worker Node N ─────┐
│ kubelet │ kube-proxy │ CRI  │   │ kubelet │ kube-proxy │ CRI │
│   ┌──────┐  ┌──────┐         │   │   ┌──────┐  ┌──────┐       │
│   │ Pod  │  │ Pod  │  ...    │   │   │ Pod  │  │ Pod  │  ...  │
│   └──────┘  └──────┘         │   │   └──────┘  └──────┘       │
└──────────────────────────────┘   └──────────────────────────────┘
```

### 2.2 Composants du Control Plane

| Composant | Rôle |
|---|---|
| **kube-apiserver** | Point d'entrée REST de toutes les opérations. Authentifie, autorise, valide, persiste dans etcd. |
| **etcd** | Base clé-valeur distribuée (Raft). Source de vérité. À sauvegarder ! |
| **kube-scheduler** | Décide sur quel nœud placer un Pod en attente (filtres + scoring). |
| **kube-controller-manager** | Boucles de contrôle : Deployment, ReplicaSet, Node, Endpoint, Job… |
| **cloud-controller-manager** | Intégration cloud (LBs, volumes, nodes) — optionnel. |

### 2.3 Composants du Worker

| Composant | Rôle |
|---|---|
| **kubelet** | Agent qui parle à l'API et au container runtime, gère le cycle de vie des Pods sur le nœud. |
| **kube-proxy** | Programme les règles réseau (iptables/IPVS/eBPF) pour les Services. |
| **Container Runtime (CRI)** | containerd, CRI-O (Docker n'est plus supporté depuis K8s 1.24). |

### 2.4 Boucle de réconciliation

Tous les contrôleurs suivent ce pattern :

1. Lire l'**état désiré** depuis l'API (ex. Deployment veut 3 replicas).
2. Lire l'**état observé** (ex. il y a 2 pods running).
3. **Agir** pour converger (créer 1 pod).
4. Recommencer indéfiniment.

C'est cette boucle qui rend Kubernetes **auto-cicatrisant**.

### 2.5 CRI / CNI / CSI

- **CRI** (Container Runtime Interface) : containerd, CRI-O.
- **CNI** (Container Network Interface) : Calico, Cilium, Flannel, Weave.
- **CSI** (Container Storage Interface) : drivers pour AWS EBS, Azure Disk, Ceph, NFS, etc.

---

## 3. Installation et clusters

### 3.1 Local

| Outil | Cas d'usage |
|---|---|
| **Minikube** | Cluster 1 nœud, hyperviseur ou Docker |
| **Kind** (Kubernetes IN Docker) | Multi-nœuds rapide, idéal CI |
| **k3s / k3d** | Léger (Rancher), production edge possible |
| **Docker Desktop** | Cluster intégré (1 nœud) |
| **MicroK8s** | Ubuntu, snap |

```bash
# Kind multi-nœuds
kind create cluster --config=kind-config.yaml
```

`kind-config.yaml` :
```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
  - role: worker
```

### 3.2 Managed

| Provider | Service |
|---|---|
| AWS | **EKS** |
| Azure | **AKS** |
| Google | **GKE** (le plus mature) |
| OVHCloud, Scaleway, DigitalOcean | offres managées équivalentes |

Avantages : control plane managé, mises à jour gérées, intégration LBs/IAM/stockage natif.

### 3.3 Self-managed

- **kubeadm** : bootstrap officiel, requiert config OS/réseau préalable.
- **kops** : pour AWS principalement.
- **kubespray** : playbooks Ansible, multi-cloud/bare-metal.
- **Talos** / **Bottlerocket** : OS immuables conçus pour K8s.

---

## 4. kubectl en profondeur

### 4.1 Configuration

`~/.kube/config` peut contenir plusieurs **contexts** (cluster + user + namespace).

```bash
kubectl config get-contexts
kubectl config use-context prod-eu
kubectl config set-context --current --namespace=monapp
```

Outil recommandé : **kubectx** + **kubens** (changement rapide).

### 4.2 Commandes essentielles

```bash
kubectl get pods -A                       # tous namespaces
kubectl get pods -o wide
kubectl get pods -l app=nginx
kubectl get pods --field-selector status.phase=Running

kubectl describe pod <name>
kubectl logs <pod> -f --tail=100
kubectl logs <pod> -c <container> --previous

kubectl exec -it <pod> -- /bin/sh
kubectl port-forward svc/web 8080:80

kubectl apply -f deploy.yaml
kubectl delete -f deploy.yaml
kubectl rollout status deploy/web
kubectl rollout undo deploy/web

kubectl explain pod.spec.containers.resources
kubectl api-resources
kubectl auth can-i delete pods -n prod
```

### 4.3 Output et formats

```bash
kubectl get pod web -o yaml
kubectl get pod web -o json | jq '.status.podIP'
kubectl get pods -o jsonpath='{.items[*].metadata.name}'
kubectl get pods -o custom-columns=NAME:.metadata.name,IP:.status.podIP
```

### 4.4 Debug

```bash
kubectl debug pod/web -it --image=busybox --target=web
kubectl debug node/node1 -it --image=alpine
```

---

## 5. Pods

### 5.1 Définition

Un **Pod** est l'unité atomique de déploiement K8s : 1 ou plusieurs conteneurs partageant **réseau** (IP, ports) et **volumes**.

### 5.2 Manifest minimal

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels: { app: web }
spec:
  containers:
    - name: nginx
      image: nginx:1.27-alpine
      ports: [{ containerPort: 80 }]
      resources:
        requests: { cpu: "100m", memory: "128Mi" }
        limits:   { cpu: "500m", memory: "256Mi" }
```

### 5.3 Cycle de vie

`Pending → Running → (Succeeded | Failed)` ; `Unknown` si plus de communication kubelet.

### 5.4 Probes

| Probe | Effet si échec |
|---|---|
| **liveness** | Kubelet **redémarre** le conteneur |
| **readiness** | Pod retiré des Endpoints du Service |
| **startup** | Désactive les autres probes tant qu'elle n'a pas réussi |

```yaml
livenessProbe:
  httpGet: { path: /healthz, port: 8080 }
  initialDelaySeconds: 10
  periodSeconds: 10
readinessProbe:
  httpGet: { path: /ready, port: 8080 }
  periodSeconds: 5
startupProbe:
  httpGet: { path: /healthz, port: 8080 }
  failureThreshold: 30
  periodSeconds: 10
```

### 5.5 Init containers

Exécutés **avant** les conteneurs principaux, séquentiellement.

```yaml
spec:
  initContainers:
    - name: wait-db
      image: busybox
      command: ['sh', '-c', 'until nc -z db 5432; do sleep 2; done']
  containers:
    - name: app
      image: myapp:1.0
```

### 5.6 Multi-container patterns

- **Sidecar** : conteneur compagnon (ex. log shipper, proxy).
- **Ambassador** : représente un service externe pour le conteneur principal.
- **Adapter** : adapte la sortie du conteneur principal (ex. format de logs).

### 5.7 Pod lifecycle hooks

```yaml
lifecycle:
  postStart:
    exec: { command: ['/bin/sh', '-c', 'echo started'] }
  preStop:
    exec: { command: ['/bin/sh', '-c', 'nginx -s quit; sleep 5'] }
```

---

## 6. Workloads

### 6.1 Tableau récapitulatif

| Workload | Cas d'usage | Persistance | Identité stable |
|---|---|---|---|
| **Deployment** | Apps stateless | Non | Non |
| **StatefulSet** | DB, brokers | Oui (PVC) | Oui (`pod-0`, `pod-1`…) |
| **DaemonSet** | 1 pod par nœud (logs, monitoring) | Selon usage | Non |
| **Job** | Tâche batch ponctuelle | Non | Non |
| **CronJob** | Job planifié | Non | Non |
| **ReplicaSet** | Bas niveau (utilisé par Deployment) | Non | Non |

### 6.2 Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: web }
spec:
  replicas: 3
  selector: { matchLabels: { app: web } }
  strategy:
    type: RollingUpdate
    rollingUpdate: { maxSurge: 1, maxUnavailable: 0 }
  template:
    metadata: { labels: { app: web } }
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports: [{ containerPort: 80 }]
```

Opérations :
```bash
kubectl rollout status deploy/web
kubectl rollout history deploy/web
kubectl rollout undo deploy/web --to-revision=2
kubectl scale deploy/web --replicas=5
```

### 6.3 StatefulSet

- Pods nommés `name-0`, `name-1`… (ordonnés).
- Création/suppression séquentielles.
- `volumeClaimTemplates` : un PVC par pod, conservé même si le pod est recréé.
- Nécessite un **Headless Service** (`clusterIP: None`) pour DNS stable.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata: { name: postgres }
spec:
  serviceName: postgres
  replicas: 3
  selector: { matchLabels: { app: postgres } }
  template:
    metadata: { labels: { app: postgres } }
    spec:
      containers:
        - name: pg
          image: postgres:16
          ports: [{ containerPort: 5432 }]
          volumeMounts: [{ name: data, mountPath: /var/lib/postgresql/data }]
  volumeClaimTemplates:
    - metadata: { name: data }
      spec:
        accessModes: [ReadWriteOnce]
        resources: { requests: { storage: 20Gi } }
        storageClassName: standard
```

### 6.4 DaemonSet

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata: { name: node-exporter, namespace: monitoring }
spec:
  selector: { matchLabels: { app: node-exporter } }
  template:
    metadata: { labels: { app: node-exporter } }
    spec:
      hostNetwork: true
      containers:
        - name: node-exporter
          image: prom/node-exporter:v1.8.0
          ports: [{ containerPort: 9100, hostPort: 9100 }]
```

### 6.5 Job / CronJob

```yaml
apiVersion: batch/v1
kind: Job
metadata: { name: migration }
spec:
  backoffLimit: 3
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: m
          image: myapp:1.0
          command: ["python", "manage.py", "migrate"]
---
apiVersion: batch/v1
kind: CronJob
metadata: { name: backup }
spec:
  schedule: "0 2 * * *"
  successfulJobsHistoryLimit: 3
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: dump
              image: postgres:16
              command: ["pg_dumpall", "-h", "postgres"]
```

---

## 7. Services et networking

### 7.1 Types de Service

| Type | Accessibilité | Cas d'usage |
|---|---|---|
| **ClusterIP** (défaut) | Interne au cluster | Communication inter-pods |
| **NodePort** | Port ouvert sur chaque nœud (30000-32767) | Dev, debug |
| **LoadBalancer** | LB cloud externe | Production cloud |
| **ExternalName** | Alias DNS vers un nom externe | Migration progressive |
| **Headless** (`clusterIP: None`) | Pas de VIP, DNS renvoie les IPs des pods | StatefulSets |

```yaml
apiVersion: v1
kind: Service
metadata: { name: web }
spec:
  type: ClusterIP
  selector: { app: web }
  ports:
    - port: 80
      targetPort: 8080
      protocol: TCP
```

### 7.2 DNS interne

CoreDNS fournit : `<service>.<namespace>.svc.cluster.local`.

Depuis un pod du namespace `app`, on peut joindre `db.app` ou `db.app.svc.cluster.local`.

### 7.3 Endpoints

Le contrôleur Endpoint maintient automatiquement la liste des IPs de pods *Ready* pour chaque Service. EndpointSlices remplacent progressivement (scalabilité).

### 7.4 NetworkPolicy

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: deny-all-ingress, namespace: prod }
spec:
  podSelector: {}
  policyTypes: [Ingress]
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: allow-web-from-lb, namespace: prod }
spec:
  podSelector: { matchLabels: { app: web } }
  policyTypes: [Ingress]
  ingress:
    - from:
        - namespaceSelector: { matchLabels: { name: ingress } }
      ports: [{ port: 8080 }]
```

> Nécessite un CNI qui les supporte (Calico, Cilium…).

---

## 8. Ingress

### 8.1 Concept

Un **Ingress** route le trafic HTTP/HTTPS externe vers des Services internes, basé sur l'hôte et le chemin.

Nécessite un **Ingress Controller** (Nginx, Traefik, HAProxy, AWS ALB, GCE).

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  tls:
    - hosts: [app.example.com]
      secretName: app-tls
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service: { name: api, port: { number: 8080 } }
          - path: /
            pathType: Prefix
            backend:
              service: { name: web, port: { number: 80 } }
```

### 8.2 cert-manager

Automatise les certificats Let's Encrypt via ACME :

```bash
helm install cert-manager jetstack/cert-manager -n cert-manager \
  --create-namespace --set installCRDs=true
```

```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata: { name: letsencrypt, namespace: default }
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: ops@example.com
    privateKeySecretRef: { name: letsencrypt-key }
    solvers:
      - http01: { ingress: { class: nginx } }
```

### 8.3 Gateway API (futur)

Nouvelle API (GA depuis K8s 1.30) plus expressive, basée sur 3 ressources : `GatewayClass`, `Gateway`, `HTTPRoute`. Destinée à remplacer Ingress à terme.

---

## 9. ConfigMaps et Secrets

### 9.1 ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata: { name: app-config }
data:
  LOG_LEVEL: debug
  app.properties: |
    server.port=8080
    feature.flag.x=true
```

Consommation :

```yaml
envFrom:
  - configMapRef: { name: app-config }
# ou
env:
  - name: LOG_LEVEL
    valueFrom:
      configMapKeyRef: { name: app-config, key: LOG_LEVEL }
# ou en volume
volumes:
  - name: config
    configMap: { name: app-config }
volumeMounts:
  - name: config
    mountPath: /etc/app
```

### 9.2 Secret

Mêmes mécanismes que ConfigMap, données encodées en **base64** (≠ chiffrées !).

```yaml
apiVersion: v1
kind: Secret
metadata: { name: db-creds }
type: Opaque
stringData:
  username: admin
  password: S3cret!
```

### 9.3 Chiffrement at-rest

Configurer `EncryptionConfiguration` sur l'API server :

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources: [secrets]
    providers:
      - aescbc:
          keys:
            - { name: key1, secret: <base64-32bytes> }
      - identity: {}
```

### 9.4 Solutions externes

- **External Secrets Operator** : synchronise Secrets depuis Vault, AWS Secrets Manager, GCP Secret Manager.
- **Sealed Secrets** (Bitnami) : Secrets chiffrés stockables en Git.
- **SOPS** + **Helm Secrets** : encryption avec KMS/PGP.

---

## 10. Volumes et stockage

### 10.1 Types de volumes

- `emptyDir` : éphémère, partagé entre conteneurs d'un pod.
- `hostPath` : monte un chemin du nœud (à éviter en prod).
- `configMap` / `secret` : projetés en fichiers.
- `persistentVolumeClaim` : volume persistant.
- `projected` : combinaison.

### 10.2 PV / PVC / StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata: { name: fast-ssd }
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "4000"
reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: data }
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: fast-ssd
  resources: { requests: { storage: 50Gi } }
```

### 10.3 Access modes

| Mode | Description |
|---|---|
| `ReadWriteOnce` (RWO) | Monté en R/W par 1 seul nœud |
| `ReadOnlyMany` (ROX) | R/O par plusieurs nœuds |
| `ReadWriteMany` (RWX) | R/W par plusieurs nœuds (NFS, CephFS) |
| `ReadWriteOncePod` (RWOP) | 1 seul pod (K8s 1.27+) |

---

## 11. Namespaces, labels, sélecteurs

### 11.1 Namespaces

Isolation logique. Par défaut : `default`, `kube-system`, `kube-public`, `kube-node-lease`.

```bash
kubectl create namespace prod
kubectl apply -f app.yaml -n prod
kubectl get all -n prod
```

Quotas :

```yaml
apiVersion: v1
kind: ResourceQuota
metadata: { name: rq, namespace: prod }
spec:
  hard:
    pods: "50"
    requests.cpu: "20"
    requests.memory: 40Gi
    limits.cpu: "40"
    limits.memory: 80Gi
```

### 11.2 Labels et annotations

- **Labels** : clé/valeur pour **identifier**/sélectionner (`app=web`, `env=prod`).
- **Annotations** : métadonnées **non-sélectionnables** (description, hash, owner).

Selecteurs :

```bash
kubectl get pods -l 'env=prod,tier=frontend'
kubectl get pods -l 'env in (prod,staging),tier!=db'
```

---

## 12. RBAC et sécurité

### 12.1 RBAC

4 ressources :

- **Role** : permissions dans un **namespace**.
- **ClusterRole** : permissions cluster-wide.
- **RoleBinding** / **ClusterRoleBinding** : lie un sujet (User, Group, ServiceAccount) à un rôle.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata: { name: pod-reader, namespace: dev }
rules:
  - apiGroups: [""]
    resources: [pods, pods/log]
    verbs: [get, list, watch]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata: { name: alice-pod-reader, namespace: dev }
subjects:
  - kind: User
    name: alice@example.com
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### 12.2 ServiceAccount

```yaml
apiVersion: v1
kind: ServiceAccount
metadata: { name: app-sa, namespace: prod }
---
# Dans le Pod
spec:
  serviceAccountName: app-sa
```

### 12.3 Pod Security Standards (PSS)

Trois profils : `privileged`, `baseline`, `restricted`. Appliqués via labels de namespace :

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: prod
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

### 12.4 SecurityContext

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  capabilities:
    drop: [ALL]
  seccompProfile: { type: RuntimeDefault }
```

### 12.5 OPA Gatekeeper / Kyverno

Policies "as code" : interdire `latest`, exiger des labels, des requests/limits, scanner d'images, etc.

```yaml
# Kyverno
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata: { name: require-labels }
spec:
  validationFailureAction: enforce
  rules:
    - name: require-team-label
      match: { any: [{ resources: { kinds: [Deployment] } }] }
      validate:
        message: "Le label 'team' est requis"
        pattern:
          metadata: { labels: { team: "?*" } }
```

---

## 13. Resource management et autoscaling

### 13.1 Requests / Limits

- **Requests** : minimum garanti, utilisé par le scheduler.
- **Limits** : maximum autorisé. CPU throttlé, RAM = OOMKilled si dépassé.

### 13.2 QoS Classes

| Class | Condition |
|---|---|
| **Guaranteed** | requests == limits pour tous conteneurs et toutes ressources |
| **Burstable** | au moins une request définie, mais != limits |
| **BestEffort** | aucune request/limit |

Les BestEffort sont évincés en premier.

### 13.3 HorizontalPodAutoscaler

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: web }
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target: { type: Utilization, averageUtilization: 70 }
    - type: Resource
      resource:
        name: memory
        target: { type: Utilization, averageUtilization: 80 }
```

Pré-requis : `metrics-server` installé.

### 13.4 VerticalPodAutoscaler

Ajuste les **requests** d'un pod automatiquement (mode `Off`, `Initial`, `Auto`).

### 13.5 Cluster Autoscaler

Ajoute/supprime des nœuds selon les pods Pending. Sur EKS : **Karpenter** est aujourd'hui souvent préféré.

### 13.6 KEDA

Autoscaling **event-driven** (Kafka lag, RabbitMQ queue, Prometheus query…).

---

## 14. Helm

**Helm** = gestionnaire de paquets K8s (charts = templates).

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install my-redis bitnami/redis -n cache --create-namespace \
  --set auth.password=changeme
helm upgrade my-redis bitnami/redis --reuse-values --version 19.0.0
helm rollback my-redis 1
helm uninstall my-redis -n cache
helm list -A
helm get values my-redis
```

### Structure d'un chart

```
mychart/
├── Chart.yaml
├── values.yaml
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── _helpers.tpl
│   └── NOTES.txt
└── charts/        # subcharts
```

`templates/deployment.yaml` :
```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: {{ include "mychart.fullname" . }} }
spec:
  replicas: {{ .Values.replicaCount }}
  selector: { matchLabels: { app: {{ include "mychart.name" . }} } }
  template:
    metadata: { labels: { app: {{ include "mychart.name" . }} } }
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

---

## 15. Observabilité

### 15.1 Pile classique

| Couche | Outils |
|---|---|
| Métriques | Prometheus + Grafana |
| Logs | Loki / EFK (Elasticsearch + Fluent Bit + Kibana) |
| Traces | Jaeger / Tempo + OpenTelemetry |
| Alerting | Alertmanager / Grafana Alerts |

### 15.2 kube-prometheus-stack

```bash
helm install monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace
```

Inclut Prometheus, Alertmanager, Grafana, node-exporter, kube-state-metrics, ServiceMonitor CRDs.

### 15.3 Logs

Pattern recommandé : stdout/stderr → driver de log → agent (Fluent Bit, Vector) → backend.

---

## 16. Operators et CRDs

### 16.1 CRD (Custom Resource Definition)

Étend l'API K8s avec vos propres ressources :

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata: { name: backups.acme.com }
spec:
  group: acme.com
  scope: Namespaced
  names: { kind: Backup, plural: backups, singular: backup, shortNames: [bk] }
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
                schedule: { type: string }
                target:   { type: string }
```

### 16.2 Operator

Un Operator = CRD + Controller qui réconcilie cet objet. Écrits en Go (**kubebuilder**, **Operator SDK**) ou en Python (**Kopf**), Ansible…

Exemples : **prometheus-operator**, **cert-manager**, **postgres-operator**, **strimzi (Kafka)**.

---

## 17. Service Mesh

Couche L7 entre vos services : mTLS, observabilité, traffic management, retry, circuit breaker.

| Mesh | Caractéristiques |
|---|---|
| **Istio** | Riche, complexe, Envoy sidecar |
| **Linkerd** | Léger, Rust micro-proxy, plus simple |
| **Cilium Service Mesh** | Sans sidecar (eBPF) |
| **Consul Connect** | HashiCorp, multi-plateforme |

---

## 18. GitOps

Principe : **Git est la source de vérité**. Un agent dans le cluster synchronise les manifests Git → cluster.

| Outil | Approche |
|---|---|
| **ArgoCD** | UI riche, multi-cluster, applications, sync policies |
| **Flux** | Plus orienté GitOps "pur", composé d'opérateurs (kustomize-controller, helm-controller…) |

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata: { name: web, namespace: argocd }
spec:
  project: default
  source:
    repoURL: https://github.com/acme/k8s
    targetRevision: main
    path: apps/web/overlays/prod
  destination:
    server: https://kubernetes.default.svc
    namespace: prod
  syncPolicy:
    automated: { prune: true, selfHeal: true }
```

---

## 19. Troubleshooting

### 19.1 Méthodologie

```bash
kubectl get pods -n <ns>           # status
kubectl describe pod <p>           # events
kubectl logs <p> --previous        # logs du conteneur crashé
kubectl get events -n <ns> --sort-by=.lastTimestamp
kubectl exec -it <p> -- sh
```

### 19.2 Erreurs fréquentes

| Status | Cause probable | Remède |
|---|---|---|
| **Pending** | Pas de nœud assez gros, PVC non bound, taints | `describe`, vérifier requests, autoscaler |
| **ImagePullBackOff** | Image inexistante, registry privé sans imagePullSecret | Vérifier tag, secret docker-registry |
| **CrashLoopBackOff** | App crash, mauvais command, healthcheck KO | Logs, `--previous` |
| **OOMKilled** | Limit mémoire dépassée | Augmenter limits ou optimiser app |
| **Init:Error** | initContainer en échec | `logs <pod> -c <init>` |
| **CreateContainerConfigError** | Secret/ConfigMap absent | `describe` pour le détail |
| **NodeNotReady** | kubelet down, disk full, réseau | `kubectl describe node`, ssh node |

### 19.3 Ephemeral debug container

```bash
kubectl debug -it web-abcde --image=nicolaka/netshoot --target=web
```

---

## 20. Bonnes pratiques en production

1. **Toujours** définir `requests` et `limits`.
2. **Probes** liveness + readiness sur chaque container.
3. **PodDisruptionBudget** pour les services critiques.
4. **TopologySpreadConstraints** ou anti-affinity pour répartir entre AZ/nœuds.
5. **Image immuable**, jamais `latest`. Scanner avec trivy/grype.
6. **runAsNonRoot**, **readOnlyRootFilesystem**, drop ALL capabilities.
7. **NetworkPolicies** par défaut deny-all.
8. **RBAC** principe du moindre privilège.
9. **Backups etcd** + tests de restauration.
10. **Multi-AZ** control plane, ≥ 3 nœuds masters.
11. **Upgrades réguliers** (1 version mineure tous les 4 mois).
12. **GitOps** + revue de code obligatoire.
13. **Observabilité dès J1** (logs, metrics, traces).
14. **Tests E2E** post-déploiement.
15. **Limites de namespace** (ResourceQuota, LimitRange).

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata: { name: web-pdb }
spec:
  minAvailable: 2
  selector: { matchLabels: { app: web } }
```

```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels: { app: web }
```

---

## Conclusion

Vous avez désormais une vue complète et opérationnelle de Kubernetes. Étapes recommandées :

1. Pratiquez les [exercices avancés](./EXERCICES_AVANCES.md).
2. Préparez votre entretien avec le [guide d'interview](./INTERVIEW.md).
3. Visez les certifications **CKAD**, **CKA**, puis **CKS**.
4. Contribuez à un opérateur open-source ou écrivez le vôtre.

