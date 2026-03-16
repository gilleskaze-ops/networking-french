# Module 4 — AWS Networking
### Partie Théorie

> **Cours Networking Fondamentaux · Orienté Cloud**
> Prérequis : Modules 1, 2 et 3 complétés · Compte AWS requis · Durée estimée : 3h

---

## Note pédagogique — Pourquoi AWS Networking arrive ici

Dans le monde professionnel, l'ordre logique d'apprentissage est :

```
AWS Networking → Docker Networking → Kubernetes Networking
```

On construit d'abord l'infrastructure réseau Cloud (AWS), puis on déploie les conteneurs (Docker) dessus, puis on les orchestre (Kubernetes).

Ce cours a fait le choix inverse pour une raison pratique : Docker s'installe sur ta machine en 5 minutes et tu peux pratiquer immédiatement, sans compte AWS, sans coût. Les concepts réseau de Docker (bridge, NAT, DNS, ports, isolation) sont identiques à ceux d'AWS — juste virtualisés différemment.

Maintenant que ces concepts sont ancrés, on construit l'infrastructure Cloud sur laquelle tout ça tourne réellement. Tout ce que tu as vu dans les modules précédents trouve son équivalent ici :

```
docker0 bridge      →  VPC
réseau personnalisé →  Subnet
ufw / iptables      →  Security Group
NAT Docker          →  NAT Gateway
DNS 127.0.0.11      →  Route 53 Private Hosted Zone
port forwarding     →  Load Balancer
```

---

## Glossaire

| Terme | Signification | Définition simple |
|-------|--------------|-------------------|
| **ACL** | Access Control List | Liste de règles de filtrage réseau — stateless, au niveau subnet |
| **ALB** | Application Load Balancer | Load balancer couche 7 — route selon l'URL, les headers HTTP |
| **AZ** | Availability Zone | Centre de données physiquement séparé dans une région AWS |
| **CIDR** | Classless Inter-Domain Routing | Notation pour définir la taille d'un réseau (ex: `/24`) |
| **Direct Connect** | — | Connexion réseau physique dédiée entre ton datacenter et AWS |
| **DHCP** | Dynamic Host Configuration Protocol | Attribue automatiquement les IPs dans un réseau |
| **EIP** | Elastic IP | IP publique fixe réservée dans ton compte AWS |
| **ENI** | Elastic Network Interface | Interface réseau virtuelle attachée à une EC2 |
| **Flow Logs** | — | Capture du trafic réseau d'un VPC pour audit et debug |
| **IGW** | Internet Gateway | Passerelle entre un VPC et Internet |
| **NACLs** | Network Access Control Lists | Firewall stateless au niveau subnet dans AWS |
| **NAT Gateway** | Network Address Translation Gateway | Permet aux instances privées d'accéder à Internet en sortie |
| **NLB** | Network Load Balancer | Load balancer couche 4 — ultra-rapide, TCP/UDP |
| **PrivateLink** | — | Accès privé à un service AWS ou tiers sans passer par Internet |
| **Region** | — | Zone géographique AWS composée de plusieurs AZ (ex: eu-west-1) |
| **Route Table** | Table de routage | Règles qui définissent où envoyer le trafic dans un VPC |
| **Security Group** | — | Firewall stateful attaché à une ressource AWS |
| **Subnet** | Sous-réseau | Subdivision d'un VPC dans une AZ spécifique |
| **Target Group** | — | Ensemble de ressources (EC2, conteneurs) vers lesquelles un LB route le trafic |
| **Transit Gateway** | — | Hub réseau central pour connecter plusieurs VPC et réseaux on-premise |
| **VPC** | Virtual Private Cloud | Réseau privé virtuel isolé dans AWS |
| **VPC Endpoint** | — | Accès privé aux services AWS sans passer par Internet |
| **VPC Peering** | — | Connexion réseau directe entre deux VPC |
| **VPN Gateway** | — | Passerelle pour créer un tunnel VPN entre ton réseau et AWS |

---

## Sommaire

