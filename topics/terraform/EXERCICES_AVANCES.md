# Terraform — Exercices avancés (Débutant → Expert)

> 25 exercices progressifs en français pour pratiquer Terraform (et OpenTofu).
> Cible principale AWS, mais facilement adaptables à Azure/GCP.

> Prérequis : compte AWS (free tier suffit), AWS CLI configuré, Terraform 1.5+.

---

## Niveau Débutant

### Exercice 1 — Premier provider

**Objectif** : initialiser un projet AWS, lister la version Terraform.

<details>
<summary>Solution</summary>

`main.tf` :
```hcl
terraform {
  required_version = ">= 1.5"
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
  }
}
provider "aws" { region = "eu-west-1" }
```

```bash
terraform init
terraform version
```
</details>

---

### Exercice 2 — Un bucket S3

**Objectif** : créer un bucket S3 nommé `tf-demo-<random>` avec tags.

<details>
<summary>Solution</summary>

```hcl
resource "random_pet" "suffix" { length = 2 }

resource "aws_s3_bucket" "demo" {
  bucket = "tf-demo-${random_pet.suffix.id}"
  tags   = { Owner = "me", ManagedBy = "terraform" }
}
```

(Ajouter le provider random dans `required_providers`.)

```bash
terraform init && terraform apply
```
</details>

---

### Exercice 3 — Variables

**Objectif** : paramétrer le nom de bucket et l'environnement.

<details>
<summary>Solution</summary>

```hcl
variable "env"         { type = string ; default = "dev" }
variable "bucket_name" { type = string }

resource "aws_s3_bucket" "b" {
  bucket = "${var.bucket_name}-${var.env}"
  tags   = { Env = var.env }
}
```

```bash
terraform apply -var bucket_name=acme -var env=prod
```
</details>

---

### Exercice 4 — Outputs

**Objectif** : exposer l'ARN et l'URL du bucket.

<details>
<summary>Solution</summary>

```hcl
output "arn" { value = aws_s3_bucket.b.arn }
output "url" { value = "https://${aws_s3_bucket.b.bucket}.s3.amazonaws.com" }
```
</details>

---

### Exercice 5 — Destroy

**Objectif** : détruire toutes les ressources.

<details>
<summary>Solution</summary>

```bash
terraform destroy
# ou cibler
terraform destroy -target=aws_s3_bucket.b
```
</details>

---

## Niveau Intermédiaire

### Exercice 6 — Backend S3 + DynamoDB

**Objectif** : configurer un backend distant avec verrouillage.

<details>
<summary>Solution</summary>

Voir le guide complet section 11.3. Créer le bucket et la table à la main une fois, puis :

```hcl
terraform {
  backend "s3" {
    bucket         = "acme-tfstate"
    key            = "demo/terraform.tfstate"
    region         = "eu-west-1"
    encrypt        = true
    dynamodb_table = "tf-locks"
  }
}
```

```bash
terraform init -migrate-state
```
</details>

---

### Exercice 7 — VPC custom

**Objectif** : créer un VPC `10.0.0.0/16` avec 2 subnets publics et 2 privés.

<details>
<summary>Solution</summary>

```hcl
locals {
  azs = ["eu-west-1a", "eu-west-1b"]
}

resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  tags                 = { Name = "demo" }
}

resource "aws_subnet" "public" {
  for_each                = toset(local.azs)
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.${index(local.azs, each.key) + 1}.0/24"
  availability_zone       = each.key
  map_public_ip_on_launch = true
  tags                    = { Name = "public-${each.key}" }
}

resource "aws_subnet" "private" {
  for_each          = toset(local.azs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.${index(local.azs, each.key) + 11}.0/24"
  availability_zone = each.key
  tags              = { Name = "private-${each.key}" }
}
```
</details>

---

### Exercice 8 — EC2 + Security Group

**Objectif** : déployer une EC2 t3.micro avec SG ouvert sur 22 et 80.

<details>
<summary>Solution</summary>

