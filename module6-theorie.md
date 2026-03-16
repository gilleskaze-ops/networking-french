# Module 6 — Networking as Code avec Terraform
### Partie Théorie

> **Cours Networking Fondamentaux · Orienté Cloud**
> Prérequis : Modules 1 à 5 complétés · Terraform installé · Compte AWS · Durée estimée : 3h

---

## Glossaire

| Terme | Signification | Définition simple |
|-------|--------------|-------------------|
| **Backend** | — | Endroit où Terraform stocke le state (local, S3, Terraform Cloud) |
| **Data Source** | — | Permet de lire des ressources existantes sans les gérer avec Terraform |
| **HCL** | HashiCorp Configuration Language | Langage de Terraform — déclaratif, lisible par les humains |
| **IaC** | Infrastructure as Code | Gérer l'infrastructure comme du code — versionnable, reproductible |
| **Local** | — | Variable calculée dans Terraform — évite la répétition |
| **Module** | — | Ensemble de ressources Terraform réutilisables — comme une fonction |
| **Output** | — | Valeur exportée par un module Terraform — partageable entre modules |
| **Plan** | — | Prévisualisation des changements Terraform avant application |
| **Provider** | — | Plugin Terraform qui sait parler à une API (AWS, GCP, Azure...) |
| **Remote State** | — | State Terraform stocké à distance (S3) — partageable en équipe |
| **Resource** | — | Ressource infrastructure gérée par Terraform (VPC, EC2, SG...) |
| **State** | — | Fichier qui mappe les ressources Terraform aux ressources réelles AWS |
| **Terraform** | — | Outil IaC open source de HashiCorp — déclare l'infrastructure souhaitée |
| **tflint** | — | Linter pour Terraform — détecte les erreurs avant le plan/apply |
| **Variable** | — | Paramètre configurable d'un module ou d'une configuration Terraform |
| **Workspace** | — | Environnement isolé dans Terraform — permet prod/staging/dev séparés |

---

## Sommaire

