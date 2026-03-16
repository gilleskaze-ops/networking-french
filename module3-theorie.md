# Module 3 — Docker Networking
### Partie Théorie

> **Cours Networking Fondamentaux · Orienté Cloud**
> Prérequis : Modules 1 et 2 complétés · Docker installé · Durée estimée : 2h

---

## Glossaire

| Terme | Signification | Définition simple |
|-------|--------------|-------------------|
| **Bridge** | — | Réseau virtuel local créé par Docker — permet aux conteneurs de se parler entre eux |
| **CNI** | Container Network Interface | Standard qui définit comment les plugins réseau fonctionnent avec les conteneurs |
| **Container** | Conteneur | Processus isolé qui partage le noyau de l'OS hôte — plus léger qu'une VM |
| **docker0** | — | Interface réseau bridge créée automatiquement par Docker sur l'hôte |
| **Docker Compose** | — | Outil pour définir et lancer plusieurs conteneurs ensemble via un fichier YAML |
| **EXPOSE** | — | Instruction Dockerfile qui documente le port utilisé — ne publie rien sans `-p` |
| **Image** | — | Modèle immuable à partir duquel on crée des conteneurs |
| **Ingress** | — | Point d'entrée réseau qui route le trafic vers les bons conteneurs |
| **Macvlan** | — | Mode réseau Docker qui donne une adresse MAC propre à chaque conteneur |
| **NAT** | Network Address Translation | Traduction d'adresse — permet aux conteneurs d'accéder à Internet |
| **Network namespace** | — | Isolation réseau au niveau du noyau Linux — chaque conteneur a le sien |
| **Overlay** | — | Réseau virtuel qui s'étend sur plusieurs hôtes Docker |
| **Port binding** | — | Association entre un port de l'hôte et un port du conteneur (`-p 8080:80`) |
| **Registry** | — | Dépôt d'images Docker (ex: Docker Hub, ECR AWS) |
| **Service discovery** | — | Mécanisme par lequel les conteneurs se trouvent mutuellement par nom |
| **veth** | Virtual Ethernet | Paire de câbles réseau virtuels reliant un conteneur au bridge Docker |
| **VM** | Virtual Machine | Machine virtuelle avec son propre noyau OS — plus lourd qu'un conteneur |

---

## Sommaire

