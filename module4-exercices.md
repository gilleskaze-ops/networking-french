# Module 4 — AWS Networking
### Partie Exercices

> Complète la partie théorie (`module4-theorie.md`) avant de faire ces exercices.
> Prérequis : compte AWS (Free Tier suffisant pour la plupart des exercices)
> Durée estimée : 3h à 4h
> ⚠️ Certains exercices créent des ressources AWS payantes — bien noter les sections concernées.

---

## Exercice 1 — Créer et explorer un VPC

**Objectif** : créer un VPC complet avec subnets publics et privés sur deux AZ, comprendre la structure réseau AWS.

### Partie A — Via la console AWS

```
1. AWS Console → VPC → Create VPC

2. Choisir "VPC and more" (crée tout automatiquement) :
   Name tag prefix : lab-networking
   IPv4 CIDR       : 10.0.0.0/16
   AZs             : 2
   Public subnets  : 2 (un par AZ)
   Private subnets : 2 (un par AZ)
   NAT gateways    : None (pour éviter les coûts)
   VPC endpoints   : None

3. Examiner ce qui a été créé :
   → VPC : lab-networking-vpc
   → 4 subnets (2 publics, 2 privés)
   → Internet Gateway attachée
   → Route Tables (une pour publics, une pour privés)
```

### Partie B — Via AWS CLI

```bash
# Configurer l'AWS CLI d'abord
aws configure
# AWS Access Key ID : [ta clé]
# AWS Secret Access Key : [ta clé secrète]
# Default region : eu-central-1
# Default output format : json

# Créer un VPC
VPC_ID=$(aws ec2 create-vpc \
  --cidr-block 10.0.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=lab-vpc}]' \
  --query 'Vpc.VpcId' \
  --output text)
echo "VPC créé : $VPC_ID"

# Créer un subnet public
SUBNET_PUBLIC=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.1.0/24 \
  --availability-zone eu-central-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=lab-subnet-public-1a}]' \
  --query 'Subnet.SubnetId' \
  --output text)
echo "Subnet public : $SUBNET_PUBLIC"

# Créer un subnet privé
SUBNET_PRIVATE=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.10.0/24 \
  --availability-zone eu-central-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=lab-subnet-private-1a}]' \
  --query 'Subnet.SubnetId' \
  --output text)
echo "Subnet privé : $SUBNET_PRIVATE"

# Créer et attacher une Internet Gateway
IGW_ID=$(aws ec2 create-internet-gateway \
  --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=lab-igw}]' \
  --query 'InternetGateway.InternetGatewayId' \
  --output text)
aws ec2 attach-internet-gateway \
  --internet-gateway-id $IGW_ID \
  --vpc-id $VPC_ID
echo "IGW attachée : $IGW_ID"

# Créer une Route Table publique
RT_PUBLIC=$(aws ec2 create-route-table \
  --vpc-id $VPC_ID \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=lab-rt-public}]' \
  --query 'RouteTable.RouteTableId' \
  --output text)

# Ajouter la route vers Internet
aws ec2 create-route \
  --route-table-id $RT_PUBLIC \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id $IGW_ID

# Associer la RT au subnet public
aws ec2 associate-route-table \
  --route-table-id $RT_PUBLIC \
  --subnet-id $SUBNET_PUBLIC

# Vérifier les Route Tables
aws ec2 describe-route-tables \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'RouteTables[*].{ID:RouteTableId,Routes:Routes[*].{Dest:DestinationCidrBlock,Target:GatewayId}}' \
  --output table
```

**Questions :**

1. Quelle est la différence entre la Route Table du subnet public et celle du subnet privé ?
2. La règle `local` est-elle présente dans les deux Route Tables ? Peut-on la supprimer ?
3. Combien d'IPs utilisables a chaque subnet `/24` dans AWS ?
4. Pourquoi crée-t-on des subnets dans deux AZ différentes plutôt qu'une seule ?

