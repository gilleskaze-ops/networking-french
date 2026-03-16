# Networking Fondamentaux — Orienté Cloud
### Cours complet · 6 Modules · Du réseau de base à l'Infrastructure as Code

---

## À propos de ce cours

Ce cours a été construit de façon interactive et progressive, en partant des fondamentaux absolus du réseau jusqu'à l'infrastructure Cloud en production codée avec Terraform.

**Philosophie** : comprendre le *pourquoi* avant le *comment*. Chaque concept est ancré dans un cas d'usage réel Cloud avant d'être expliqué techniquement. Chaque module contient un glossaire complet avec tous les acronymes développés et définis en français.

**Public cible** : ingénieurs en reconversion vers le Cloud, développeurs qui veulent comprendre l'infrastructure, futurs Solutions Architects.

---

## Structure du cours

```
module1-theorie.md      + module1-exercices.md
module2-theorie.md      + module2-exercices.md
module3-theorie.md      + module3-exercices.md
module4-theorie.md      + module4-exercices.md
module5-theorie.md      + module5-exercices.md
module6-theorie.md      + module6-exercices.md
```

Chaque module est divisé en deux fichiers séparés :
- **Théorie** : cours complet avec glossaire, exemples annotés, schémas texte, liens Cloud
- **Exercices** : exercices pratiques avec outils réels, réponses cachées, quiz final

---

## Les 6 modules

### Module 1 — Les bases absolues du réseau
> *Prérequis : aucun · Durée : 2h30*

Le fondement de tout. Sans ces bases, rien de ce qui suit n'a de sens.

```
Contenu théorie (15 sections) :
  → C'est quoi un réseau (LAN, WAN)
  → Adresses IP (IPv4, IPv6, classes IP, privé vs public)
  → Masques de sous-réseau
  → CIDR — notation universelle dans le Cloud
  → TCP et UDP — comment les données voyagent
  → DNS — comment les noms deviennent des adresses
  → HTTP, HTTPS et les ports
  → DHCP — attribution automatique des IPs
  → ICMP — tester si une machine est joignable
  → Modèle OSI et TCP/IP — les couches réseau
  → NTP — synchroniser les horloges (indispensable pour les logs Cloud)
  → NAT — comment les IPs privées accèdent à Internet
  → IPv6 — le successeur d'IPv4
  → TLS — comment fonctionne le chiffrement
  → Proxy et Reverse Proxy

Exercices (14 exercices + quiz 16 questions) :
  → dig, curl, openssl, ping, /etc/hosts
  → Calcul CIDR, conception plan d'adressage VPC
  → Inspection certificats TLS
  → Configuration nginx reverse proxy
  → Observer NAT, DHCP, OSI en pratique
```

---

### Module 2 — Le réseau dans Linux
> *Prérequis : Module 1 · Durée : 3h*

Linux est l'OS de tout serveur Cloud. Maîtriser ses outils réseau, c'est maîtriser le diagnostic en production.

```
Contenu théorie (9 sections) :
  → Le voyage d'un paquet de bout en bout
    (ARP, Switch, Default Gateway, NAT, hops sur Internet)
  → Les interfaces réseau (eth0, wlan0, lo, ENI AWS)
  → La table de routage (ip route, Default Gateway, Route Tables AWS)
  → Commandes CLI essentielles
    (ip, ping, traceroute, ss, curl, netstat)
  → Le firewall sous Linux (iptables, ufw, Security Groups AWS)
  → Outils d'analyse avancés (Wireshark, tcpdump, mtr, netcat)
  → Configuration DNS locale (/etc/hosts, /etc/resolv.conf)
  → Ports éphémères — comment l'OS les gère
  → NTP en pratique (timedatectl, chronyc)
  → nmap — scanner et auditer un réseau
  → SSH avancé (tunneling, port forwarding, bastion, clés)

Exercices (17 exercices + quiz 15 questions) :
  → ip, ss, traceroute, mtr, netcat, nmap
  → Wireshark + tcpdump (captures réseau réelles)
  → ufw (règles firewall)
  → SSH tunneling et bastion
  → Diagnostic réseau méthodique étape par étape

Outils installés :
  sudo apt install wireshark tcpdump mtr netcat-openbsd nmap traceroute
```

