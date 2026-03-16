# Module 6 — Networking as Code avec Terraform
### Partie Exercices

> Complète la partie théorie (`module6-theorie.md`) avant de faire ces exercices.
> Prérequis : Terraform installé, AWS CLI configuré, compte AWS
> Durée estimée : 3h à 4h
> ⚠️ Certaines ressources sont payantes. Toujours `terraform destroy` après les exercices.
> 💡 Coût estimé si tu détruies dans l'heure : < $1

---

## Installation et configuration

```bash
# Vérifier Terraform
terraform version
# → Terraform v1.6.x ou plus récent

# Vérifier AWS CLI
aws sts get-caller-identity
# → affiche ton compte AWS

# Installer tflint
curl -s https://raw.githubusercontent.com/terraform-linters/tflint/master/install_linux.sh | bash

# Structure de travail pour tous les exercices
mkdir -p ~/terraform-networking-lab
cd ~/terraform-networking-lab
```

---

## Exercice 1 — Premier VPC en Terraform

**Objectif** : créer un VPC complet avec subnets publics et privés — reproduire en code ce qu'on a fait manuellement dans le Module 4.

```bash
mkdir -p ~/terraform-networking-lab/ex1-vpc
cd ~/terraform-networking-lab/ex1-vpc
```

```hcl
# versions.tf
terraform {
  required_version = ">= 1.0"
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

```hcl
# variables.tf
variable "environment" {
  type    = string
  default = "lab"
}

variable "vpc_cidr" {
  type    = string
  default = "10.0.0.0/16"
}
```

```hcl
# main.tf
locals {
  common_tags = {
    Environment = var.environment
    ManagedBy   = "terraform"
    Project     = "networking-lab"
  }
}

data "aws_availability_zones" "available" {
  state = "available"
}

resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true
  tags = merge(local.common_tags, { Name = "${var.environment}-vpc" })
}

resource "aws_subnet" "public" {
  count             = 2
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index + 1)
  availability_zone = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true
  tags = merge(local.common_tags, {
    Name = "${var.environment}-subnet-public-${count.index + 1}"
    Type = "public"
  })
}

resource "aws_subnet" "private" {
  count             = 2
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index + 10)
  availability_zone = data.aws_availability_zones.available.names[count.index]
  tags = merge(local.common_tags, {
    Name = "${var.environment}-subnet-private-${count.index + 1}"
    Type = "private"
  })
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  tags   = merge(local.common_tags, { Name = "${var.environment}-igw" })
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }
  tags = merge(local.common_tags, { Name = "${var.environment}-rt-public" })
}

resource "aws_route_table_association" "public" {
  count          = length(aws_subnet.public)
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}
```

```hcl
# outputs.tf
output "vpc_id" {
  value = aws_vpc.main.id
}

output "public_subnet_ids" {
  value = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  value = aws_subnet.private[*].id
}

output "public_subnet_cidrs" {
  value = aws_subnet.public[*].cidr_block
}
```

```bash
# Initialiser et appliquer
terraform init
terraform fmt
terraform validate
terraform plan

# Observer le plan — combien de ressources vont être créées ?
terraform apply

# Voir les outputs
terraform output

# Vérifier dans AWS CLI
VPC_ID=$(terraform output -raw vpc_id)
aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'Subnets[*].{ID:SubnetId,CIDR:CidrBlock,AZ:AvailabilityZone,Public:MapPublicIpOnLaunch}' \
  --output table

# Vérifier les Route Tables
aws ec2 describe-route-tables \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'RouteTables[*].{ID:RouteTableId,Routes:Routes[*].{Dest:DestinationCidrBlock,Target:join(`,`,[GatewayId,NatGatewayId]|[?@])}}' \
  --output table

# Voir le state Terraform
terraform state list

# Inspecter une ressource dans le state
terraform state show aws_vpc.main