<details>
<summary>Voir les réponses</summary>

1. La RT du subnet public a une route `0.0.0.0/0 → igw-xxx` (accès Internet). La RT du subnet privé n'a que la route `local` — aucun accès Internet direct.
2. Oui, présente dans les deux. Non, la règle `local` est permanente et non supprimable — c'est elle qui permet la communication interne au VPC.
3. **251** — AWS réserve 5 adresses par subnet (réseau, routeur, DNS, futur, broadcast).
4. Pour la **haute disponibilité** — si l'AZ `1a` tombe en panne, les ressources dans l'AZ `1b` continuent de fonctionner. Un subnet est dans une seule AZ — sans multi-AZ, une panne AZ = panne totale.

</details>

---

## Exercice 2 — Security Groups : architecture en couches

**Objectif** : créer une architecture Security Groups qui respecte le principe du moindre privilège — chaque ressource n'accepte que le trafic strictement nécessaire.

```bash
# Variables (adapte selon ton VPC)
VPC_ID="vpc-xxxxxxxx"  # remplace par ton VPC ID

# 1. Créer le SG pour le Load Balancer (public)
SG_ALB=$(aws ec2 create-security-group \
  --group-name "sg-alb" \
  --description "Security Group ALB" \
  --vpc-id $VPC_ID \
  --query 'GroupId' --output text)

# Autoriser HTTP et HTTPS depuis Internet
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ALB \
  --protocol tcp --port 80 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ALB \
  --protocol tcp --port 443 --cidr 0.0.0.0/0

echo "SG ALB : $SG_ALB"

# 2. Créer le SG pour les EC2 App (privé)
SG_APP=$(aws ec2 create-security-group \
  --group-name "sg-app" \
  --description "Security Group App Servers" \
  --vpc-id $VPC_ID \
  --query 'GroupId' --output text)

# Autoriser le trafic app UNIQUEMENT depuis l'ALB
aws ec2 authorize-security-group-ingress \
  --group-id $SG_APP \
  --protocol tcp --port 3000 \
  --source-group $SG_ALB   # ← SG comme source, pas une IP

echo "SG App : $SG_APP"

# 3. Créer le SG pour la BDD (très privé)
SG_DB=$(aws ec2 create-security-group \
  --group-name "sg-db" \
  --description "Security Group Database" \
  --vpc-id $VPC_ID \
  --query 'GroupId' --output text)

# Autoriser PostgreSQL UNIQUEMENT depuis les EC2 App
aws ec2 authorize-security-group-ingress \
  --group-id $SG_DB \
  --protocol tcp --port 5432 \
  --source-group $SG_APP   # ← SG comme source

echo "SG DB : $SG_DB"

# 4. Vérifier les règles créées
echo "=== Règles SG ALB ==="
aws ec2 describe-security-groups \
  --group-ids $SG_ALB \
  --query 'SecurityGroups[0].IpPermissions' \
  --output table

echo "=== Règles SG App ==="
aws ec2 describe-security-groups \
  --group-ids $SG_APP \
  --query 'SecurityGroups[0].IpPermissions' \
  --output table

echo "=== Règles SG DB ==="
aws ec2 describe-security-groups \
  --group-ids $SG_DB \
  --query 'SecurityGroups[0].IpPermissions' \
  --output table

# 5. Ajouter SSH au SG App pour l'administration
# (depuis ton IP uniquement)
MON_IP=$(curl -s ifconfig.me)/32
aws ec2 authorize-security-group-ingress \
  --group-id $SG_APP \
  --protocol tcp --port 22 \
  --cidr $MON_IP
echo "SSH autorisé depuis : $MON_IP"
```

**Questions :**