1. [Terraform en bref — orienté réseau](#1-terraform-en-bref--orienté-réseau)
2. [VPC et Subnets](#2-vpc-et-subnets)
3. [Connectivité Internet](#3-connectivité-internet)
4. [Security Groups](#4-security-groups)
5. [Modules Terraform réseau](#5-modules-terraform-réseau)
6. [Route 53 et DNS](#6-route-53-et-dns)
7. [ALB en Terraform](#7-alb-en-terraform)
8. [VPC Peering](#8-vpc-peering)
9. [Outputs et Remote State](#9-outputs-et-remote-state)
10. [Outils et bonnes pratiques](#10-outils-et-bonnes-pratiques)

---

## 1. Terraform en bref — orienté réseau

### Pourquoi c'est important

Dans les modules précédents, tu as créé des VPC, des Security Groups, des Route Tables — à la main via la console AWS ou l'AWS CLI. C'est bien pour apprendre, mais catastrophique en production :

```
Problèmes de l'infrastructure manuelle :
  → Pas de versioning — impossible de savoir qui a changé quoi
  → Pas de reproductibilité — créer le même réseau en staging
    prend des heures et risque d'être différent
  → Pas d'audit — impossible de voir l'historique des changements
  → Pas de rollback — si tu casses quelque chose, comment revenir ?
  → Drift — la réalité diverge de la documentation

Avec Terraform :
  → Tout est dans des fichiers .tf versionés dans Git
  → Un `terraform apply` recrée exactement le même réseau
    en staging, prod, dev — identique
  → Chaque changement est reviewé comme du code (Pull Request)
  → Rollback = `git revert` + `terraform apply`
  → Terraform détecte le drift entre le code et la réalité
```

### Les concepts essentiels

**Provider** — le plugin qui sait parler à AWS :

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "eu-central-1"
}
```

**Resource** — une ressource AWS que Terraform gère :

```hcl
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"

  tags = {
    Name = "main-vpc"
  }
}
#  └──┬──┘  └──┬──┘
#  type     nom local
#  (AWS)    (dans Terraform)
```

**Variable** — paramètre configurable :

```hcl
variable "vpc_cidr" {
  description = "CIDR block du VPC"
  type        = string
  default     = "10.0.0.0/16"
}

# Utilisation
resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr
}
```

**Output** — valeur exportée :

```hcl
output "vpc_id" {
  description = "ID du VPC créé"
  value       = aws_vpc.main.id
}
```

**Local** — valeur calculée, évite la répétition :

```hcl
locals {
  environment = "production"
  common_tags = {
    Environment = local.environment
    Project     = "mon-app"
    ManagedBy   = "terraform"
  }
}

resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  tags       = local.common_tags
}
```

### Le workflow Terraform

```bash
# 1. Initialiser — télécharge les providers
terraform init

# 2. Formater le code
terraform fmt

# 3. Valider la syntaxe
terraform validate

# 4. Prévisualiser les changements
terraform plan

# Output plan :
# + aws_vpc.main     ← sera créé
# ~ aws_subnet.pub   ← sera modifié
# - aws_sg.old       ← sera supprimé

# 5. Appliquer
terraform apply

# 6. Voir ce qui est géré
terraform state list

# 7. Détruire (lab uniquement !)
terraform destroy
```

### Le State — la mémoire de Terraform

Le **state** est un fichier JSON (`terraform.tfstate`) qui mappe chaque ressource Terraform à sa ressource réelle AWS :

```json
{
  "resources": [
    {
      "type": "aws_vpc",
      "name": "main",
      "instances": [{
        "attributes": {
          "id": "vpc-0abc123def456",
          "cidr_block": "10.0.0.0/16"
        }
      }]
    }
  ]
}
```

```
⚠️ Le state est critique :
  → Ne jamais le modifier manuellement
  → Ne jamais le committer dans Git (contient des secrets)
  → En équipe → stocker dans S3 + DynamoDB (Remote State)
```

---

## 2. VPC et Subnets

### Pourquoi c'est important

Le VPC et ses subnets sont la première chose à créer dans toute architecture AWS — et la plus répétée. En Terraform, on les code une fois, on les paramètre, et on les réutilise pour tous les environnements.

### VPC de base

```hcl
# variables.tf
variable "environment" {
  type    = string
  default = "production"
}

variable "vpc_cidr" {
  type    = string
  default = "10.0.0.0/16"
}

# main.tf
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true   # important pour RDS, EKS
  enable_dns_support   = true

  tags = {
    Name        = "${var.environment}-vpc"
    Environment = var.environment
  }
}
```

### Subnets multi-AZ

Pour créer des subnets dans plusieurs AZ sans répétition, on utilise `for_each` :

```hcl
# variables.tf
variable "public_subnet_cidrs" {
  type    = list(string)
  default = ["10.0.1.0/24", "10.0.2.0/24"]
}

variable "private_subnet_cidrs" {
  type    = list(string)
  default = ["10.0.10.0/24", "10.0.11.0/24"]
}

variable "availability_zones" {
  type    = list(string)
  default = ["eu-central-1a", "eu-central-1b"]
}

# main.tf
resource "aws_subnet" "public" {
  count             = length(var.public_subnet_cidrs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.public_subnet_cidrs[count.index]
  availability_zone = var.availability_zones[count.index]

  # Les instances lancées ici reçoivent une IP publique automatiquement
  map_public_ip_on_launch = true

  tags = {
    Name        = "${var.environment}-subnet-public-${count.index + 1}"
    Environment = var.environment
    Type        = "public"
  }
}

resource "aws_subnet" "private" {
  count             = length(var.private_subnet_cidrs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.private_subnet_cidrs[count.index]
  availability_zone = var.availability_zones[count.index]

  tags = {
    Name        = "${var.environment}-subnet-private-${count.index + 1}"
    Environment = var.environment
    Type        = "private"
  }
}
```

### Data Source — récupérer les AZ disponibles

Au lieu de hardcoder les AZ, on peut les récupérer dynamiquement :

```hcl
# Récupère automatiquement les AZ disponibles dans la région
data "aws_availability_zones" "available" {
  state = "available"
}

# Utilisation
resource "aws_subnet" "public" {
  count             = 2
  availability_zone = data.aws_availability_zones.available.names[count.index]
  # ...
}
```

### Référencer des ressources entre elles

```hcl
# aws_vpc.main.id → l'ID du VPC créé ci-dessus
# Terraform résout les dépendances automatiquement

resource "aws_subnet" "public" {
  vpc_id     = aws_vpc.main.id      # référence le VPC
  cidr_block = "10.0.1.0/24"
  # ...
}
```

---

## 3. Connectivité Internet

### Pourquoi c'est important

On a vu dans le Module 4 que c'est la Route Table qui fait la différence entre un subnet public et privé. En Terraform, on code explicitement cette logique — ce qui la rend visible, reviewable et reproductible.

### Internet Gateway

```hcl
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name        = "${var.environment}-igw"
    Environment = var.environment
  }
}
```

### Route Tables publiques

```hcl
# Route Table pour les subnets publics
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = {
    Name        = "${var.environment}-rt-public"
    Environment = var.environment
  }
}

# Associer la RT publique à chaque subnet public
resource "aws_route_table_association" "public" {
  count          = length(aws_subnet.public)
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}
```

### NAT Gateway et Elastic IP

```hcl
# Une Elastic IP par NAT Gateway (une par AZ)
resource "aws_eip" "nat" {
  count  = length(var.public_subnet_cidrs)
  domain = "vpc"

  tags = {
    Name        = "${var.environment}-eip-nat-${count.index + 1}"
    Environment = var.environment
  }

  # L'EIP doit être créée après l'IGW
  depends_on = [aws_internet_gateway.main]
}

# Une NAT Gateway par subnet public (donc par AZ)
resource "aws_nat_gateway" "main" {
  count         = length(var.public_subnet_cidrs)
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id

  tags = {
    Name        = "${var.environment}-nat-${count.index + 1}"
    Environment = var.environment
  }
}
```

### Route Tables privées

```hcl
# Une Route Table par AZ (pour utiliser la NAT GW locale)
resource "aws_route_table" "private" {
  count  = length(var.private_subnet_cidrs)
  vpc_id = aws_vpc.main.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main[count.index].id
    # Chaque subnet privé utilise la NAT GW de sa propre AZ
  }

  tags = {
    Name        = "${var.environment}-rt-private-${count.index + 1}"
    Environment = var.environment
  }
}

resource "aws_route_table_association" "private" {
  count          = length(aws_subnet.private)
  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private[count.index].id
}
```

---

## 4. Security Groups

### Pourquoi c'est important

Les Security Groups en Terraform permettent de coder exactement les règles d'accès qu'on a définies dans le Module 4 — et surtout de les **référencer entre elles**. Un SG peut avoir comme source un autre SG, et Terraform gère les dépendances automatiquement.

### Security Group de base

```hcl
resource "aws_security_group" "alb" {
  name        = "${var.environment}-sg-alb"
  description = "Security Group pour le Load Balancer"
  vpc_id      = aws_vpc.main.id

  ingress {
    description = "HTTP depuis Internet"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "HTTPS depuis Internet"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    description = "Tout le trafic sortant"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"    # tous les protocoles
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name        = "${var.environment}-sg-alb"
    Environment = var.environment
  }
}
```

### Référencer un SG depuis un autre SG

C'est le pattern clé du Module 4 — en Terraform il est naturel :

```hcl
# SG pour les EC2 app — accepte uniquement depuis l'ALB
resource "aws_security_group" "app" {
  name        = "${var.environment}-sg-app"
  description = "Security Group pour les EC2 App"
  vpc_id      = aws_vpc.main.id

  ingress {
    description     = "Trafic depuis l'ALB uniquement"
    from_port       = 3000
    to_port         = 3000
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
    # ↑ référence le SG ALB — Terraform crée ALB avant App
  }

  ingress {
    description = "SSH depuis mon IP uniquement"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["${var.my_ip}/32"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "${var.environment}-sg-app"
  }
}

# SG pour la RDS — accepte uniquement depuis les EC2 app
resource "aws_security_group" "database" {
  name        = "${var.environment}-sg-database"
  description = "Security Group pour la base de données"
  vpc_id      = aws_vpc.main.id

  ingress {
    description     = "PostgreSQL depuis EC2 App uniquement"
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id]
    # ↑ référence le SG App
  }

  tags = {
    Name = "${var.environment}-sg-database"
  }
}
```

### Règles dynamiques avec for_each

Pour des règles multiples sans répétition :

```hcl
# variables.tf
variable "app_ingress_rules" {
  type = list(object({
    port        = number
    protocol    = string
    description = string
    cidr        = string
  }))
  default = [
    { port = 80,   protocol = "tcp", description = "HTTP",  cidr = "0.0.0.0/0" },
    { port = 443,  protocol = "tcp", description = "HTTPS", cidr = "0.0.0.0/0" },
    { port = 8080, protocol = "tcp", description = "Alt HTTP", cidr = "10.0.0.0/8" },
  ]
}

# main.tf — Règles séparées avec aws_security_group_rule
resource "aws_security_group" "app_dynamic" {
  name   = "${var.environment}-sg-app-dynamic"
  vpc_id = aws_vpc.main.id
}

resource "aws_security_group_rule" "app_ingress" {
  for_each = {
    for rule in var.app_ingress_rules :
    rule.description => rule
  }

  type              = "ingress"
  security_group_id = aws_security_group.app_dynamic.id
  from_port         = each.value.port
  to_port           = each.value.port
  protocol          = each.value.protocol
  cidr_blocks       = [each.value.cidr]
  description       = each.key
}
```

---

## 5. Modules Terraform réseau

### Pourquoi c'est important

Un module Terraform est un ensemble de ressources packagées et réutilisables — comme une fonction en programmation. Au lieu de copier-coller le même code VPC pour prod, staging et dev, on crée un module `vpc` qu'on appelle trois fois avec des paramètres différents.

C'est ainsi que travaillent les équipes Platform Engineering en entreprise — des modules réseau stables, documentés, partagés entre toutes les équipes.

### Structure d'un module

```
modules/
  vpc/
    main.tf       ← les ressources
    variables.tf  ← les inputs
    outputs.tf    ← les outputs
    README.md     ← documentation

environments/
  production/
    main.tf       ← appelle le module
    terraform.tfvars
  staging/
    main.tf
    terraform.tfvars
```

### Créer un module VPC complet

```hcl
# modules/vpc/variables.tf

variable "environment" {
  description = "Nom de l'environnement (production, staging, dev)"
  type        = string
}

variable "vpc_cidr" {
  description = "CIDR block du VPC"
  type        = string
}

variable "public_subnet_cidrs" {
  description = "Liste des CIDRs pour les subnets publics"
  type        = list(string)
}

variable "private_subnet_cidrs" {
  description = "Liste des CIDRs pour les subnets privés"
  type        = list(string)
}

variable "enable_nat_gateway" {
  description = "Créer une NAT Gateway (coût supplémentaire)"
  type        = bool
  default     = true
}

variable "single_nat_gateway" {
  description = "Une seule NAT Gateway au lieu d'une par AZ (économique)"
  type        = bool
  default     = false
}
```

```hcl
# modules/vpc/main.tf

locals {
  nat_gateway_count = var.single_nat_gateway ? 1 : length(var.public_subnet_cidrs)
}

data "aws_availability_zones" "available" {
  state = "available"
}

resource "aws_vpc" "this" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name        = "${var.environment}-vpc"
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}

resource "aws_subnet" "public" {
  count             = length(var.public_subnet_cidrs)
  vpc_id            = aws_vpc.this.id
  cidr_block        = var.public_subnet_cidrs[count.index]
  availability_zone = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true

  tags = {
    Name        = "${var.environment}-subnet-public-${count.index + 1}"
    Environment = var.environment
    Type        = "public"
  }
}

resource "aws_subnet" "private" {
  count             = length(var.private_subnet_cidrs)
  vpc_id            = aws_vpc.this.id
  cidr_block        = var.private_subnet_cidrs[count.index]
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = {
    Name        = "${var.environment}-subnet-private-${count.index + 1}"
    Environment = var.environment
    Type        = "private"
  }
}

resource "aws_internet_gateway" "this" {
  vpc_id = aws_vpc.this.id
  tags = {
    Name        = "${var.environment}-igw"
    Environment = var.environment
  }
}

resource "aws_eip" "nat" {
  count  = var.enable_nat_gateway ? local.nat_gateway_count : 0
  domain = "vpc"
  depends_on = [aws_internet_gateway.this]
  tags = {
    Name = "${var.environment}-eip-nat-${count.index + 1}"
  }
}

resource "aws_nat_gateway" "this" {
  count         = var.enable_nat_gateway ? local.nat_gateway_count : 0
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id
  tags = {
    Name = "${var.environment}-nat-${count.index + 1}"
  }
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.this.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.this.id
  }
  tags = { Name = "${var.environment}-rt-public" }
}

resource "aws_route_table_association" "public" {
  count          = length(aws_subnet.public)
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table" "private" {
  count  = length(var.private_subnet_cidrs)
  vpc_id = aws_vpc.this.id

  dynamic "route" {
    for_each = var.enable_nat_gateway ? [1] : []
    content {
      cidr_block     = "0.0.0.0/0"
      nat_gateway_id = var.single_nat_gateway ? aws_nat_gateway.this[0].id : aws_nat_gateway.this[count.index].id
    }
  }

  tags = { Name = "${var.environment}-rt-private-${count.index + 1}" }
}

resource "aws_route_table_association" "private" {
  count          = length(aws_subnet.private)
  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private[count.index].id
}
```

```hcl
# modules/vpc/outputs.tf

output "vpc_id" {
  description = "ID du VPC"
  value       = aws_vpc.this.id
}

output "vpc_cidr" {
  description = "CIDR du VPC"
  value       = aws_vpc.this.cidr_block
}

output "public_subnet_ids" {
  description = "IDs des subnets publics"
  value       = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  description = "IDs des subnets privés"
  value       = aws_subnet.private[*].id
}

output "nat_gateway_ids" {
  description = "IDs des NAT Gateways"
  value       = aws_nat_gateway.this[*].id
}
```

### Utiliser le module dans plusieurs environnements

```hcl
# environments/production/main.tf

module "vpc" {
  source = "../../modules/vpc"

  environment          = "production"
  vpc_cidr             = "10.0.0.0/16"
  public_subnet_cidrs  = ["10.0.1.0/24", "10.0.2.0/24"]
  private_subnet_cidrs = ["10.0.10.0/24", "10.0.11.0/24"]
  enable_nat_gateway   = true
  single_nat_gateway   = false  # une NAT GW par AZ en prod
}

# environments/staging/main.tf

module "vpc" {
  source = "../../modules/vpc"

  environment          = "staging"
  vpc_cidr             = "10.1.0.0/16"
  public_subnet_cidrs  = ["10.1.1.0/24"]
  private_subnet_cidrs = ["10.1.10.0/24"]
  enable_nat_gateway   = true
  single_nat_gateway   = true  # une seule NAT GW en staging (économique)
}
```

---

## 6. Route 53 et DNS

### Pourquoi c'est important

On a vu dans le Module 4 que Route 53 permet de nommer les ressources internes et d'éviter d'utiliser des IPs. En Terraform, on code ces enregistrements DNS directement avec l'infrastructure — si tu crées une RDS, son enregistrement DNS est créé en même temps.

### Hosted Zone privée

```hcl
# Zone DNS privée pour les communications internes
resource "aws_route53_zone" "private" {
  name = "prod.internal"

  vpc {
    vpc_id = module.vpc.vpc_id
  }

  tags = {
    Name        = "prod-internal-zone"
    Environment = var.environment
  }
}

# Enregistrement pour la base de données
resource "aws_route53_record" "database" {
  zone_id = aws_route53_zone.private.zone_id
  name    = "database.prod.internal"
  type    = "A"
  ttl     = 300
  records = [aws_db_instance.main.address]
  # ↑ référence directe à l'IP de la RDS
}

# Enregistrement CNAME pour le cache
resource "aws_route53_record" "cache" {
  zone_id = aws_route53_zone.private.zone_id
  name    = "cache.prod.internal"
  type    = "CNAME"
  ttl     = 300
  records = [aws_elasticache_cluster.main.cache_nodes[0].address]
}
```

### Hosted Zone publique et Alias Record

```hcl
# Zone publique
data "aws_route53_zone" "public" {
  name         = "monapp.com"
  private_zone = false
}

# Alias Record vers un ALB (pas de CNAME à la racine du domaine)
resource "aws_route53_record" "app" {
  zone_id = data.aws_route53_zone.public.zone_id
  name    = "monapp.com"
  type    = "A"

  alias {
    name                   = aws_lb.main.dns_name
    zone_id                = aws_lb.main.zone_id
    evaluate_target_health = true
  }
}

# Sous-domaine www
resource "aws_route53_record" "www" {
  zone_id = data.aws_route53_zone.public.zone_id
  name    = "www.monapp.com"
  type    = "CNAME"
  ttl     = 300
  records = ["monapp.com"]
}
```

---

## 7. ALB en Terraform

### Pourquoi c'est important

Un ALB AWS nécessite plusieurs ressources Terraform interconnectées. Comprendre comment elles s'articulent, c'est comprendre l'architecture complète d'exposition d'une application.

### Architecture complète ALB

```hcl
# Security Group pour l'ALB
resource "aws_security_group" "alb" {
  name   = "${var.environment}-sg-alb"
  vpc_id = module.vpc.vpc_id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# Le Load Balancer
resource "aws_lb" "main" {
  name               = "${var.environment}-alb"
  internal           = false          # public
  load_balancer_type = "application"  # ALB
  security_groups    = [aws_security_group.alb.id]
  subnets            = module.vpc.public_subnet_ids  # subnets publics

  enable_deletion_protection = var.environment == "production"

  tags = {
    Name        = "${var.environment}-alb"
    Environment = var.environment
  }
}

# Target Group — groupe de cibles (EC2, conteneurs...)
resource "aws_lb_target_group" "app" {
  name     = "${var.environment}-tg-app"
  port     = 3000
  protocol = "HTTP"
  vpc_id   = module.vpc.vpc_id

  health_check {
    path                = "/health"
    healthy_threshold   = 3
    unhealthy_threshold = 2
    timeout             = 5
    interval            = 30
  }
}

# Listener HTTP → redirect vers HTTPS
resource "aws_lb_listener" "http" {
  load_balancer_arn = aws_lb.main.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type = "redirect"
    redirect {
      port        = "443"
      protocol    = "HTTPS"
      status_code = "HTTP_301"
    }
  }
}

# Listener HTTPS → forward vers le Target Group
resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.main.arn
  port              = 443
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS13-1-2-2021-06"
  certificate_arn   = aws_acm_certificate.main.arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app.arn
  }
}

