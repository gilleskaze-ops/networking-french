# Module 5 — Kubernetes Networking
### Partie Théorie

> **Cours Networking Fondamentaux · Orienté Cloud**
> Prérequis : Modules 1 à 4 complétés · Minikube installé · Durée estimée : 3h

---

## Glossaire

| Terme | Signification | Définition simple |
|-------|--------------|-------------------|
| **CNI** | Container Network Interface | Standard qui définit comment les plugins réseau connectent les pods |
| **ClusterIP** | — | Type de Service K8s — IP virtuelle interne, accessible uniquement dans le cluster |
| **CoreDNS** | — | Serveur DNS interne de Kubernetes — résout les noms de services en IPs |
| **DaemonSet** | — | Ressource K8s qui déploie un pod sur chaque nœud du cluster |
| **Deployment** | — | Ressource K8s qui gère le déploiement et la mise à l'échelle de pods |
| **EKS** | Elastic Kubernetes Service | Service AWS de Kubernetes managé |
| **Ingress** | — | Règles de routage HTTP/HTTPS vers des Services K8s |
| **Ingress Controller** | — | Composant qui implémente les règles Ingress (nginx, traefik, ALB...) |
| **kube-proxy** | — | Composant K8s qui maintient les règles réseau (iptables) sur chaque nœud |
| **Namespace** | — | Isolation logique dans un cluster K8s — comme des dossiers pour les ressources |
| **Network Policy** | — | Règles qui contrôlent quel trafic est autorisé entre les pods |
| **Node** | Nœud | Machine (VM ou physique) qui fait partie du cluster K8s |
| **NodePort** | — | Type de Service K8s — expose un port sur chaque nœud du cluster |
| **Pod** | — | Unité de base K8s — un ou plusieurs conteneurs partageant le même réseau |
| **Service** | — | Ressource K8s qui expose des pods sur le réseau de façon stable |
| **Service Mesh** | — | Couche réseau qui gère la communication entre services (Istio, Linkerd) |
| **VPC CNI** | — | Plugin réseau AWS qui donne aux pods des IPs directement du VPC |

---

## Sommaire