```hcl
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]
  filter { name = "name"  values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"] }
}

resource "aws_security_group" "web" {
  name   = "web-sg"
  vpc_id = aws_vpc.main.id

  ingress { from_port = 22 ; to_port = 22 ; protocol = "tcp" ; cidr_blocks = ["0.0.0.0/0"] }
  ingress { from_port = 80 ; to_port = 80 ; protocol = "tcp" ; cidr_blocks = ["0.0.0.0/0"] }
  egress  { from_port = 0  ; to_port = 0  ; protocol = "-1"  ; cidr_blocks = ["0.0.0.0/0"] }
}

resource "aws_instance" "web" {
  ami                    = data.aws_ami.ubuntu.id
  instance_type          = "t3.micro"
  subnet_id              = values(aws_subnet.public)[0].id
  vpc_security_group_ids = [aws_security_group.web.id]
  user_data = <<-EOT
    #!/bin/bash
    apt update && apt install -y nginx
    echo "<h1>Hi from $(hostname)</h1>" > /var/www/html/index.html
  EOT
}

output "ip" { value = aws_instance.web.public_ip }
```
</details>

---

### Exercice 9 — count vs for_each

**Objectif** : créer 3 instances avec `count`, puis refactor en `for_each` sur une map.

<details>
<summary>Solution</summary>

```hcl
# Version count
resource "aws_instance" "c" {
  count         = 3
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"
  tags          = { Name = "web-${count.index}" }
}

# Version for_each
variable "instances" {
  default = {
    web1 = { type = "t3.micro" }
    web2 = { type = "t3.small" }
    web3 = { type = "t3.micro" }
  }
}

resource "aws_instance" "fe" {
  for_each      = var.instances
  ami           = data.aws_ami.ubuntu.id
  instance_type = each.value.type
  tags          = { Name = each.key }
}
```
</details>

---

### Exercice 10 — Module réutilisable

**Objectif** : créer un module local `s3_bucket` qui prend `name`, `versioning`, `tags`.

<details>
<summary>Solution</summary>

`modules/s3_bucket/variables.tf` :
```hcl
variable "name"       { type = string }
variable "versioning" { type = bool, default = true }
variable "tags"       { type = map(string), default = {} }
```

`modules/s3_bucket/main.tf` :
```hcl
resource "aws_s3_bucket" "this" {
  bucket = var.name
  tags   = var.tags
}

resource "aws_s3_bucket_versioning" "v" {
  bucket = aws_s3_bucket.this.id
  versioning_configuration { status = var.versioning ? "Enabled" : "Suspended" }
}
```

`modules/s3_bucket/outputs.tf` :
```hcl
output "arn" { value = aws_s3_bucket.this.arn }
output "id"  { value = aws_s3_bucket.this.id }
```

Usage :
```hcl
module "logs" {
  source     = "./modules/s3_bucket"
  name       = "acme-logs"
  versioning = true
  tags       = { Env = "prod" }
}
```
</details>

---

### Exercice 11 — Dynamic block

**Objectif** : générer dynamiquement les règles ingress d'un SG depuis une liste.

<details>
<summary>Solution</summary>

Voir guide complet section 16.
</details>

---

### Exercice 12 — Multi-environnement par dossier

**Objectif** : structurer un repo `envs/{dev,prod}` partageant des modules.

<details>
<summary>Solution</summary>

```
infra/
├── modules/
│   └── vpc/
├── envs/
│   ├── dev/
│   │   ├── main.tf      (module "vpc" { source="../../modules/vpc" cidr="10.0.0.0/16" })
│   │   └── backend.tf
│   └── prod/
│       ├── main.tf      (cidr="10.10.0.0/16")
│       └── backend.tf   (key="prod/...")
```

```bash
cd envs/prod && terraform init && terraform apply
```
</details>

---

## Niveau Avancé

### Exercice 13 — Import d'une ressource existante

**Objectif** : importer une instance EC2 créée à la console dans le state.

<details>
<summary>Solution</summary>

Méthode classique :
```bash
terraform import aws_instance.legacy i-0123abcd
# Puis ajouter dans .tf un bloc resource conforme
terraform plan        # doit montrer "no changes"
```