# IMPORTANT : détruire après l'exercice
terraform destroy
```

**Questions :**

1. La fonction `cidrsubnet(var.vpc_cidr, 8, count.index + 1)` — que calcule-t-elle pour `count.index = 0` avec `vpc_cidr = "10.0.0.0/16"` ?
2. Combien de ressources Terraform ont été créées au total ?
3. Dans le state, que contient l'attribut `id` de `aws_vpc.main` ?
4. Que se passe-t-il si tu relances `terraform apply` sans rien changer ?

<details>
<summary>Voir les réponses</summary>

1. `cidrsubnet("10.0.0.0/16", 8, 1)` = `10.0.1.0/24` — elle découpe le VPC `/16` en subnets `/24` et prend le Nème subnet.
2. VPC + 2 subnets publics + 2 subnets privés + IGW + 1 Route Table + 2 associations = **9 ressources**.
3. L'ID AWS réel du VPC — quelque chose comme `vpc-0abc123def456789`. C'est le lien entre le code Terraform et la ressource réelle AWS.
4. Terraform affiche `No changes. Infrastructure is up-to-date.` — c'est l'idempotence. Il compare le state avec AWS et ne fait rien si c'est identique.

</details>

---

## Exercice 2 — Security Groups en couches

**Objectif** : implémenter l'architecture Security Groups du Module 4 en Terraform — ALB → App → DB avec référencement de SGs.

```bash
mkdir -p ~/terraform-networking-lab/ex2-sg
cd ~/terraform-networking-lab/ex2-sg
```

```hcl
# versions.tf + provider (même qu'exercice 1)

# variables.tf
variable "environment" { default = "lab" }
variable "my_ip" {
  description = "Ton IP publique pour SSH"
  type        = string
  # Trouve ta valeur avec : curl ifconfig.me
}
```

```hcl
# main.tf

# Récupérer un VPC existant via data source
# (utilise le VPC de l'exercice 1 si encore actif, sinon crée-en un)
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  tags       = { Name = "${var.environment}-vpc-sg-lab" }
}

# SG pour l'ALB
resource "aws_security_group" "alb" {
  name        = "${var.environment}-sg-alb"
  description = "Security Group ALB"
  vpc_id      = aws_vpc.main.id

  ingress {
    description = "HTTP"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "HTTPS"
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

  tags = { Name = "${var.environment}-sg-alb" }
}

# SG pour les EC2 App
resource "aws_security_group" "app" {
  name        = "${var.environment}-sg-app"
  description = "Security Group EC2 App"
  vpc_id      = aws_vpc.main.id

  ingress {
    description     = "App depuis ALB uniquement"
    from_port       = 3000
    to_port         = 3000
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
  }

  ingress {
    description = "SSH depuis mon IP"
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

  tags = { Name = "${var.environment}-sg-app" }
}

# SG pour la BDD
resource "aws_security_group" "database" {
  name        = "${var.environment}-sg-database"
  description = "Security Group RDS"
  vpc_id      = aws_vpc.main.id

  ingress {
    description     = "PostgreSQL depuis App uniquement"
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id]
  }

  tags = { Name = "${var.environment}-sg-database" }
}
```

```hcl
# outputs.tf
output "sg_alb_id"      { value = aws_security_group.alb.id }
output "sg_app_id"      { value = aws_security_group.app.id }
output "sg_database_id" { value = aws_security_group.database.id }
```

```bash
# Créer un terraform.tfvars
echo 'my_ip = "REMPLACE_PAR_TON_IP"' > terraform.tfvars
# Trouver ton IP : curl ifconfig.me

terraform init && terraform apply

# Vérifier les règles créées
SG_APP=$(terraform output -raw sg_app_id)
aws ec2 describe-security-groups \
  --group-ids $SG_APP \
  --query 'SecurityGroups[0].IpPermissions' \
  --output table

# Observer que la source de la règle port 3000 est un SG ID, pas un CIDR
SG_DB=$(terraform output -raw sg_database_id)
aws ec2 describe-security-groups \
  --group-ids $SG_DB \
  --query 'SecurityGroups[0].IpPermissions[0].UserIdGroupPairs' \
  --output table

# Modifier une règle et voir le plan
# Dans main.tf, changer le port 3000 en 8080 sur sg_app
# Puis : terraform plan
# → Observer le changement in-place

terraform destroy
```

**Questions :**

1. Terraform crée-t-il les SGs dans un ordre particulier ? Pourquoi ?
2. Dans AWS CLI, comment apparaît la source d'une règle qui référence un autre SG (vs une règle CIDR) ?
3. Si tu changes le port de l'app de 3000 à 8080, Terraform modifie-t-il le SG en place ou le recrée ?
4. Pourquoi le SG `database` n'a-t-il pas de règle `egress` explicite ?

<details>
<summary>Voir les réponses</summary>

1. **Oui** — Terraform analyse les dépendances. `aws_security_group.app` référence `aws_security_group.alb.id`, donc ALB est créé en premier. `database` référence `app.id`, donc app est créé avant database. C'est la **dependency graph** automatique de Terraform.
2. La règle avec un SG source apparaît dans `UserIdGroupPairs` avec un `GroupId` (ex: `sg-0abc123`). Une règle CIDR apparaît dans `IpRanges` avec un `CidrIp`. Ce sont deux champs différents dans l'API AWS.
3. Terraform **modifie en place** (`~`) — il met à jour la règle existante sans recréer le SG. Si la modification nécessitait de recréer (ex: changer le nom), il afficherait `-/+` (destroy + create).
4. Par défaut, AWS autorise tout le trafic sortant sur un Security Group sans règle egress explicite. La BDD n'initie pas de connexions vers l'extérieur — pas besoin de règle outbound. Si on l'ajoutait explicitement ce serait `from_port=0, to_port=0, protocol="-1"`.

</details>

---

## Exercice 3 — Module VPC réutilisable

**Objectif** : créer un module VPC complet et l'utiliser pour deux environnements différents.

```bash
mkdir -p ~/terraform-networking-lab/ex3-module/{modules/vpc,environments/staging}
cd ~/terraform-networking-lab/ex3-module
```

```bash
# Créer le module VPC
cat > modules/vpc/variables.tf << 'EOF'
variable "environment"           { type = string }
variable "vpc_cidr"              { type = string }
variable "public_subnet_cidrs"   { type = list(string) }
variable "private_subnet_cidrs"  { type = list(string) }
variable "enable_nat_gateway"    { type = bool; default = false }
EOF

cat > modules/vpc/main.tf << 'EOF'
data "aws_availability_zones" "available" { state = "available" }

resource "aws_vpc" "this" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true
  tags = { Name = "${var.environment}-vpc", Environment = var.environment, ManagedBy = "terraform" }
}