1. Pourquoi utilise-t-on `--source-group $SG_ALB` au lieu de `--cidr 0.0.0.0/0` pour le SG App ?
2. Si on ajoute 5 nouvelles EC2 avec le SG ALB, doivent-on modifier la règle du SG App ?
3. La RDS avec le SG DB peut-elle être contactée depuis ton laptop directement ? Pourquoi ?
4. Quel est l'intérêt d'autoriser SSH uniquement depuis `$MON_IP/32` plutôt que `0.0.0.0/0` ?

<details>
<summary>Voir les réponses</summary>

1. Référencer un SG comme source est **dynamique** — toute ressource qui reçoit ce SG peut automatiquement accéder à l'app, sans modifier les règles. Avec un CIDR, il faudrait mettre à jour les règles à chaque changement d'IP.
2. **Non** — la règle autorise tout trafic provenant d'une ressource avec `SG_ALB`, peu importe leur nombre. C'est la force du pattern "SG comme source".
3. **Non** — ton laptop n'a pas le `SG_APP` attaché. La règle sur le `SG_DB` autorise uniquement les ressources avec `SG_APP`. Depuis Internet, il est impossible d'atteindre la RDS directement.
4. `/32` = une seule IP (la tienne). `0.0.0.0/0` autoriserait SSH depuis n'importe qui sur Internet — risque d'attaques brute-force massif. Toujours restreindre SSH à des IPs connues.

</details>

---

## Exercice 3 — Route Tables : rendre un subnet public

**Objectif** : comprendre concrètement que c'est la Route Table qui fait la différence entre un subnet public et privé.

```bash
VPC_ID="vpc-xxxxxxxx"
IGW_ID="igw-xxxxxxxx"  # ton Internet Gateway

# 1. Voir les Route Tables actuelles
aws ec2 describe-route-tables \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'RouteTables[*].{ID:RouteTableId,Name:Tags[?Key==`Name`].Value|[0],Routes:Routes[*].{Dest:DestinationCidrBlock,Target:join(`,`,[GatewayId,NatGatewayId,TransitGatewayId]|[?@!=null])}}' \
  --output table

# 2. Créer un subnet "neutre" (ni public ni privé encore)
SUBNET_TEST=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.5.0/24 \
  --availability-zone eu-central-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=lab-subnet-test}]' \
  --query 'Subnet.SubnetId' \
  --output text)
echo "Subnet test : $SUBNET_TEST"

# 3. Voir sa Route Table actuelle (Main RT par défaut)
aws ec2 describe-route-tables \
  --filters "Name=association.subnet-id,Values=$SUBNET_TEST" \
  --query 'RouteTables[*].Routes' \
  --output table

# 4. Créer une Route Table publique
RT_PUBLIC=$(aws ec2 create-route-table \
  --vpc-id $VPC_ID \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=lab-rt-public-test}]' \
  --query 'RouteTable.RouteTableId' \
  --output text)

# 5. Ajouter la route Internet → rend le subnet PUBLIC
aws ec2 create-route \
  --route-table-id $RT_PUBLIC \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id $IGW_ID

# 6. Associer la RT publique au subnet
aws ec2 associate-route-table \
  --route-table-id $RT_PUBLIC \
  --subnet-id $SUBNET_TEST

# 7. Vérifier → le subnet est maintenant PUBLIC
aws ec2 describe-route-tables \
  --route-table-ids $RT_PUBLIC \
  --query 'RouteTables[0].Routes' \
  --output table

# 8. Maintenant "privatiser" le subnet
# Créer une RT privée (sans route 0.0.0.0/0)
RT_PRIVATE=$(aws ec2 create-route-table \
  --vpc-id $VPC_ID \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=lab-rt-private-test}]' \
  --query 'RouteTable.RouteTableId' \
  --output text)

# Remplacer l'association
ASSOC_ID=$(aws ec2 describe-route-tables \
  --route-table-ids $RT_PUBLIC \
  --query 'RouteTables[0].Associations[0].RouteTableAssociationId' \
  --output text)

aws ec2 replace-route-table-association \
  --association-id $ASSOC_ID \
  --route-table-id $RT_PRIVATE

echo "Subnet maintenant PRIVÉ"

# 9. Nettoyer
aws ec2 delete-subnet --subnet-id $SUBNET_TEST
aws ec2 delete-route-table --route-table-id $RT_PUBLIC
aws ec2 delete-route-table --route-table-id $RT_PRIVATE
```