# Règle de routing par path
resource "aws_lb_listener_rule" "api" {
  listener_arn = aws_lb_listener.https.arn
  priority     = 100

  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.api.arn
  }

  condition {
    path_pattern {
      values = ["/api/*"]
    }
  }
}
```

---

## 8. VPC Peering

### Pourquoi c'est important

Le VPC Peering manuel (Module 4) nécessite plusieurs étapes — créer la connexion, l'accepter, mettre à jour les Route Tables des deux VPC. En Terraform, tout ça est déclaré en une fois et appliqué dans le bon ordre automatiquement.

```hcl
# VPC A et VPC B déjà créés via le module vpc

# Connexion de peering
resource "aws_vpc_peering_connection" "prod_to_staging" {
  vpc_id      = module.vpc_prod.vpc_id     # requester
  peer_vpc_id = module.vpc_staging.vpc_id  # accepter
  auto_accept = true  # accepte automatiquement (même compte AWS)

  tags = {
    Name = "prod-to-staging-peering"
  }
}

# Route dans VPC prod vers VPC staging
resource "aws_route" "prod_to_staging" {
  count                     = length(module.vpc_prod.private_subnet_ids)
  route_table_id            = module.vpc_prod.private_route_table_ids[count.index]
  destination_cidr_block    = module.vpc_staging.vpc_cidr
  vpc_peering_connection_id = aws_vpc_peering_connection.prod_to_staging.id
}