resource "aws_subnet" "public" {
  count             = length(var.public_subnet_cidrs)
  vpc_id            = aws_vpc.this.id
  cidr_block        = var.public_subnet_cidrs[count.index]
  availability_zone = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true
  tags = { Name = "${var.environment}-public-${count.index + 1}", Type = "public" }
}

resource "aws_subnet" "private" {
  count             = length(var.private_subnet_cidrs)
  vpc_id            = aws_vpc.this.id
  cidr_block        = var.private_subnet_cidrs[count.index]
  availability_zone = data.aws_availability_zones.available.names[count.index]
  tags = { Name = "${var.environment}-private-${count.index + 1}", Type = "private" }
}

resource "aws_internet_gateway" "this" {
  vpc_id = aws_vpc.this.id
  tags   = { Name = "${var.environment}-igw" }
}

resource "aws_eip" "nat" {
  count      = var.enable_nat_gateway ? 1 : 0
  domain     = "vpc"
  depends_on = [aws_internet_gateway.this]
}

resource "aws_nat_gateway" "this" {
  count         = var.enable_nat_gateway ? 1 : 0
  allocation_id = aws_eip.nat[0].id
  subnet_id     = aws_subnet.public[0].id
  tags          = { Name = "${var.environment}-nat" }
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.this.id
  route { cidr_block = "0.0.0.0/0"; gateway_id = aws_internet_gateway.this.id }
  tags = { Name = "${var.environment}-rt-public" }
}

resource "aws_route_table_association" "public" {
  count          = length(aws_subnet.public)
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table" "private" {
  vpc_id = aws_vpc.this.id
  dynamic "route" {
    for_each = var.enable_nat_gateway ? [1] : []
    content {
      cidr_block     = "0.0.0.0/0"
      nat_gateway_id = aws_nat_gateway.this[0].id
    }
  }
  tags = { Name = "${var.environment}-rt-private" }
}