**Questions :**

1. Avant d'associer la RT publique, quelle Route Table était utilisée par le subnet ?
2. Qu'est-ce qui change concrètement dans le routage quand tu ajoutes `0.0.0.0/0 → igw` ?
3. Peut-on changer la Route Table d'un subnet sans redémarrer les EC2 qui s'y trouvent ?
4. Deux subnets peuvent-ils partager la même Route Table ?

<details>
<summary>Voir les réponses</summary>

1. La **Main Route Table** du VPC — celle associée par défaut à tous les subnets sans RT explicite. Elle contient uniquement la règle `local`.
2. Avant : tout trafic non-local est abandonné (pas de route). Après : tout trafic non-local est envoyé à l'IGW, qui le route vers Internet. Le subnet devient joignable depuis Internet si les ressources ont une IP publique et le SG approprié.
3. **Oui** — le changement de Route Table est immédiat et ne nécessite pas de redémarrage. Les connexions existantes peuvent être brièvement interrompues, mais les EC2 elles-mêmes ne redémarrent pas.
4. **Oui** — plusieurs subnets peuvent partager la même Route Table. C'est souvent le cas pour tous les subnets publics (même RT) et tous les subnets privés d'une même AZ.

</details>

---

## Exercice 4 — Flow Logs : voir le trafic réseau

**Objectif** : activer les Flow Logs sur un VPC et analyser le trafic réseau capturé.

> ⚠️ Cet exercice génère des logs dans CloudWatch — coût minime mais non nul.

```bash
VPC_ID="vpc-xxxxxxxx"

# 1. Créer un Log Group CloudWatch
aws logs create-log-group \
  --log-group-name /aws/vpc/flowlogs/lab

# 2. Créer un rôle IAM pour Flow Logs
# (permet à VPC de publier dans CloudWatch)
cat > flow-logs-trust.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "vpc-flow-logs.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
EOF

ROLE_ARN=$(aws iam create-role \
  --role-name VPCFlowLogsRole \
  --assume-role-policy-document file://flow-logs-trust.json \
  --query 'Role.Arn' --output text)

# Attacher la politique CloudWatch Logs
aws iam attach-role-policy \
  --role-name VPCFlowLogsRole \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchLogsFullAccess

# 3. Activer les Flow Logs sur le VPC
FLOW_LOG_ID=$(aws ec2 create-flow-logs \
  --resource-type VPC \
  --resource-ids $VPC_ID \
  --traffic-type ALL \
  --log-destination-type cloud-watch-logs \
  --log-group-name /aws/vpc/flowlogs/lab \
  --deliver-logs-permission-arn $ROLE_ARN \
  --query 'FlowLogIds[0]' --output text)
echo "Flow Log activé : $FLOW_LOG_ID"

# 4. Générer du trafic pour créer des logs
# Lance une EC2 dans le VPC (ou utilise une existante)
# et génère quelques connexions (ping, curl...)

# 5. Après quelques minutes, voir les logs
aws logs describe-log-streams \
  --log-group-name /aws/vpc/flowlogs/lab \
  --query 'logStreams[*].logStreamName' \
  --output table

# Lire les logs (remplace par ton log stream)
aws logs get-log-events \
  --log-group-name /aws/vpc/flowlogs/lab \
  --log-stream-name "eni-xxxxxxxxx-all" \
  --query 'events[*].message' \
  --output text | head -20

# 6. Analyser les patterns
# Chercher les REJECT (connexions bloquées)
aws logs filter-log-events \
  --log-group-name /aws/vpc/flowlogs/lab \
  --filter-pattern "REJECT" \
  --query 'events[*].message' \
  --output text

# Chercher le trafic SSH (port 22)
aws logs filter-log-events \
  --log-group-name /aws/vpc/flowlogs/lab \
  --filter-pattern "22" \
  --query 'events[*].message' \
  --output text

# 7. Nettoyer
aws ec2 delete-flow-logs --flow-log-ids $FLOW_LOG_ID
aws logs delete-log-group --log-group-name /aws/vpc/flowlogs/lab
rm flow-logs-trust.json
```

