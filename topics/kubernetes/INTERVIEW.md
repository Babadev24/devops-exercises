# Kubernetes — Préparation à l'entretien

> ~60 questions/réponses en français par niveau pour un entretien DevOps/SRE/Cloud focalisé Kubernetes.

---

## Junior

<details>
<summary>1. Qu'est-ce que Kubernetes ?</summary>

Un orchestrateur open-source de conteneurs créé par Google et maintenu par la CNCF. Il automatise le déploiement, la mise à l'échelle, la résilience et la mise à jour d'applications conteneurisées sur un cluster de machines.
</details>

<details>
<summary>2. Différence entre conteneur, Pod et Deployment ?</summary>

- **Conteneur** : une instance d'image (runtime containerd/CRI-O).
- **Pod** : 1+ conteneur(s) partageant réseau et volumes. Unité atomique K8s.
- **Deployment** : abstraction de plus haut niveau qui gère un ReplicaSet, lui-même gérant N Pods. Supporte rolling updates et rollbacks.
</details>

<details>
<summary>3. Qu'est-ce qu'un Service ? Citez les types.</summary>

Une abstraction réseau stable pour accéder à un ensemble de Pods (sélectionnés par labels). Types :
- **ClusterIP** (défaut) : IP interne
- **NodePort** : port sur chaque nœud
- **LoadBalancer** : LB cloud externe
- **ExternalName** : alias DNS
- **Headless** : pas de VIP, DNS renvoie les IPs des Pods
</details>

<details>
<summary>4. À quoi sert un Namespace ?</summary>

Isolation logique au sein d'un cluster : éviter les collisions de noms, appliquer des RBAC / quotas / NetworkPolicies différents par équipe ou environnement.
</details>

<details>
<summary>5. ConfigMap vs Secret ?</summary>

Mêmes APIs/usages, mais le Secret stocke des données sensibles (encodées en base64 par défaut, chiffrables at-rest). Les ConfigMaps stockent de la configuration non sensible.
</details>

<details>
<summary>6. À quoi sert kubectl ?</summary>

CLI principal pour interagir avec l'API Server de Kubernetes : créer/lister/supprimer des ressources, lire les logs, exec dans un container, debug, etc.
</details>

<details>
<summary>7. Qu'est-ce qu'un label ?</summary>

Une paire clé/valeur attachée à un objet, utilisée pour sélectionner des sous-ensembles (par Services, Deployments, HPA, etc.). Exemple : `app=web, env=prod`.
</details>

<details>
<summary>8. Comment exposer une app hors du cluster ?</summary>

- Service `NodePort` (dev)
- Service `LoadBalancer` (cloud)
- **Ingress** + Ingress Controller (HTTP/HTTPS, plus flexible)
- Gateway API (plus moderne)
</details>

<details>
<summary>9. Quel est le rôle d'etcd ?</summary>

Base clé-valeur distribuée (consensus Raft) qui stocke **tout l'état** du cluster (objets, ConfigMaps, Secrets, RBAC…). Source de vérité. À sauvegarder absolument.
</details>

<details>
<summary>10. Différence entre `kubectl apply` et `kubectl create` ?</summary>

- `create` : impératif, échoue si la ressource existe déjà.
- `apply` : déclaratif, crée ou met à jour, conserve les annotations de "last-applied-configuration". Préféré dans les workflows GitOps.
</details>

---

## Intermédiaire

<details>
<summary>11. Décrivez l'architecture du Control Plane.</summary>

- **kube-apiserver** : point d'entrée REST, auth/authz.
- **etcd** : base d'état.
- **kube-scheduler** : assigne les Pods aux nœuds.
- **kube-controller-manager** : boucles de réconciliation.
- (**cloud-controller-manager** : intégration cloud)
</details>

<details>
<summary>12. Comment fonctionne le scheduler ?</summary>