resource "aws_route_table_association" "private" {
  count          = length(aws_subnet.private)
  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private.id
}
EOF

cat > modules/vpc/outputs.tf << 'EOF'
output "vpc_id"              { value = aws_vpc.this.id }
output "vpc_cidr"            { value = aws_vpc.this.cidr_block }
output "public_subnet_ids"   { value = aws_subnet.public[*].id }
output "private_subnet_ids"  { value = aws_subnet.private[*].id }
output "igw_id"              { value = aws_internet_gateway.this.id }
EOF
```

```hcl
# environments/staging/main.tf
terraform {
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
  }
}

provider "aws" { region = "eu-central-1" }

module "vpc_staging" {
  source = "../../modules/vpc"

  environment          = "staging"
  vpc_cidr             = "10.1.0.0/16"
  public_subnet_cidrs  = ["10.1.1.0/24"]
  private_subnet_cidrs = ["10.1.10.0/24"]
  enable_nat_gateway   = false  # économie en staging
}

output "staging_vpc_id"     { value = module.vpc_staging.vpc_id }
output "staging_subnet_ids" { value = module.vpc_staging.public_subnet_ids }
```

```bash
cd environments/staging
terraform init
terraform plan
terraform apply

# Voir les outputs du module
terraform output

# Vérifier les ressources créées
aws ec2 describe-vpcs \
  --filters "Name=tag:Environment,Values=staging" \
  --query 'Vpcs[*].{ID:VpcId,CIDR:CidrBlock}' \
  --output table

# Tester en changeant enable_nat_gateway = true
# Observer le plan — quelles ressources sont ajoutées ?
# terraform plan → noter les + aws_eip, aws_nat_gateway, route

terraform destroy
```

**Questions :**

1. Quand tu passes `enable_nat_gateway = false`, quelles ressources ne sont pas créées ?
2. Comment Terraform sait-il dans quel ordre créer les ressources du module ?
3. Si deux équipes utilisent ce module simultanément (staging et prod), y a-t-il un conflit ?
4. Quelle est la différence entre `count` et `for_each` dans le module — quand utiliser l'un ou l'autre ?

<details>
<summary>Voir les réponses</summary>

1. `aws_eip.nat`, `aws_nat_gateway.this`, et la route `0.0.0.0/0 → nat_gateway_id` dans la Route Table privée. La Route Table privée est quand même créée mais sans route Internet.
2. Terraform construit un **graphe de dépendances** en analysant les références entre ressources. `aws_nat_gateway` référence `aws_eip.nat[0].id` → EIP créée en premier. `aws_route_table_association` référence `aws_route_table.private.id` → RT créée avant l'association.
3. **Non** — chaque configuration Terraform a son propre state. Staging et prod ont des states séparés, des ressources AWS séparées (VPCs différents), aucun conflit.
4. **`count`** quand les ressources sont identiques sauf l'index (même type, configuration similaire). **`for_each`** quand les ressources ont des configurations distinctes identifiées par une clé (ex: différents SGs avec des règles différentes). `for_each` est plus stable — si tu supprimes un élément au milieu, Terraform ne recrée pas tout.

</details>

---

## Exercice 4 — Route 53 DNS privé

**Objectif** : créer une zone DNS privée et des enregistrements pour les services internes.

```bash
mkdir -p ~/terraform-networking-lab/ex4-dns
cd ~/terraform-networking-lab/ex4-dns
```

```hcl
# main.tf
provider "aws" { region = "eu-central-1" }

# VPC minimal pour le lab
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true
  tags = { Name = "lab-dns-vpc" }
}

# Zone DNS privée
resource "aws_route53_zone" "private" {
  name = "lab.internal"

  vpc {
    vpc_id = aws_vpc.main.id
  }

  tags = { Name = "lab-internal-zone" }
}

# Enregistrements DNS internes
resource "aws_route53_record" "database" {
  zone_id = aws_route53_zone.private.zone_id
  name    = "database.lab.internal"
  type    = "A"
  ttl     = 300
  records = ["10.0.20.5"]    # IP fictive de la BDD
}