1. [Le modèle réseau Kubernetes](#1-le-modèle-réseau-kubernetes)
2. [Les Pods et leurs IPs](#2-les-pods-et-leurs-ips)
3. [Les Services](#3-les-services)
4. [DNS dans Kubernetes](#4-dns-dans-kubernetes)
5. [Ingress](#5-ingress)
6. [Network Policies](#6-network-policies)
7. [CNI Plugins](#7-cni-plugins)
8. [Kubernetes Networking sur AWS (EKS)](#8-kubernetes-networking-sur-aws-eks)

---

## 1. Le modèle réseau Kubernetes

### Pourquoi c'est important

Kubernetes a un modèle réseau fondamentalement différent de Docker. Dans Docker, les conteneurs sont isolés par défaut et tu dois explicitement créer des réseaux pour les faire communiquer. Dans Kubernetes, c'est l'inverse — **tout pod peut parler à tout autre pod par défaut**, et c'est toi qui ajoutes des restrictions ensuite.

Comprendre ce modèle, c'est comprendre pourquoi K8s a besoin de Network Policies pour la sécurité, et pourquoi les Services existent pour stabiliser la communication.

### Les 4 règles fondamentales du modèle réseau K8s

Kubernetes impose ces règles à tous les plugins réseau (CNI) :

```
Règle 1 : Chaque pod a une IP unique dans le cluster
          → pas de partage d'IP entre pods
          → un pod = une identité réseau

Règle 2 : Tous les pods peuvent se parler directement
          → sans NAT
          → l'IP source n'est jamais modifiée
          → un pod dans le nœud A voit l'IP réelle d'un pod dans le nœud B

Règle 3 : Les agents du nœud peuvent parler à tous les pods
          → kubelet, kube-proxy ont accès à tous les pods du nœud

Règle 4 : Les pods qui s'exécutent sur le réseau hôte (hostNetwork: true)
          voient les IPs des pods comme si ils étaient sur le nœud
```

### Différence fondamentale avec Docker

```
DOCKER :
  Conteneurs isolés par défaut
  Tu crées un réseau pour les faire communiquer
  NAT pour la communication inter-hôtes
  IPs dans une plage Docker séparée (172.17.x.x)

KUBERNETES :
  Pods accessibles par défaut (flat network)
  Tous les pods dans le même espace d'adressage
  Pas de NAT entre pods
  IPs uniques dans tout le cluster
```

### Vue d'ensemble de l'architecture réseau K8s

```
Cluster Kubernetes
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  Nœud 1 (192.168.1.10)      Nœud 2 (192.168.1.11)     │
│  ┌─────────────────────┐    ┌─────────────────────┐    │
│  │  Pod A  10.244.0.2  │    │  Pod C  10.244.1.2  │    │
│  │  Pod B  10.244.0.3  │    │  Pod D  10.244.1.3  │    │
│  └─────────────────────┘    └─────────────────────┘    │
│                                                         │
│  Pod A (10.244.0.2) → Pod D (10.244.1.3) ✅            │
│  Communication directe, sans NAT                        │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 2. Les Pods et leurs IPs

### Pourquoi c'est important

Le pod est l'unité de base de Kubernetes. Comprendre son modèle réseau — comment il obtient son IP, pourquoi cette IP est éphémère, comment les conteneurs d'un même pod se parlent — est la fondation de tout ce qui suit.

### Ce qu'est un pod réseau

Un pod est un groupe de conteneurs qui **partagent le même namespace réseau**. Ils ont :
- La même adresse IP
- Les mêmes interfaces réseau
- La même table de routage
- Le même `localhost`

```
Pod (IP: 10.244.0.2)
  ├── Conteneur app (écoute sur :3000)
  ├── Conteneur sidecar (écoute sur :9090)
  └── Réseau partagé
      → app peut joindre sidecar via localhost:9090
      → de l'extérieur : 10.244.0.2:3000 et 10.244.0.2:9090
```

### Un pod = une interface réseau

Chaque pod reçoit une interface réseau virtuelle (via le CNI) et une IP unique dans le cluster. Cette IP est attribuée depuis le CIDR du pod network — défini à la création du cluster.

```bash
# Voir les IPs des pods
kubectl get pods -o wide

# Output :
NAME          READY   STATUS    IP            NODE
frontend-xyz  1/1     Running   10.244.0.4    node-1
api-abc       1/1     Running   10.244.0.5    node-1
database-def  1/1     Running   10.244.1.2    node-2
```

### Le problème de l'éphémérité des IPs

Les IPs de pods sont **éphémères** — elles changent à chaque fois qu'un pod est recréé :

```
Pod "api" démarre    → IP 10.244.0.5
Pod "api" crashe     → K8s recrée le pod
Pod "api" redémarre  → IP 10.244.0.8  ← différente !

Si "frontend" hardcode l'IP 10.244.0.5 → connexion cassée ❌

Solution : les Services (voir section 3)
→ IP stable même quand les pods changent ✅
```

### Communication entre conteneurs d'un même pod

Les conteneurs d'un même pod partagent `localhost` — ils communiquent comme deux processus sur la même machine :

```yaml
# Pod avec deux conteneurs
spec:
  containers:
  - name: app
    image: mon-api
    ports:
    - containerPort: 3000

  - name: log-shipper
    image: fluentd
    # peut joindre l'app via localhost:3000
    # sans passer par le réseau K8s
```

---

## 3. Les Services

### Pourquoi c'est important

Les pods ont des IPs éphémères — elles changent à chaque recréation. Les Services résolvent ce problème en fournissant une **IP virtuelle stable** (ClusterIP) qui reste constante même quand les pods en dessous changent. Un Service fait le pont entre les clients (stables) et les pods (éphémères).

### Comment fonctionne un Service

Un Service utilise des **selectors** pour trouver les pods qu'il doit exposer. Il maintient une liste d'endpoints — les IPs des pods sélectionnés — et distribue le trafic entre eux.

```yaml
# Deployment — crée 3 pods avec le label app=api
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api    # ← label que le Service va sélectionner
    spec:
      containers:
      - name: api
        image: mon-api
        ports:
        - containerPort: 3000
---
# Service — sélectionne les pods avec app=api
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  selector:
    app: api        # ← sélectionne les pods avec ce label
  ports:
  - port: 80        # port du Service
    targetPort: 3000 # port des pods
```

```
Service "api-service" (ClusterIP: 10.96.0.1)
  │
  ├── Pod api-xyz (10.244.0.5:3000)
  ├── Pod api-abc (10.244.0.6:3000)
  └── Pod api-def (10.244.1.2:3000)

frontend appelle http://api-service:80
→ Service distribue vers l'un des 3 pods (round-robin)
→ même si les pods changent, l'adresse du Service reste ✅
```

### ClusterIP — Service interne

**ClusterIP** est le type par défaut. Il crée une IP virtuelle accessible uniquement depuis l'intérieur du cluster.

```yaml
spec:
  type: ClusterIP    # défaut si non spécifié
  selector:
    app: api
  ports:
  - port: 80
    targetPort: 3000
```

```
Accessible depuis : pods dans le cluster ✅
Accessible depuis : l'extérieur du cluster ❌
Cas d'usage : communication interne (api → database)
```

### NodePort — exposer sur les nœuds

**NodePort** expose le Service sur un port de chaque nœud du cluster. Depuis l'extérieur, tu peux joindre le Service via `<IP-du-nœud>:<NodePort>`.

```yaml
spec:
  type: NodePort
  selector:
    app: frontend
  ports:
  - port: 80          # port interne du Service
    targetPort: 3000  # port des pods
    nodePort: 30080   # port sur les nœuds (30000-32767)
```

```
Nœud 1 (192.168.1.10) : port 30080 ouvert
Nœud 2 (192.168.1.11) : port 30080 ouvert

Accès externe : http://192.168.1.10:30080 ✅
               http://192.168.1.11:30080 ✅

Inconvénient : port dans la plage 30000-32767
               pas propre pour la production
               pas de load balancing externe
```

### LoadBalancer — exposer via un Load Balancer externe

**LoadBalancer** crée un Load Balancer externe (ex: ALB sur AWS) qui route le trafic vers les nœuds K8s.

```yaml
spec:
  type: LoadBalancer
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 3000
```

```
Internet
    │
    ▼
AWS Load Balancer (IP publique attribuée automatiquement)
    │
    ├── Nœud 1 : NodePort 30080
    └── Nœud 2 : NodePort 30080
         │
         └── Pods frontend

kubectl get service frontend
→ EXTERNAL-IP: 54.12.34.56  ← IP du Load Balancer AWS
```

### ExternalName — alias DNS

**ExternalName** fait pointer un Service vers un nom DNS externe — utile pour intégrer des services extérieurs au cluster.

```yaml
spec:
  type: ExternalName
  externalName: ma-rds.xxxx.eu-central-1.rds.amazonaws.com
```

```
Dans les pods : connexion à "database" (nom du Service)
→ K8s résout "database" → ma-rds.xxxx.rds.amazonaws.com
→ transparent pour l'application ✅
```

### kube-proxy — le mécanisme sous-jacent

**kube-proxy** tourne sur chaque nœud et maintient les règles **iptables** qui implémentent les Services. Quand un pod envoie une requête vers un ClusterIP, iptables intercepte et redirige vers l'un des pods cibles.

```
Pod A envoie requête vers 10.96.0.1:80 (ClusterIP)
    │
    ▼
iptables (géré par kube-proxy) intercepte
    │
    ▼
Redirige vers 10.244.0.5:3000 (Pod réel)
    │
    ▼
Pod B reçoit la requête sur son IP réelle ✅

Le ClusterIP n'existe pas physiquement —
c'est une règle iptables sur chaque nœud.
```

---

## 4. DNS dans Kubernetes

### Pourquoi c'est important

Dans le Module 3, on a vu le DNS Docker — les conteneurs se trouvent par leur nom de service. Kubernetes va plus loin avec un système DNS hiérarchique complet. Comprendre le format DNS K8s est essentiel pour débugger les problèmes de communication entre services et pour comprendre comment les applications doivent être configurées.

### CoreDNS — le DNS interne de K8s

**CoreDNS** est le serveur DNS qui tourne dans chaque cluster Kubernetes. Il résout automatiquement les noms des Services en leurs ClusterIPs.

```bash
# CoreDNS tourne comme un Deployment dans le namespace kube-system
kubectl get pods -n kube-system | grep coredns

# Output :
coredns-5d78c9869d-abc12   1/1   Running   kube-system
coredns-5d78c9869d-def34   1/1   Running   kube-system
# → toujours deux replicas pour la haute disponibilité
```

### Le format DNS K8s

Chaque Service reçoit automatiquement un nom DNS dans ce format :

```
<service>.<namespace>.svc.cluster.local

Exemple :
  Service "api" dans le namespace "production"
  → api.production.svc.cluster.local

  Service "database" dans le namespace "production"
  → database.production.svc.cluster.local

  Service "redis" dans le namespace "default"
  → redis.default.svc.cluster.local
```

### Résolution DNS depuis un pod

Depuis l'intérieur d'un pod, tu n'as pas besoin d'écrire le nom complet — K8s ajoute automatiquement les suffixes DNS :

```bash
# Depuis un pod dans le namespace "production"

# Ces trois formes fonctionnent pour joindre le service "api"
# dans le MÊME namespace :
curl http://api
curl http://api.production
curl http://api.production.svc.cluster.local

# Pour joindre un service dans un AUTRE namespace :
# le nom court ne suffit plus
curl http://database.staging             # ✅ autre namespace
curl http://database.staging.svc.cluster.local  # ✅ forme complète
```

```bash
# Voir la config DNS dans un pod
kubectl exec mon-pod -- cat /etc/resolv.conf

# Output :
nameserver 10.96.0.10           ← IP du Service CoreDNS
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

### DNS pour les pods (moins utilisé)

Les pods eux-mêmes ont aussi des entrées DNS, mais basées sur leurs IPs (avec des tirets) :

```
Pod IP : 10.244.0.5
DNS    : 10-244-0-5.default.pod.cluster.local

→ rarement utilisé directement
→ on préfère toujours passer par les Services
```

### Débugger le DNS K8s

```bash
# Lancer un pod de debug avec les outils réseau
kubectl run debug --image=nicolaka/netshoot --rm -it

# Dans le pod debug :
nslookup kubernetes.default          # service K8s lui-même
nslookup api.production              # ton service
dig api.production.svc.cluster.local # forme complète

# Tester la résolution complète
nslookup api
# → Server: 10.96.0.10 (CoreDNS)
# → Address: 10.96.1.45 (ClusterIP du service api)
```

---

## 5. Ingress

### Pourquoi c'est important

Un Service de type LoadBalancer crée un Load Balancer externe pour **chaque** Service. Si tu as 10 microservices, tu crées 10 Load Balancers — coûteux et difficile à gérer. **Ingress** est la solution : un seul point d'entrée qui route le trafic vers les bons Services selon l'URL ou le hostname.

C'est exactement le même concept que le reverse proxy qu'on a vu dans le Module 1 (nginx) et l'ALB du Module 4 — mais au niveau K8s.

### Ingress vs Service LoadBalancer

```
SERVICE LOADBALANCER (sans Ingress) :
  service-frontend  → LB 54.1.1.1
  service-api       → LB 54.1.1.2
  service-auth      → LB 54.1.1.3
  → 3 Load Balancers, 3 IPs, 3 factures ❌

INGRESS (un seul LB) :
  Ingress Controller → LB 54.1.1.1 (un seul)
    /           → service-frontend
    /api/*      → service-api
    /auth/*     → service-auth
  → 1 Load Balancer, 1 IP, routing intelligent ✅
```

### L'Ingress Controller

Un **Ingress** est juste une ressource K8s qui définit des règles. Pour qu'elles soient appliquées, il faut un **Ingress Controller** — un composant qui lit ces règles et configure réellement le Load Balancer.

```
Ingress (règles)     →     Ingress Controller    →    Traffic
kind: Ingress              nginx-ingress               Internet
  /api → service-api       traefik                         │
  /    → service-web       AWS ALB Controller              ▼
                                                     Load Balancer
```

**Ingress Controllers populaires :**
- **nginx-ingress** : le plus utilisé, très configurable
- **traefik** : moderne, auto-découverte
- **AWS Load Balancer Controller** : crée des ALB natifs AWS pour EKS

### Routing par path

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: monapp.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

```
monapp.com/api/users  → api-service:80
monapp.com/api/orders → api-service:80
monapp.com/           → frontend-service:80
monapp.com/about      → frontend-service:80
```

### Routing par hostname

```yaml
spec:
  rules:
  - host: api.monapp.com        # sous-domaine API
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80

  - host: admin.monapp.com      # sous-domaine admin
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: admin-service
            port:
              number: 80
```

### TLS avec Ingress

```yaml
spec:
  tls:
  - hosts:
    - monapp.com
    secretName: monapp-tls-secret   # Secret K8s contenant le certificat

  rules:
  - host: monapp.com
    http:
      paths:
      - path: /
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

```bash
# Créer le secret TLS
kubectl create secret tls monapp-tls-secret \
  --cert=monapp.crt \
  --key=monapp.key

# Avec cert-manager (renouvellement automatique Let's Encrypt)
# cert-manager crée et renouvelle automatiquement les certificats TLS
# Équivalent K8s de ACM sur AWS
```

### Lien avec AWS ALB sur EKS

Sur EKS, l'**AWS Load Balancer Controller** traduit les ressources Ingress K8s en ALB AWS natifs :

```yaml
# Ingress avec annotations AWS
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  rules:
  - host: monapp.com
    http:
      paths:
      - path: /
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

```
kubectl apply → AWS Load Balancer Controller lit l'Ingress
             → crée un ALB AWS dans le VPC
             → configure les Target Groups avec les IPs des pods
             → associe le certificat ACM si configuré
```

---

## 6. Network Policies

### Pourquoi c'est important

Par défaut dans Kubernetes, **tout pod peut parler à tout autre pod** — il n'y a aucune isolation réseau. C'est pratique pour développer, mais inacceptable en production. Les **Network Policies** sont l'équivalent des Security Groups AWS — elles contrôlent quel trafic est autorisé entre les pods.

### Le modèle par défaut — tout ouvert

```
Par défaut :
  Pod frontend  ────►  Pod api      ✅
  Pod frontend  ────►  Pod database ✅  ← dangereux !
  Pod api       ────►  Pod database ✅
  Pod database  ────►  Pod api      ✅  ← inutile

Avec Network Policies :
  Pod frontend  ────►  Pod api      ✅
  Pod frontend  ────►  Pod database ❌  (bloqué)
  Pod api       ────►  Pod database ✅
  Pod database  ────►  Pod api      ❌  (bloqué)
```

### Structure d'une Network Policy

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-network-policy
  namespace: production
spec:
  podSelector:              # à quels pods s'applique cette policy ?
    matchLabels:
      app: api

  policyTypes:
  - Ingress                 # règles de trafic entrant
  - Egress                  # règles de trafic sortant

  ingress:
  - from:
    - podSelector:          # autorise uniquement les pods frontend
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 3000

  egress:
  - to:
    - podSelector:          # autorise uniquement vers les pods database
        matchLabels:
          app: database
    ports:
    - protocol: TCP
      port: 5432
```

### Architecture Network Policies — pattern 3-tiers

```yaml
# Policy pour frontend : peut contacter api, rien d'autre
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-policy
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: api
    ports:
    - port: 80

# Policy pour api : reçoit depuis frontend, contacte database
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-policy
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - port: 5432

# Policy pour database : reçoit uniquement depuis api
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-policy
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: api
    ports:
    - port: 5432
```

### Network Policies cross-namespace

```yaml
# Autoriser le trafic depuis un namespace spécifique
ingress:
- from:
  - namespaceSelector:
      matchLabels:
        environment: production
    podSelector:
      matchLabels:
        app: frontend
```

> **Important** : les Network Policies nécessitent un CNI qui les supporte. **Flannel** ne supporte pas les Network Policies. **Calico** et **Weave** les supportent. Sur EKS, le VPC CNI supporte les Network Policies depuis 2023.

---

## 7. CNI Plugins

### Pourquoi c'est important

Le CNI est le composant qui implémente concrètement les règles réseau de Kubernetes. Sans CNI, les pods n'ont pas d'IP, ils ne peuvent pas communiquer. Le choix du CNI impacte les performances, les fonctionnalités disponibles (Network Policies, chiffrement...) et l'intégration avec l'infrastructure.

### Le rôle du CNI

Quand un pod démarre, Kubernetes appelle le plugin CNI qui :

```
1. Crée une interface réseau virtuelle pour le pod
2. Attribue une IP depuis le pod CIDR
3. Configure le routage pour que le pod soit joignable
4. Configure les règles pour que le pod puisse joindre les autres

Sans CNI → pods sans IP → rien ne fonctionne
```

### Les CNI principaux

**Flannel** — le plus simple
```
→ Réseau overlay simple (VXLAN)
→ Facile à configurer
→ Ne supporte PAS les Network Policies
→ Bon pour : dev, apprentissage, clusters simples
→ Utilisé par : minikube par défaut
```

**Calico** — le plus utilisé en production
```
→ Réseau overlay OU routage BGP natif
→ Supporte les Network Policies ✅
→ Hautes performances
→ Fonctionnalités avancées (chiffrement, observabilité)
→ Bon pour : production, environnements sécurisés
→ Utilisé par : EKS, GKE, la plupart des clusters prod
```

**Weave Net**
```
→ Réseau mesh automatique
→ Supporte les Network Policies ✅
→ Chiffrement intégré
→ Bon pour : clusters multi-cloud
```

**AWS VPC CNI** — spécifique EKS
```
→ Donne aux pods des IPs directement du VPC AWS
→ Les pods sont des "vraies" machines sur le VPC
→ Intégration native avec ALB, Security Groups
→ Supporte les Network Policies (depuis 2023)
→ Uniquement sur EKS
→ Voir section 8 pour les détails
```

### Tableau comparatif

| CNI | Network Policies | Performance | Complexité | Cas d'usage |
|-----|-----------------|-------------|------------|-------------|
| Flannel | ❌ | Bonne | Simple | Dev, apprentissage |
| Calico | ✅ | Excellente | Moyenne | Production |
| Weave | ✅ | Bonne | Moyenne | Multi-cloud |
| AWS VPC CNI | ✅ | Excellente | Faible sur EKS | EKS uniquement |

---

## 8. Kubernetes Networking sur AWS (EKS)

### Pourquoi c'est important

Quand tu déploies K8s sur AWS avec EKS, le networking change fondamentalement par rapport à un cluster local. Les pods obtiennent des IPs directement du VPC — ils sont des citoyens à part entière de ton réseau AWS. Comprendre cette intégration, c'est comprendre comment K8s et AWS Networking (Module 4) s'articulent.

### AWS VPC CNI — des IPs pods dans le VPC

Sur EKS, le plugin **AWS VPC CNI** donne à chaque pod une IP directement depuis le CIDR du VPC :

```
VPC : 10.0.0.0/16

Nœud EC2 (10.0.10.5) dans subnet-privé-1a (10.0.10.0/24)
  ├── Pod A : 10.0.10.42   ← IP dans le subnet du nœud
  ├── Pod B : 10.0.10.43   ← IP dans le subnet du nœud
  └── Pod C : 10.0.10.44   ← IP dans le subnet du nœud
```

```
Conséquences :
  ✅ Pas d'overlay réseau → performance maximale
  ✅ Les pods sont visibles comme des hôtes dans le VPC
  ✅ Les Security Groups s'appliquent directement aux pods
  ✅ Les Flow Logs capturent le trafic des pods
  ⚠️ Chaque nœud consomme plusieurs IPs du subnet
     → planifier les CIDRs plus grands qu'avec un cluster local
```

### Dimensionnement des subnets pour EKS

```
Chaque instance EC2 a une limite d'ENIs et d'IPs par ENI.

Exemple : m5.large
  → 3 ENIs maximum
  → 10 IPs par ENI
  → Maximum 30 pods par nœud (avec AWS VPC CNI)

Pour 10 nœuds m5.large :
  → jusqu'à 300 pods
  → 300 IPs consommées dans le subnet
  → Subnet /24 (251 IPs) insuffisant !
  → Utiliser /22 (1019 IPs) ou plus grand

Bonne pratique :
  Subnets dédiés pour les nœuds EKS, séparés des autres ressources
  subnet-eks-1a : 10.0.100.0/22  (1019 IPs)
  subnet-eks-1b : 10.0.104.0/22  (1019 IPs)
```

### AWS Load Balancer Controller — Ingress natif AWS

L'**AWS Load Balancer Controller** est un composant installé dans EKS qui traduit les ressources Ingress K8s en ressources AWS natives :

```
Ingress K8s avec annotation alb
    │
    ▼
AWS Load Balancer Controller
    │
    ├── Crée un ALB dans le VPC
    ├── Crée des Target Groups avec les IPs des pods
    ├── Configure les Listeners et les règles de routing
    └── Associe le certificat ACM si spécifié

kubectl get ingress
→ ADDRESS: k8s-default-app-xxxxx.eu-central-1.elb.amazonaws.com
→ C'est l'ALB AWS créé automatiquement ✅
```

### Security Groups pour les pods EKS

Sur EKS avec VPC CNI, tu peux attacher des Security Groups **directement aux pods** — pas seulement aux nœuds :

```yaml
# SecurityGroupPolicy (ressource EKS)
apiVersion: vpcresources.k8s.aws/v1beta1
kind: SecurityGroupPolicy
metadata:
  name: api-sg-policy
spec:
  podSelector:
    matchLabels:
      app: api
  securityGroups:
    groupIds:
    - sg-xxxxxxxx    # SG AWS attaché aux pods "api"
```

```
Avant (Security Groups sur le nœud) :
  SG nœud autorise port 5432
  → TOUS les pods du nœud peuvent joindre la RDS ⚠️

Avec Security Groups par pod :
  SG attaché uniquement aux pods "api"
  → seuls les pods "api" peuvent joindre la RDS ✅
  → même isolation que les Security Groups AWS Module 4
    mais au niveau pod K8s
```

### Architecture complète EKS + AWS Networking

```
Internet
    │
    ▼
Route 53 (monapp.com → ALB)
    │
    ▼
ALB (créé par AWS LB Controller depuis Ingress K8s)
subnet-public-1a / subnet-public-1b
    │
    ▼
Security Group ALB : 443 depuis 0.0.0.0/0
    │
    ▼
Pods frontend (10.0.100.x) — Security Group pods-frontend
subnet-eks-1a / subnet-eks-1b
    │
    ▼
Security Group pods-api : 3000 depuis SG pods-frontend
    │
    ▼
Pods api (10.0.104.x) — Security Group pods-api
    │
    ▼
Security Group RDS : 5432 depuis SG pods-api
    │
    ▼
RDS PostgreSQL (10.0.20.5)
subnet-db-1a / subnet-db-1b
```

---

## Pour aller plus loin — Service Mesh

### Mention : Istio et Linkerd

Un **Service Mesh** est une couche réseau supplémentaire qui gère la communication entre services de façon transparente — chiffrement mTLS, observabilité, circuit breakers, retries, traffic management.

```
Sans Service Mesh :
  Pod A → Pod B (trafic en clair dans le cluster)
  Pas de métriques réseau détaillées
  Chaque app gère ses retries/timeouts

Avec Service Mesh (Istio) :
  Pod A → [Envoy sidecar] → [Envoy sidecar] → Pod B
  Chiffrement mTLS automatique entre tous les pods
  Métriques réseau détaillées (latence, erreurs, throughput)
  Retries, timeouts, circuit breakers configurables sans code

Cas d'usage : microservices complexes en production,
              conformité sécurité (chiffrement interne obligatoire)
Coût : complexité opérationnelle importante
→ réservé aux architectures matures
```

---

## Récapitulatif du Module 5

| Concept | Ce qu'il faut retenir |
|---------|----------------------|
| **Modèle réseau K8s** | Tout pod a une IP unique. Pod-to-pod sans NAT. Tout ouvert par défaut. |
| **Pod** | Unité réseau de base. IP éphémère. Conteneurs du même pod partagent localhost. |
| **ClusterIP** | IP virtuelle stable pour un Service. Interne au cluster uniquement. |
| **NodePort** | Expose un Service sur un port de chaque nœud (30000-32767). |
| **LoadBalancer** | Crée un Load Balancer externe. Sur AWS = ALB ou NLB via AWS LB Controller. |
| **kube-proxy** | Maintient les règles iptables qui implémentent les Services. |
| **CoreDNS** | DNS interne K8s. Résout `service.namespace.svc.cluster.local`. |
| **Ingress** | Un seul LB pour plusieurs Services. Routing par URL/hostname. |
| **Ingress Controller** | Composant qui implémente les règles Ingress (nginx, ALB...). |
| **Network Policy** | Contrôle le trafic entre pods. Équivalent Security Groups pour K8s. |
| **CNI** | Plugin qui donne des IPs aux pods et configure le réseau. |
| **AWS VPC CNI** | Sur EKS : pods ont des IPs du VPC. Performance maximale. |
| **AWS LB Controller** | Traduit les Ingress K8s en ALB AWS natifs. |
| **Service Mesh** | Couche réseau avancée (mTLS, observabilité). Istio, Linkerd. |

---

*→ Exercices pratiques : voir `module5-exercices.md`*
*→ Module suivant : Module 6 — Networking as Code avec Terraform*