Phase 1 — **Filtering** : éliminer les nœuds non éligibles (resources, nodeSelector, taints, affinité, PV…). Phase 2 — **Scoring** : noter les nœuds restants (spread, image locality, etc.). Le mieux noté est choisi. Personnalisable via le framework de scheduling.
</details>

<details>
<summary>13. Différence entre Deployment et StatefulSet ?</summary>

| Critère | Deployment | StatefulSet |
|---|---|---|
| Identité pods | éphémère, aléatoire | stable (`name-0`, `name-1`) |
| Ordre create/delete | aucun | séquentiel |
| Stockage | partagé/aucun | PVC dédié par pod |
| DNS | via Service | via Headless Service |
| Usage | apps stateless | bases, brokers, quorum |
</details>

<details>
<summary>14. À quoi sert un DaemonSet ?</summary>

Garantir qu'**un pod tourne sur chaque nœud** (ou un sous-ensemble via nodeSelector/affinity). Cas d'usage : agents de logs (Fluent Bit), métriques (node-exporter), CNI, stockage.
</details>

<details>
<summary>15. Différence liveness vs readiness probe ?</summary>

- **liveness** : si elle échoue, kubelet **redémarre** le conteneur.
- **readiness** : si elle échoue, le pod est **retiré des Endpoints** du Service (mais pas redémarré).
- **startup** : désactive les deux autres tant qu'elle n'a pas réussi (utile pour apps à démarrage lent).
</details>

<details>
<summary>16. Qu'est-ce que QoS class ?</summary>

Trois classes :
- **Guaranteed** : requests == limits sur toutes ressources.
- **Burstable** : requests définies mais ≠ limits.
- **BestEffort** : rien de défini.

En cas de pression mémoire, les BestEffort sont évincés en premier, puis Burstable.
</details>

<details>
<summary>17. Différence requests vs limits ?</summary>

- **requests** : minimum garanti, utilisé par le scheduler pour le placement.
- **limits** : maximum autorisé. CPU au-delà est throttlé, RAM au-delà déclenche **OOMKill**.
</details>

<details>
<summary>18. Comment fonctionne le HPA ?</summary>

Il interroge un Metrics API (metrics-server pour CPU/RAM, custom/external metrics pour le reste), calcule le ratio `currentValue / targetValue`, et ajuste `replicas` du Deployment cible (entre min et max). Période par défaut 15s.
</details>

<details>
<summary>19. Qu'est-ce qu'un Ingress et pourquoi a-t-on besoin d'un Ingress Controller ?</summary>

Un Ingress = ressource déclarative décrivant les règles de routage HTTP/HTTPS. L'Ingress Controller (Nginx, Traefik, ALB…) lit ces ressources et configure le **vrai** reverse proxy qui traite le trafic. Sans Controller, l'Ingress ne fait rien.
</details>

<details>
<summary>20. Comment fonctionne le DNS dans le cluster ?</summary>

**CoreDNS** s'exécute en pod(s) dans `kube-system`. Tous les Pods sont configurés avec `nameserver <ClusterIP CoreDNS>`. Format des entrées : `<svc>.<ns>.svc.cluster.local`. Pour StatefulSet : `<pod>.<svc>.<ns>.svc.cluster.local`.
</details>

<details>
<summary>21. Quels types de volumes connaissez-vous ?</summary>

`emptyDir`, `hostPath`, `configMap`, `secret`, `projected`, `persistentVolumeClaim`, `csi` (drivers cloud : EBS, Azure Disk, GCP PD, Ceph, NFS).
</details>

<details>
<summary>22. PV, PVC, StorageClass : différences ?</summary>

- **PV** : ressource concrète (un volume).
- **PVC** : demande de volume par l'utilisateur (taille, mode d'accès).
- **StorageClass** : "type" de stockage avec provisioner dynamique. Quand un PVC référence une SC, un PV est créé automatiquement.
</details>

<details>
<summary>23. RBAC : Role vs ClusterRole ?</summary>