**Questions :**

1. Dans un log Flow Logs, que signifie `REJECT` vs `ACCEPT` ?
2. Si tu vois des entrées `REJECT` sur le port 22 depuis des IPs inconnues — qu'est-ce que ça indique ?
3. Quelle est la différence entre Flow Logs et tcpdump ?
4. Dans quel scénario de debug les Flow Logs sont-ils plus utiles que les logs applicatifs ?

<details>
<summary>Voir les réponses</summary>

1. `ACCEPT` = trafic autorisé par les Security Groups / NACLs. `REJECT` = trafic bloqué par un SG ou NACL.
2. Des **tentatives de connexion SSH depuis Internet** — des robots qui scannent en permanence toutes les IPs publiques. C'est normal sur Internet, c'est pour ça qu'on restreint SSH à une IP spécifique.
3. **tcpdump** capture le contenu complet des paquets (payload, données). **Flow Logs** capture uniquement les métadonnées (qui parle à qui, sur quel port, accepté ou refusé) — sans le contenu. Flow Logs est moins intrusif et scalable sur des milliers d'instances.
4. Quand le problème est réseau (connexion qui n'arrive pas) et non applicatif. Si les logs de l'app ne montrent rien → le paquet n'est peut-être jamais arrivé. Flow Logs te dit si le trafic a été `ACCEPT` ou `REJECT` — tu sais immédiatement si c'est un SG qui bloque.

</details>

---

## Exercice 5 — VPC Peering : connecter deux VPC

**Objectif** : créer une connexion de peering entre deux VPC et vérifier la communication inter-VPC.

```bash
# 1. Créer un deuxième VPC
VPC_B=$(aws ec2 create-vpc \
  --cidr-block 10.1.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=lab-vpc-b}]' \
  --query 'Vpc.VpcId' --output text)

SUBNET_B=$(aws ec2 create-subnet \
  --vpc-id $VPC_B \
  --cidr-block 10.1.1.0/24 \
  --availability-zone eu-central-1a \
  --query 'Subnet.SubnetId' --output text)

echo "VPC B : $VPC_B"
echo "Subnet B : $SUBNET_B"

# 2. Créer la connexion de peering
PEERING_ID=$(aws ec2 create-vpc-peering-connection \
  --vpc-id $VPC_ID \
  --peer-vpc-id $VPC_B \
  --tag-specifications 'ResourceType=vpc-peering-connection,Tags=[{Key=Name,Value=lab-peering-a-b}]' \
  --query 'VpcPeeringConnection.VpcPeeringConnectionId' \
  --output text)
echo "Peering créé : $PEERING_ID"

# 3. Accepter la connexion de peering
aws ec2 accept-vpc-peering-connection \
  --vpc-peering-connection-id $PEERING_ID

# 4. Vérifier le statut
aws ec2 describe-vpc-peering-connections \
  --vpc-peering-connection-ids $PEERING_ID \
  --query 'VpcPeeringConnections[0].Status.Code' \
  --output text
# → active ✅

# 5. Mettre à jour les Route Tables des DEUX VPC
# RT du VPC A → route vers VPC B via peering
RT_A=$(aws ec2 describe-route-tables \
  --filters "Name=vpc-id,Values=$VPC_ID" "Name=association.main,Values=true" \
  --query 'RouteTables[0].RouteTableId' --output text)

aws ec2 create-route \
  --route-table-id $RT_A \
  --destination-cidr-block 10.1.0.0/16 \
  --vpc-peering-connection-id $PEERING_ID

# RT du VPC B → route vers VPC A via peering
RT_B=$(aws ec2 describe-route-tables \
  --filters "Name=vpc-id,Values=$VPC_B" "Name=association.main,Values=true" \
  --query 'RouteTables[0].RouteTableId' --output text)

aws ec2 create-route \
  --route-table-id $RT_B \
  --destination-cidr-block 10.0.0.0/16 \
  --vpc-peering-connection-id $PEERING_ID

echo "Routes ajoutées dans les deux VPC"

# 6. Vérifier les routes
echo "=== Routes VPC A ==="
aws ec2 describe-route-tables \
  --route-table-ids $RT_A \
  --query 'RouteTables[0].Routes' \
  --output table

echo "=== Routes VPC B ==="
aws ec2 describe-route-tables \
  --route-table-ids $RT_B \
  --query 'RouteTables[0].Routes' \
  --output table

# 7. Nettoyer
aws ec2 delete-vpc-peering-connection \
  --vpc-peering-connection-id $PEERING_ID
aws ec2 delete-subnet --subnet-id $SUBNET_B
aws ec2 delete-vpc --vpc-id $VPC_B
```

**Questions :**

1. Après avoir créé le peering, les deux VPC peuvent-ils immédiatement communiquer ?
2. Si VPC A est en peering avec VPC B, et VPC B avec VPC C — VPC A peut-il joindre VPC C ?
3. Pourquoi les CIDR de VPC A (10.0.0.0/16) et VPC B (10.1.0.0/16) doivent-ils être différents ?
4. Quand utiliser VPC Peering vs Transit Gateway ?

<details>
<summary>Voir les réponses</summary>

1. **Non** — trois étapes sont nécessaires : créer le peering, accepter le peering, **et** mettre à jour les Route Tables des deux VPC. Sans les routes, les paquets ne savent pas comment atteindre l'autre VPC.
2. **Non** — le peering est **non transitif**. A↔B et B↔C ne crée pas A↔C. Il faudrait créer un peering A↔C séparé.
3. S'ils se chevauchaient (ex: les deux en 10.0.0.0/16), AWS ne saurait pas si un paquet vers 10.0.1.42 doit rester dans le VPC local ou partir vers le VPC distant. Le routing serait ambigu et le peering refusé.
4. **VPC Peering** pour 2-5 VPC avec peu de connexions (simple, gratuit, faible latence). **Transit Gateway** pour 5+ VPC, besoin de transitivité, connexion on-premise — plus complexe et coûteux mais beaucoup plus flexible.

</details>

---

## Exercice 6 — Route 53 : DNS public et privé

**Objectif** : créer des enregistrements DNS publics et une Hosted Zone privée pour les communications internes.

```bash
# PARTIE A — Hosted Zone privée (interne au VPC)

# 1. Créer une Hosted Zone privée
HZ_ID=$(aws route53 create-hosted-zone \
  --name "prod.internal" \
  --vpc VPCRegion=eu-central-1,VPCId=$VPC_ID \
  --caller-reference "lab-$(date +%s)" \
  --hosted-zone-config Comment="Zone privée lab",PrivateZone=true \
  --query 'HostedZone.Id' --output text)
echo "Hosted Zone : $HZ_ID"

# 2. Créer des enregistrements DNS internes
cat > dns-records.json << EOF
{
  "Changes": [
    {
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "database.prod.internal",
        "Type": "A",
        "TTL": 300,
        "ResourceRecords": [{"Value": "10.0.20.5"}]
      }
    },
    {
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "cache.prod.internal",
        "Type": "A",
        "TTL": 300,
        "ResourceRecords": [{"Value": "10.0.20.10"}]
      }
    },
    {
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "api.prod.internal",
        "Type": "CNAME",
        "TTL": 300,
        "ResourceRecords": [{"Value": "database.prod.internal"}]
      }
    }
  ]
}
EOF

aws route53 change-resource-record-sets \
  --hosted-zone-id $HZ_ID \
  --change-batch file://dns-records.json

# 3. Vérifier les enregistrements créés
aws route53 list-resource-record-sets \
  --hosted-zone-id $HZ_ID \
  --query 'ResourceRecordSets[*].{Name:Name,Type:Type,Value:ResourceRecords[0].Value}' \
  --output table

# 4. Tester la résolution (depuis une EC2 dans le VPC)
# Sur une EC2 dans le VPC :
# dig database.prod.internal
# → doit retourner 10.0.20.5

# PARTIE B — Tester les routing policies

# 5. Créer deux enregistrements weighted
cat > weighted-records.json << EOF
{
  "Changes": [
    {
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "app.prod.internal",
        "Type": "A",
        "SetIdentifier": "v1",
        "Weight": 90,
        "TTL": 60,
        "ResourceRecords": [{"Value": "10.0.10.1"}]
      }
    },
    {
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "app.prod.internal",
        "Type": "A",
        "SetIdentifier": "v2",
        "Weight": 10,
        "TTL": 60,
        "ResourceRecords": [{"Value": "10.0.10.2"}]
      }
    }
  ]
}
EOF

aws route53 change-resource-record-sets \
  --hosted-zone-id $HZ_ID \
  --change-batch file://weighted-records.json

echo "Weighted routing créé : 90% vers 10.0.10.1, 10% vers 10.0.10.2"

# 6. Nettoyer
aws route53 delete-hosted-zone --id $HZ_ID
rm dns-records.json weighted-records.json
```

**Questions :**

1. Quelle est la différence entre une Hosted Zone publique et privée ?
2. Pourquoi utilise-t-on `database.prod.internal` au lieu de l'IP `10.0.20.5` dans le code ?
3. Le weighted routing 90/10 dans l'étape 5 — à quel cas d'usage réel correspond-il ?
4. Si le TTL est de 60 secondes et que tu changes l'IP de `database.prod.internal` — combien de temps faut-il pour que toutes les EC2 voient le changement ?

<details>
<summary>Voir les réponses</summary>

1. **Zone publique** : résolvable depuis Internet, pour les domaines accessibles au public (monapp.com). **Zone privée** : résolvable uniquement depuis les VPC associés, pour les communications internes (database.prod.internal). Les noms internes ne sont pas visibles depuis Internet.
2. Si l'IP change (migration, scaling, remplacement d'instance), il suffit de mettre à jour l'enregistrement DNS — **aucun code à modifier**. Avec une IP hardcodée, il faudrait modifier et redéployer toutes les applications qui l'utilisent.
3. **Déploiement canary** — 10% du trafic va vers la nouvelle version (v2) pour valider qu'elle fonctionne correctement, 90% reste sur la version stable (v1). Si v2 présente des problèmes, seuls 10% des utilisateurs sont impactés.
4. **60 secondes maximum** — le TTL est la durée pendant laquelle les resolvers cachent la réponse. Après 60 secondes, ils redemandent à Route 53 et obtiennent la nouvelle IP. C'est pour ça qu'on baisse le TTL avant une migration importante.