resource "aws_route53_record" "cache" {
  zone_id = aws_route53_zone.private.zone_id
  name    = "cache.lab.internal"
  type    = "A"
  ttl     = 60
  records = ["10.0.20.10"]
}

resource "aws_route53_record" "api" {
  zone_id = aws_route53_zone.private.zone_id
  name    = "api.lab.internal"
  type    = "CNAME"
  ttl     = 300
  records = ["database.lab.internal"]  # CNAME vers database
}

# Weighted routing — simulation blue/green
resource "aws_route53_record" "app_v1" {
  zone_id        = aws_route53_zone.private.zone_id
  name           = "app.lab.internal"
  type           = "A"
  ttl            = 60
  records        = ["10.0.10.1"]
  set_identifier = "v1"

  weighted_routing_policy {
    weight = 90
  }
}

resource "aws_route53_record" "app_v2" {
  zone_id        = aws_route53_zone.private.zone_id
  name           = "app.lab.internal"
  type           = "A"
  ttl            = 60
  records        = ["10.0.10.2"]
  set_identifier = "v2"

  weighted_routing_policy {
    weight = 10
  }
}
```

```hcl
# outputs.tf
output "zone_id"   { value = aws_route53_zone.private.zone_id }
output "zone_name" { value = aws_route53_zone.private.name }
```

```bash
terraform init && terraform apply

# Voir les enregistrements créés
ZONE_ID=$(terraform output -raw zone_id)
aws route53 list-resource-record-sets \
  --hosted-zone-id $ZONE_ID \
  --query 'ResourceRecordSets[*].{Name:Name,Type:Type,Value:ResourceRecords[0].Value,Weight:Weight}' \
  --output table

# Simuler un déploiement blue/green :
# Changer les weights à 50/50 puis 0/100
# terraform plan → observer le changement in-place

# Simuler une migration d'IP BDD :
# Changer l'IP de database de 10.0.20.5 à 10.0.20.99
# terraform plan → observer le changement

terraform destroy
```

**Questions :**

1. La zone DNS privée `lab.internal` est-elle résolvable depuis Internet ? Pourquoi ?
2. Que se passe-t-il si tu changes le TTL de `database` de 300 à 60 — Terraform recrée l'enregistrement ?
3. Pour migrer la BDD vers une nouvelle IP sans interruption, quelle procédure Terraform recommanderais-tu ?
4. Le weighted routing 90/10 — quel cas d'usage concret justifie cette configuration ?

<details>
<summary>Voir les réponses</summary>

1. **Non** — c'est une zone privée associée au VPC. Elle n'est résolvable que depuis les ressources dans ce VPC (EC2, Lambda, RDS...). Internet n'a aucune visibilité sur ces enregistrements.
2. Terraform **modifie en place** (`~`) — il met à jour le TTL sans recréer l'enregistrement. Pas d'interruption DNS.
3. Procédure en 3 étapes : **1.** Baisser le TTL à 60s (`terraform apply`) et attendre que les caches expirent. **2.** Changer l'IP vers la nouvelle BDD (`terraform apply`). **3.** Remonter le TTL à 300s après validation. Le TTL court garantit une propagation rapide du changement.
4. **Canary deployment** / déploiement progressif : 90% du trafic va sur la version stable (v1), 10% sur la nouvelle version (v2). Si v2 présente des problèmes, seuls 10% des utilisateurs sont impactés. On augmente progressivement le weight de v2 jusqu'à 100% quand on est confiant.

</details>

---

## Exercice 5 — Remote State et partage entre équipes

**Objectif** : configurer le Remote State S3 et partager les outputs réseau entre deux configurations Terraform.

```bash
mkdir -p ~/terraform-networking-lab/ex5-remote-state/{networking,application}
cd ~/terraform-networking-lab/ex5-remote-state
```

```bash
# Étape 1 : Créer le bucket S3 pour le state
# (à faire UNE SEULE FOIS par compte AWS)
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="terraform-state-lab-$ACCOUNT_ID"