---

### Module 3 — Docker Networking
> *Prérequis : Modules 1 et 2 · Docker installé · Durée : 2h30*

> **Note pédagogique** : dans le monde réel, l'ordre est AWS → Docker → Kubernetes. On a commencé par Docker car il se pratique immédiatement en local sans compte AWS. Tous les concepts Docker (bridge, NAT, DNS, ports) ont leurs équivalents directs dans AWS.

```
Contenu théorie (7 sections) :
  → Docker en bref — conteneur vs VM, contexte réseau
  → Comment Docker gère le réseau
    (docker0, paires veth, NAT via iptables)
  → Les modes réseau (bridge, host, none, overlay, macvlan)
  → DNS dans Docker — pourquoi les réseaux personnalisés
  → Docker Compose networking — service discovery par nom
  → Ports et exposition (-p, EXPOSE, 0.0.0.0 vs 127.0.0.1)
  → Debug réseau dans les conteneurs

Exercices (8 exercices + quiz 12 questions) :
  → docker network create/inspect/connect
  → docker exec + nicolaka/netshoot
  → Docker Compose multi-réseaux (isolation par couche)
  → tcpdump dans les conteneurs
  → nmap sur les conteneurs
  → Debug architecture cassée (méthode complète)
  → Architecture complète avec monitoring réseau

Image de debug indispensable :
  docker run --rm --network mon-reseau nicolaka/netshoot
```

---

### Module 4 — AWS Networking
> *Prérequis : Modules 1 à 3 · Compte AWS · Durée : 3h*

Le cœur du métier Solutions Architect. Toute l'infrastructure Cloud repose sur ces concepts.

```
Contenu théorie (16 sections en 5 parties) :

  PARTIE 1 — Les fondations
  → VPC (CIDR, par défaut vs personnalisé, planification)
  → Subnets, AZ et résilience (architecture multi-AZ standard)
  → Route Tables (local, priorité, public vs privé)
  → ENI (IP privée/publique/EIP, multi-ENI)

  PARTIE 2 — Connectivité Internet
  → Internet Gateway (3 conditions simultanées)
  → NAT Gateway + Elastic IP (multi-AZ, coûts)

  PARTIE 3 — Sécurité réseau
  → Security Groups (stateful, pattern SG→SG, architecture en couches)
  → Network ACLs (stateless, vs SG)
  → VPC Flow Logs (équivalent Cloud de tcpdump)

  PARTIE 4 — Connectivité avancée
  → VPC Peering (non transitif, limitations)
  → PrivateLink + VPC Endpoints (Gateway S3 gratuit, Interface payant)
  → Transit Gateway, VPN, Direct Connect (mention)

  PARTIE 5 — Services réseau managés
  → ALB + NLB (composants, health checks, couches OSI)
  → Route 53 (hosted zones, routing policies, Alias records)
  → DHCP Option Sets (lien Module 1/2)
  → AWS Network Firewall + Global Accelerator (mention)

Exercices (6 exercices + quiz 12 questions) :
  → Création VPC complet (console + AWS CLI)
  → Architecture Security Groups en couches
  → Route Tables public/privé
  → Flow Logs + analyse CloudWatch
  → VPC Peering entre deux VPC
  → Route 53 DNS privé + weighted routing
```

---

### Module 5 — Kubernetes Networking
> *Prérequis : Modules 1 à 4 · Minikube installé · Durée : 3h*

K8s a son propre modèle réseau — flat network, Services, Ingress. Indispensable pour les architectures conteneurisées modernes.

