# Terraform — Préparation à l'entretien

> ~55 questions/réponses en français par niveau pour un entretien DevOps/Cloud/SRE focalisé Terraform/OpenTofu.

---

## Junior

<details>
<summary>1. Qu'est-ce que Terraform ?</summary>

Outil **Infrastructure as Code** open-source (HashiCorp) qui permet de décrire son infrastructure en HCL (HashiCorp Configuration Language) et de la créer/modifier/détruire via des **providers** (AWS, Azure, GCP, Kubernetes, etc.) de manière **déclarative**.
</details>

<details>
<summary>2. Qu'est-ce que l'IaC ?</summary>

**Infrastructure as Code** : décrire l'infra (VMs, réseaux, DBs…) dans du code versionné. Avantages : reproductibilité, audit, code review, rollback, documentation vivante.
</details>

<details>
<summary>3. Différence déclaratif vs impératif ?</summary>

- **Déclaratif** (Terraform) : on décrit l'état souhaité, l'outil calcule les actions.
- **Impératif** (shell, boto3) : on décrit les actions étape par étape.
</details>

<details>
<summary>4. Cycle de vie d'une commande Terraform ?</summary>

`init` → `fmt`/`validate` → `plan` → `apply` → `destroy`.

- `init` : télécharge providers et initialise le backend.
- `plan` : montre les changements.
- `apply` : applique.
- `destroy` : détruit.
</details>

<details>
<summary>5. Qu'est-ce qu'un provider ?</summary>

Un plugin (binaire séparé) qui sait parler à une API (AWS, Azure, GitHub, Cloudflare…). Téléchargé à `terraform init`. Configuré via le bloc `provider`.
</details>

<details>
<summary>6. Qu'est-ce qu'une resource ?</summary>

Un bloc HCL qui déclare un objet à gérer (ex. `aws_instance`, `kubernetes_deployment`). Référence : `<type>.<nom>.<attribut>`.
</details>

<details>
<summary>7. À quoi sert le state ?</summary>

Fichier JSON qui mappe chaque ressource HCL à son objet réel (ID, attributs). Permet à Terraform de savoir ce qu'il a créé et de calculer les diffs au prochain plan.
</details>

<details>
<summary>8. Différence variable / output / local ?</summary>

- **variable** : input du module.
- **output** : sortie exposée (utile pour modules et inspection).
- **local** : calcul intermédiaire interne au module.
</details>

<details>
<summary>9. Comment passer une valeur à une variable ?</summary>

Par ordre de priorité croissant :
1. Default
2. `.tfvars` auto-chargés
3. `-var-file=prod.tfvars`
4. `-var "key=value"`
5. `TF_VAR_key=value` (env)
</details>

<details>
<summary>10. Qu'est-ce qu'un data source ?</summary>

Lecture d'une ressource **existante** (non gérée par ce state) pour utiliser ses attributs dans le code. Ex. : `data "aws_ami" "ubuntu" {...}`.
</details>

---

## Intermédiaire

<details>
<summary>11. Différence count vs for_each ?</summary>