aws s3 mb s3://$BUCKET_NAME --region eu-central-1
aws s3api put-bucket-versioning \
  --bucket $BUCKET_NAME \
  --versioning-configuration Status=Enabled

echo "Bucket créé : $BUCKET_NAME"
```

```hcl
# networking/main.tf
terraform {
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
  }
  backend "s3" {
    # Remplace ACCOUNT_ID par ton ID de compte AWS
    bucket = "terraform-state-lab-ACCOUNT_ID"
    key    = "networking/terraform.tfstate"
    region = "eu-central-1"
  }
}

provider "aws" { region = "eu-central-1" }

resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  tags = { Name = "shared-vpc", ManagedBy = "terraform-networking" }
}

resource "aws_subnet" "app" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.10.0/24"
  availability_zone = "eu-central-1a"
  tags              = { Name = "subnet-app" }
}

resource "aws_security_group" "app" {
  name   = "sg-app-shared"
  vpc_id = aws_vpc.main.id
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  tags = { Name = "sg-app-shared" }
}
```

```hcl
# networking/outputs.tf
output "vpc_id"         { value = aws_vpc.main.id }
output "app_subnet_id"  { value = aws_subnet.app.id }
output "sg_app_id"      { value = aws_security_group.app.id }
output "vpc_cidr"       { value = aws_vpc.main.cidr_block }
```

```hcl
# application/main.tf
terraform {
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
  }
  backend "s3" {
    bucket = "terraform-state-lab-ACCOUNT_ID"
    key    = "application/terraform.tfstate"
    region = "eu-central-1"
  }
}

provider "aws" { region = "eu-central-1" }

# Lire le state de la configuration networking
data "terraform_remote_state" "networking" {
  backend = "s3"
  config = {
    bucket = "terraform-state-lab-ACCOUNT_ID"
    key    = "networking/terraform.tfstate"
    region = "eu-central-1"
  }
}

# Utiliser les outputs du networking
locals {
  vpc_id    = data.terraform_remote_state.networking.outputs.vpc_id
  subnet_id = data.terraform_remote_state.networking.outputs.app_subnet_id
  sg_app_id = data.terraform_remote_state.networking.outputs.sg_app_id
}