Méthode 1.5+ avec `import` block :
```hcl
import {
  to = aws_instance.legacy
  id = "i-0123abcd"
}
resource "aws_instance" "legacy" {
  ami           = "ami-..."
  instance_type = "t3.micro"
  # autres attributs réels
}
```
```bash
terraform plan -generate-config-out=generated.tf   # 1.5+
terraform apply
```
</details>

---

### Exercice 14 — `moved` blocks

**Objectif** : refactor une ressource sans la détruire.

<details>
<summary>Solution</summary>

```hcl
moved {
  from = aws_instance.web
  to   = module.compute.aws_instance.web
}
```

`terraform plan` doit indiquer "0 to add, 0 to change, 0 to destroy" + "Move".
</details>

---

### Exercice 15 — OPA / conftest

**Objectif** : interdire les SGs ouverts au monde avec une policy Rego sur le plan JSON.

<details>
<summary>Solution</summary>

Voir guide complet section 20.3. Workflow :
```bash
terraform plan -out=tfplan
terraform show -json tfplan > plan.json
conftest test plan.json --policy policies/
```
</details>

---

### Exercice 16 — terratest minimal

**Objectif** : tester en Go que le module S3 crée bien un bucket et l'arn est non vide.

<details>
<summary>Solution</summary>

Voir guide complet section 18.3.
</details>

---

### Exercice 17 — EKS

**Objectif** : déployer un EKS minimal en utilisant le module communautaire `terraform-aws-modules/eks/aws`.

<details>
<summary>Solution</summary>

```hcl
module "vpc" {
  source = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"
  name = "eks-vpc"
  cidr = "10.0.0.0/16"
  azs             = ["eu-west-1a","eu-west-1b","eu-west-1c"]
  private_subnets = ["10.0.1.0/24","10.0.2.0/24","10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24","10.0.102.0/24","10.0.103.0/24"]
  enable_nat_gateway = true
}

module "eks" {
  source = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"
  cluster_name    = "demo"
  cluster_version = "1.30"
  vpc_id          = module.vpc.vpc_id
  subnet_ids      = module.vpc.private_subnets
  eks_managed_node_groups = {
    default = {
      instance_types = ["t3.medium"]
      min_size = 2 ; max_size = 5 ; desired_size = 2
    }
  }
}
```

```bash
aws eks update-kubeconfig --name demo --region eu-west-1
kubectl get nodes
```
</details>

---

### Exercice 18 — Multi-providers AWS + Cloudflare

**Objectif** : créer un LB AWS et un record DNS Cloudflare pointant dessus.

<details>
<summary>Solution</summary>

```hcl
terraform {
  required_providers {
    aws        = { source = "hashicorp/aws" }
    cloudflare = { source = "cloudflare/cloudflare" }
  }
}

provider "cloudflare" { api_token = var.cf_token }

resource "aws_lb" "alb" { /* ... */ }

resource "cloudflare_record" "app" {
  zone_id = var.cf_zone_id
  name    = "app"
  type    = "CNAME"
  content = aws_lb.alb.dns_name
  proxied = true
}
```
</details>

---

### Exercice 19 — Workspaces vs structure dossier

**Objectif** : comparer en local : créer 2 workspaces (`dev`, `prod`) et observer où sont les states.

<details>
<summary>Solution</summary>

Avec backend local :
```bash
terraform workspace new dev
terraform workspace new prod
ls -R terraform.tfstate.d/      # un dossier par workspace
```

Conclusion : pour des envs très différents, préférez la structure en dossiers.
</details>

---

### Exercice 20 — `lifecycle` create_before_destroy

**Objectif** : créer un SG dont le rename ne casse pas la prod.

<details>
<summary>Solution</summary>

```hcl
resource "aws_security_group" "web" {
  name_prefix = "web-"        # évite le conflit de nom unique
  lifecycle { create_before_destroy = true }
  ingress { ... }
}
```
</details>

---

## Niveau Expert

### Exercice 21 — Drift detection scheduled

**Objectif** : pipeline GitHub Actions qui exécute `terraform plan -detailed-exitcode` chaque nuit et alerte si exitcode = 2 (changements).