- `count` : index numérique. Si on supprime un élément au milieu, **toutes les ressources suivantes** sont recréées (décalage d'index).
- `for_each` : itère sur set/map ; chaque ressource a une clé stable. **Préféré**.
</details>

<details>
<summary>12. Qu'est-ce qu'un module ?</summary>

Un dossier de `.tf` réutilisable. Le dossier racine est lui-même un module. Permet de factoriser et composer l'infra.
</details>

<details>
<summary>13. Quelles sources de modules connaissez-vous ?</summary>

Local (`./modules/x`), Git, GitHub, Terraform Registry (public ou privé), S3, HTTP. Toujours **pinner une version** (`?ref=v1.2.0` ou `version = "~> 5.0"`).
</details>

<details>
<summary>14. Qu'est-ce qu'un backend ?</summary>

Lieu de stockage du state. **Local** (fichier) par défaut, **distant** (S3+DynamoDB, GCS, Azure Blob, Terraform Cloud) pour la collaboration et le verrouillage.
</details>

<details>
<summary>15. À quoi sert le state locking ?</summary>

Éviter que deux personnes/CI exécutent un apply simultanément, ce qui corromprait le state. Sur S3 : utilisé via DynamoDB (table `LockID`).
</details>

<details>
<summary>16. Comment importer une ressource existante ?</summary>

- Classique : `terraform import <addr> <id>` puis écrire le code conforme.
- 1.5+ : `import` block dans le code + `terraform plan -generate-config-out=`.
</details>

<details>
<summary>17. Qu'est-ce que le drift ?</summary>

Écart entre l'état réel et le state Terraform, causé par des modifications manuelles ou par d'autres outils. Détectable via `terraform plan`. Solution : réconcilier via le code (modifier le HCL ou `import` pour les nouvelles ressources).
</details>

<details>
<summary>18. À quoi sert `terraform fmt` ?</summary>

Reformate les fichiers `.tf` selon le style canonique. À mettre en pre-commit et bloquant en CI (`terraform fmt -check`).
</details>

<details>
<summary>19. Différence `provisioner` vs `user_data` ?</summary>

- `user_data` : injecté au démarrage de l'instance (cloud-init). Préféré, simple, idempotent.
- `provisioner` (`remote-exec`, `local-exec`) : Terraform se connecte et exécute des commandes. Fragile, **déconseillé** par HashiCorp.

Pour de la config plus poussée : Packer (image pré-cuite) ou Ansible après Terraform.
</details>

<details>
<summary>20. Qu'est-ce que `lifecycle.create_before_destroy` ?</summary>

Crée la nouvelle ressource avant de détruire l'ancienne, pour éviter un downtime. Utile pour les SGs/LBs renommés (sinon AWS refuse à cause du conflit de nom).
</details>

<details>
<summary>21. À quoi sert `lifecycle.ignore_changes` ?</summary>

Ignorer certains attributs lors du diff. Utile quand un autre système modifie ces attributs (autoscaling, opérateur K8s, tags ajoutés par AWS Config…).
</details>

<details>
<summary>22. Qu'est-ce qu'un workspace Terraform CLI ?</summary>

Plusieurs **states** isolés sur le **même code**. Pratique pour de l'éphémère. Pour des envs très différents (variables, providers), préférer une structure **par dossier**.
</details>

<details>
<summary>23. À quoi sert `terraform state mv` ?</summary>

Renommer une ressource dans le state (suite à un rename du code, déplacement vers un module) sans la recréer. Alternative 1.5+ : `moved` blocks dans le code (préférée car versionnée).
</details>

<details>
<summary>24. Comment gérer plusieurs régions/comptes AWS ?</summary>

Plusieurs blocs `provider "aws"` avec des **alias**, puis référencer dans chaque ressource avec `provider = aws.us`. Pour multi-account : `assume_role` dans le provider.
</details>

<details>
<summary>25. Comment générer un fichier dynamiquement ?</summary>

`templatefile("user_data.tpl", { name = "web", port = 8080 })` qui interpole les variables dans un fichier template.
</details>

---

## Senior

<details>
<summary>26. Comment structurer un repo Terraform à grande échelle ?</summary>

- `modules/` : modules réutilisables, versionnés.
- `envs/{dev,staging,prod}/` : compositions par environnement, **states séparés**.
- `.github/workflows/` : CI/CD avec OIDC.
- `policies/` : OPA/conftest.
- `Makefile` ou `taskfile` pour les commandes courantes.
- README + `terraform-docs` auto pour chaque module.
- **Terragrunt** pour les cas extrêmes (DRY backend, multi-account).
</details>

<details>
<summary>27. Comment gérer les secrets en Terraform ?</summary>

- **Jamais** dans le code (ils finissent dans le state).
- Lire depuis Vault, AWS Secrets Manager, GCP Secret Manager via `data` sources.
- Variables marquées `sensitive = true`.
- Backend chiffré (S3 SSE-KMS).
- IAM minimal pour le runner Terraform.
- OIDC pour CI/CD (pas de clés long-terme).
</details>

<details>
<summary>28. Comment sécuriser le state ?</summary>

- Backend distant chiffré + KMS.
- Bucket S3 privé, versionné, MFA delete.
- Lock DynamoDB.
- IAM strict (lecture/écriture limitée).
- Pas de download local non chiffré.
- Audit logs S3 + CloudTrail.
</details>

<details>
<summary>29. Différence Terraform vs Ansible ?</summary>

- **Terraform** : provisioning d'infra immuable, déclaratif, state.
- **Ansible** : configuration de systèmes vivants, orchestration, agentless, pas de state.

Complémentaires : Terraform crée les VMs, Ansible configure l'OS et déploie l'app.
</details>

<details>
<summary>30. Terraform vs CloudFormation ?</summary>

| | Terraform | CloudFormation |
|---|---|---|
| Cloud | Multi | AWS only |
| Langage | HCL | YAML/JSON |
| State | Géré explicitement | Service AWS |
| Modules | Oui (registry) | Nested stacks |
| Drift detection | plan | Service AWS |
| Communauté | Énorme | AWS-centrée |
</details>

<details>
<summary>31. Terraform vs Pulumi ?</summary>

- **Terraform** : DSL HCL, plus simple, vaste communauté.
- **Pulumi** : vrais langages (TS/Python/Go/.NET), boucles natives, tests "vrais".

Pulumi est puissant mais ajoute une couche de complexité. Pour les équipes IaC pures, Terraform reste le standard.
</details>

<details>
<summary>32. Comment tester du Terraform ?</summary>

- `terraform fmt -check` + `terraform validate` (bloquant en CI).
- **tflint** (lint statique).
- **tfsec / checkov / KICS** (sécurité).
- **conftest + OPA** sur le plan JSON.
- **terraform test** (1.6+) : tests natifs HCL.
- **Terratest** (Go) : tests end-to-end provisionnant réellement.
- **Infracost** pour estimer le coût des changements.
</details>

<details>
<summary>33. Comment intégrer Terraform dans une CI/CD ?</summary>

- `plan` sur PR (commenté dans la PR).
- Revue obligatoire.
- `apply` automatique après merge (ou manuel selon criticité).
- OIDC pour les credentials.
- Lock state via DynamoDB.
- Notifications Slack.
- Drift detection nightly.
- Outils SaaS : Atlantis, Spacelift, env0, Terraform Cloud.
</details>

<details>
<summary>34. Comment refactorer du Terraform sans recréer les ressources ?</summary>

- `moved` blocks (1.5+) : versionner les renommages dans le code.
- `terraform state mv` : ad-hoc.
- `import` blocks : faire entrer une ressource existante dans le state.

Toujours vérifier avec `terraform plan` : doit afficher "no changes".
</details>

<details>
<summary>35. Qu'est-ce qu'OpenTofu ?</summary>

Fork open-source de Terraform créé en 2023 suite au changement de licence (BSL) de HashiCorp. Maintenu par la Linux Foundation. **Drop-in compatible** (binaire `tofu`). Communauté active, divergences progressives (chiffrement de state natif, etc.).
</details>

<details>
<summary>36. Quand utiliser Terragrunt ?</summary>

- Multi-account / multi-region nécessitant **DRY** (backend, providers).
- Beaucoup d'environnements isolés.
- Besoin d'orchestrer plusieurs modules ensemble (`run-all apply`).

Inconvénient : couche supplémentaire à maintenir.
</details>

<details>
<summary>37. Comment gérer la dépendance entre 2 layers Terraform ?</summary>

- **Remote state data source** : `data "terraform_remote_state" "network" { backend = "s3" ... }`.
- **Outputs Terraform Cloud / Spacelift** consommés cross-projets.
- Variables d'env publiées dans un store (SSM Parameter Store, Consul).

Préférer : passer des valeurs **explicites** via variables d'env ou data sources (`aws_vpc { tags ... }`), plutôt que de coupler les states.
</details>

<details>
<summary>38. Que se passe-t-il si le state est corrompu ?</summary>

- S3 versioning : restaurer une version précédente.
- `terraform state push` (en dernier recours).
- Backup régulier hors S3 (snapshot manuel).
- `terraform refresh` pour resynchroniser depuis le cloud (mais pas de garantie complète).
- En urgence : reconstruire en réimportant les ressources.
</details>

<details>
<summary>39. Comment Terraform calcule-t-il le plan ?</summary>

1. Parse le HCL.
2. Construit un graphe de dépendances (DAG).
3. Pour chaque ressource : compare `desired (HCL+var)` vs `current (state)` vs `refreshed (API)`.
4. Génère une liste d'actions : `+` (create), `~` (update), `-` (destroy), `-/+` (replace).
5. Affiche les diffs.

L'apply exécute les actions en respectant le graphe (parallélisme configurable avec `-parallelism`).
</details>

<details>
<summary>40. À quoi sert `depends_on` explicite ?</summary>

Forcer une dépendance que Terraform ne peut détecter via les références (ex. : effet de bord d'une IAM policy sur une autre ressource). À utiliser **avec parcimonie** ; les dépendances implicites sont préférables.
</details>

---

## Architecte / Lead

<details>
<summary>41. Scénario : un collègue modifie manuellement un SG en console AWS. Comment gérez-vous ?</summary>

1. `terraform plan` détectera le drift.
2. Décider : la modif est-elle légitime ?
   - **Non** : `terraform apply` rétablit l'état désiré.
   - **Oui** : adapter le code HCL pour refléter, faire PR.
3. Long terme : restreindre les accès IAM en prod (lecture seule), drift detection automatique, alertes.
</details>

<details>
<summary>42. Comment migrer une ressource d'un state à un autre ?</summary>

```bash
# Source
terraform state mv -state-out=/tmp/x.tfstate <addr> <addr>
# Destination
cd ../other-project
terraform init
terraform state push /tmp/x.tfstate    # ou import manuellement
```

Solution moderne : utiliser `moved` blocks + restructurer le code.
</details>

<details>
<summary>43. Quelle stratégie de versioning pour les modules ?</summary>

- SemVer strict (MAJOR.MINOR.PATCH).
- Tags Git correspondant aux releases.
- Changelog tenu à jour.
- Tests automatisés sur les exemples (terratest).
- Documentation auto via `terraform-docs`.
- Pinning strict en prod (`?ref=v1.2.0`), `~>` minor en dev.
</details>

<details>
<summary>44. Comment gérer la migration d'un provider major ?</summary>

1. Lire le changelog et les guides d'upgrade.
2. Tester en staging d'abord.
3. Pinner précisément (`= 4.67.0`) puis migrer vers `~> 5.0`.
4. `terraform plan` pour identifier les changements.
5. Préparer un rollback (snapshot state).
6. Communiquer le timing à l'équipe.
</details>

<details>
<summary>45. Pourquoi éviter les très gros states ?</summary>

- `plan` lent (relisant chaque ressource).
- Risque accru de corruption.
- Blast radius : un apply qui foire peut tout casser.
- Lock plus long.
- Limites API cloud (rate-limit).

**Solution** : découper en **layers** (network, security, compute, app) avec des states séparés, reliés par data sources / remote state.
</details>

<details>
<summary>46. Comment gérer plusieurs équipes sur Terraform ?</summary>

- Séparation des states par domaine/équipe.
- Modules partagés versionnés (registry interne).
- RBAC sur le backend / TF Cloud workspaces.
- Conventions de naming, tags, structure (Backstage scaffolders).
- Code review obligatoire.
- Policies as code communes.
- Onboarding documenté.
</details>

<details>
<summary>47. Quels patterns pour réduire les coûts Terraform ?</summary>

- **Infracost** en PR pour visualiser le coût.
- Modules économes par défaut (single-AZ en dev).
- Variables `enable_*` pour activer/désactiver des composants coûteux selon l'env.
- Tags Cost-Center / Owner systématiques pour FinOps.
- Schedule auto-destroy pour les envs éphémères (CI).
</details>

<details>
<summary>48. Comment garantir la conformité (PCI, SOC2…) avec Terraform ?</summary>

- Policies OPA / Sentinel obligatoires (CIS Benchmarks).
- Tags obligatoires (DataClassification, Owner).
- Chiffrement at-rest et in-transit forcé.
- Backup/snapshot configurés.
- Logging activé (CloudTrail, VPC Flow Logs).
- Pull requests + traçabilité Git.
- Audits réguliers (tfsec, checkov, Prowler en complément).
</details>

<details>
<summary>49. Que pensez-vous de OpenTofu vs Terraform ?</summary>

- **OpenTofu** : licence MPL 2.0 réellement open-source, gouvernance Linux Foundation, communautaire.
- **Terraform** : licence BSL depuis 2023 (restrictive pour outils concurrents), reste leader d'écosystème.

Pour un projet vendor-neutral, **OpenTofu** est un choix pertinent. Compatibilité globalement OK jusqu'à 1.6, divergences progressives.
</details>

<details>
<summary>50. Quel futur pour Terraform ?</summary>

- **OpenTofu** rivalisera de plus en plus, surtout dans la sphère open-source / Europe.
- **Crossplane** challenge sur l'angle K8s-native / GitOps.
- **Pulumi** continue de monter dans les équipes dev-heavy.
- **AI assistée** (Terraform Cloud + LLM) pour générer/auditer du HCL.
- Convergence avec **policy-as-code** native.
- Renforcement testing (terraform test, mocking).
</details>

---

## Scénarios pratiques

<details>
<summary>51. Vous héritez d'un Terraform de 5000 lignes monolithique. Plan d'action ?</summary>

1. Snapshot du state, backup.
2. Mettre la CI en place (plan only) pour stopper le drift.
3. Cartographier les ressources (`terraform state list`).
4. Identifier les **layers naturels** (network, IAM, compute…).
5. Extraire un layer en module, migrer son state via `moved` blocks et `state mv`.
6. Tester en staging d'abord.
7. Itérer layer par layer.
8. Ajouter tests + policies à chaque étape.
</details>

<details>
<summary>52. Comment provisionner un nouveau micro-service en 1 commande ?</summary>

- Module "service template" qui prend `name`, `port`, `replicas`, `db` etc. et crée tout le nécessaire (ECR repo, ECS service, ALB target group, IAM, DNS).
- Backstage / Cookiecutter pour générer le repo applicatif + le fichier `.tf` correspondant.
- Pipeline auto-déclenchée à la création de la PR Terraform.
</details>

<details>
<summary>53. Que faire si `terraform apply` plante au milieu ?</summary>

- Le state est mis à jour pour les ressources déjà créées. Le lock est libéré.
- Relancer `terraform apply` : Terraform continuera où il s'est arrêté.
- Si une ressource est dans un état corrompu : `terraform taint <addr>` (déprécié) ou `-replace=<addr>`.
- Vérifier les API cloud pour ressources orphelines (créées hors state).
</details>

<details>
<summary>54. Comment éviter de tout détruire par accident ?</summary>

- `lifecycle { prevent_destroy = true }` sur ressources critiques.
- IAM : compte CI sans permission delete sur la prod.
- Workflow CI : approvals obligatoires pour `destroy`.
- Pas de `terraform destroy` en local sur prod.
- Backups réguliers + tests de restauration.
</details>

<details>
<summary>55. Quel outil pour générer la doc d'un module ?</summary>

`terraform-docs` :

```bash
terraform-docs markdown table . > README.md
```

Intégrable en pre-commit hook ou CI.
</details>

---

## Conseils

- Maîtrisez **state**, **modules**, **for_each**, **lifecycle**.
- Comprenez les compromis : **workspaces** vs dossiers, **count** vs **for_each**, **provisioners** vs cloud-init.
- Sachez parler de **CI/CD** (OIDC, plan en PR, policies).
- Préparez 1 anecdote de **refactor de state** ou **migration**.
- Restez à jour sur **OpenTofu** et le débat licence.
- Lisez les guides d'upgrade des providers AWS/Azure (changements fréquents).