# Exemple : créer une ressource dans le VPC partagé
resource "aws_security_group" "web" {
  name   = "sg-web-app"
  vpc_id = local.vpc_id    # ← VPC créé par l'équipe networking

  ingress {
    from_port   = 80
    to_port     = 80
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
```

```hcl
# application/outputs.tf
output "vpc_id_used"    { value = local.vpc_id }
output "subnet_id_used" { value = local.subnet_id }
```

```bash
# Déployer networking d'abord
cd networking
# Modifier le bucket name dans backend "s3" avec ton ACCOUNT_ID
terraform init
terraform apply

# Déployer l'application qui lit le state du networking
cd ../application
# Modifier le bucket name dans backend "s3" avec ton ACCOUNT_ID
terraform init
terraform apply

# Voir que l'application utilise le VPC créé par networking
terraform output

# Simuler un changement réseau :
# Dans networking/ : changer le CIDR du subnet de /24 à /23
# terraform plan → observer l'impact
# Dans application/ : terraform plan → pas d'impact direct
#   (utilise juste l'ID, pas le CIDR)

# Nettoyer (ordre inverse !)
cd application && terraform destroy
cd ../networking && terraform destroy

# Supprimer le bucket S3 (vider d'abord)
aws s3 rm s3://$BUCKET_NAME --recursive
aws s3 rb s3://$BUCKET_NAME
```

**Questions :**

1. Pourquoi détruit-on `application` avant `networking` et pas l'inverse ?
2. Si l'équipe networking change le CIDR du VPC, l'équipe application doit-elle faire un `terraform apply` ?
3. Quelle est la différence entre un `output` et un `data "terraform_remote_state"` ?
4. Pourquoi active-t-on le versioning sur le bucket S3 du state ?

<details>
<summary>Voir les réponses</summary>

1. `application` dépend des ressources créées par `networking` (utilise le VPC ID). Si on détruisait `networking` en premier, les ressources de `application` deviendraient orphelines dans AWS — impossible à détruire proprement via Terraform.
2. **Non** — `data.terraform_remote_state` lit les **outputs** du state, pas les ressources elles-mêmes. L'application lit l'ID du VPC, pas son CIDR. Un changement de CIDR impacte les ressources réseau, pas les outputs qui exposent l'ID.
3. Un `output` exporte une valeur d'une configuration Terraform. `data "terraform_remote_state"` lit les outputs d'une **autre** configuration Terraform depuis son state. C'est le mécanisme de partage de données entre équipes.
4. Le versioning garde l'**historique de tous les états** du state. Si un `terraform apply` casse quelque chose, on peut récupérer l'ancienne version du state depuis S3 et faire un rollback. Sans versioning, une erreur peut corrompre définitivement le state.

</details>

---

## Quiz final — Module 6

Réponds sans regarder le cours.

1. Quelle est la différence entre `terraform plan` et `terraform apply` ?
2. Le state Terraform — pourquoi ne faut-il jamais le committer dans Git ?
3. Tu as un VPC créé manuellement dans AWS. Comment l'intégrer dans Terraform sans le détruire et recréer ?
4. Quelle est la différence entre `count` et `for_each` ?
5. Un collègue fait un `terraform apply` en même temps que toi. Que se passe-t-il sans Remote State avec DynamoDB ?
6. `terraform plan` affiche `~` devant une ressource. Que signifie ce symbole ?
7. Comment référencer l'ID d'un Security Group créé dans le même fichier Terraform ?
8. Quelle commande Terraform utilises-tu pour voir toutes les ressources gérées par le state ?
9. Tu veux créer la même infrastructure en production et staging. Quelle approche Terraform utilises-tu ?
10. `depends_on` — quand en as-tu besoin alors que Terraform gère les dépendances automatiquement ?

<details>
<summary>Voir les réponses</summary>

1. `terraform plan` **prévisualise** les changements sans rien faire dans AWS — lecture seule, sûr à lancer à tout moment. `terraform apply` **exécute** les changements et modifie réellement l'infrastructure AWS.
2. Le state peut contenir des **secrets** (mots de passe RDS, clés API...) en clair. De plus, un state commité dans Git peut être désynchronisé avec la réalité si quelqu'un fait un apply sans commiter. Le state doit être dans un backend sécurisé (S3 + chiffrement).
3. `terraform import <resource_type>.<name> <aws_id>` — importe la ressource dans le state sans la recréer. Ex: `terraform import aws_vpc.main vpc-0abc123`. Ensuite il faut écrire le bloc `resource` correspondant dans le code.
4. **`count`** crée N ressources identiques identifiées par un index numérique. Si tu supprimes un élément au milieu, Terraform renumérote et peut recréer des ressources. **`for_each`** crée des ressources identifiées par une clé string — plus stable, une suppression n'affecte pas les autres.
5. Les deux applies pourraient corrompre le state — chacun lit la version initiale, fait ses modifications, et la dernière écriture écrase l'autre. La table DynamoDB sert de **verrou distribué** : le premier apply verrouille, le second attend ou échoue avec un message d'erreur.
6. `~` signifie **modification in-place** — la ressource sera mise à jour sans être détruite et recrée. Moins risqué que `-/+` (destroy + create) qui peut causer une interruption.
7. `aws_security_group.mon_sg.id` — référence directe par `type.name.attribut`. Terraform détecte automatiquement cette dépendance et crée le SG référencé en premier.
8. `terraform state list` — liste toutes les ressources dans le state avec leur adresse Terraform.
9. **Modules Terraform** — un module `vpc` paramétré avec `environment`, `vpc_cidr`, etc. Appelé deux fois avec des valeurs différentes dans `environments/production/main.tf` et `environments/staging/main.tf`.
10. `depends_on` est nécessaire quand la dépendance est **implicite** et non visible dans le code. Exemple : une EC2 qui doit être créée après qu'une politique IAM soit propagée, ou une ressource qui dépend d'une autre via un effet de bord non capturé par les références directes.

</details>

---

*→ Retour au cours : `module6-theorie.md`*
*→ Cours terminé ! Consulte le README général pour la suite.*