# Route dans VPC staging vers VPC prod
resource "aws_route" "staging_to_prod" {
  count                     = length(module.vpc_staging.private_subnet_ids)
  route_table_id            = module.vpc_staging.private_route_table_ids[count.index]
  destination_cidr_block    = module.vpc_prod.vpc_cidr
  vpc_peering_connection_id = aws_vpc_peering_connection.prod_to_staging.id
}
```

---

## 9. Outputs et Remote State

### Pourquoi c'est important

En entreprise, l'infrastructure réseau est souvent gérée par une équipe Platform Engineering — et les équipes applicatives ont besoin des IDs de VPC et subnets pour déployer leurs services. Le Remote State permet de partager ces informations entre configurations Terraform séparées.

### Remote State — stocker le state dans S3

```hcl
# backend.tf — dans chaque configuration Terraform

terraform {
  backend "s3" {
    bucket         = "mon-entreprise-terraform-state"
    key            = "networking/production/terraform.tfstate"
    region         = "eu-central-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
    # ↑ DynamoDB pour le verrouillage (évite les apply simultanés)
  }
}
```

### Créer le bucket S3 et la table DynamoDB pour le state

```hcl
# infrastructure/state-backend/main.tf

resource "aws_s3_bucket" "terraform_state" {
  bucket = "mon-entreprise-terraform-state"

  lifecycle {
    prevent_destroy = true  # ne jamais détruire le bucket de state !
  }
}

resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  versioning_configuration {
    status = "Enabled"  # garde l'historique de tous les states
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

resource "aws_dynamodb_table" "terraform_lock" {
  name         = "terraform-state-lock"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
}
```

### Lire le state d'une autre configuration

```hcl
# Dans la configuration "application" — lit le VPC créé par "networking"

data "terraform_remote_state" "networking" {
  backend = "s3"
  config = {
    bucket = "mon-entreprise-terraform-state"
    key    = "networking/production/terraform.tfstate"
    region = "eu-central-1"
  }
}

# Utiliser les outputs de la configuration networking
resource "aws_instance" "app" {
  subnet_id              = data.terraform_remote_state.networking.outputs.private_subnet_ids[0]
  vpc_security_group_ids = [data.terraform_remote_state.networking.outputs.sg_app_id]
  # ...
}
```

---

## 10. Outils et bonnes pratiques

### Outils essentiels

```bash
# terraform fmt — formater automatiquement le code
terraform fmt -recursive

# terraform validate — vérifier la syntaxe
terraform validate

# terraform plan — prévisualiser sans appliquer
terraform plan -out=tfplan        # sauvegarder le plan
terraform apply tfplan            # appliquer le plan sauvegardé

# terraform state — gérer le state
terraform state list              # voir toutes les ressources
terraform state show aws_vpc.main # voir une ressource spécifique
terraform state mv                # renommer une ressource dans le state
terraform import aws_vpc.main vpc-0abc123  # importer une ressource existante

# tflint — linter avancé
tflint --init
tflint

# terraform-docs — générer la doc automatiquement
terraform-docs markdown . > README.md
```

### Bonnes pratiques réseau en Terraform

```
Structure de fichiers recommandée :
  main.tf       → ressources principales
  variables.tf  → toutes les variables
  outputs.tf    → tous les outputs
  versions.tf   → versions des providers
  locals.tf     → valeurs locales calculées
  backend.tf    → configuration du remote state

Nommage :
  → Utiliser des tirets pour les noms AWS (aws-vpc-prod)
  → Utiliser des underscores pour les noms Terraform (aws_vpc.main)
  → Toujours tagger avec Environment, ManagedBy=terraform, Project

Sécurité :
  → Jamais de credentials AWS dans le code
  → Utiliser des variables pour les CIDRs sensibles
  → terraform.tfvars dans .gitignore
  → Toujours enable_deletion_protection en production

Coûts :
  → NAT Gateway : ~$32/mois par gateway
    → single_nat_gateway = true en dev/staging
  → ALB : ~$16/mois minimum
    → partager un ALB entre plusieurs services si possible
  → EIP inutilisée : ~$3.6/mois
    → toujours libérer les EIPs non attachées
```

### Terragrunt — mention

```
Terragrunt est un wrapper autour de Terraform qui résout
des problèmes courants dans les grandes organisations :

→ DRY (Don't Repeat Yourself) pour les backends S3
→ Dépendances entre modules (apply dans le bon ordre)
→ Génération de code boilerplate
→ Locks et retries automatiques

Cas d'usage : 10+ environnements, équipes multiples
Pour ce cours : Terraform seul est suffisant
```

### Terraform Cloud — mention

```
Terraform Cloud est le SaaS de HashiCorp pour gérer
le state et les runs Terraform :

→ State centralisé et sécurisé
→ Plans et applies depuis l'UI ou l'API
→ Intégration GitHub/GitLab (plan sur PR, apply sur merge)
→ Variables et secrets chiffrés
→ Historique des runs

Alternative open source : Atlantis (auto-hébergé)
```

---

## Récapitulatif du Module 6

| Concept | Ce qu'il faut retenir |
|---------|----------------------|
| **IaC** | Infrastructure déclarée en code — versionnable, reproductible, auditable. |
| **Provider** | Plugin Terraform qui parle à une API (AWS, GCP...). |
| **Resource** | Ressource infrastructure gérée par Terraform. |
| **State** | Fichier qui mappe le code Terraform aux ressources réelles. Stocker dans S3 en équipe. |
| **Variable** | Paramètre configurable — permet la réutilisation. |
| **Output** | Valeur exportée — partageable entre modules et configurations. |
| **Local** | Valeur calculée localement — évite la répétition. |
| **count** | Créer plusieurs ressources similaires avec un index. |
| **for_each** | Créer des ressources à partir d'une map ou d'un set. |
| **Module** | Ensemble de ressources packagées et réutilisables. |
| **Remote State** | State stocké dans S3 — partageable entre équipes. |
| **Data Source** | Lire des ressources existantes sans les gérer. |
| **depends_on** | Forcer un ordre de création quand Terraform ne le détecte pas. |
| **terraform fmt** | Formater le code automatiquement. |
| **terraform plan** | Prévisualiser les changements avant apply. |
| **tflint** | Linter — détecte les erreurs de config avant le plan. |

---

*→ Exercices pratiques : voir `module6-exercices.md`*
*→ Cours terminé — récapitulatif général dans `README.md`*