</details>

---

## Quiz final — Module 4

Réponds sans regarder le cours.

1. Quelle est la différence entre un subnet public et un subnet privé dans AWS ?
2. Une EC2 dans un subnet privé peut-elle accéder à Internet ? Comment ?
3. Les Security Groups sont stateful — qu'est-ce que ça signifie concrètement ?
4. Quelle est la différence entre un Security Group et un Network ACL ?
5. Tu veux qu'une EC2 app puisse joindre une RDS mais pas l'inverse. Comment configures-tu les Security Groups ?
6. Tu vois dans les Flow Logs : `10.0.1.42 → 10.0.20.5 port 5432 REJECT`. Que s'est-il passé ?
7. Deux VPC en peering — VPC A peut-il atteindre les ressources de VPC C via VPC B ?
8. Quelle est la différence entre VPC Peering et Transit Gateway ?
9. Quelle est la différence entre un Gateway Endpoint et un Interface Endpoint ?
10. Une EC2 a une IP publique mais n'est pas accessible depuis Internet. Quelles sont les 3 conditions à vérifier ?
11. Quelle routing policy Route 53 utiliserais-tu pour envoyer les utilisateurs européens vers des serveurs en Europe ?
12. Une NAT Gateway est dans l'AZ `eu-central-1a`. Les EC2 du subnet privé dans `eu-central-1b` peuvent-elles l'utiliser ? Est-ce une bonne pratique ?