```
Contenu théorie (8 sections) :
  → Le modèle réseau K8s (4 règles, différence avec Docker)
  → Les Pods et leurs IPs (éphémérité, namespace réseau partagé)
  → Les Services
    (ClusterIP, NodePort, LoadBalancer, ExternalName, kube-proxy)
  → DNS dans Kubernetes
    (CoreDNS, format service.namespace.svc.cluster.local)
  → Ingress (vs LoadBalancer, Ingress Controllers, routing, TLS)
  → Network Policies (tout ouvert par défaut, pattern 3-tiers)
  → CNI Plugins (Flannel, Calico, Weave, AWS VPC CNI)
  → K8s sur AWS/EKS
    (VPC CNI, dimensionnement subnets, AWS LB Controller,
     Security Groups par pod, architecture complète)

Exercices (6 exercices + quiz 12 questions) :
  → kubectl get pods -o wide (IPs, éphémérité)
  → Services ClusterIP/NodePort + endpoints + iptables
  → DNS K8s cross-namespace avec CoreDNS
  → Ingress routing path + hostname avec nginx
  → Network Policies architecture 3-tiers
  → Debug complet (bug intentionnel selector)

Outils K8s :
  minikube start --cni=calico
  kubectl, nicolaka/netshoot, kubectl port-forward
  minikube addons enable ingress
```

---

### Module 6 — Networking as Code avec Terraform
> *Prérequis : Modules 1 à 5 · Terraform installé · Compte AWS · Durée : 3h*

Coder l'infrastructure réseau AWS — versionnable, reproductible, partageable en équipe.

```
Contenu théorie (10 sections) :
  → Terraform en bref orienté réseau
    (providers, resources, state, variables, outputs, locals)
  → VPC et Subnets (aws_vpc, aws_subnet, count, cidrsubnet)
  → Connectivité Internet (IGW, NAT GW, EIP, Route Tables)
  → Security Groups (règles statiques, SG→SG, for_each dynamique)
  → Modules Terraform réseau
    (module VPC réutilisable, single_nat_gateway, multi-env)
  → Route 53 et DNS (zone privée, Alias records, weighted routing)
  → ALB en Terraform (SG + LB + Target Group + Listeners)
  → VPC Peering en Terraform
  → Outputs et Remote State (S3 + DynamoDB, partage inter-équipes)
  → Outils et bonnes pratiques
    (terraform fmt/validate/plan, tflint, Terragrunt mention)

Exercices (5 exercices + quiz 10 questions) :
  → Premier VPC avec cidrsubnet et data sources
  → Security Groups en couches avec référencement SG→SG
  → Module VPC réutilisable (staging vs production)
  → Route 53 zone privée + blue/green weighted routing
  → Remote State S3 + partage networking/application

Outils Terraform :
  terraform init / fmt / validate / plan / apply / destroy
  terraform state list / show / import
  tflint
```

---

## Correspondances entre modules

Ce cours est conçu pour que chaque concept se retrouve à plusieurs niveaux :

| Concept | Module 1 | Module 2 | Module 3 | Module 4 | Module 5 | Module 6 |
|---------|----------|----------|----------|----------|----------|----------|
| **IP / Réseau** | ✅ Bases | ✅ ip addr | ✅ docker0 | ✅ VPC/ENI | ✅ Pod IPs | ✅ aws_vpc |
| **DNS** | ✅ Bases | ✅ resolv.conf | ✅ 127.0.0.11 | ✅ Route 53 | ✅ CoreDNS | ✅ aws_route53 |
| **Firewall** | ✅ Bases | ✅ ufw/iptables | ✅ docker network | ✅ Security Groups | ✅ Network Policies | ✅ aws_sg |
| **NAT** | ✅ Bases | ✅ table NAT | ✅ NAT Docker | ✅ NAT Gateway | ✅ VPC CNI | ✅ aws_nat_gateway |
| **Load Balancer** | ✅ Reverse Proxy | — | ✅ ports -p | ✅ ALB/NLB | ✅ Ingress | ✅ aws_lb |
| **Debug réseau** | ✅ dig/curl | ✅ tcpdump/nmap | ✅ netshoot | ✅ Flow Logs | ✅ kubectl exec | ✅ terraform plan |