- **Role** : permissions dans un seul namespace.
- **ClusterRole** : permissions cluster-wide (ou non namespacées comme nodes).
- Bindings (`RoleBinding`/`ClusterRoleBinding`) lient un sujet (User/Group/ServiceAccount) à un rôle.
</details>

<details>
<summary>24. Qu'est-ce qu'un ServiceAccount ?</summary>

Une identité utilisée par les pods pour s'authentifier auprès de l'API. Si non spécifié, `default` du namespace est utilisé. On y attache des RBAC pour autoriser des opérations (ex. un opérateur qui crée des Deployments).
</details>

<details>
<summary>25. Comment installer une application avec Helm ?</summary>

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install my-redis bitnami/redis -n cache --create-namespace --set auth.password=xxx
```

Helm = templating + gestionnaire de releases (upgrade, rollback, uninstall).
</details>

<details>
<summary>26. Pourquoi éviter `hostPath` en prod ?</summary>

- Couple le pod à un nœud spécifique.
- Pas de portabilité.
- Risque sécurité (accès au filesystem du nœud).
- Pas de gestion centralisée.

À remplacer par un PV via CSI ou un volume cloud-native.
</details>

<details>
<summary>27. À quoi sert un PodDisruptionBudget ?</summary>

Définit le **nombre minimum de pods** d'un groupe qui doivent rester disponibles lors de perturbations volontaires (drain, upgrade nœud). Garantit la résilience pendant les opérations de maintenance.
</details>

<details>
<summary>28. Comment garantir la haute disponibilité d'une application ?</summary>

- ≥ 2 réplicas + PDB.
- Anti-affinity / TopologySpreadConstraints sur AZ ou nœud.
- Probes liveness/readiness.
- HPA pour absorber les pics.
- StorageClass multi-AZ (ou réplication).
- Cluster multi-AZ avec masters HA.
- Tests de chaos régulièrement.
</details>

<details>
<summary>29. Qu'est-ce qu'une NetworkPolicy ?</summary>

Une règle de filtrage L3/L4 entre pods (par labels, namespace, IP). Par défaut tout est autorisé ; une fois qu'une policy sélectionne un pod, **seul ce qui est explicitement autorisé** passe. Nécessite un CNI compatible (Calico, Cilium…).
</details>

<details>
<summary>30. À quoi sert un Headless Service ?</summary>

`clusterIP: None`. Aucun VIP/équilibrage. Le DNS renvoie directement les IPs des pods. Utilisé avec les StatefulSets pour adresser individuellement chaque pod.
</details>

---

## Senior

<details>
<summary>31. Comment fonctionne le rolling update d'un Deployment ?</summary>

Le Deployment crée un nouveau ReplicaSet et scale-up progressivement (limité par `maxSurge`) tout en scale-down l'ancien (limité par `maxUnavailable`). Les pods sont validés via readiness probes avant d'être ajoutés aux Endpoints.
</details>

<details>
<summary>32. Comment faire un déploiement blue/green ou canary ?</summary>

- **Blue/green** : 2 Deployments (`v1`, `v2`), bascule via le sélecteur du Service.
- **Canary natif K8s** : 2 Deployments avec replicas proportionnels (90/10).
- **Outils** : Argo Rollouts, Flagger (Flux) — gèrent les étapes, le poids, les analyses Prometheus, l'auto-promotion ou rollback.
- **Service Mesh** (Istio, Linkerd) pour split de trafic L7 précis.
</details>

<details>
<summary>33. Que se passe-t-il lors d'un OOMKilled ?</summary>

Le conteneur a dépassé sa `memory.limit`. Le kernel envoie SIGKILL. Le conteneur est redémarré selon le `restartPolicy` (en CrashLoopBackOff si récurrent). Solution : augmenter la limit, fixer une fuite mémoire, profiler.
</details>

<details>
<summary>34. Comment debugger un Pod CrashLoopBackOff ?</summary>

```bash
kubectl describe pod <p>            # events, exit code
kubectl logs <p> --previous         # logs du run précédent
kubectl logs <p> -c <container>     # multi-container
kubectl debug -it <p> --image=busybox --target=<c>
```

Vérifier : variables d'env, command/args, probes (trop strictes ?), permissions, secrets manquants, OOM, healthcheck qui échoue.
</details>

<details>
<summary>35. Comment gérer les secrets en production ?</summary>

- Chiffrement at-rest activé sur l'API server.
- Pas de secrets dans Git (sauf SealedSecrets / SOPS).
- **External Secrets Operator** synchronisant depuis Vault/AWS SM/GCP SM.
- IAM via IRSA (EKS) ou Workload Identity (GKE) pour éviter les secrets longue durée.
- Rotation automatique.
- Audit logs sur les accès aux secrets.
</details>

<details>
<summary>36. Comment scaler un cluster ?</summary>

- **Cluster Autoscaler** : ajoute/supprime des nœuds en fonction des pods Pending.
- **Karpenter** (AWS) : choisit dynamiquement les types d'instances optimaux.
- **HPA / VPA / KEDA** pour les pods.
- Surveillance : `kube-state-metrics`, dashboards capacité.
</details>

<details>
<summary>37. Qu'est-ce qu'un Operator ?</summary>

Une combinaison CRD + Controller qui encode l'expertise opérationnelle d'une application (deploy, backup, upgrade, recovery). Exemples : prometheus-operator, cert-manager, postgres-operator, strimzi. Écrits avec Kubebuilder, Operator SDK, Kopf.
</details>

<details>
<summary>38. Différence entre Helm et Kustomize ?</summary>

- **Helm** : templates (Go template) + values + gestion de release. Très puissant mais verbose.
- **Kustomize** : overlays purement YAML, patch déclaratif. Intégré à kubectl. Plus simple, sans logique de release.
- **Combinaison** : Helm pour les charts complexes + Kustomize pour customiser sans forker.
</details>

<details>
<summary>39. Service mesh : pourquoi, quand ?</summary>

Apporte mTLS entre services, retry/circuit-breaker, observabilité L7, traffic-splitting, policies de sécurité — **sans modifier le code**. Coût : complexité, latence (sidecar), ressources. À envisager quand le nombre de services dépasse ~20, ou pour la conformité (chiffrement obligatoire).
</details>

<details>
<summary>40. Comment gérer le cycle de vie des nœuds (drain, cordon) ?</summary>

```bash
kubectl cordon <node>             # marquer non-schedulable
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
# maintenance...
kubectl uncordon <node>
```

PDBs assurent la disponibilité pendant le drain.
</details>

<details>
<summary>41. GitOps : ArgoCD vs Flux ?</summary>

| Critère | ArgoCD | Flux |
|---|---|---|
| UI | Riche | Minimaliste (CLI) |
| Multi-cluster | Natif | Via Fleet/Karmada |
| Helm/Kustomize | OK | OK (controllers dédiés) |
| Image automation | Plugin | Natif (image-reflector + automation-controller) |
| Approche | App-centric | Toolkit modulaire |
</details>

<details>
<summary>42. Quelle stratégie de backup K8s recommandez-vous ?</summary>

- **etcd snapshot** réguliers + restauration testée.
- **Velero** pour sauvegarder les ressources K8s + les PVs (snapshots cloud).
- Backup base de données **applicatif** (pg_dump, mysqldump…) en plus.
- Stockage hors-cluster, multi-région.
- Procédure DR testée trimestriellement.
</details>

<details>
<summary>43. Comment sécuriser un cluster Kubernetes ?</summary>

- RBAC minimal, audit logs activés.
- Pod Security Standards `restricted` par défaut.
- NetworkPolicies deny-all par défaut.
- Chiffrement etcd at-rest.
- Image scanning (Trivy) en CI + admission policy (Kyverno).
- Pas de `hostPath`, `hostNetwork`, `privileged` sauf justification.
- TLS partout (mTLS via mesh).
- Auth via OIDC, MFA.
- Mises à jour régulières (versions support upstream).
- Outils : kube-bench (CIS), falco (runtime), trivy-operator.
</details>

<details>
<summary>44. Différence entre Docker (runtime) et containerd ?</summary>

Docker était une suite complète (CLI + daemon + runtime). Depuis K8s 1.24, le shim docker a été supprimé. K8s utilise désormais directement un runtime CRI : **containerd** (extrait de Docker) ou **CRI-O**. Cela n'empêche pas les images Docker de fonctionner — elles respectent le standard OCI.
</details>

<details>
<summary>45. Comment fonctionne le scheduling avec taints/tolerations ?</summary>

- **Taint** sur un nœud : `key=value:effect` (`NoSchedule`, `PreferNoSchedule`, `NoExecute`).
- Aucun pod n'y est planifié sauf s'il a une **toleration** correspondante.
- Usage : isoler GPU, nœuds dédiés équipe, control plane.
</details>

<details>
<summary>46. Différence affinity vs nodeSelector ?</summary>

- `nodeSelector` : exact match labels (simple).
- `affinity` : expressions plus riches (`In`, `NotIn`, `Exists`), `required` ou `preferred`, anti-affinity entre pods.
</details>

<details>
<summary>47. Quel est l'impact d'un Pod sans `resources` définies ?</summary>

QoS = BestEffort → premier évincé en pression. Le scheduler n'a aucune info pour placer correctement. Risque de "noisy neighbor". Toujours définir au minimum `requests`.
</details>

<details>
<summary>48. Comment fonctionne kube-proxy ?</summary>

Programme les règles réseau de chaque nœud pour les Services. Modes :
- **iptables** : par défaut, scalable jusqu'à des milliers de services.
- **IPVS** : meilleures perfs au-delà.
- **eBPF** (Cilium) : remplace kube-proxy, plus rapide.
</details>

<details>
<summary>49. Différence entre Ingress et Gateway API ?</summary>

Gateway API (GA en 1.30) propose un modèle plus riche : séparation des rôles (infra, app, dev), support TCP/UDP, expression plus claire, extensibilité. Destinée à remplacer Ingress.
</details>

<details>
<summary>50. Que vérifie un audit Kubernetes (kube-bench) ?</summary>

Conformité au benchmark **CIS Kubernetes** : permissions fichiers du control plane, options de l'API server (anonymous-auth disabled, audit logs, encryption), kubelet, RBAC, etc. À exécuter régulièrement.
</details>

---

## Architecte / SRE

<details>
<summary>51. Comment dimensionner un cluster K8s ?</summary>

Analyser :
- **Workloads** : nombre de pods, requests CPU/RAM cumulées.
- **Headroom** : prévoir 30 % de marge.
- **Multi-AZ** : minimum 3 zones.
- **Taille nœud** : équilibre entre flexibilité (petits nœuds) et efficacité (gros).
- **etcd** : sur disque SSD dédié, faible latence (< 10 ms fsync).
- **API server** : ressources et nombre de masters (3 ou 5 pour quorum).
</details>

<details>
<summary>52. Stratégie multi-cluster ?</summary>

Approches :
- **Cluster par environnement** (dev/staging/prod) : isolation simple.
- **Multi-région** (DR, latence) : ArgoCD/Flux pour synchroniser.
- **Fleet / Karmada / kcp** pour gestion centralisée.
- **Service mesh multi-cluster** (Istio) pour communications inter-cluster.
- Trade-off : complexité vs résilience.
</details>

<details>
<summary>53. Comment gérer la conformité multi-tenant ?</summary>

- **Namespace par tenant** + RBAC isolant.
- **ResourceQuota** + **LimitRange**.
- **NetworkPolicies** deny-all par défaut.
- **OPA/Gatekeeper** ou **Kyverno** pour les policies.
- Pour isolation forte : **clusters virtuels** (vCluster, Capsule) ou clusters dédiés.
</details>

<details>
<summary>54. Comment optimiser les coûts d'un cluster ?</summary>

- Right-sizing : VPA + recommendations.
- Spot/Preemptible nodes pour workloads tolérants.
- Cluster Autoscaler / Karpenter pour scale-down agressif.
- Goldilocks, kube-resource-report pour visualiser.
- Bin packing : optimiser les requests.
- Réservations capacity sur cloud.
- Consolider les environnements non-prod.
</details>

<details>
<summary>55. Que pensez-vous des "operators" maison vs charts Helm ?</summary>

- Chart Helm = packaging et deploy. Suffit pour la plupart des cas.
- Operator = nécessaire dès qu'il faut **du jour 2** automatisé (backup, failover, scale event-driven, reconcile continu). Surcoût de développement et maintenance.
- Règle : commencer par Helm, basculer en operator si l'opérationnel devient complexe.
</details>

<details>
<summary>56. Comment gérez-vous les CRDs et leurs versions ?</summary>

- Versionner via `v1alpha1`, `v1beta1`, `v1`.
- Conversion webhook pour migrer entre versions.
- Stocker la version `storage`.
- Tests d'upgrade systématiques.
- Documentation claire des breaking changes.
</details>

<details>
<summary>57. Scénario : la latence API server explose à 5s. Que faites-vous ?</summary>

1. Vérifier metrics API server (`apiserver_request_duration_seconds`).
2. Vérifier etcd (latence fsync, leader change).
3. Identifier les clients agressifs (`apiserver_request_total` par user-agent) — souvent un controller boggué.
4. Activer **APF** (API Priority and Fairness) si pas déjà fait.
5. Augmenter `--max-requests-inflight` / scaling vertical des masters.
6. À long terme : sharding, replicas etcd.
</details>

<details>
<summary>58. Comment migrer une app legacy vers K8s ?</summary>

1. Conteneuriser (Dockerfile).
2. Externaliser config (12-factor) → ConfigMap/Secret.
3. Logs vers stdout.
4. Gérer le stockage (PVC ou externe).
5. Health endpoints pour probes.
6. Manifests + Helm chart.
7. Déployer en parallèle (strangler fig), test canary.
8. Décommissionner l'ancien.
</details>

<details>
<summary>59. Comment surveiller la santé d'un cluster ?</summary>

- **Metrics** : kube-state-metrics + node-exporter + cAdvisor → Prometheus.
- **Logs** : Loki / EFK.
- **Traces** : OpenTelemetry → Tempo/Jaeger.
- **Events** : `kubectl get events`, k8s-event-exporter.
- **Audit logs** : envoi vers SIEM.
- **Synthetic checks** : Blackbox exporter.
- **SLO/SLI** : burn rate alerts.
</details>

<details>
<summary>60. Quel futur pour Kubernetes ?</summary>

- **eBPF** partout (Cilium, observabilité, sécurité).
- **WASM** comme runtime alternatif aux conteneurs (Spin, WasmEdge).
- **Gateway API** remplaçant Ingress.
- **AI workloads** (KServe, KubeFlow, NVIDIA operators).
- **Edge** (k3s, KubeEdge) à grande échelle.
- **Multi-cluster** standardisé (Karmada, Kratix).
- **Sustainability** (carbon-aware scheduling).
- Simplification de l'expérience développeur (Backstage, internal dev platforms).
</details>

---

## Conseils

- Maîtrisez `kubectl` jusqu'au bout des doigts ; entraînez-vous sur **killer.sh** (simulateur CKAD/CKA).
- Ayez 1-2 incidents concrets à raconter avec votre démarche.
- Comprenez **pourquoi** une feature existe, pas seulement comment.
- Soyez à jour des releases récentes (Sidecars natifs en 1.29, Gateway API en 1.30, etc.).
- Préparez les questions qui croisent les sujets : "comment exposez-vous une app et gérez vous les secrets et l'observabilité ?".