<details>
<summary>Voir les réponses</summary>

1. La différence vient de la **Route Table** : subnet public a `0.0.0.0/0 → IGW` (accès Internet bidirectionnel), subnet privé a `0.0.0.0/0 → NAT GW` (sortie seulement) ou aucune route (isolé).
2. **Oui, via une NAT Gateway** dans un subnet public. L'EC2 envoie le trafic à la NAT GW via sa Route Table, la NAT GW traduit l'IP privée en IP publique et accède à Internet. La réponse revient à la NAT GW qui la retransmet à l'EC2.
3. **Stateful** = le SG se souvient des connexions établies. Si tu autorises le trafic entrant sur le port 443, la réponse sortante est automatiquement autorisée sans règle outbound explicite.
4. **Security Group** : stateful, au niveau de la ressource (ENI), règles Allow uniquement, toutes évaluées. **Network ACL** : stateless, au niveau du subnet, règles Allow ET Deny, évaluées dans l'ordre numérique (s'arrête à la première correspondance).
5. SG de la RDS : `Inbound port 5432 depuis SG-App`. SG de l'EC2 App : `Outbound All traffic` (par défaut). La RDS n'a aucune règle permettant d'initier des connexions vers l'EC2 — le trafic ne peut aller que dans un sens.
6. L'EC2 (`10.0.1.42`) a essayé de contacter la RDS (`10.0.20.5`) sur le port PostgreSQL (5432) mais la connexion a été **bloquée** par un Security Group ou un NACL. La RDS ne peut pas être jointe — vérifier les règles inbound du SG de la RDS.
7. **Non** — le peering est **non transitif**. A peut joindre B, B peut joindre C, mais A ne peut pas joindre C via B. Il faut un peering direct A↔C ou utiliser un Transit Gateway.
8. **VPC Peering** : connexion point-à-point, non transitif, gratuit, max ~125 peerings. **Transit Gateway** : hub central, transitif, coûteux, supporte des milliers de connexions VPC + on-premise.
9. **Gateway Endpoint** : pour S3 et DynamoDB uniquement, gratuit, ajoute une route dans la RT. **Interface Endpoint** : pour tous les services AWS (SSM, Secrets Manager...), crée une ENI avec IP privée dans ton subnet, payant (~$7/mois/AZ).
10. Trois conditions simultanées : (1) le subnet doit avoir une route `0.0.0.0/0 → IGW`, (2) l'EC2 doit avoir une IP publique ou Elastic IP, (3) le Security Group doit autoriser le trafic entrant sur le port concerné.
11. **Geolocation routing** — route le trafic selon la localisation géographique de l'utilisateur. Les IPs européennes sont dirigées vers les serveurs européens. Utile aussi pour la conformité RGPD.
12. **Oui, techniquement** — mais c'est une **mauvaise pratique**. Si l'AZ `1a` tombe en panne, la NAT Gateway aussi — les EC2 dans `1b` perdent leur accès Internet. La bonne pratique : une NAT Gateway par AZ, chaque subnet privé d'une AZ utilise la NAT GW de sa propre AZ.

</details>

---

*→ Retour au cours : `module4-theorie.md`*
*→ Module suivant : Module 5 — Kubernetes Networking*
