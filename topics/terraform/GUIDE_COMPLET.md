# Terraform — Le Guide Complet : de 0 à Hero

> Livre de référence en français pour maîtriser Terraform (et OpenTofu), de la première ressource à la prod multi-cloud.

---

## Table des matières

1. [Introduction à l'IaC](#1-introduction-à-liac)
2. [Architecture Terraform](#2-architecture-terraform)
3. [Installation](#3-installation)
4. [Langage HCL](#4-langage-hcl)
5. [Workflow de base](#5-workflow-de-base)
6. [Providers](#6-providers)
7. [Resources](#7-resources)
8. [Variables](#8-variables)
9. [Outputs et locals](#9-outputs-et-locals)
10. [Data sources](#10-data-sources)
11. [State Terraform](#11-state-terraform)
12. [Modules](#12-modules)
13. [Workspaces](#13-workspaces)
14. [Provisioners](#14-provisioners)
15. [Fonctions built-in](#15-fonctions-built-in)
16. [Dynamic blocks & for](#16-dynamic-blocks)
17. [Lifecycle](#17-lifecycle)
18. [Testing](#18-testing)
19. [CI/CD Terraform](#19-cicd-terraform)
20. [Sécurité](#20-sécurité)
21. [Bonnes pratiques](#21-bonnes-pratiques)
22. [Patterns avancés](#22-patterns-avancés)
23. [Migration et OpenTofu](#23-migration-et-opentofu)

---

## 1. Introduction à l'IaC

### 1.1 IaC, c'est quoi ?

**Infrastructure as Code** : décrire son infra (réseaux, VMs, DBs, K8s clusters…) sous forme de fichiers texte versionnables, qui produisent la même infra à chaque exécution.

Bénéfices :
- **Reproductibilité** (dev, staging, prod identiques).
- **Versionning** Git.
- **Code review** + tests.
- **Documentation vivante**.
- **Auditable**, **réversible**.

### 1.2 Déclaratif vs impératif

- **Impératif** (Bash, Python boto3) : "fais ceci, puis cela". L'utilisateur gère l'état.
- **Déclaratif** (Terraform, CloudFormation) : "voilà ce que je veux". L'outil calcule le diff.

### 1.3 Terraform

Créé par **HashiCorp** (2014), écrit en Go, ~110 providers officiels (AWS, Azure, GCP, Kubernetes, GitHub, Datadog…). Modèle :

```
HCL ──► terraform plan ──► graphe de diff ──► terraform apply ──► API cloud
                                            │
                                            └──► state (source de vérité)
```

### 1.4 Terraform vs alternatives

| Outil | Approche | Force | Limite |
|---|---|---|---|
| **Terraform / OpenTofu** | HCL, déclaratif, multi-cloud | Écosystème, multi-provider | State à gérer |
| **CloudFormation** | YAML/JSON, AWS only | Intégration AWS native | Verbosité, non portable |
| **Pulumi** | TS/Python/Go/.NET | Boucles, tests natifs | Coût Pulumi Cloud, courbe |
| **CDK (AWS/k8s)** | TS/Python/Java | Sucre syntaxique | Toujours produit du CFN |
| **Crossplane** | YAML, K8s-native | GitOps, contrôleurs | Maturité variable |
| **Ansible** | YAML, impératif (ish) | Config + provisioning OK | Pas de state ; pas idéal cloud massif |

> En 2023, **OpenTofu** a forké Terraform après le changement de licence HashiCorp (BSL). Communauté Linux Foundation, drop-in compatible (jusqu'aux versions récentes).

---

## 2. Architecture Terraform

```
┌────────────┐  HCL    ┌──────────────────┐
│  *.tf      │────────►│  Terraform Core  │
└────────────┘         │  - Parser HCL    │
                       │  - Graphe        │
                       │  - Planner       │
                       │  - Executor      │
                       └─┬────┬───────────┘
                         │    │
                  plugins│    │state
                         ▼    ▼
                  ┌──────────┐  ┌──────────┐
                  │ Provider │  │ Backend  │
                  │ AWS/GCP  │  │ S3/local │
                  └──────────┘  └──────────┘
                         │
                         ▼
                  ┌──────────┐
                  │ API Cloud│
                  └──────────┘
```

- **Core** : parsing, graph, plan/apply.
- **Providers** : binaires séparés (`terraform-provider-aws`), téléchargés à `init`.
- **State** : représentation persistante de l'infra gérée.
- **Backend** : où le state est stocké (local, S3, GCS, Azure blob, Terraform Cloud).

---

## 3. Installation

### 3.1 Binaire direct

```bash
# Linux/macOS
wget https://releases.hashicorp.com/terraform/1.9.5/terraform_1.9.5_linux_amd64.zip
unzip terraform_1.9.5_linux_amd64.zip && sudo mv terraform /usr/local/bin/
terraform version
```

Windows : télécharger le zip, mettre dans PATH (ou via Chocolatey/Scoop).

### 3.2 Versionner

**tfenv** (Linux/macOS) ou **asdf** (multiplatform) pour gérer plusieurs versions :

```bash
brew install tfenv
tfenv install 1.9.5
tfenv use 1.9.5
```

### 3.3 OpenTofu

```bash
# Drop-in replacement
brew install opentofu
tofu version
```

Toutes les commandes ci-dessous fonctionnent en remplaçant `terraform` par `tofu`.

---

## 4. Langage HCL

### 4.1 Bases

```hcl
# Commentaire
// Commentaire C-style
/* Bloc */

resource "aws_instance" "web" {
  ami           = "ami-0abcd1234"
  instance_type = var.instance_type

  tags = {
    Name = "web-${terraform.workspace}"
    Env  = var.env
  }
}
```

### 4.2 Types

| Type | Exemple |
|---|---|
| `string` | `"hello"` |
| `number` | `42`, `3.14` |
| `bool` | `true` |
| `list(any)` | `["a","b","c"]` |
| `set(string)` | `toset(["a","b"])` |
| `map(string)` | `{ a = "1", b = "2" }` |
| `object` | `{ name = string, age = number }` |
| `tuple` | `["a", 1, true]` |

### 4.3 Heredoc

```hcl
user_data = <<-EOT
  #!/bin/bash
  echo "Hello ${var.name}"
  apt update && apt install -y nginx
EOT
```

### 4.4 Expressions

```hcl
# Ternaire
size = var.env == "prod" ? "large" : "small"

# Interpolation
name = "${var.app}-${var.env}"

# Splat
ips = aws_instance.web[*].public_ip

# Index
first = aws_instance.web[0].id

# For
upper_names = [for n in var.names : upper(n)]
filtered    = [for n in var.names : n if length(n) > 3]
to_map      = { for k, v in var.servers : k => v.ip }
```

---

## 5. Workflow de base

### 5.1 Commandes essentielles

```bash
terraform init                # télécharge providers, initialise backend
terraform fmt -recursive      # reformate les .tf
terraform validate            # validité syntaxique
terraform plan                # voir les changements
terraform plan -out=tfplan    # sauver le plan
terraform apply               # appliquer
terraform apply tfplan        # appliquer le plan sauvé
terraform destroy             # tout détruire
terraform show                # voir le state
terraform output              # voir les outputs
terraform output -json
terraform state list
terraform state show aws_instance.web
terraform refresh             # resync state ↔ réel
terraform graph | dot -Tpng > graph.png
terraform console             # REPL HCL
terraform providers
```

### 5.2 Cycle de vie d'un projet

1. Écrire `main.tf`, `variables.tf`, `outputs.tf`.
2. `terraform init` (une fois).
3. `terraform fmt && terraform validate`.
4. `terraform plan` (à chaque modif).
5. Revue de code.
6. `terraform apply`.
7. État partagé via backend distant.

### 5.3 Exemple minimal AWS

```hcl
# main.tf
terraform {
  required_version = ">= 1.5"
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
  }
}

provider "aws" {
  region = "eu-west-1"
}

resource "aws_instance" "web" {
  ami           = "ami-0c1bc246476a5572b"
  instance_type = "t3.micro"

  tags = { Name = "demo" }
}

output "public_ip" {
  value = aws_instance.web.public_ip
}
```

---

## 6. Providers

### 6.1 Configuration

```hcl
terraform {
  required_providers {
    aws        = { source = "hashicorp/aws",        version = "~> 5.0" }
    kubernetes = { source = "hashicorp/kubernetes", version = "~> 2.30" }
    helm       = { source = "hashicorp/helm",       version = "~> 2.13" }
    cloudflare = { source = "cloudflare/cloudflare", version = "~> 4.40" }
  }
}

provider "aws" {
  region = "eu-west-1"
}
```

### 6.2 Providers multiples (alias)

```hcl
provider "aws" {
  region = "eu-west-1"
}

provider "aws" {
  alias  = "us"
  region = "us-east-1"
}

resource "aws_s3_bucket" "logs_us" {
  provider = aws.us
  bucket   = "my-logs-us"
}
```

### 6.3 Versions

Opérateurs : `~> 5.0` (>=5.0, <6.0), `>= 1.2`, `= 5.10.1`.

**Toujours pinner** en production. Vérouiller via `.terraform.lock.hcl` (commit).

---

## 7. Resources

### 7.1 Syntaxe

```hcl
resource "<type>" "<nom_local>" {
  argument = value
  ...
}
```

Référence : `<type>.<nom>.<attribut>` (ex. `aws_instance.web.id`).

### 7.2 Meta-arguments

| Meta | Rôle |
|---|---|
| `count` | Créer N ressources (index `count.index`) |
| `for_each` | Itérer sur set/map (clé `each.key`, valeur `each.value`) |
| `depends_on` | Dépendance explicite |
| `provider` | Choisir un alias |
| `lifecycle` | create_before_destroy, prevent_destroy, ignore_changes |

### 7.3 `count` vs `for_each`

```hcl
# count : index numérique
resource "aws_instance" "w" {
  count         = 3
  ami           = var.ami
  instance_type = "t3.micro"
  tags = { Name = "web-${count.index}" }
}

# for_each sur set/map : clés stables (préféré)
resource "aws_instance" "w" {
  for_each      = toset(["web1", "web2", "web3"])
  ami           = var.ami
  instance_type = "t3.micro"
  tags = { Name = each.key }
}
```

> **for_each** est généralement préféré : les clés stables évitent qu'un remove en milieu de liste recrée toutes les suivantes (cas avec count).

### 7.4 Dépendances

Implicites : Terraform les détecte via les références. Explicites :

```hcl
resource "aws_iam_role_policy_attachment" "p" {
  role       = aws_iam_role.r.name
  policy_arn = aws_iam_policy.pol.arn
  depends_on = [aws_iam_role.r, aws_iam_policy.pol]    # rarement nécessaire
}
```

---

## 8. Variables

### 8.1 Définition

```hcl
variable "env" {
  type        = string
  default     = "dev"
  description = "Environnement"
  validation {
    condition     = contains(["dev", "staging", "prod"], var.env)
    error_message = "env doit être dev, staging ou prod."
  }
  sensitive = false
}

variable "instance_count" {
  type    = number
  default = 1
}

variable "tags" {
  type    = map(string)
  default = {}
}

variable "subnets" {
  type = list(object({
    cidr = string
    az   = string
  }))
}
```

### 8.2 Sources et précédence

Du plus faible au plus fort :

1. Defaults
2. Fichier `terraform.tfvars` (auto-chargé)
3. Fichiers `*.auto.tfvars` (auto-chargés)
4. `-var-file=prod.tfvars` (CLI)
5. `-var "env=prod"` (CLI)
6. `TF_VAR_env=prod` (env var)

### 8.3 Exemple `.tfvars`

```hcl
# prod.tfvars
env            = "prod"
instance_count = 5
tags = {
  Owner = "platform"
  Cost  = "eng"
}
```

```bash
terraform apply -var-file=prod.tfvars
```

### 8.4 Sensitive

```hcl
variable "db_password" {
  type      = string
  sensitive = true
}
```

Les valeurs sensibles ne s'affichent pas dans le plan/apply (mais restent dans le state — voir section sécurité).

---

## 9. Outputs et locals

### 9.1 Outputs

```hcl
output "vpc_id" {
  value       = aws_vpc.main.id
  description = "ID du VPC créé"
  sensitive   = false
}

output "endpoint" {
  value     = "${aws_lb.app.dns_name}:${var.port}"
}
```

```bash
terraform output vpc_id
terraform output -json
```

### 9.2 Locals

```hcl
locals {
  name_prefix = "${var.app}-${var.env}"
  common_tags = merge(var.tags, {
    Env       = var.env
    ManagedBy = "terraform"
  })
}

resource "aws_instance" "w" {
  ami           = var.ami
  instance_type = "t3.micro"
  tags          = merge(local.common_tags, { Name = "${local.name_prefix}-web" })
}
```

---

## 10. Data sources

Lire des ressources existantes (non gérées par ce state).

```hcl
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }
}

data "aws_vpc" "default" {
  default = true
}

resource "aws_instance" "w" {
  ami           = data.aws_ami.ubuntu.id
  subnet_id     = tolist(data.aws_subnets.default.ids)[0]
  instance_type = "t3.micro"
}
```

---

## 11. State Terraform

### 11.1 Qu'est-ce que c'est ?

Un fichier JSON (`terraform.tfstate`) qui :

- Mappe chaque ressource HCL à son objet réel (ID cloud).
- Stocke les attributs (pour calculer les diffs).
- Permet à Terraform de connaître ce qu'il a créé.

**Sans state, Terraform ne sait rien** de ce qu'il a fait précédemment.

### 11.2 Local vs Remote

| | Local | Remote |
|---|---|---|
| Lieu | `./terraform.tfstate` | S3, GCS, Azure blob, TF Cloud |
| Concurrent | Conflits | Verrouillage natif |
| Sécurité | Fichier en clair | Chiffrement at-rest + IAM |
| Collaboration | Très difficile | Adapté |

### 11.3 Backend S3 + DynamoDB lock

```hcl
terraform {
  backend "s3" {
    bucket         = "acme-tfstate"
    key            = "prod/network/terraform.tfstate"
    region         = "eu-west-1"
    encrypt        = true
    dynamodb_table = "tf-locks"
    kms_key_id     = "alias/tf-state"
  }
}
```

Création du bucket et de la table :

```bash
aws s3api create-bucket --bucket acme-tfstate --region eu-west-1
aws s3api put-bucket-versioning --bucket acme-tfstate --versioning-configuration Status=Enabled
aws s3api put-bucket-encryption --bucket acme-tfstate --server-side-encryption-configuration \
  '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"}}]}'

aws dynamodb create-table --table-name tf-locks \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST
```

### 11.4 Manipulations utiles

```bash
terraform state list
terraform state show aws_instance.w
terraform state mv aws_instance.w aws_instance.web        # renommer
terraform state rm aws_instance.w                          # retirer du state (pas du cloud)
terraform import aws_instance.w i-0123abcd                 # importer existant
terraform state pull > state.json                          # télécharger
terraform state push state.json                            # uploader (dangereux)
terraform force-unlock <LOCK_ID>                           # libérer un lock bloqué
```

### 11.5 Drift

État réel ≠ state. Causes : changements manuels en console, autres outils. Détecter avec `terraform plan` (montre les diffs). Toujours réconcilier en réappliquant via le code (ou `import` si nouvelle ressource).

---

## 12. Modules

### 12.1 Définition

Un **module** = un dossier contenant `.tf`. Le dossier racine (`.`) est lui-même un module ("root module").

### 12.2 Structure recommandée

```
modules/
└── vpc/
    ├── main.tf
    ├── variables.tf
    ├── outputs.tf
    ├── versions.tf
    └── README.md
```

### 12.3 Utilisation

```hcl
module "vpc" {
  source = "./modules/vpc"

  name            = "prod"
  cidr            = "10.0.0.0/16"
  azs             = ["eu-west-1a", "eu-west-1b", "eu-west-1c"]
  public_subnets  = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  private_subnets = ["10.0.11.0/24", "10.0.12.0/24", "10.0.13.0/24"]
}

resource "aws_instance" "w" {
  subnet_id = module.vpc.private_subnet_ids[0]
}
```

### 12.4 Sources de modules

```hcl
source = "./modules/vpc"                                   # local
source = "git::https://github.com/acme/tf-vpc.git//?ref=v1.2.0"  # Git
source = "github.com/acme/tf-vpc"                          # GitHub
source = "terraform-aws-modules/vpc/aws"                   # Registry public
source = "terraform-aws-modules/vpc/aws//modules/subnets"  # sous-module
source = "app.terraform.io/acme/vpc/aws"                   # private registry
```

### 12.5 Versionnement

Toujours pinner avec `?ref=v1.2.0` (Git) ou `version = "~> 5.0"` (registry).

### 12.6 Composition

```hcl
module "vpc"     { source = "..." ; ... }
module "cluster" { source = "..." ; vpc_id = module.vpc.id ; ... }
module "app"     { source = "..." ; cluster_endpoint = module.cluster.endpoint }
```

---

## 13. Workspaces

### 13.1 Workspaces CLI

```bash
terraform workspace list
terraform workspace new staging
terraform workspace select prod
terraform workspace show
```

Chaque workspace a son propre **state** isolé (`env:/staging/...`).

```hcl
resource "aws_instance" "w" {
  instance_type = terraform.workspace == "prod" ? "t3.large" : "t3.micro"
  tags = { Env = terraform.workspace }
}
```

### 13.2 Limites

Tous les workspaces partagent le **même code**. Pour des différences plus profondes, préférer une structure **par dossier** :

```
infra/
├── modules/
│   ├── network/
│   └── compute/
├── envs/
│   ├── dev/
│   │   ├── main.tf
│   │   └── backend.tf
│   ├── staging/
│   └── prod/
```

C'est généralement la **bonne pratique**.

### 13.3 Terraform Cloud workspaces

Tout autre concept : un workspace TFC = un dépôt + un set de variables + un state distant. À ne pas confondre avec les workspaces CLI.

---

## 14. Provisioners

### 14.1 Types

```hcl
resource "aws_instance" "w" {
  ami           = var.ami
  instance_type = "t3.micro"

  connection {
    type        = "ssh"
    user        = "ubuntu"
    private_key = file("~/.ssh/id_rsa")
    host        = self.public_ip
  }

  provisioner "remote-exec" {
    inline = [
      "sudo apt update",
      "sudo apt install -y nginx",
    ]
  }

  provisioner "file" {
    source      = "app.conf"
    destination = "/tmp/app.conf"
  }

  provisioner "local-exec" {
    command = "echo ${self.public_ip} > ips.txt"
  }
}
```

### 14.2 À éviter quand possible

HashiCorp recommande explicitement d'**éviter** les provisioners. Préférer :

- **cloud-init** / `user_data` pour bootstrap.
- **Packer** pour images AMI préconfigurées.
- **Ansible** appelé après Terraform.

Les provisioners sont fragiles, non idempotents, et complexifient l'état.

---

## 15. Fonctions built-in

Catégories : **string**, **numeric**, **collection**, **encoding**, **filesystem**, **date/time**, **hash/crypto**, **IP**, **type conversion**, **specialized**.

### Exemples

```hcl
# Strings
upper("hello")              # "HELLO"
format("%-10s : %d", "k", 1)
join(",", ["a","b","c"])    # "a,b,c"
split(",", "a,b,c")
replace("foo-bar", "-", "_")
trimspace("  hello  ")
regex("^\\d+", "123abc")    # "123"

# Collections
length([1,2,3])             # 3
contains(["a","b"], "a")    # true
merge({a=1}, {b=2})         # {a=1, b=2}
lookup({a=1}, "a", "def")
distinct([1,1,2,2,3])
flatten([[1,2],[3,4]])
toset([...])
zipmap(["a","b"], [1,2])

# Encoding
jsondecode(file("data.json"))
yamldecode(file("config.yml"))
base64encode("hello")
templatefile("user_data.tpl", { name = "web" })

# Filesystem
file("script.sh")
fileexists("config.yml")
fileset(".", "**/*.tf")

# Type
tolist(toset(["a","b"]))
tostring(42)
tonumber("3.14")

# Hash
sha256("hello")
filemd5("script.sh")

# Network
cidrsubnet("10.0.0.0/16", 8, 2)   # "10.0.2.0/24"
cidrhost("10.0.0.0/24", 5)        # "10.0.0.5"

# Date
timestamp()
formatdate("YYYY-MM-DD", timestamp())
```

### templatefile

`user_data.tpl` :
```bash
#!/bin/bash
echo "Hello ${name}"
apt install -y ${join(" ", packages)}
```

```hcl
user_data = templatefile("${path.module}/user_data.tpl", {
  name     = "web"
  packages = ["nginx", "htop"]
})
```

---

## 16. Dynamic blocks

Utiles quand un bloc imbriqué doit être généré dynamiquement.

```hcl
resource "aws_security_group" "sg" {
  name = "web"

  dynamic "ingress" {
    for_each = var.ingress_rules
    content {
      from_port   = ingress.value.from
      to_port     = ingress.value.to
      protocol    = ingress.value.protocol
      cidr_blocks = ingress.value.cidrs
    }
  }
}
```

Avec :
```hcl
variable "ingress_rules" {
  type = list(object({ from = number, to = number, protocol = string, cidrs = list(string) }))
  default = [
    { from = 80,  to = 80,  protocol = "tcp", cidrs = ["0.0.0.0/0"] },
    { from = 443, to = 443, protocol = "tcp", cidrs = ["0.0.0.0/0"] },
  ]
}
```

---

## 17. Lifecycle

```hcl
resource "aws_instance" "w" {
  # ...

  lifecycle {
    create_before_destroy = true        # éviter downtime sur recréation
    prevent_destroy       = true        # refuse destroy (sécurité)
    ignore_changes        = [tags["LastModified"]]
    replace_triggered_by  = [null_resource.config_version]
  }
}
```

### `ignore_changes`

Utile quand un attribut est modifié par un autre système (autoscaling, opérateur K8s, etc.).

### `create_before_destroy`

Indispensable pour les ressources avec un `name` unique (LBs, security groups) lors d'un rename forçant un replace.

---

## 18. Testing

### 18.1 Builtin `terraform test` (1.6+)

```hcl
# tests/vpc.tftest.hcl
run "create_vpc" {
  command = plan
  variables {
    cidr = "10.0.0.0/16"
  }
  assert {
    condition     = aws_vpc.main.cidr_block == "10.0.0.0/16"
    error_message = "Le CIDR est incorrect"
  }
}
```

```bash
terraform test
```

### 18.2 Static analysis

```bash
# Format & validate
terraform fmt -check -recursive
terraform validate

# Lint
tflint --recursive

# Security
tfsec .
checkov -d .

# Cost
infracost breakdown --path .
```

### 18.3 Terratest (Go)

```go
package test

import (
    "testing"
    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/stretchr/testify/assert"
)

func TestVPC(t *testing.T) {
    opts := &terraform.Options{TerraformDir: "../examples/basic"}
    defer terraform.Destroy(t, opts)
    terraform.InitAndApply(t, opts)
    vpcID := terraform.Output(t, opts, "vpc_id")
    assert.NotEmpty(t, vpcID)
}
```

---

## 19. CI/CD Terraform

### 19.1 GitHub Actions

```yaml
name: terraform
on: [pull_request, push]
jobs:
  plan:
    runs-on: ubuntu-latest
    permissions:
      id-token: write   # OIDC AWS
      contents: read
      pull-requests: write
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/gh-tf
          aws-region: eu-west-1
      - uses: hashicorp/setup-terraform@v3
        with: { terraform_version: 1.9.5 }
      - run: terraform fmt -check -recursive
      - run: terraform init
      - run: terraform validate
      - run: tflint --recursive
      - run: tfsec .
      - run: terraform plan -out=tfplan -no-color
      - if: github.event_name == 'push' && github.ref == 'refs/heads/main'
        run: terraform apply -auto-approve tfplan
```

### 19.2 Atlantis

Serveur qui écoute les webhooks GitHub/GitLab, exécute `terraform plan/apply` selon les commentaires PR (`atlantis plan`, `atlantis apply`). Auto-hébergé, gratuit.

### 19.3 Terraform Cloud / Spacelift / Env0

SaaS gérant : runs distants, state, RBAC, policies (Sentinel/OPA), notifications, drift detection.

### 19.4 OpenTofu en CI

Remplacer `setup-terraform` par `setup-opentofu` et `terraform` par `tofu`.

---

## 20. Sécurité

### 20.1 Secrets

- **Jamais** de secrets dans les `.tf` (ils finissent dans le state).
- Utiliser : variables d'env (`TF_VAR_db_password`), provider Vault, AWS SM (data source).
- `sensitive = true` cache dans logs mais **pas dans le state**.
- Chiffrer le backend (S3 SSE-KMS).

```hcl
data "aws_secretsmanager_secret_version" "db" {
  secret_id = "prod/db/master"
}

locals {
  db_password = jsondecode(data.aws_secretsmanager_secret_version.db.secret_string)["password"]
}
```

### 20.2 IAM / credentials du runner

- Pas de clés long-terme : **OIDC** (GitHub Actions ↔ AWS/GCP/Azure).
- Rôle minimal (lecture/écriture seulement sur les ressources nécessaires).
- Séparer les rôles `plan` (RO) et `apply` (RW).

### 20.3 Policies as code

- **OPA / conftest** : valider les plans (interdire `0.0.0.0/0`, exiger des tags…).
- **Checkov, tfsec, KICS** : règles prêtes à l'emploi.
- **Sentinel** (TF Cloud Enterprise) : policy framework propriétaire.

Exemple conftest (Rego) :

```rego
package main
deny[msg] {
  resource := input.resource_changes[_]
  resource.type == "aws_security_group_rule"
  cidr := resource.change.after.cidr_blocks[_]
  cidr == "0.0.0.0/0"
  msg := sprintf("SG ouvert au monde : %s", [resource.address])
}
```

```bash
terraform show -json tfplan > plan.json
conftest test plan.json
```

---

## 21. Bonnes pratiques

1. **Structure de repo** par environnement (`envs/dev`, `envs/prod`) + modules réutilisables.
2. **Backend distant** + lock obligatoire.
3. **`.terraform.lock.hcl`** commité.
4. **Versions pinées** (Terraform, providers, modules).
5. **Naming conventions** cohérentes (`${env}-${app}-${resource}`).
6. **Tags** systématiques (Env, Owner, Cost, ManagedBy=terraform).
7. **Pas de provisioners** sauf cas extrême.
8. **`for_each` > `count`** (clés stables).
9. **Modules petits et composables** (un module = une responsabilité).
10. **`terraform fmt` + `validate`** dans la CI (bloquant).
11. **Plan sur PR**, **apply après merge** (jamais d'apply manuel local en prod).
12. **OIDC** pour les credentials cloud.
13. **Drift detection** régulière (`plan` en mode read-only).
14. **Documentation** : `terraform-docs` pour générer le README de chaque module.
15. **Tests** (terraform test, terratest, conftest).
16. **Pas de logique métier dans Terraform** ; pas de "stringly typed" excessif.

---

## 22. Patterns avancés

### 22.1 Multi-environnement (dossiers)

```
envs/prod/
├── main.tf          # appelle modules
├── backend.tf       # backend S3 spécifique
├── terraform.tfvars # valeurs prod
└── providers.tf
```

Chaque env a son propre state, ses propres credentials. Le code commun vit dans `modules/`.

### 22.2 Terragrunt

Wrapper autour de Terraform pour DRY :

```hcl
# envs/prod/network/terragrunt.hcl
include "root" { path = find_in_parent_folders() }
terraform { source = "${get_repo_root()}/modules/network" }
inputs = {
  cidr = "10.0.0.0/16"
  env  = "prod"
}
```

Génère automatiquement le backend, les versions, etc. Très utilisé en multi-account.

### 22.3 Multi-account AWS

Pattern :
- Un compte par environnement.
- Rôles `OrganizationAccountAccessRole`.
- AssumeRole dans les providers :

```hcl
provider "aws" {
  region = "eu-west-1"
  assume_role {
    role_arn = "arn:aws:iam::222233334444:role/TerraformExec"
  }
}
```

### 22.4 Blue/green sur infra

`create_before_destroy` + `lifecycle.ignore_changes` + LBs avec `target_group_arn` paramétré.

### 22.5 Refactor du state

Renommer une ressource ou la déplacer entre modules :

```bash
terraform state mv 'aws_instance.web' 'module.app.aws_instance.web'
```

Pour migrer entre states distincts :

```bash
# Source
terraform state mv -state-out=/tmp/dst.tfstate 'aws_instance.web' 'aws_instance.web'

# Destination
terraform state push /tmp/dst.tfstate
```

(En 1.5+ : utiliser **`moved` blocks** dans le code, plus sûr et code-as-doc.)

```hcl
moved {
  from = aws_instance.web
  to   = module.app.aws_instance.web
}
```

### 22.6 `import` blocks (1.5+)

```hcl
import {
  to = aws_instance.web
  id = "i-0abcd1234"
}

resource "aws_instance" "web" {
  # config conforme à l'instance existante
}
```

`terraform plan` génère le diff, `apply` enregistre dans le state.

---

## 23. Migration et OpenTofu

### 23.1 Versions Terraform

Suivre la doc d'upgrade entre versions mineures. Backup du state avant. Tester en staging.

### 23.2 Providers

Lire les changelogs ; les majors AWS introduisent souvent des breaking changes. `terraform init -upgrade`.

### 23.3 OpenTofu

Fork open-source de Terraform (Linux Foundation) depuis le passage à BSL en août 2023. Compatible avec Terraform jusqu'à 1.6. Divergences à partir de 1.7+ (chiffrement de state natif, etc.).

Migration : remplacer le binaire, garder le code. Vérifier `.terraform.lock.hcl` car il diffère.

```bash
brew install opentofu
tofu init
tofu plan
```

---

## Conclusion

Terraform / OpenTofu est l'outil pivot de l'IaC en 2025. Maîtrisez :

1. Le **workflow** plan/apply + state distant.
2. **Modules** composables, pinned.
3. **CI/CD avec OIDC** + policies (tfsec, conftest).
4. **Multi-environnement** + multi-account.
5. **Refactoring du state** (moved, import blocks).

Pratiquez les [exercices avancés](./EXERCICES_AVANCES.md) puis préparez votre entretien avec le [guide](./INTERVIEW.md).