**Partie 1 — Les fondations**
1. [VPC — ton réseau privé dans AWS](#1-vpc--ton-réseau-privé-dans-aws)
2. [Subnets, Availability Zones et résilience](#2-subnets-availability-zones-et-résilience)
3. [Route Tables — diriger le trafic](#3-route-tables--diriger-le-trafic)
4. [ENI — l'interface réseau d'une EC2](#4-eni--linterface-réseau-dune-ec2)

**Partie 2 — Connectivité Internet**
5. [Internet Gateway](#5-internet-gateway)
6. [NAT Gateway et Elastic IP](#6-nat-gateway-et-elastic-ip)

**Partie 3 — Sécurité réseau**
7. [Security Groups](#7-security-groups)
8. [Network ACLs](#8-network-acls)
9. [VPC Flow Logs](#9-vpc-flow-logs)

**Partie 4 — Connectivité avancée**
10. [VPC Peering](#10-vpc-peering)
11. [PrivateLink et VPC Endpoints](#11-privatelink-et-vpc-endpoints)
12. [Transit Gateway, VPN et Direct Connect](#12-transit-gateway-vpn-et-direct-connect)

**Partie 5 — Services réseau managés**
13. [ALB et NLB](#13-alb-et-nlb)
14. [Route 53](#14-route-53)
15. [DHCP Option Sets](#15-dhcp-option-sets)
16. [AWS Network Firewall et Global Accelerator](#16-aws-network-firewall-et-global-accelerator)

---

## Partie 1 — Les fondations

## 1. VPC — ton réseau privé dans AWS

### Pourquoi c'est important

Le VPC est **la brique fondamentale de toute architecture AWS**. Chaque ressource que tu crées — EC2, RDS, Lambda, EKS — vit dans un VPC. Sans VPC, pas de réseau. Sans réseau, rien ne fonctionne.

Comprendre le VPC, c'est comprendre comment isoler tes environnements, comment contrôler qui peut parler à qui, et comment connecter ton infrastructure Cloud au monde extérieur.

### Ce qu'est un VPC

Un **VPC** (Virtual Private Cloud) est un réseau privé virtuel que tu crées dans AWS, dans une région spécifique. Il est complètement isolé des autres VPC — même ceux d'autres clients AWS sur la même infrastructure physique.

```
Région AWS eu-central-1 (Frankfurt)
┌─────────────────────────────────────────────────────┐
│                                                     │
│  Ton VPC (10.0.0.0/16)        VPC d'un autre client │
│  ┌─────────────────────┐      ┌──────────────────┐  │
│  │  tes ressources     │      │  ses ressources  │  │
│  │  isolées            │      │  isolées         │  │
│  └─────────────────────┘      └──────────────────┘  │
│                                                     │
└─────────────────────────────────────────────────────┘
Les deux VPC coexistent sur la même infra physique
mais sont complètement invisibles l'un pour l'autre
```

### Choisir le bon CIDR pour son VPC

Le CIDR du VPC définit l'espace d'adressage disponible pour toutes tes ressources. C'est une décision importante — difficile à changer après coup.

```
Règles à respecter :
→ Utiliser une plage privée (10.x, 172.16-31.x, 192.168.x)
→ Prendre suffisamment grand pour la croissance
→ Éviter les chevauchements avec d'autres réseaux
  (on-premise, autres VPC avec lesquels tu voudras peut-être peerer)

Recommandations pratiques :
/16 → 65 536 adresses — standard pour un VPC de production
/20 → 4 096 adresses  — suffisant pour un petit projet
/24 → 256 adresses    — trop petit, à éviter pour un VPC

Exemple d'architecture multi-environnements sans chevauchement :
  prod    : 10.0.0.0/16
  staging : 10.1.0.0/16
  dev     : 10.2.0.0/16
```

### VPC par défaut vs VPC personnalisé

AWS crée automatiquement un **VPC par défaut** dans chaque région — avec des subnets publics préconfigurés et une Internet Gateway attachée. C'est pratique pour débuter mais **jamais utilisé en production** :

```
VPC par défaut :
  CIDR : 172.31.0.0/16
  Subnets : publics uniquement, un par AZ
  IGW : attachée automatiquement
  Inconvénient : toutes les EC2 ont une IP publique par défaut ⚠️

VPC personnalisé (production) :
  CIDR : défini par toi
  Subnets : publics ET privés, architecture maîtrisée
  IGW : attachée si nécessaire
  Avantage : contrôle total sur l'isolation et la sécurité ✅
```

---

## 2. Subnets, Availability Zones et résilience

### Pourquoi c'est important

Un VPC seul ne suffit pas — il faut le subdiviser en subnets. La façon dont tu découpes ton VPC en subnets définit **qui peut accéder à quoi** et **à quel niveau de résilience** ton architecture résiste aux pannes.

### Subnets — subdiviser le VPC

Un **subnet** est une subdivision du VPC dans une **Availability Zone** spécifique. Chaque subnet a son propre CIDR — un sous-ensemble du CIDR du VPC.

```
VPC : 10.0.0.0/16 (région eu-central-1)
  │
  ├── subnet-public-1a  : 10.0.1.0/24  (AZ eu-central-1a)
  ├── subnet-public-1b  : 10.0.2.0/24  (AZ eu-central-1b)
  ├── subnet-private-1a : 10.0.10.0/24 (AZ eu-central-1a)
  ├── subnet-private-1b : 10.0.11.0/24 (AZ eu-central-1b)
  ├── subnet-db-1a      : 10.0.20.0/24 (AZ eu-central-1a)
  └── subnet-db-1b      : 10.0.21.0/24 (AZ eu-central-1b)
```

**Règle importante** : un subnet existe dans **une seule AZ**. Une AZ peut contenir plusieurs subnets.

### Availability Zones — la résilience réseau

Une **Availability Zone** est un datacenter physiquement séparé dans une région AWS. Les AZ d'une même région sont connectées par des fibres à très faible latence mais sont suffisamment distantes pour ne pas subir la même panne.

```
Région eu-central-1 (Frankfurt)
  ├── AZ eu-central-1a  ← datacenter A
  ├── AZ eu-central-1b  ← datacenter B
  └── AZ eu-central-1c  ← datacenter C

Si l'AZ "a" tombe en panne :
  → les ressources dans "b" et "c" continuent de fonctionner
  → le trafic bascule automatiquement (si architecture multi-AZ)
```

### Subnet public vs subnet privé

La distinction public/privé **ne vient pas d'un paramètre** sur le subnet lui-même — elle vient de la **Route Table** qui lui est associée :

```
Subnet PUBLIC :
  Route Table contient : 0.0.0.0/0 → Internet Gateway
  → les ressources peuvent recevoir du trafic depuis Internet
  → utilisé pour : Load Balancers, NAT Gateway, Bastion hosts

Subnet PRIVÉ :
  Route Table contient : 0.0.0.0/0 → NAT Gateway
  → les ressources peuvent initier des connexions vers Internet
    (mises à jour, appels API externes)
  → mais ne sont PAS joignables depuis Internet
  → utilisé pour : EC2 app servers, EKS nodes

Subnet ISOLÉ (BDD) :
  Route Table : pas de route 0.0.0.0/0
  → aucun accès Internet dans aucune direction
  → utilisé pour : RDS, ElastiCache, données sensibles
```

### Architecture réseau standard AWS

```
Internet
    │
    ▼
Internet Gateway
    │
    ├── Subnet Public 1a ──────────────── Subnet Public 1b
    │   [ALB, NAT GW]                    [ALB, NAT GW]
    │         │                                │
    ├── Subnet Privé 1a ──────────────── Subnet Privé 1b
    │   [EC2 App, EKS]                   [EC2 App, EKS]
    │         │                                │
    └── Subnet DB 1a ──────────────────── Subnet DB 1b
        [RDS Primary]                    [RDS Standby]
```

Chaque couche est dupliquée sur deux AZ — si une AZ tombe, l'autre prend le relais.

---

## 3. Route Tables — diriger le trafic

### Pourquoi c'est important

La Route Table est le **GPS du VPC**. C'est elle qui décide où va chaque paquet — vers Internet, vers un autre subnet, vers un VPC distant, ou nulle part. C'est aussi la Route Table qui fait la différence entre un subnet public et un subnet privé.

### Fonctionnement

Chaque subnet est associé à une Route Table. Quand un paquet quitte une ressource dans ce subnet, AWS consulte la Route Table pour savoir où l'envoyer.

```
Route Table — subnet public :
Destination    Target
10.0.0.0/16    local       ← trafic interne au VPC → direct
0.0.0.0/0      igw-xxx     ← tout le reste → Internet Gateway

Route Table — subnet privé :
Destination    Target
10.0.0.0/16    local       ← trafic interne au VPC → direct
0.0.0.0/0      nat-xxx     ← tout le reste → NAT Gateway

Route Table — subnet BDD :
Destination    Target
10.0.0.0/16    local       ← trafic interne uniquement
(pas de 0.0.0.0/0)         ← aucun accès Internet ✅
```

### La règle "local" — toujours présente

Chaque Route Table contient automatiquement une règle `local` pour le CIDR du VPC. Elle ne peut pas être supprimée — c'est ce qui permet à toutes les ressources du VPC de se parler directement.

```
10.0.0.0/16 → local

Cette règle dit : "tout trafic vers une IP dans 10.0.0.0/16
passe directement par le réseau interne du VPC"
Pas besoin de routeur, pas de NAT — communication directe.
```

### Main Route Table vs Route Tables personnalisées

```
Main Route Table :
  → créée automatiquement avec le VPC
  → associée à tous les subnets qui n'ont pas de RT dédiée
  → par défaut : uniquement la règle local

Bonne pratique :
  → ne jamais modifier la Main Route Table
  → créer une RT dédiée par type de subnet
  → associer explicitement chaque subnet à sa RT
```

### Ordre de priorité des routes

AWS applique la règle la **plus spécifique** en premier :

```
Destination    Target
10.0.0.0/16    local
10.0.5.0/24    vpn-gateway  ← plus spécifique que /16
0.0.0.0/0      igw-xxx

Un paquet vers 10.0.5.42 :
→ 10.0.5.0/24 est plus spécifique que 10.0.0.0/16
→ Il part vers vpn-gateway, pas vers local
```

---

## 4. ENI — l'interface réseau d'une EC2

### Pourquoi c'est important

On a vu dans le Module 2 que chaque machine Linux a des interfaces réseau (`eth0`, `wlan0`...). Sur AWS, l'équivalent est l'**ENI**. C'est l'ENI qui donne à une EC2 son IP privée, son IP publique (optionnelle), et son adresse MAC. Comprendre l'ENI, c'est comprendre comment une EC2 est réellement connectée au réseau AWS.

### Ce qu'est une ENI

Une **ENI** (Elastic Network Interface) est une interface réseau virtuelle qu'on attache à une EC2. Elle vit dans un subnet spécifique et hérite de ses propriétés réseau.

```
EC2 instance
  └── ENI (eth0)
        ├── IP privée      : 10.0.1.42 (attribuée par DHCP AWS)
        ├── IP publique    : 54.12.34.56 (optionnelle)
        ├── Adresse MAC    : 0a:1b:2c:3d:4e:5f
        ├── Security Group : sg-xxxxxxxx
        └── Subnet         : subnet-public-1a
```

### Propriétés d'une ENI

```
IP privée primaire :
  → attribuée automatiquement par DHCP AWS au lancement
  → reste fixe pendant toute la vie de l'instance
  → libérée quand l'instance est terminée

IPs privées secondaires :
  → tu peux en ajouter plusieurs sur la même ENI
  → utile pour héberger plusieurs services sur la même instance

IP publique :
  → optionnelle, attribuée automatiquement si le subnet le permet
  → CHANGE à chaque redémarrage de l'instance ⚠️
  → pour une IP fixe → utiliser une Elastic IP

Elastic IP (EIP) :
  → IP publique fixe que tu réserves dans ton compte
  → reste la même même si tu redémarres ou changes d'instance
  → coût : gratuit si attachée à une instance en cours
            payant si réservée mais non utilisée (~$4/mois)
```

### Plusieurs ENIs sur une EC2

Une EC2 peut avoir plusieurs ENIs — utile pour des cas avancés :

```
EC2 avec deux ENIs :
  eth0 → ENI dans subnet public  (trafic entrant depuis Internet)
  eth1 → ENI dans subnet privé   (trafic vers la BDD)

Cas d'usage :
  → appliance réseau (firewall, proxy)
  → séparer le trafic de gestion du trafic applicatif
  → migration : détacher une ENI d'une instance et l'attacher à une autre
    → l'IP privée et le Security Group suivent l'ENI ✅
```

---

## Partie 2 — Connectivité Internet

## 5. Internet Gateway

### Pourquoi c'est important

Une Internet Gateway (IGW) est ce qui relie ton VPC à Internet. Sans IGW, ton VPC est complètement isolé — aucune ressource ne peut ni recevoir ni envoyer de trafic vers Internet. C'est la porte d'entrée et de sortie de ton infrastructure Cloud.

### Ce que fait une IGW

```
L'IGW fait deux choses :

1. Routage bidirectionnel
   → permet au trafic d'entrer dans le VPC depuis Internet
   → permet au trafic de sortir du VPC vers Internet

2. NAT pour les IPs publiques
   → traduit l'IP publique d'une ressource en son IP privée
   → et vice versa

Exemple :
  EC2 (IP privée 10.0.1.42, IP publique 54.12.34.56)
  Trafic entrant depuis Internet vers 54.12.34.56
  → IGW traduit 54.12.34.56 en 10.0.1.42 et livre à l'EC2 ✅
```

### Propriétés importantes

```
→ Une seule IGW par VPC (one-to-one)
→ Hautement disponible par défaut — pas de gestion
→ Pas de bande passante limite — AWS gère
→ Gratuite — pas de coût pour l'IGW elle-même
  (tu paies le transfert de données, pas l'IGW)

Conditions pour qu'une ressource soit accessible depuis Internet :
  1. Subnet avec route 0.0.0.0/0 → IGW ✅
  2. EC2 avec une IP publique ou EIP ✅
  3. Security Group autorisant le trafic entrant ✅
  Les trois conditions sont nécessaires simultanément.
```

---

## 6. NAT Gateway et Elastic IP

### Pourquoi c'est important

Tes EC2 dans les subnets privés n'ont pas d'IP publique — elles ne sont pas joignables depuis Internet. Mais elles ont parfois besoin d'accéder à Internet en **sortie** : télécharger des mises à jour, appeler des APIs externes, pousser des logs vers un service externe. La NAT Gateway résout exactement ce problème.

### NAT Gateway — sortie Internet pour les ressources privées

```
EC2 privée (10.0.10.42)
    │  "je veux télécharger une mise à jour"
    ▼
Route Table subnet privé : 0.0.0.0/0 → nat-xxxxxxxx
    ▼
NAT Gateway (dans subnet PUBLIC, EIP: 54.99.88.77)
    │  remplace 10.0.10.42 par 54.99.88.77
    ▼
Internet Gateway
    ▼
Internet ✅

Réponse revient vers 54.99.88.77
→ NAT Gateway sait que c'est pour 10.0.10.42
→ retransmet à l'EC2 privée ✅
```

### Architecture multi-AZ avec NAT Gateway

```
⚠️ Une NAT Gateway est dans une seule AZ.
   Si l'AZ tombe, les ressources privées de cette AZ
   perdent leur accès Internet.

Bonne pratique : une NAT Gateway par AZ

  Subnet Public 1a → NAT GW 1a (EIP: 54.x.x.1)
  Subnet Public 1b → NAT GW 1b (EIP: 54.x.x.2)

  Route Table subnet privé 1a : 0.0.0.0/0 → NAT GW 1a
  Route Table subnet privé 1b : 0.0.0.0/0 → NAT GW 1b

Coût : ~$0.045/heure + $0.045/Go
       → 2 NAT GW = ~$65/mois minimum
       → décision architecturale à prendre selon le budget
```

### Elastic IP

```
Une Elastic IP est une IP publique fixe que tu réserves
dans ton compte AWS — indépendamment de toute instance.

Cas d'usage :
  → IP fixe pour une NAT Gateway
  → IP fixe pour un serveur qui doit être whitelisté
    chez des partenaires
  → Migration sans downtime : détacher l'EIP d'une instance
    et l'attacher à une autre

Coût :
  → Gratuit si attachée à une instance en cours d'exécution
  → $0.005/heure si réservée mais non utilisée (pénalité AWS)
  → Libère toujours tes EIPs inutilisées !
```

---

## Partie 3 — Sécurité réseau

## 7. Security Groups

### Pourquoi c'est important

Les Security Groups sont **le mécanisme de sécurité réseau le plus utilisé sur AWS**. Ils contrôlent exactement quel trafic peut entrer et sortir de chaque ressource. Mal configurés, ils exposent des services critiques à Internet. Bien configurés, ils créent une défense en profondeur où chaque couche ne peut parler qu'à la couche adjacente.

### Ce qu'est un Security Group

Un **Security Group** est un firewall virtuel **stateful** attaché à une ressource AWS (EC2, RDS, ALB, Lambda...). Il filtre le trafic selon des règles inbound (entrant) et outbound (sortant).

```
Stateful = il se souvient des connexions

Si tu autorises le trafic entrant sur le port 443,
la réponse sortante est automatiquement autorisée
→ pas besoin de règle outbound explicite pour les réponses

C'est la différence fondamentale avec les Network ACLs
qui sont stateless (pas de mémoire des connexions).
```

### Règles inbound et outbound

```
Par défaut, un Security Group :
  → Inbound  : DENY tout (aucune connexion entrante autorisée)
  → Outbound : ALLOW tout (toutes les connexions sortantes autorisées)

Chaque règle précise :
  → Protocole : TCP, UDP, ICMP, ou All
  → Port(s)   : 80, 443, 22, ou une plage
  → Source    : IP/CIDR, ou un autre Security Group
```

### Référencer un Security Group comme source

C'est le **pattern le plus puissant et recommandé** des Security Groups AWS :

```
Au lieu de mettre une plage IP comme source,
tu mets l'ID d'un autre Security Group.

Exemple :
  SG-ALB       : attaché au Load Balancer
  SG-App       : attaché aux EC2 app
  SG-DB        : attaché à la RDS

  Règle sur SG-App :
    Inbound port 3000 depuis SG-ALB
    → seules les ressources avec SG-ALB peuvent joindre l'app
    → si tu ajoutes/supprimes une EC2 dans le SG-ALB,
      la règle s'adapte automatiquement ✅
    → pas besoin de gérer des plages d'IPs ✅

  Règle sur SG-DB :
    Inbound port 5432 depuis SG-App
    → seules les EC2 app peuvent joindre la RDS
    → la RDS n'est jamais accessible depuis Internet ✅
```

### Architecture Security Groups — pattern standard

```
Internet
    │
    │  443/TCP depuis 0.0.0.0/0
    ▼
[ALB]  ←── SG-ALB
    │
    │  3000/TCP depuis SG-ALB
    ▼
[EC2 App]  ←── SG-App
    │
    │  5432/TCP depuis SG-App
    ▼
[RDS]  ←── SG-DB

Chaque flèche = une règle inbound dans le SG de destination
Aucune ressource privée n'est accessible depuis Internet ✅
```

### Limites des Security Groups

```
→ Maximum 5 SG par interface réseau (ENI)
→ Maximum 60 règles inbound + 60 outbound par SG
→ Un SG est régional — ne peut pas couvrir plusieurs régions
→ S'applique à la ressource, pas au subnet
  (différence avec les NACLs)
```

---

## 8. Network ACLs

### Pourquoi c'est important

Les Network ACLs (NACLs) sont la deuxième couche de sécurité réseau dans AWS — au niveau du subnet, avant que le trafic atteigne les Security Groups. Elles sont **stateless** et fonctionnent comme un filtre de dernier recours.

### NACLs vs Security Groups

```
                    Security Group      Network ACL
Niveau             Ressource (ENI)      Subnet
Stateful/stateless Stateful ✅          Stateless ⚠️
Règles allow/deny  Allow uniquement     Allow ET Deny
Ordre des règles   Toutes évaluées      Numéro croissant (s'arrête à la 1ère match)
Par défaut         Deny tout entrant    Allow tout (NACL par défaut)
                   Allow tout sortant
```

### Stateless — la conséquence importante

```
Avec un NACL stateless :
  Si tu autorises le trafic entrant sur le port 443,
  tu DOIS AUSSI autoriser le trafic sortant
  sur les ports éphémères (1024-65535) pour les réponses.

Règles NACL pour autoriser HTTPS entrant :
  Inbound  : Allow TCP 443 depuis 0.0.0.0/0
  Outbound : Allow TCP 1024-65535 vers 0.0.0.0/0
  ← sans cette règle outbound, les réponses sont bloquées ⚠️
```

### Quand utiliser les NACLs

```
Cas d'usage typiques :
  → Bloquer une IP malveillante rapidement
    (plus rapide que modifier tous les SG)
  → Règle Deny explicite — impossible avec les SG
    (les SG n'ont que des Allow)
  → Conformité / audit : couche de sécurité supplémentaire

En pratique :
  → La plupart des architectures AWS utilisent principalement
    les Security Groups et laissent les NACLs en configuration
    par défaut (Allow all)
  → Les NACLs deviennent importantes pour les réglementations
    strictes (PCI-DSS, HIPAA, etc.)
```

---

## 9. VPC Flow Logs

### Pourquoi c'est important

Quand un service ne répond pas, quand tu suspectes une attaque, quand tu dois auditer les accès réseau — les Flow Logs sont ton `tcpdump` dans le Cloud. Ils capturent les métadonnées de tout le trafic qui circule dans ton VPC et te permettent de comprendre ce qui se passe réellement sur ton réseau.

### Ce que capturent les Flow Logs

Les Flow Logs ne capturent pas le **contenu** des paquets (comme tcpdump) — ils capturent les **métadonnées** :

```
Version  AccountID  InterfaceID  SrcAddr      DstAddr      SrcPort  DstPort  Protocol  Packets  Bytes   Start       End         Action  Status
2        123456789  eni-abc123   10.0.1.42    10.0.20.5    52341    5432     6         10       1240    1697123456  1697123516  ACCEPT  OK
2        123456789  eni-abc123   203.0.113.5  10.0.1.10    45123    22       6         5        300     1697123456  1697123476  REJECT  OK

Chaque ligne = une connexion réseau avec :
  → IPs source et destination
  → Ports source et destination
  → Protocole (6=TCP, 17=UDP, 1=ICMP)
  → Nombre de paquets et octets
  → Action : ACCEPT ou REJECT
```

### Niveaux de capture

```
Flow Logs peuvent être activés à trois niveaux :

VPC entier    → tout le trafic de tous les subnets et ENIs
Subnet        → tout le trafic d'un subnet spécifique
ENI           → tout le trafic d'une interface réseau spécifique
```

### Destinations des Flow Logs

```
→ CloudWatch Logs : requêtes et alertes en temps réel
→ S3 : archivage long terme, analyse avec Athena
→ Kinesis Data Firehose : streaming vers des outils tiers

Exemple d'utilisation avec Athena (requête SQL sur les logs S3) :
  "Montre-moi toutes les connexions REJECT des dernières 24h"
  "Quelle EC2 a le plus de trafic sortant ?"
  "Y a-t-il des tentatives de connexion sur le port 22 ?"
```

### Flow Logs dans la pratique de debug

```
Problème : "mon EC2 ne peut pas joindre la RDS"

Sans Flow Logs : tu testes au hasard, tu modifies des SG...

Avec Flow Logs :
  Tu cherches les flux entre l'IP de l'EC2 et l'IP de la RDS
  → Si tu vois REJECT → c'est un SG ou NACL qui bloque
  → Si tu ne vois rien → le paquet ne part même pas
    (problème de routage ou de config réseau sur l'EC2)
  → Si tu vois ACCEPT → le problème est dans l'application,
    pas dans le réseau
```

---

## Partie 4 — Connectivité avancée

## 10. VPC Peering

### Pourquoi c'est important

Par défaut, deux VPC sont complètement isolés l'un de l'autre — même s'ils appartiennent au même compte AWS. Le **VPC Peering** crée une connexion réseau directe entre deux VPC, permettant à leurs ressources de communiquer comme si elles étaient sur le même réseau.

### Comment fonctionne le VPC Peering

```
VPC A (10.0.0.0/16)  ←──── Peering ────►  VPC B (10.1.0.0/16)
  EC2: 10.0.1.42                             RDS: 10.1.20.5

EC2 dans VPC A peut joindre RDS dans VPC B directement
→ trafic passe par l'infrastructure AWS (privé, pas Internet)
→ faible latence, pas de coût de bande passante supplémentaire
  (sauf si cross-région)
```

### Configurer le peering

```
3 étapes nécessaires :

1. Créer la connexion de peering
   VPC A envoie une demande → VPC B accepte

2. Mettre à jour les Route Tables des DEUX VPC
   Route Table VPC A : 10.1.0.0/16 → pcx-xxxxxxxx (peering)
   Route Table VPC B : 10.0.0.0/16 → pcx-xxxxxxxx (peering)

3. Mettre à jour les Security Groups
   SG de la RDS dans VPC B :
     Inbound 5432 depuis 10.0.0.0/16 (CIDR du VPC A)
```

### Limitations importantes

```
→ Non transitif : A↔B et B↔C ne signifie PAS A↔C
  Pour A↔C, il faut créer un peering A↔C séparé

→ Pas de chevauchement CIDR : les deux VPC doivent avoir
  des plages d'IPs différentes — d'où l'importance de bien
  planifier ses CIDRs dès le début

→ Maximum 125 peerings actifs par VPC

→ Cross-région possible mais avec coût de transfert
```

### Quand utiliser le peering

```
✅ Communication entre environnements (prod ↔ monitoring)
✅ Architecture multi-comptes (compte prod ↔ compte shared services)
✅ Nombre de VPC limité (< 10)

❌ Nombreux VPC à connecter → utiliser Transit Gateway
❌ Besoin de transitivité → utiliser Transit Gateway
```

---

## 11. PrivateLink et VPC Endpoints

### Pourquoi c'est important

Par défaut, quand une EC2 dans un subnet privé accède à S3, le trafic passe par Internet (via la NAT Gateway). C'est sous-optimal — tu paies des coûts de NAT, le trafic sort de ton réseau AWS, et il y a une latence supplémentaire. **VPC Endpoints** et **PrivateLink** résolvent ça en gardant le trafic dans le réseau AWS.

### Gateway Endpoints — S3 et DynamoDB

Les **Gateway Endpoints** sont des endpoints spéciaux pour S3 et DynamoDB. Gratuits, ils ajoutent une entrée dans la Route Table qui redirige le trafic vers ces services directement dans le réseau AWS.

```
SANS Gateway Endpoint :
  EC2 (subnet privé)
    → NAT Gateway
    → Internet
    → S3 (s3.amazonaws.com)
  Coût : transfert NAT + latence Internet

AVEC Gateway Endpoint :
  EC2 (subnet privé)
    → Gateway Endpoint
    → S3 directement dans le réseau AWS
  Coût : gratuit + latence minimale ✅

Configuration :
  Route Table subnet privé :
    pl-xxxxxx (S3) → vpce-xxxxxxxx
```

### Interface Endpoints (PrivateLink)

Les **Interface Endpoints** (basés sur PrivateLink) créent une ENI dans ton subnet avec une IP privée. Cette ENI représente le service AWS dans ton réseau.

```
Services supportés (exemples) :
  → EC2 API, SSM, Secrets Manager
  → CloudWatch, CloudTrail
  → STS, KMS
  → Et des centaines d'autres...

SANS Interface Endpoint :
  EC2 → NAT GW → Internet → secretsmanager.amazonaws.com

AVEC Interface Endpoint :
  EC2 → ENI privée (10.0.1.99) → Secrets Manager
  Trafic reste dans AWS, jamais sur Internet ✅

Coût : ~$0.01/heure par AZ + $0.01/Go
```

### PrivateLink — exposer ses propres services

PrivateLink permet aussi d'exposer un **service interne** à d'autres VPC ou clients AWS, sans VPC Peering, sans exposer l'infrastructure :

```
Ton service (dans ton VPC)
    │
    │ NLB devant ton service
    │
    └── PrivateLink Endpoint Service

Clients (dans leur VPC)
    │
    └── Interface Endpoint → ton service
        Ils accèdent à ton service par une IP privée
        dans leur propre VPC, sans voir ton infrastructure ✅
```

---

## 12. Transit Gateway, VPN et Direct Connect

### Transit Gateway — le hub réseau central

Quand tu as de nombreux VPC à connecter, le peering devient ingérable (N*(N-1)/2 connexions). Le **Transit Gateway** est un hub central auquel tu connectes tous tes VPC et réseaux on-premise.

```
SANS Transit Gateway (5 VPC = 10 peerings) :
  VPC-A ↔ VPC-B ↔ VPC-C ↔ VPC-D ↔ VPC-E
  + VPC-A↔C, A↔D, A↔E, B↔D, B↔E, C↔E...

AVEC Transit Gateway :
  VPC-A ──┐
  VPC-B ──┤
  VPC-C ──┼── Transit Gateway ── On-premise
  VPC-D ──┤
  VPC-E ──┘
  5 connexions au lieu de 10, et transitif ✅
```

### VPN Gateway — connexion sécurisée vers AWS

Un **VPN Gateway** crée un tunnel VPN IPsec chiffré entre ton réseau on-premise (bureau, datacenter) et ton VPC AWS.

```
Ton datacenter ──── VPN IPsec ────► VPC AWS
(Customer Gateway)               (Virtual Private Gateway)

Cas d'usage : migration progressive vers le Cloud,
              connexion d'une filiale à l'infrastructure AWS
Bande passante : jusqu'à 1.25 Gbps par tunnel
Coût : ~$36/mois + transfert de données
```

### Direct Connect — fibre dédiée vers AWS

**Direct Connect** est une connexion réseau physique dédiée entre ton datacenter et AWS — sans passer par Internet.

```
Ton datacenter ──── Fibre dédiée ────► AWS Direct Connect Location ────► VPC
                    (1G ou 10G)

Avantages vs VPN :
  → Bande passante garantie (1G, 10G, 100G)
  → Latence prévisible et faible
  → Pas de fluctuations Internet
  → Coût de transfert réduit

Cas d'usage : grandes entreprises avec volumes de données
              importants, applications sensibles à la latence
Coût : élevé (port + partenaire + transfert)
```

---

## Partie 5 — Services réseau managés

## 13. ALB et NLB

### Pourquoi c'est important

On a vu les Load Balancers en théorie dans le Module 1 (couches OSI). Dans ce module, on voit leur configuration concrète dans AWS — les composants qui les constituent et comment ils distribuent le trafic vers tes ressources.

### Les composants d'un Load Balancer AWS

```
Load Balancer AWS = 3 composants :

1. Listener
   → écoute sur un port/protocole (ex: TCP 443)
   → reçoit les connexions entrantes

2. Rules (règles)
   → conditions de routage
   → ex: /api/* → Target Group API
         /static/* → Target Group Static

3. Target Group
   → ensemble de ressources qui reçoivent le trafic
   → EC2, conteneurs ECS, Lambdas, IPs
   → health checks pour retirer les cibles défaillantes
```

### ALB — Application Load Balancer (couche 7)

```
L'ALB lit le contenu HTTP — il peut router selon :
  → l'URL : /api/* → serveurs API, /images/* → serveurs statiques
  → les headers HTTP : Host, User-Agent, etc.
  → les cookies : sticky sessions
  → la méthode HTTP : GET, POST...

Fonctionnalités supplémentaires :
  → SSL termination (certificat TLS géré par ACM)
  → HTTP/2 et WebSocket
  → Authentification (Cognito, OIDC)
  → Intégration WAF

Cas d'usage : applications web, APIs REST, microservices
```

### NLB — Network Load Balancer (couche 4)

```
Le NLB ne lit que TCP/UDP — il route selon :
  → l'IP source
  → le port
  → le protocole

Caractéristiques :
  → ultra-faible latence (< 1ms)
  → millions de requêtes par seconde
  → IP fixe par AZ (utile pour les whitelists)
  → préserve l'IP source du client

Cas d'usage : jeux en ligne, streaming, IoT,
              applications temps réel, bases de données
```

### Health Checks — détecter les cibles défaillantes

```
Chaque Target Group a des health checks configurés :
  → Protocole : HTTP, HTTPS, TCP
  → Path      : /health, /ping
  → Seuil     : 3 succès consécutifs → healthy
                2 échecs consécutifs → unhealthy

Si une cible est unhealthy :
  → le Load Balancer arrête d'envoyer du trafic vers elle
  → automatiquement, sans intervention manuelle
  → quand elle redevient healthy → trafic reprend ✅
```

---

## 14. Route 53

### Pourquoi c'est important

Route 53 est le service DNS d'AWS. Il fait bien plus que résoudre des noms — il permet de contrôler intelligemment où le trafic est dirigé : vers quelle région, vers quelle ressource, en fonction de la santé des services et de la localisation des utilisateurs.

### Hosted Zones

```
Hosted Zone publique :
  → gère le DNS pour un domaine public (ex: monapp.com)
  → accessible depuis Internet
  → enregistrements A, CNAME, MX, TXT...

Hosted Zone privée :
  → gère le DNS pour des noms internes dans un VPC
  → ex: database.prod.internal → 10.0.20.5
  → accessible uniquement depuis les VPC associés
  → les ressources se parlent par nom, jamais par IP ✅
```

### Routing Policies — diriger le trafic intelligemment

```
Simple :
  monapp.com → 54.12.34.56
  Basique, pas de logique

Weighted (pondéré) :
  monapp.com → EC2-v1 (90%) + EC2-v2 (10%)
  Utile pour les déploiements progressifs (blue/green, canary)

Latency-based :
  monapp.com → région eu-west-1 (pour utilisateurs Europe)
              → région us-east-1 (pour utilisateurs Amérique)
  Route vers la région avec la latence la plus faible

Failover :
  monapp.com → Primary (EC2 principale)
              → Secondary (EC2 de secours si primary HS)
  Bascule automatique en cas de panne

Geolocation :
  monapp.com → serveurs Europe (pour IPs européennes)
              → serveurs USA (pour IPs américaines)
  Conformité RGPD : garder les données des EU en EU

Health Checks :
  Route 53 vérifie périodiquement si tes endpoints répondent
  → retire automatiquement les endpoints défaillants
  → combinable avec toutes les routing policies
```

### Alias Records — spécificité AWS

```
Un enregistrement Alias est une extension AWS du DNS standard.
Il permet de pointer directement vers une ressource AWS
(ALB, CloudFront, S3 website, etc.) sans connaître son IP.

Avantage vs CNAME :
  → CNAME ne peut pas être à la racine du domaine (apex)
  → Alias peut : monapp.com → ALB ✅ (pas juste www.monapp.com)
  → Gratuit (pas de facturation pour les requêtes Alias)
```

---

## 15. DHCP Option Sets

### Pourquoi c'est important

On a vu DHCP dans le Module 1 et NTP dans le Module 2. Sur AWS, le **DHCP Option Set** est l'équivalent de la configuration DHCP de ta Fritz!Box — il définit les paramètres réseau que chaque EC2 reçoit automatiquement au démarrage : serveur DNS, nom de domaine, serveurs NTP.

### Ce qu'est un DHCP Option Set

```
Chaque VPC a un DHCP Option Set associé.
Quand une EC2 démarre, elle fait une requête DHCP
et reçoit automatiquement :

  domain-name-servers : 10.0.0.2 (DNS AWS = VPC CIDR + 2)
  domain-name         : eu-central-1.compute.internal
  ntp-servers         : 169.254.169.123 (Amazon Time Sync)
```

### Personnaliser le DHCP Option Set

```
Cas d'usage en entreprise :
  → Utiliser ses propres serveurs DNS internes
    (Active Directory, Bind9 on-premise)
  → Utiliser ses propres serveurs NTP
  → Définir un domaine DNS interne personnalisé
    ex: corp.monentreprise.com au lieu de .internal

Configuration :
  domain-name-servers : 10.0.0.2, 192.168.1.10 (DNS on-premise)
  domain-name         : corp.monentreprise.com
  ntp-servers         : 169.254.169.123

Important :
  → Un seul DHCP Option Set par VPC
  → Modification prend effet au prochain renouvellement DHCP
    (ou redémarrage de l'instance)
  → Sauvegarder la config avant de modifier !
```

---

## 16. AWS Network Firewall et Global Accelerator

### AWS Network Firewall — filtrage avancé au niveau VPC

```
AWS Network Firewall est un firewall managé qui s'insère
dans le flux réseau du VPC — avant même d'atteindre
les Security Groups.

Capacités (au-delà des Security Groups) :
  → Inspection stateful des paquets (Deep Packet Inspection)
  → Filtrage par domaine : bloquer *.malware.com
  → Règles IDS/IPS (détection d'intrusion)
  → Filtering du trafic chiffré TLS (avec décryptage)

Cas d'usage :
  → Conformité stricte (PCI-DSS, HIPAA)
  → Bloquer l'accès à des domaines malveillants
  → Audit détaillé du trafic réseau

Coût : élevé (~$400/mois par endpoint)
→ réservé aux architectures enterprise avec exigences strictes
```

### Global Accelerator — accélération réseau mondiale

```
Global Accelerator optimise le routage du trafic
en utilisant le réseau backbone privé d'AWS
au lieu d'Internet public.

Sans Global Accelerator :
  Utilisateur (Tokyo) → Internet public → ALB (Frankfurt)
  → nombreux sauts, latence variable

Avec Global Accelerator :
  Utilisateur (Tokyo) → Edge AWS le plus proche (Tokyo)
  → backbone privé AWS → ALB (Frankfurt)
  → latence réduite et stable

Cas d'usage :
  → Applications gaming avec joueurs mondiaux
  → Applications financières sensibles à la latence
  → Failover multi-région ultra-rapide (< 30 secondes)

Coût : ~$18/mois par accelerator + $0.01/Go
```

---

## Récapitulatif du Module 4

| Concept | Ce qu'il faut retenir |
|---------|----------------------|
| **VPC** | Réseau privé isolé dans AWS. Un VPC par environnement. CIDR à bien planifier. |
| **Subnet** | Subdivision du VPC dans une AZ. Public = route vers IGW. Privé = route vers NAT GW. |
| **AZ** | Datacenter physiquement séparé. Toujours déployer sur 2+ AZ pour la résilience. |
| **Route Table** | Définit où va le trafic. C'est elle qui fait la différence public/privé. |
| **ENI** | Interface réseau virtuelle d'une EC2. Porte l'IP, la MAC et le Security Group. |
| **Internet Gateway** | Relie le VPC à Internet. Une par VPC. Fait le NAT pour les IPs publiques. |
| **NAT Gateway** | Sortie Internet pour les subnets privés. Une par AZ pour la résilience. |
| **Elastic IP** | IP publique fixe. Gratuite si utilisée, payante si réservée et inutilisée. |
| **Security Group** | Firewall stateful par ressource. Référencer un SG comme source = pattern clé. |
| **Network ACL** | Firewall stateless par subnet. Stateless = rules inbound ET outbound nécessaires. |
| **Flow Logs** | Capture les métadonnées du trafic réseau. Équivalent Cloud de tcpdump. |
| **VPC Peering** | Connexion directe entre deux VPC. Non transitif. Pas de CIDR overlap. |
| **Gateway Endpoint** | Accès privé S3/DynamoDB sans passer par Internet. Gratuit. |
| **Interface Endpoint** | Accès privé aux services AWS via ENI privée. Basé sur PrivateLink. |
| **Transit Gateway** | Hub central pour connecter de nombreux VPC. Transitif. |
| **VPN Gateway** | Tunnel IPsec chiffré vers un réseau on-premise. |
| **Direct Connect** | Fibre dédiée vers AWS. Bande passante garantie, faible latence. |
| **ALB** | Load Balancer couche 7. Route selon l'URL/headers. SSL termination. |
| **NLB** | Load Balancer couche 4. Ultra-rapide. IP fixe. Préserve l'IP source. |
| **Route 53** | DNS AWS. Routing policies : weighted, latency, failover, geolocation. |
| **DHCP Option Set** | Configure DNS et NTP pour tout le VPC. |

---

*→ Exercices pratiques : voir `module4-exercices.md`*
*→ Module suivant : Module 5 — Kubernetes Networking*