---

## Outils par module — vue d'ensemble

```
MODULE 1  → curl, dig, ping, openssl, nginx
MODULE 2  → ip, ss, traceroute, mtr, netcat, nmap,
             tcpdump, Wireshark, ufw, ssh,
             timedatectl, resolvectl
MODULE 3  → docker, docker compose, nicolaka/netshoot
MODULE 4  → aws cli, console AWS
MODULE 5  → kubectl, minikube, nicolaka/netshoot
MODULE 6  → terraform, tflint, aws cli
```

---

## Prérequis d'installation

```bash
# Module 1 & 2 — outils réseau Linux
sudo apt update
sudo apt install -y \
  curl wget dig nmap traceroute mtr \
  tcpdump wireshark netcat-openbsd \
  ufw openssh-client openssh-server \
  nginx chrony tflint

# Ajouter son user au groupe wireshark
sudo usermod -aG wireshark $USER

# Module 3 — Docker
# https://docs.docker.com/engine/install/ubuntu/
docker --version
docker compose version

# Image de debug réseau
docker pull nicolaka/netshoot

# Module 4 — AWS CLI
# https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html
aws --version
aws configure

# Module 5 — Kubernetes local
# https://minikube.sigs.k8s.io/docs/start/
minikube version
kubectl version

# Module 6 — Terraform
# https://developer.hashicorp.com/terraform/install
terraform version

# tflint
curl -s https://raw.githubusercontent.com/terraform-linters/tflint/master/install_linux.sh | bash
```

---

## Parcours de certification recommandé

Ce cours couvre la partie réseau de ces certifications :

```
1. AWS Solutions Architect Associate (SAA-C03)
   Modules couverts : 1, 2, 4, 6
   Modules complémentaires à préparer : EC2, S3, IAM, RDS, Lambda

2. Terraform Associate (003)
   Modules couverts : 6
   Compléments : state avancé, providers, testing

3. CKA — Certified Kubernetes Administrator
   Modules couverts : 1, 2, 5
   Compléments : cluster administration, storage, security

4. IHK Cloud Computing
   Modules couverts : 1 à 6 (complet)
```

---

## Concepts à approfondir dans d'autres chats

Ce cours couvre uniquement le **Networking**. Pour être Solutions Architect complet :

```
Compute & Serverless
  → EC2, Auto Scaling Groups, Lambda, ECS/EKS

Storage
  → S3, EBS, EFS, Glacier, Storage Gateway

Databases
  → RDS, Aurora, DynamoDB, ElastiCache, Redshift

Security & Identity
  → IAM, KMS, Secrets Manager, CloudTrail, GuardDuty

Observability
  → CloudWatch, X-Ray, OpenTelemetry, Grafana

Architecture Patterns
  → Well-Architected Framework (5 piliers)
  → High Availability, Disaster Recovery
  → Microservices, Event-driven, Serverless

CI/CD
  → GitHub Actions, CodePipeline, ArgoCD, GitOps

Cost Management
  → Cost Explorer, Budgets, Savings Plans
```

---

## Progression suggérée

```
Semaine 1-2  →  Module 1 (théorie + tous les exercices)
Semaine 3-4  →  Module 2 (théorie + tous les exercices)
Semaine 5    →  Module 3 (théorie + tous les exercices)
Semaine 6-7  →  Module 4 (théorie + tous les exercices)
Semaine 8    →  Module 5 (théorie + tous les exercices)
Semaine 9-10 →  Module 6 (théorie + tous les exercices)

Total : ~10 semaines à raison de 1h/jour
```

**Conseil** : ne pas enchaîner les modules sans faire les exercices. La théorie sans pratique s'oublie en 48h. La pratique sans théorie ne donne pas de compréhension profonde. Les deux ensemble construisent une expertise durable.

---

*Cours construit de façon interactive — les modules peuvent être enrichis au fur et à mesure de la progression et des questions qui émergent naturellement lors de l'apprentissage.*