<details>
<summary>Solution</summary>

```yaml
on:
  schedule: [{ cron: "0 3 * * *" }]
jobs:
  drift:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
      - run: terraform init
      - id: plan
        run: |
          set +e
          terraform plan -detailed-exitcode -no-color > plan.txt
          echo "exit=$?" >> $GITHUB_OUTPUT
      - if: steps.plan.outputs.exit == '2'
        run: |
          curl -X POST -H 'Content-Type: application/json' \
            -d "{\"text\":\"⚠️ Drift detected:\\n$(cat plan.txt | head -50)\"}" \
            $SLACK_WEBHOOK
```
</details>

---

### Exercice 22 — Refactor state mv

**Objectif** : renommer plusieurs ressources et utiliser `moved` blocks.

<details>
<summary>Solution</summary>

```hcl
moved { from = aws_instance.w1 to = module.compute.aws_instance.web["w1"] }
moved { from = aws_instance.w2 to = module.compute.aws_instance.web["w2"] }
```

Si plus complexe, scripter `terraform state mv` dans un Makefile.
</details>

---

### Exercice 23 — Terragrunt minimal

**Objectif** : utiliser Terragrunt pour DRY un backend partagé sur 3 envs.

<details>
<summary>Solution</summary>

`terragrunt.hcl` racine :
```hcl
remote_state {
  backend = "s3"
  config = {
    bucket         = "acme-tfstate"
    key            = "${path_relative_to_include()}/terraform.tfstate"
    region         = "eu-west-1"
    encrypt        = true
    dynamodb_table = "tf-locks"
  }
}
```

`envs/prod/network/terragrunt.hcl` :
```hcl
include "root" { path = find_in_parent_folders() }
terraform { source = "${get_repo_root()}/modules/network" }
inputs = { cidr = "10.10.0.0/16", env = "prod" }
```

```bash
cd envs/prod/network && terragrunt apply
```
</details>

---

### Exercice 24 — Provider K8s + Helm

**Objectif** : utiliser Terraform pour installer un chart Helm (Nginx) sur un cluster existant.

<details>
<summary>Solution</summary>

```hcl
provider "kubernetes" {
  config_path = "~/.kube/config"
}
provider "helm" {
  kubernetes { config_path = "~/.kube/config" }
}

resource "helm_release" "nginx" {
  name             = "ingress-nginx"
  repository       = "https://kubernetes.github.io/ingress-nginx"
  chart            = "ingress-nginx"
  version          = "4.10.1"
  namespace        = "ingress-nginx"
  create_namespace = true
  values = [yamlencode({ controller = { replicaCount = 2 } })]
}
```
</details>

---

### Exercice 25 — Sécurité as code (tfsec + checkov + custom rules)

**Objectif** : intégrer tfsec, checkov dans la CI et ajouter une règle conftest custom interdisant les buckets S3 publics.

<details>
<summary>Solution</summary>

```yaml
- run: |
    pip install checkov
    curl -sSL https://raw.githubusercontent.com/aquasecurity/tfsec/master/scripts/install_linux.sh | bash
    tfsec .
    checkov -d . --skip-check CKV_AWS_18    # exemple skip
    terraform plan -out=tfplan && terraform show -json tfplan > plan.json
    conftest test plan.json --policy policies/
```

`policies/s3_public.rego` :
```rego
package main
deny[msg] {
  r := input.resource_changes[_]
  r.type == "aws_s3_bucket_public_access_block"
  r.change.after.block_public_acls == false
  msg := sprintf("S3 bucket %s autorise des ACLs publiques", [r.address])
}
```
</details>

---

## Pour aller plus loin

- Maîtrisez **Terragrunt** pour les très gros déploiements multi-account.
- Construisez votre propre **module communautaire** et publiez-le sur le Terraform Registry.
- Mettez en place un **pipeline GitOps** avec Atlantis ou Terraform Cloud.
- Comparez **OpenTofu** vs **Terraform** sur un projet réel.
- Apprenez **Crossplane** comme alternative cloud-native.