1. [Docker en bref — le contexte](#1-docker-en-bref--le-contexte)
2. [Comment Docker gère le réseau](#2-comment-docker-gère-le-réseau)
3. [Les modes réseau Docker](#3-les-modes-réseau-docker)
4. [DNS dans Docker](#4-dns-dans-docker)
5. [Docker Compose networking](#5-docker-compose-networking)
6. [Ports et exposition de services](#6-ports-et-exposition-de-services)
7. [Debug réseau dans les conteneurs](#7-debug-réseau-dans-les-conteneurs)

---

## 1. Docker en bref — le contexte

### Pourquoi ce contexte est nécessaire

Avant de comprendre comment Docker gère le réseau, il faut comprendre ce qu'est un conteneur et en quoi il diffère d'une machine virtuelle. La différence architecturale entre les deux explique directement pourquoi Docker a besoin de son propre système réseau.

### Conteneur vs Machine Virtuelle

Une **machine virtuelle** émule un ordinateur complet — elle a son propre noyau OS, ses propres drivers, sa propre mémoire allouée. C'est comme avoir un deuxième ordinateur qui tourne à l'intérieur du premier.

Un **conteneur** est un processus isolé qui partage le noyau de l'OS hôte. Il n'a pas son propre OS — il emprunte celui de la machine hôte. C'est beaucoup plus léger.

```
MACHINE VIRTUELLE                    CONTENEUR
┌─────────────────────┐              ┌─────────────────────┐
│   App               │              │   App               │
├─────────────────────┤              ├─────────────────────┤
│   OS invité         │              │   Libs / dépendances│
│   (Linux/Windows)   │              └──────────┬──────────┘
├─────────────────────┤                         │ partage
│   Hyperviseur       │              ┌──────────┴──────────┐
├─────────────────────┤              │   Noyau OS hôte     │
│   OS hôte           │              ├─────────────────────┤
├─────────────────────┤              │   Hardware          │
│   Hardware          │              └─────────────────────┘
└─────────────────────┘

Démarrage : 30-60 secondes           Démarrage : < 1 seconde
Taille     : plusieurs Go             Taille    : quelques Mo à quelques centaines de Mo
Isolation  : complète (noyau séparé) Isolation  : partielle (noyau partagé)
```

### Les concepts Docker essentiels

**Image** : un modèle immuable — comme un template. Elle contient le code, les libs, la config. On ne modifie pas une image, on en crée des conteneurs.

**Conteneur** : une instance en cours d'exécution d'une image. Tu peux lancer 10 conteneurs à partir de la même image — ils sont indépendants.

```bash
# Lancer un conteneur nginx
docker run -d --name mon-nginx nginx

# Lancer 3 conteneurs à partir de la même image
docker run -d --name web1 nginx
docker run -d --name web2 nginx
docker run -d --name web3 nginx

# Lister les conteneurs en cours
docker ps

# Arrêter et supprimer
docker stop mon-nginx
docker rm mon-nginx
```

### Pourquoi Docker a besoin de son propre réseau

Un conteneur est isolé — il a son propre **namespace réseau** (espace réseau isolé au niveau du noyau Linux). Ça signifie qu'il a ses propres interfaces réseau, sa propre table de routage, ses propres règles firewall.

```
Hôte Linux
├── Interface eth0 (192.168.178.10)  ← réseau physique de l'hôte
├── Interface docker0 (172.17.0.1)   ← bridge Docker
│
├── Conteneur A
│   └── Interface eth0 (172.17.0.2)  ← réseau Docker privé
│
└── Conteneur B
    └── Interface eth0 (172.17.0.3)  ← réseau Docker privé
```

Chaque conteneur voit son propre `eth0` — mais ce n'est pas le vrai `eth0` de l'hôte. C'est une interface virtuelle connectée au bridge Docker. C'est ce mécanisme qu'on va détailler dans la section suivante.

---

## 2. Comment Docker gère le réseau

### Pourquoi c'est important

Quand un conteneur démarre, il ne tombe pas du ciel sur un réseau. Docker crée toute une infrastructure réseau virtuelle pour lui — interfaces, bridge, règles NAT, DNS. Comprendre cette infrastructure, c'est comprendre pourquoi deux conteneurs peuvent ou ne peuvent pas se parler, et comment le trafic sort vers Internet.

### L'interface docker0 — le bridge central

Quand tu installes Docker, il crée automatiquement une interface réseau sur l'hôte appelée **docker0**. C'est un **bridge réseau virtuel** — comme un switch virtuel auquel tous les conteneurs se connectent par défaut.

```bash
# Voir l'interface docker0 sur l'hôte
ip addr show docker0

# Output typique :
docker0: <BROADCAST,MULTICAST,UP,LOWER_UP>
    inet 172.17.0.1/16 brd 172.17.255.255
    # → Docker est le "routeur" de ce réseau privé 172.17.0.0/16
```

### Les paires veth — les câbles virtuels

Pour connecter chaque conteneur au bridge docker0, Docker crée une **paire veth** (virtual ethernet) — deux interfaces virtuelles reliées comme les deux extrémités d'un câble :

```
Hôte                              Conteneur
                    ┌─────────────────────────────┐
docker0 ──── vethXXX│vethYYY ── eth0 (172.17.0.2) │
(bridge)    (hôte)  │(conteneur)                   │
                    └─────────────────────────────┘

Ce qui circule dans vethXXX ressort dans vethYYY — et vice versa.
C'est un câble virtuel entre l'hôte et le conteneur.
```

```bash
# Voir les paires veth créées par Docker
ip link show | grep veth

# Voir tous les réseaux Docker
docker network ls

# Inspecter le réseau bridge par défaut
docker network inspect bridge
```

### NAT automatique — comment les conteneurs accèdent à Internet

Les conteneurs ont des IPs privées (`172.17.0.x`) — non routables sur Internet. Docker configure automatiquement des règles **iptables** sur l'hôte pour faire du NAT :

```
Conteneur (172.17.0.2)
    │
    │  "je veux joindre google.com"
    ▼
docker0 bridge (172.17.0.1)
    │
    │  iptables MASQUERADE
    │  remplace 172.17.0.2 par l'IP de l'hôte (192.168.178.10)
    ▼
Interface eth0 de l'hôte
    │
    ▼
Internet → Google ✅
```

```bash
# Voir les règles iptables créées par Docker
sudo iptables -t nat -L -n | grep DOCKER

# Output :
# MASQUERADE  all  172.17.0.0/16  →  anywhere
# ← Docker masque les IPs des conteneurs avec l'IP de l'hôte
```

### Isolation par défaut

Par défaut, deux conteneurs sur le réseau bridge par défaut peuvent se parler **par IP** mais **pas par nom**. Et ils sont isolés du reste du réseau hôte sauf via les ports explicitement publiés.

```
Réseau bridge par défaut (172.17.0.0/16) :

  Conteneur A (172.17.0.2) ──ping──► Conteneur B (172.17.0.3) ✅
  Conteneur A               ──ping──► "mon-conteneur-b"        ❌ (pas de DNS)
  Internet                  ──────►  Conteneur A               ❌ (pas de port publié)
  Conteneur A               ──────►  Internet                  ✅ (NAT automatique)
```

C'est précisément pour avoir le DNS entre conteneurs qu'on crée des **réseaux personnalisés** — qu'on verra en section 4.

---

## 3. Les modes réseau Docker

### Pourquoi c'est important

Docker propose plusieurs modes réseau selon le niveau d'isolation et de performance dont tu as besoin. Choisir le mauvais mode peut causer des problèmes de sécurité (trop ouvert) ou de connectivité (trop isolé). Chaque mode a son cas d'usage précis.

### Bridge — le mode par défaut

Le mode **bridge** est celui qu'on vient de décrire — chaque conteneur obtient une IP privée sur un réseau virtuel isolé. C'est le mode par défaut quand tu lances `docker run` sans préciser de réseau.

```bash
# Ces deux commandes sont équivalentes
docker run nginx
docker run --network bridge nginx

# Créer son propre réseau bridge (recommandé — active le DNS)
docker network create mon-reseau
docker run --network mon-reseau nginx
```

```
Cas d'usage : applications multi-conteneurs isolées
              (API + BDD + Cache sur le même hôte)
Avantages   : isolation, DNS entre conteneurs (réseau perso)
Inconvénients : légère overhead réseau (NAT)
```

### Host — partager le réseau de l'hôte

Le mode **host** supprime l'isolation réseau — le conteneur partage directement l'interface réseau de l'hôte. Il n'y a plus de NAT, plus de bridge — le conteneur utilise les ports de l'hôte directement.

```bash
docker run --network host nginx
# nginx écoute sur le port 80 de l'hôte directement
# pas besoin de -p 80:80
```

```
SANS host mode :          AVEC host mode :
Conteneur:80              Hôte:80 = Conteneur:80
    ↕ NAT                 (même interface réseau)
Hôte:8080

Cas d'usage   : performances maximales, monitoring réseau
Avantages     : pas de NAT → latence minimale
Inconvénients : pas d'isolation réseau
                un port occupé par le conteneur = occupé sur l'hôte
                Linux uniquement (pas sur Mac/Windows)
```

### None — isolation totale

Le mode **none** désactive complètement le réseau du conteneur. Il n'a aucune interface réseau sauf le loopback.

```bash
docker run --network none alpine ping google.com
# → ping: google.com: Name or service not known ❌
# Aucun accès réseau possible
```

```
Cas d'usage   : traitement de données sensibles sans accès réseau
                tests d'isolation
Avantages     : sécurité maximale
Inconvénients : aucune communication possible
```

### Overlay — réseau multi-hôtes

Le mode **overlay** crée un réseau virtuel qui s'étend sur **plusieurs hôtes Docker**. Les conteneurs sur des machines différentes peuvent se parler comme s'ils étaient sur le même réseau local.

```
Hôte 1 (192.168.1.10)          Hôte 2 (192.168.1.11)
┌──────────────────┐            ┌──────────────────┐
│  Conteneur A     │            │  Conteneur B     │
│  10.0.0.2        │◄──────────►│  10.0.0.3        │
│  (réseau overlay)│            │  (réseau overlay)│
└──────────────────┘            └──────────────────┘
         ▼                               ▼
    eth0 physique                   eth0 physique
    trafic encapsulé (VXLAN)
```

```bash
# Nécessite Docker Swarm ou un orchestrateur
docker network create --driver overlay mon-overlay

Cas d'usage   : Docker Swarm, applications distribuées multi-hôtes
Avantages     : transparence réseau entre hôtes
Inconvénients : nécessite un orchestrateur, overhead VXLAN
```

### Macvlan — IP dédiée sur le réseau physique

Le mode **macvlan** donne au conteneur sa propre adresse MAC et IP sur le réseau physique — comme si c'était une vraie machine sur le réseau local.

```bash
docker network create \
  --driver macvlan \
  --subnet 192.168.178.0/24 \
  --gateway 192.168.178.1 \
  -o parent=eth0 \
  mon-macvlan

docker run --network mon-macvlan \
           --ip 192.168.178.200 \
           nginx
# Le conteneur est visible sur le réseau local comme une vraie machine
# IP 192.168.178.200 directement accessible depuis n'importe quel appareil du réseau
```

```
Cas d'usage   : intégration dans un réseau existant
                appareils IoT, legacy apps qui ont besoin d'une IP fixe
Avantages     : pas de NAT, IP directe sur le réseau physique
Inconvénients : configuration complexe, nécessite accès promiscuous
```

### Tableau comparatif

| Mode | Isolation | DNS entre conteneurs | Accès Internet | Cas d'usage |
|------|-----------|---------------------|----------------|-------------|
| **bridge** (perso) | ✅ Bonne | ✅ Oui | ✅ Via NAT | Applications multi-conteneurs |
| **bridge** (défaut) | ✅ Bonne | ❌ Non | ✅ Via NAT | Conteneurs isolés simples |
| **host** | ❌ Aucune | — | ✅ Direct | Performance maximale |
| **none** | ✅ Totale | ❌ Non | ❌ Non | Isolation sécurité |
| **overlay** | ✅ Bonne | ✅ Oui | ✅ Via NAT | Multi-hôtes / Swarm |
| **macvlan** | ✅ Bonne | ❌ Non | ✅ Direct | IP physique dédiée |

---

## 4. DNS dans Docker

### Pourquoi c'est important

Dans une vraie application, tes conteneurs ne peuvent pas se parler par IP — les IPs Docker changent à chaque redémarrage. Il faut se parler **par nom**. Le DNS interne de Docker est le mécanisme qui rend ça possible — et c'est l'une des différences les plus importantes entre le réseau bridge par défaut et un réseau personnalisé.

### Le problème du bridge par défaut

Sur le réseau bridge par défaut de Docker, les conteneurs peuvent se pinguer par IP mais **pas par nom** :

```bash
# Lancer deux conteneurs sur le bridge par défaut
docker run -d --name app nginx
docker run -d --name db postgres

# Depuis app, essayer de joindre db par nom
docker exec app ping db
# → ping: db: Name or service not known ❌

# Par IP ça marche (mais l'IP change à chaque redémarrage)
docker exec app ping 172.17.0.3  # ✅ mais fragile
```

C'est une limitation délibérée du bridge par défaut — il n'a pas de serveur DNS intégré.

### La solution — les réseaux personnalisés

Quand tu crées un **réseau bridge personnalisé**, Docker active automatiquement un **serveur DNS embarqué** (`127.0.0.11`) qui résout les noms des conteneurs sur ce réseau.

```bash
# Créer un réseau personnalisé
docker network create mon-app

# Lancer les conteneurs sur ce réseau
docker run -d --name app --network mon-app nginx
docker run -d --name db  --network mon-app postgres

# Maintenant la résolution DNS fonctionne !
docker exec app ping db
# → PING db (172.18.0.3) ✅

docker exec app curl http://db:5432
# → le nom "db" se résout automatiquement ✅
```

### Comment fonctionne le DNS Docker

```
Conteneur "app" cherche l'IP de "db"
    │
    │  requête DNS vers 127.0.0.11
    ▼
Serveur DNS Docker embarqué (127.0.0.11)
    │
    │  "db" est un conteneur sur ce réseau → IP = 172.18.0.3
    ▼
Conteneur "app" reçoit 172.18.0.3 ✅
```

```bash
# Vérifier le serveur DNS dans un conteneur
docker exec app cat /etc/resolv.conf

# Output :
nameserver 127.0.0.11     ← serveur DNS Docker
options ndots:0
```

### Les alias réseau

Tu peux donner des **alias** à un conteneur — des noms supplémentaires par lesquels il peut être contacté :

```bash
docker run -d \
  --name ma-base-postgres \
  --network mon-app \
  --network-alias db \
  --network-alias database \
  postgres

# Maintenant "ma-base-postgres", "db" et "database" pointent tous
# vers le même conteneur
docker exec app ping database  # ✅
docker exec app ping db        # ✅
```

**Cas d'usage Cloud** : en production sur AWS ECS ou EKS, le service discovery fonctionne exactement sur ce principe — les services se trouvent par nom, pas par IP.

---

## 5. Docker Compose networking

### Pourquoi c'est important

En développement et en production, tu ne lances jamais des conteneurs un par un avec `docker run`. Tu utilises **Docker Compose** pour définir toute ton architecture dans un fichier YAML et lancer tout d'un coup. Docker Compose crée automatiquement un réseau dédié pour ton application — et tous les services peuvent se parler par leur nom de service.

### Le réseau automatique par projet

Quand tu lances `docker compose up`, Docker crée automatiquement un réseau bridge personnalisé nommé `[projet]_default`. Tous les services définis dans le `compose.yml` sont automatiquement connectés à ce réseau.

```yaml
# compose.yml
services:
  frontend:
    image: nginx
    ports:
      - "80:80"

  api:
    image: node:20
    ports:
      - "3000:3000"

  database:
    image: postgres
    environment:
      POSTGRES_PASSWORD: secret
```

```bash
docker compose up -d

# Docker crée automatiquement :
# - Réseau : monprojet_default
# - Conteneurs : monprojet-frontend-1, monprojet-api-1, monprojet-database-1
# - Tous connectés au même réseau → DNS entre eux ✅

# frontend peut contacter api par son nom de service :
# http://api:3000

# api peut contacter database par son nom de service :
# postgres://database:5432
```

### Communication entre services par nom

```yaml
# compose.yml — exemple complet avec networking explicite
services:
  frontend:
    image: nginx
    ports:
      - "80:80"           # exposé vers l'extérieur
    networks:
      - web

  api:
    image: mon-api
    # PAS de ports: → non accessible depuis l'extérieur
    environment:
      DB_HOST: database   # ← nom du service, pas une IP
      DB_PORT: 5432
    networks:
      - web
      - backend

  database:
    image: postgres
    # PAS de ports: → jamais exposé vers l'extérieur
    environment:
      POSTGRES_PASSWORD: secret
    networks:
      - backend           # accessible uniquement depuis le backend

networks:
  web:        # réseau frontend ↔ api
  backend:    # réseau api ↔ database
```

```
Internet
    │
    │ port 80
    ▼
[frontend]  ──── réseau "web" ────► [api]
                                      │
                                      │ réseau "backend"
                                      ▼
                                   [database]

Internet ne peut PAS atteindre api ou database directement ✅
```

### Isolation réseau dans Compose

En définissant plusieurs réseaux dans Compose, tu crées des zones d'isolation — exactement comme les subnets publics/privés dans AWS :

```
Réseau "web"     → frontend + api
Réseau "backend" → api + database

database ne peut pas être contacté par frontend directement
→ seul api peut parler à database ✅
C'est exactement la même logique que les Security Groups AWS
```

### Variables d'environnement et service discovery

```yaml
services:
  api:
    image: mon-api
    environment:
      # On utilise le nom du service, jamais une IP
      REDIS_HOST: cache
      REDIS_PORT: 6379
      DB_HOST: database
      DB_PORT: 5432
      DB_NAME: myapp

  cache:
    image: redis

  database:
    image: postgres
```

> Le nom du service dans `compose.yml` devient automatiquement un nom DNS résolvable pour tous les autres services du même réseau. C'est le **service discovery** de Docker Compose.

---

## 6. Ports et exposition de services

### Pourquoi c'est important

Par défaut, aucun port d'un conteneur n'est accessible depuis l'extérieur. Tu dois explicitement décider quels ports exposer et sur quelle interface de l'hôte. C'est un mécanisme de sécurité — et comprendre comment il fonctionne évite les erreurs d'exposition involontaire en production.

### La différence entre EXPOSE et -p

C'est l'une des confusions les plus fréquentes avec Docker :

```
EXPOSE dans un Dockerfile :
  → Documentation uniquement
  → Dit "ce conteneur écoute sur ce port"
  → Ne publie RIEN sur l'hôte
  → Pas de port accessible depuis l'extérieur

-p (ou --publish) dans docker run :
  → Publie réellement le port sur l'hôte
  → Crée une règle iptables de redirection
  → Le port devient accessible depuis l'extérieur
```

```dockerfile
# Dockerfile
FROM nginx
EXPOSE 80    # documentation — ne fait rien de concret
```

```bash
# Sans -p : port 80 du conteneur inaccessible depuis l'hôte
docker run nginx

# Avec -p : port 80 du conteneur accessible sur le port 8080 de l'hôte
docker run -p 8080:80 nginx
curl http://localhost:8080  # ✅
```

### Syntaxe complète de -p

```bash
# Format : -p [interface_hote:]port_hote:port_conteneur[/protocole]

# Port simple
docker run -p 8080:80 nginx
# → localhost:8080 → conteneur:80

# Port UDP
docker run -p 53:53/udp dns-server

# Lier à une interface spécifique
docker run -p 127.0.0.1:8080:80 nginx
# → accessible UNIQUEMENT depuis localhost (pas depuis le réseau local)

docker run -p 0.0.0.0:8080:80 nginx
# → accessible depuis TOUTES les interfaces (défaut)

# Plusieurs ports
docker run -p 80:80 -p 443:443 nginx

# Port aléatoire sur l'hôte
docker run -p 80 nginx
# Docker choisit un port libre sur l'hôte
docker port <container_id>  # voir quel port a été choisi
```

### 0.0.0.0 vs 127.0.0.1 — sécurité

C'est un point de sécurité critique :

```
-p 8080:80        → bind sur 0.0.0.0 par défaut
                    accessible depuis Internet si hôte exposé ⚠️

-p 127.0.0.1:8080:80 → bind uniquement sur loopback
                        accessible UNIQUEMENT depuis l'hôte local ✅
                        idéal pour les services de dev ou internes
```

```bash
# Vérifier sur quelle interface un port est bindé
ss -tlnp | grep 8080

# 0.0.0.0:8080  → accessible depuis partout
# 127.0.0.1:8080 → loopback uniquement
```

**En production sur AWS :**

```
Tu n'exposes JAMAIS les ports directement sur une EC2 publique.
Le trafic passe par un Load Balancer (ALB) qui est le seul
point d'entrée public. Les conteneurs écoutent sur leurs ports
internes, accessibles uniquement depuis le Load Balancer.

EC2 Security Group :
  → autorise port 80/443 depuis l'ALB uniquement
  → bloque tout le reste ✅
```

### Comment Docker crée les règles de redirection

Quand tu fais `-p 8080:80`, Docker ajoute automatiquement des règles iptables :

```bash
# Voir les règles de redirection créées par Docker
sudo iptables -t nat -L DOCKER -n

# Output :
DNAT  tcp  0.0.0.0/0  0.0.0.0/0  tcp dpt:8080  to:172.17.0.2:80
# ← tout trafic TCP sur le port 8080 de l'hôte
#   est redirigé vers le port 80 du conteneur (172.17.0.2)
```

---

## 7. Debug réseau dans les conteneurs

### Pourquoi c'est important

Les problèmes réseau dans Docker sont fréquents et peuvent être subtils — un conteneur qui ne peut pas joindre un autre, un port qui n'est pas accessible, un DNS qui ne résout pas. Avoir les bons outils et la bonne méthode de diagnostic fait la différence entre 5 minutes et 5 heures de débogage.

### docker network inspect — radiographier un réseau

```bash
# Voir tous les réseaux Docker
docker network ls

# Output :
NETWORK ID     NAME              DRIVER    SCOPE
abc123def456   bridge            bridge    local
xyz789uvw012   host              host      local
def456abc789   none              null      local
ghi789jkl012   mon-app_default   bridge    local

# Inspecter un réseau en détail
docker network inspect mon-app_default

# Output (simplifié) :
{
  "Name": "mon-app_default",
  "Driver": "bridge",
  "IPAM": {
    "Config": [{"Subnet": "172.18.0.0/16", "Gateway": "172.18.0.1"}]
  },
  "Containers": {
    "abc123": {
      "Name": "mon-app-api-1",
      "IPv4Address": "172.18.0.2/16"
    },
    "def456": {
      "Name": "mon-app-database-1",
      "IPv4Address": "172.18.0.3/16"
    }
  }
}
```

### docker exec — entrer dans un conteneur pour débugger

```bash
# Ouvrir un shell dans un conteneur
docker exec -it mon-conteneur bash
docker exec -it mon-conteneur sh    # si bash n'est pas dispo

# Exécuter une commande sans shell interactif
docker exec mon-conteneur ping database
docker exec mon-conteneur curl http://api:3000/health
docker exec mon-conteneur cat /etc/resolv.conf
docker exec mon-conteneur nslookup database
```

### Installer des outils réseau dans un conteneur

Les images Docker sont minimalistes — elles n'ont souvent pas les outils réseau installés. Deux approches :

```bash
# Approche 1 : installer dans le conteneur en cours
docker exec -it mon-conteneur bash
apt-get update && apt-get install -y \
  iputils-ping curl netcat-openbsd dnsutils nmap

# Approche 2 : utiliser une image de debug dédiée
docker run --rm --network mon-app nicolaka/netshoot
# netshoot contient tous les outils réseau préinstallés
# ping, curl, nc, nmap, tcpdump, dig, ss, ip, etc.
```

### Méthode de diagnostic étape par étape

```bash
# Problème : "mon conteneur api ne peut pas joindre database"

# Étape 1 — Les deux conteneurs sont-ils sur le même réseau ?
docker network inspect mon-app | grep -A3 "Containers"

# Étape 2 — Les IPs sont-elles correctes ?
docker exec api ip addr show

# Étape 3 — Le DNS résout-il correctement ?
docker exec api nslookup database
docker exec api cat /etc/resolv.conf

# Étape 4 — La machine répond-elle au ping ?
docker exec api ping -c 3 database

# Étape 5 — Le port est-il ouvert ?
docker exec api nc -zv database 5432

# Étape 6 — L'application répond-elle ?
docker exec api curl http://database:5432

# Étape 7 — Voir les logs du conteneur cible
docker logs database --tail 50
```

### tcpdump dans un conteneur

Pour capturer le trafic réseau d'un conteneur :

```bash
# Option 1 : depuis l'hôte, capturer sur l'interface veth du conteneur
# Trouver l'interface veth correspondant au conteneur
CONTAINER_ID=$(docker inspect mon-conteneur --format '{{.Id}}')
# Voir les interfaces veth sur l'hôte
ip link show | grep veth

# Capturer le trafic sur cette interface
sudo tcpdump -i vethXXXXXX -n

# Option 2 : depuis l'intérieur du conteneur (avec tcpdump installé)
docker exec -it mon-conteneur bash
tcpdump -i eth0 -n

# Option 3 : avec netshoot (le plus simple)
docker run --rm \
  --network container:mon-conteneur \
  nicolaka/netshoot \
  tcpdump -i eth0 -n port 5432
```

### nmap sur les conteneurs

```bash
# Scanner les ports d'un conteneur depuis l'hôte
docker inspect mon-conteneur \
  --format '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}'
# → 172.18.0.2

# Scanner cette IP (accessible depuis l'hôte via le bridge)
nmap -sV 172.18.0.2

# Scanner tous les conteneurs d'un réseau Docker
nmap -sn 172.18.0.0/16

# Depuis un conteneur netshoot — scanner un autre conteneur
docker run --rm --network mon-app nicolaka/netshoot \
  nmap -p 80,443,3000,5432 database
```

### Les erreurs courantes et leurs causes

```
Erreur : "connection refused" sur le port X
Cause  : le service dans le conteneur n'écoute pas sur ce port
         ou n'est pas encore démarré
Fix    : docker logs <conteneur> pour voir si le service a démarré

Erreur : "Name or service not known"
Cause  : DNS ne résout pas le nom
         → les conteneurs sont sur des réseaux différents
         → ou sur le bridge par défaut (pas de DNS)
Fix    : mettre les conteneurs sur le même réseau personnalisé

Erreur : "No route to host"
Cause  : les conteneurs sont sur des réseaux isolés
Fix    : connecter le conteneur au bon réseau
         docker network connect mon-reseau mon-conteneur

Erreur : port accessible depuis le conteneur mais pas depuis l'hôte
Cause  : le port n'est pas publié avec -p
Fix    : relancer avec -p port_hote:port_conteneur
```

---

## Récapitulatif du Module 3

| Concept | Ce qu'il faut retenir |
|---------|----------------------|
| **Conteneur** | Processus isolé qui partage le noyau hôte. Plus léger qu'une VM. |
| **docker0** | Bridge réseau virtuel créé par Docker. Hub central auquel les conteneurs se connectent. |
| **veth** | Paire de câbles virtuels reliant un conteneur au bridge docker0. |
| **NAT Docker** | Docker configure iptables automatiquement pour que les conteneurs accèdent à Internet. |
| **Bridge mode** | Mode par défaut. Réseau isolé avec NAT. DNS uniquement sur les réseaux personnalisés. |
| **Host mode** | Partage le réseau de l'hôte. Pas de NAT. Performance maximale mais sans isolation. |
| **None mode** | Isolation totale. Aucun accès réseau. |
| **Overlay mode** | Réseau multi-hôtes. Nécessite Docker Swarm ou orchestrateur. |
| **DNS Docker** | Actif uniquement sur les réseaux personnalisés. Résout les noms de conteneurs. |
| **127.0.0.11** | Serveur DNS embarqué Docker. Présent dans `/etc/resolv.conf` des conteneurs. |
| **EXPOSE** | Documentation uniquement — ne publie pas de port. |
| **-p host:container** | Publie réellement un port. Crée une règle iptables de redirection. |
| **0.0.0.0 vs 127.0.0.1** | 0.0.0.0 = accessible partout. 127.0.0.1 = loopback uniquement. |
| **Compose réseau** | Docker Compose crée automatiquement un réseau par projet avec DNS entre services. |
| **Service discovery** | Dans Compose, les services se parlent par leur nom de service (pas par IP). |
| **docker network inspect** | Voir la config d'un réseau, les conteneurs connectés, leurs IPs. |
| **netshoot** | Image Docker avec tous les outils réseau. Indispensable pour débugger. |

---

*→ Exercices pratiques : voir `module3-exercices.md`*
*→ Module suivant : Module 4 — AWS Networking + Kubernetes*
