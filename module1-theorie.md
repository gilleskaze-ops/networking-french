# Module 1 — Les bases absolues du réseau
### Partie Théorie

> **Cours Networking Fondamentaux · Orienté Cloud**
> Prérequis : aucun · Durée estimée : 1h30

---

## Glossaire

Tous les termes techniques utilisés dans ce module, expliqués en français.

| Terme | Signification | Définition simple |
|-------|--------------|-------------------|
| **ACK** | Acknowledgment | Message de confirmation : "j'ai bien reçu" |
| **ACM** | AWS Certificate Manager | Service AWS qui émet et renouvelle automatiquement les certificats TLS |
| **ALB** | Application Load Balancer | Reverse proxy AWS couche 7 — route le trafic HTTP selon l'URL |
| **CA** | Certificate Authority | Organisme de confiance qui signe les certificats TLS (ex: Let's Encrypt, DigiCert) |
| **CIDR** | Classless Inter-Domain Routing | Notation compacte pour définir la taille d'un réseau (ex: `/24`) |
| **Classe IP** | — | Ancien système de catégorisation des IPs (A=/8, B=/16, C=/24) — remplacé par CIDR en 1993 |
| **DHCP** | Dynamic Host Configuration Protocol | Attribue automatiquement une IP privée à une machine qui rejoint un réseau |
| **DNS** | Domain Name System | L'annuaire d'Internet : traduit `google.com` en adresse IP |
| **Forward Proxy** | — | Proxy placé du côté client — parle au nom du client vers Internet |
| **Header** | — | Entête ajouté par chaque couche réseau pour transporter ses informations |
| **ICMP** | Internet Control Message Protocol | Protocole de diagnostic réseau — utilisé par `ping` |
| **HTTP** | HyperText Transfer Protocol | Le protocole qui fait fonctionner le Web |
| **HTTPS** | HTTP Secure | HTTP + chiffrement TLS |
| **IP** | Internet Protocol | Le système d'adressage des machines sur un réseau |
| **IPv4** | Internet Protocol version 4 | Format d'IP classique : 4 nombres séparés par des points (ex: `192.168.1.1`) |
| **IPv6** | Internet Protocol version 6 | Format d'IP nouvelle génération en hexadécimal — 128 bits, pas besoin de NAT |
| **LAN** | Local Area Network | Réseau local — machines qui se parlent directement |
| **Let's Encrypt** | — | CA gratuite et automatisée qui émet des certificats TLS renouvelés tous les 90 jours |
| **MAC** | Media Access Control | Adresse physique unique d'une carte réseau — utilisée à la couche 2 |
| **NAT** | Network Address Translation | Traduit les IPs privées en IP publique pour accéder à Internet |
| **NLB** | Network Load Balancer | Reverse proxy AWS couche 4 — load balancing TCP/UDP ultra-rapide |
| **NTP** | Network Time Protocol | Protocole qui synchronise l'horloge d'une machine avec des serveurs de temps — UDP port 123 |
| **OSI** | Open Systems Interconnection | Modèle de référence en 7 couches décrivant comment les données voyagent |
| **PAT** | Port Address Translation | Variante de NAT où plusieurs machines partagent une IP publique grâce aux ports |
| **Port** | — | Numéro identifiant un service sur une machine (ex: 443 = HTTPS) |
| **Protocole** | — | Ensemble de règles que deux machines suivent pour se comprendre |
| **Resolver DNS** | — | Serveur qui résout les noms DNS à ta place (souvent ta box internet) |
| **Reverse Proxy** | — | Proxy placé du côté serveur — parle au nom du serveur vers les clients |
| **Routeur** | — | Appareil qui fait circuler les données entre deux réseaux différents |
| **Security Group** | — | Pare-feu virtuel AWS qui contrôle le trafic entrant/sortant d'une ressource |
| **SSL** | Secure Sockets Layer | Ancêtre de TLS — le terme est encore utilisé mais TLS l'a remplacé |
| **Subnet** | Subnetwork | Sous-réseau : une subdivision d'un réseau plus grand |
| **SYN** | Synchronize | Premier message d'une connexion TCP |
| **TCP** | Transmission Control Protocol | Transport fiable avec handshake et accusés de réception |
| **TLS** | Transport Layer Security | Protocole de chiffrement et d'authentification utilisé par HTTPS |
| **TTL** | Time To Live | Durée de vie d'une information en cache ou compteur de hops d'un paquet |
| **UDP** | User Datagram Protocol | Transport rapide sans garantie de livraison |
| **VPC** | Virtual Private Cloud | Réseau privé virtuel dans le Cloud AWS |
| **WAF** | Web Application Firewall | Pare-feu qui analyse le contenu HTTP pour bloquer les attaques |
| **WAN** | Wide Area Network | Réseau étendu — communication entre réseaux différents via Internet |

---

## Sommaire

1. [C'est quoi un réseau ?](#1-cest-quoi-un-réseau-)
2. [Les adresses IP](#2-les-adresses-ip)
3. [Les masques de sous-réseau](#3-les-masques-de-sous-réseau)
4. [CIDR](#4-cidr)
5. [TCP et UDP — comment les données voyagent](#5-tcp-et-udp--comment-les-données-voyagent)
6. [DNS — comment les noms deviennent des adresses](#6-dns--comment-les-noms-deviennent-des-adresses)
7. [HTTP, HTTPS et les ports](#7-http-https-et-les-ports)
8. [DHCP — comment une machine obtient son IP automatiquement](#8-dhcp--comment-une-machine-obtient-son-ip-automatiquement)
9. [ICMP — tester si une machine est joignable](#9-icmp--tester-si-une-machine-est-joignable)
10. [Les couches réseau — OSI et TCP/IP](#10-les-couches-réseau--osi-et-tcpip)
11. [NTP — synchroniser les horloges des machines](#11-ntp--synchroniser-les-horloges-des-machines)
12. [NAT — comment les IPs privées accèdent à Internet](#12-nat--comment-les-ips-privées-accèdent-à-internet)
13. [IPv6 — le successeur d'IPv4](#13-ipv6--le-successeur-dipv4)
14. [TLS — comment fonctionne le chiffrement](#14-tls--comment-fonctionne-le-chiffrement)
15. [Proxy et Reverse Proxy](#15-proxy-et-reverse-proxy)

---

## 1. C'est quoi un réseau ?

### Pourquoi c'est important

Quand tu déploies une application dans le Cloud, tu ne déploies pas juste du code. Tu déploies des machines qui doivent **se parler entre elles** et **parler à tes utilisateurs**. Sans réseau, rien ne fonctionne. Une base de données qui ne peut pas recevoir de connexions est inutile. Un serveur qui ne peut pas répondre aux requêtes est invisible.

Comprendre le réseau, c'est comprendre **pourquoi ton app est joignable ou pas**, pourquoi deux services arrivent ou n'arrivent pas à communiquer, et comment contrôler qui a le droit de parler à quoi.

### Ce qu'est un réseau

Un réseau, c'est un ensemble de machines capables de **s'envoyer des données**. Ces machines peuvent être dans la même pièce ou à l'autre bout du monde — ce qui change, c'est la façon dont elles sont reliées.

Internet est simplement le plus grand réseau du monde : des milliards de machines connectées entre elles via des câbles, des fibres optiques et des ondes radio, toutes en train de s'échanger des données en permanence.

### Réseau local vs réseau étendu

Il existe deux grandes catégories de réseaux :

- Le **LAN** (réseau local) regroupe des machines qui se parlent directement, sans passer par Internet. C'est rapide, c'est sécurisé, et c'est ce que tu crées dans le Cloud avec un VPC.

- Le **WAN** (réseau étendu) connecte des réseaux locaux entre eux, via Internet. Quand un utilisateur ouvre ton application depuis son téléphone, il passe par le WAN.

### À quoi ça ressemble dans le Cloud

Voici l'architecture réseau d'une application Cloud typique. Chaque flèche représente une communication réseau qui doit être explicitement autorisée :

```
[Utilisateur]
      |
      ↓  WAN — Internet (réseau public)
[Load Balancer]          ← seul élément exposé publiquement
      |
      ↓  LAN — VPC (réseau privé dans le Cloud)
[App Server 1]  [App Server 2]
      |
      ↓  Subnet isolé (encore plus privé)
[Base de données]  [Cache Redis]
```

La base de données et le cache ne sont **jamais** accessibles depuis Internet. Ils vivent dans un subnet privé, accessible uniquement par les serveurs d'application. C'est le réseau qui enforce cette règle.

---

## 2. Les adresses IP

### Pourquoi c'est important

Sur un réseau, les machines se reconnaissent grâce à leurs adresses IP. C'est leur identité réseau. Sans adresse IP, une machine ne peut ni envoyer ni recevoir de données — elle est invisible.

Dans le Cloud, chaque service que tu crées reçoit automatiquement une IP privée. Certains reçoivent aussi une IP publique. Savoir lire et manipuler ces adresses, c'est savoir où sont tes services et comment les joindre.

### IPv4 — le format classique

L'**IPv4** est le format d'adresse IP le plus utilisé aujourd'hui. Une adresse IPv4 se compose de **4 nombres** entre 0 et 255, séparés par des points :

```
192.168.1.42
```

Chacun de ces 4 nombres est appelé un **octet** — parce qu'il représente 8 bits. Au total, une adresse IPv4 = 32 bits.

Avec 32 bits, on peut créer environ **4 milliards d'adresses** différentes. Ça semble énorme, mais Internet a épuisé ce stock — d'où l'existence d'IPv6.

### IPv6 — le successeur

L'**IPv6** utilise 128 bits au lieu de 32. Il peut créer un nombre astronomique d'adresses (340 sextillions). Son format est moins lisible :

```
2001:0db8:85a3:0000:0000:8a2e:0370:7334
```

IPv6 existe et fonctionne, mais en pratique dans le Cloud on travaille encore majoritairement en IPv4. Ce cours se concentre sur IPv4.

### IPs privées vs IPs publiques

Toutes les IPs ne sont pas égales. Il en existe deux grandes familles :

Les **IPs publiques** sont uniques dans le monde entier et accessibles depuis n'importe où sur Internet. Quand ton navigateur contacte `54.12.34.56`, il atteint cette machine précise, peu importe où tu es.

Les **IPs privées** sont réservées aux réseaux internes. Elles ne sont jamais transmises sur Internet. Le même `192.168.1.1` peut exister dans des millions de réseaux différents sans conflit — parce qu'il ne sort jamais.

Les plages d'adresses privées sont définies par convention :

| Plage | Ce que ça représente | Utilisation typique dans le Cloud |
|-------|---------------------|----------------------------------|
| `10.0.0.0` à `10.255.255.255` | 16 millions d'adresses | VPC AWS, grands réseaux d'entreprise |
| `172.16.0.0` à `172.31.255.255` | 1 million d'adresses | Docker, réseaux intermédiaires |
| `192.168.0.0` à `192.168.255.255` | 65 000 adresses | Box internet, petits réseaux |

Dans ton VPC AWS, **toutes tes machines ont des IPs privées**. Seuls les éléments exposés à Internet (Load Balancer, NAT Gateway) ont une IP publique.

### Les classes IP — comprendre le vocabulaire historique

Avant CIDR, les adresses IPv4 étaient divisées en **classes fixes**. Ce système n'est plus utilisé en pratique, mais le vocabulaire reste très présent dans la documentation et les conversations réseau — donc il faut savoir le reconnaître.

| Classe | Plage | Taille réseau | Usage historique |
|--------|-------|--------------|-----------------|
| **Classe A** | `1.0.0.0` → `126.255.255.255` | `/8` — 16 millions de machines | Très grandes organisations |
| **Classe B** | `128.0.0.0` → `191.255.255.255` | `/16` — 65 536 machines | Organisations moyennes |
| **Classe C** | `192.0.0.0` → `223.255.255.255` | `/24` — 256 machines | Petits réseaux |
| **Classe D** | `224.0.0.0` → `239.255.255.255` | — | Multicast |
| **Classe E** | `240.0.0.0` → `255.255.255.255` | — | Réservé / expérimental |

Le problème de ce système : la taille était **fixe et rigide**. Une entreprise avec 300 machines ne rentrait pas dans une Classe C (256 max) et devait prendre une Classe B (65 536) — gaspillant des dizaines de milliers d'adresses. C'est l'une des raisons pour lesquelles IPv4 s'est épuisé aussi vite.

**CIDR a remplacé les classes en 1993** — en permettant n'importe quelle taille de réseau (`/25`, `/22`, `/18`...) au lieu de tailles fixes.

Le lien direct avec les plages privées qu'on utilise aujourd'hui :

```
10.0.0.0/8        →  anciennement "Classe A privée"
172.16.0.0/12     →  anciennement "Classe B privée"
192.168.0.0/16    →  anciennement "Classe C privée"
```

> Aujourd'hui dans le Cloud, on ne parle jamais de "Classe B". On dit `/16`. Mais quand tu lis de la documentation ancienne ou que tu parles à un ingénieurs réseau senior, tu entendras ces termes — et maintenant tu sais ce qu'ils signifient.

---

## 3. Les masques de sous-réseau

### Pourquoi c'est important

Sur un réseau, toutes les machines ne doivent pas forcément se parler. Ta base de données n'a pas besoin d'être au même "niveau" que ton Load Balancer. Le masque de sous-réseau est l'outil qui permet de **diviser un grand réseau en zones séparées**. C'est ce qui te permet de dire : "ces machines vivent dans la zone publique, celles-là dans la zone privée".

### Comment ça fonctionne

Une adresse IP a deux parties : une partie qui identifie le **réseau**, et une partie qui identifie la **machine** dans ce réseau.

Le masque de sous-réseau indique où se trouve la frontière entre les deux.

Prenons un exemple concret :

```
Adresse IP : 192.168.1.42
Masque     : 255.255.255.0

             192.168.1    .    42
             └──────────┘      └─┘
               réseau         machine
```

Le `255` dans le masque signifie "cette partie appartient au réseau".
Le `0` signifie "cette partie identifie la machine".

Conséquences pratiques :

- `192.168.1.5` et `192.168.1.200` sont sur le **même réseau** → elles se parlent directement.
- `192.168.1.5` et `192.168.2.5` sont sur des **réseaux différents** → elles ont besoin d'un routeur pour communiquer.

### Pourquoi c'est utile dans le Cloud

Dans AWS, chaque subnet a son propre masque. Ça définit combien de machines peuvent y vivre et quelles machines peuvent se parler directement. Un subnet public et un subnet privé sont délibérément sur des réseaux différents — c'est le masque (et les règles réseau) qui empêche qu'ils se mélangent.

---

## 4. CIDR

### Pourquoi c'est important

Le CIDR est la notation que tu verras **absolument partout** dans le Cloud : dans la console AWS, dans Terraform, dans les Security Groups, dans la documentation. C'est le langage universel pour parler de la taille d'un réseau. Impossible d'y échapper.

### Ce que c'est

**CIDR** (Classless Inter-Domain Routing) est une notation compacte qui remplace le masque de sous-réseau par un simple nombre. Ce nombre représente **combien de bits sont réservés pour identifier le réseau**.

Au lieu d'écrire `255.255.255.0`, on écrit `/24`.

```
10.0.0.0/16

└──────┘ └──┘
  IP    préfixe (nombre de bits réseau)
```

### Lire un CIDR

Le préfixe `/X` te dit directement combien d'adresses sont disponibles :

```
32 bits au total dans une IPv4

/24 → 24 bits pour le réseau, 8 bits pour les machines
    → 2^8 = 256 adresses

/16 → 16 bits pour le réseau, 16 bits pour les machines
    → 2^16 = 65 536 adresses

/32 → 32 bits pour le réseau, 0 bit pour les machines
    → 1 seule adresse (cette IP précise, rien d'autre)
```

Plus le préfixe est grand, plus le réseau est **petit**.

### Référence rapide

| CIDR | Adresses totales | Utilisables dans AWS* | Usage courant |
|------|-----------------|----------------------|---------------|
| `/8` | 16 777 216 | 16 777 211 | VPC très grand |
| `/16` | 65 536 | 65 531 | VPC standard |
| `/24` | 256 | 251 | Subnet applicatif |
| `/28` | 16 | 11 | Petit subnet (BDD) |
| `/32` | 1 | — | Une IP précise dans une règle Security Group |

> AWS réserve toujours **5 adresses** dans chaque subnet : la première (adresse réseau), les trois suivantes (routeur, DNS, usage futur), et la dernière (broadcast). Un `/24` donne donc 251 adresses utilisables, pas 256.

### Architecture Cloud typique avec CIDR

```
VPC : 10.0.0.0/16          ← ton réseau privé global (65 531 adresses)
  ├── 10.0.1.0/24          ← subnet public  (Load Balancer, NAT)
  ├── 10.0.2.0/24          ← subnet public  (redondance, autre zone géographique)
  ├── 10.0.10.0/24         ← subnet privé   (serveurs d'application)
  ├── 10.0.11.0/24         ← subnet privé   (redondance)
  └── 10.0.20.0/24         ← subnet BDD     (base de données, cache)
```

Chaque subnet est un sous-bloc du VPC. Ils ne se chevauchent pas. C'est toi qui définis ce découpage — et c'est une décision architecturale importante.

---

## 5. TCP et UDP — comment les données voyagent

### Pourquoi c'est important

Envoyer des données sur un réseau, ce n'est pas comme copier un fichier d'un dossier à un autre. Les données sont découpées en petits morceaux (**paquets**), envoyées séparément sur le réseau, et réassemblées à destination. Certains paquets peuvent se perdre. Certains peuvent arriver dans le mauvais ordre.

**TCP et UDP** sont deux façons différentes de gérer ce problème. Choisir l'un ou l'autre a des conséquences directes sur les performances et la fiabilité de ton application. Et dans le Cloud, les firewalls filtrent précisément au niveau TCP/UDP.

### Le modèle en couches

Pour bien comprendre TCP et UDP, il faut savoir qu'un réseau fonctionne en **couches**. Chaque couche a une responsabilité précise et s'appuie sur la couche en dessous :

```
┌──────────────────────────────────────────────────────┐
│  APPLICATION   HTTP, DNS, SSH, SMTP...                │  ← ton code, tes APIs
├──────────────────────────────────────────────────────┤
│  TRANSPORT     TCP, UDP                               │  ← fiabilité, ports
├──────────────────────────────────────────────────────┤
│  INTERNET      IP, ICMP                               │  ← adressage, routage
├──────────────────────────────────────────────────────┤
│  ACCÈS RÉSEAU  Ethernet, WiFi                         │  ← physique
└──────────────────────────────────────────────────────┘
```

TCP et UDP sont tous les deux dans la couche **Transport**. Ils s'occupent de faire voyager les données entre deux machines, chacun avec sa philosophie.

### TCP — la fiabilité avant tout

**TCP** (Transmission Control Protocol) est le protocole qui **garantit** que toutes les données arrivent à destination, dans le bon ordre, sans perte.

Pour ce faire, il établit d'abord une connexion via un échange en 3 étapes qu'on appelle le **handshake** :

```
Client                              Serveur
  ──── SYN ─────────────────────────►   "Je veux me connecter"
  ◄─── SYN-ACK ─────────────────────   "D'accord, je suis prêt"
  ──── ACK ─────────────────────────►   "Compris, on commence"

  ════ DONNÉES ═════════════════════►   Flux fiable et ordonné
```

Après le handshake, chaque paquet envoyé doit être confirmé par un **ACK**. Si le serveur ne reçoit pas d'ACK, il renvoie le paquet automatiquement.

**Utilisé par** : HTTP, HTTPS, SSH, PostgreSQL, MySQL, Redis, SMTP — tout ce qui ne peut pas se permettre de perdre des données.

### UDP — la vitesse avant tout

**UDP** (User Datagram Protocol) envoie les données directement, sans établir de connexion, sans vérifier si elles sont arrivées.

```
Client                              Serveur
  ──── paquet ──────────────────────►
  ──── paquet ──────────────────────►   (peut arriver dans n'importe quel ordre)
  ──── paquet ──────────────────────►   (un paquet peut se perdre sans problème)
```

C'est moins fiable, mais beaucoup plus rapide. Et pour certains usages, perdre un paquet de temps en temps n'est pas grave — un pixel manquant dans une vidéo en streaming est invisible.

**Utilisé par** : DNS, streaming vidéo, jeux en ligne, VoIP — tout ce qui a besoin de faible latence.

### Comparatif rapide

| | TCP | UDP |
|---|---|---|
| Connexion préalable | Oui (handshake) | Non |
| Garantie de livraison | Oui | Non |
| Ordre des paquets | Garanti | Non garanti |
| Vitesse | Plus lent | Plus rapide |
| Cas d'usage | API, BDD, SSH | DNS, streaming, jeux |

### Le lien avec le Cloud

Quand tu configures un **Security Group** AWS, chaque règle précise le protocole : TCP ou UDP. Autoriser le port 443 TCP ne laisse passer que du HTTPS. Autoriser le port 53 UDP laisse passer les requêtes DNS. Ce n'est pas interchangeable.

---

## 6. DNS — comment les noms deviennent des adresses

### Pourquoi c'est important

Les machines parlent en IPs. Les humains parlent en noms. Le **DNS** fait le pont entre les deux. Sans DNS, tu devrais retenir l'IP de chaque service que tu utilises — et la mettre à jour dans tout ton code à chaque fois qu'elle change.

Dans le Cloud, le DNS est encore plus crucial : il te permet de nommer tes services internes (`database.prod.internal`, `api.prod.internal`), de faire de la redirection de trafic, et même de la répartition de charge. Route 53, le service DNS d'AWS, est un outil architectural à part entière.

### Comment fonctionne une résolution DNS

Quand tu tapes `github.com` dans ton navigateur, voici ce qui se passe en coulisses, en moins de 50ms :

```
1. [Navigateur]           → vérifie son cache local
                            (a-t-il déjà résolu ce nom récemment ?)

2. [Resolver DNS]         → ton serveur DNS (souvent ta box ou 8.8.8.8)
                            pose la question à sa place

3. [Serveur Racine]       → "je ne connais pas github.com,
                            mais je sais qui gère .com"

4. [Serveur TLD .com]     → "je ne connais pas github.com,
                            mais je sais qui gère github.com"

5. [Serveur Authoritative]→ "github.com = 140.82.121.4"
                            (c'est Route 53, Cloudflare, etc.)

6. [Navigateur]           → connexion TCP vers 140.82.121.4:443
```

### Les types d'enregistrements DNS essentiels

Un serveur DNS stocke des **enregistrements** (records). Chaque type a un rôle précis :

| Type | Ce qu'il fait | Exemple concret |
|------|--------------|-----------------|
| `A` | Traduit un nom en IPv4 | `api.monapp.com` → `54.12.34.56` |
| `AAAA` | Traduit un nom en IPv6 | `api.monapp.com` → `2001:db8::1` |
| `CNAME` | Redirige un nom vers un autre nom | `www.monapp.com` → `monapp.com` |
| `MX` | Indique le serveur mail du domaine | `monapp.com` → `mail.monapp.com` |
| `TXT` | Stocke du texte libre | Vérification de domaine, anti-spam |
| `NS` | Indique quels serveurs font autorité pour ce domaine | `monapp.com` → `ns1.route53.aws` |

### Le TTL — pourquoi il faut y penser

Chaque enregistrement DNS a un **TTL** (Time To Live) exprimé en secondes. C'est la durée pendant laquelle la réponse est mise en cache par les resolvers dans le monde entier.

```
api.monapp.com   A   54.12.34.56   TTL 3600
                                       └────── résultat gardé en cache 1 heure
```

Si tu changes l'IP dans ton DNS, les utilisateurs dont le cache contient encore l'ancienne réponse continueront de se connecter à l'ancienne IP pendant encore `TTL` secondes.

**Bonne pratique** : avant une migration, baisse le TTL à 60 secondes la veille. Le changement se propagera en 1 minute au lieu de 1 heure.

### DNS privé dans le Cloud

Dans un VPC AWS, chaque machine reçoit automatiquement un nom DNS interne peu lisible :

```
ip-10-0-1-42.eu-west-1.compute.internal
```

Avec **Route 53 Private Hosted Zone**, tu peux définir tes propres noms internes :

```
database.prod.internal    →  10.0.20.5
cache.prod.internal       →  10.0.20.10
api.prod.internal         →  10.0.10.25
```

Tes services se parlent par nom, pas par IP. Si tu changes l'IP d'une machine (migration, scale-up), tu mets à jour uniquement l'enregistrement DNS — aucun code à modifier.

---

## 7. HTTP, HTTPS et les ports

### Pourquoi c'est important

HTTP et HTTPS sont les protocoles du Web — c'est via eux que ton navigateur parle à un serveur, que tes APIs se répondent, que les services Cloud échangent des données. Les comprendre, c'est comprendre le langage dans lequel s'exprime ton application.

Les **ports**, eux, sont le mécanisme qui permet à une machine de faire tourner plusieurs services en parallèle. Ils sont au cœur de toutes les règles de sécurité réseau dans le Cloud.

### Les ports — les "portes" d'une machine

Une machine n'a qu'une seule adresse IP, mais elle peut gérer des dizaines de services simultanément. Les **ports** permettent de distinguer quel service est visé.

Imagine une grande entreprise : l'adresse de l'immeuble est la même pour tout le monde, mais chaque bureau a un numéro. L'IP, c'est l'adresse de l'immeuble. Le port, c'est le numéro de bureau.

```
10.0.1.42:22    →  service SSH       (bureau 22)
10.0.1.42:80    →  serveur HTTP      (bureau 80)
10.0.1.42:443   →  serveur HTTPS     (bureau 443)
10.0.1.42:5432  →  base PostgreSQL   (bureau 5432)
```

Les ports vont de 0 à 65535. Les premiers (0-1023) sont réservés aux protocoles standard.

### Ports à connaître absolument

| Port | Protocole | Usage |
|------|-----------|-------|
| 22 | SSH / TCP | Connexion sécurisée à un serveur Linux |
| 53 | DNS / UDP+TCP | Résolution de noms de domaine |
| 80 | HTTP / TCP | Web non chiffré |
| 443 | HTTPS / TCP | Web chiffré |
| 3306 | MySQL / TCP | Base de données MySQL |
| 5432 | PostgreSQL / TCP | Base de données PostgreSQL |
| 6379 | Redis / TCP | Cache et base de données Redis |
| 27017 | MongoDB / TCP | Base de données MongoDB |
| 8080 | HTTP-alt / TCP | Applications web en développement |

### HTTP — le protocole du Web

**HTTP** (HyperText Transfer Protocol) est le protocole qui définit comment un client (navigateur, API) et un serveur échangent des données. C'est un protocole basé sur du **texte** : les requêtes et réponses sont lisibles par un humain.

Une requête HTTP ressemble à ça :

```
GET /api/users/42 HTTP/1.1
Host: api.monapp.com
Authorization: Bearer eyJhbGc...
Content-Type: application/json
```

Et une réponse :

```
HTTP/1.1 200 OK
Content-Type: application/json

{"id": 42, "name": "Alice"}
```

### Les codes de statut HTTP

Le serveur répond toujours avec un **code de statut** qui indique si la requête a réussi ou non :

| Famille | Signification | Exemples |
|---------|--------------|---------|
| `2xx` | Succès | `200 OK`, `201 Created`, `204 No Content` |
| `3xx` | Redirection | `301 Moved Permanently`, `302 Found` |
| `4xx` | Erreur côté client | `400 Bad Request`, `401 Unauthorized`, `403 Forbidden`, `404 Not Found` |
| `5xx` | Erreur côté serveur | `500 Internal Error`, `502 Bad Gateway`, `503 Service Unavailable` |

> Un `4xx` signifie que c'est le client qui a mal demandé. Un `5xx` signifie que le serveur a eu un problème pour répondre. La distinction est importante pour débugger.

### HTTP vs HTTPS

**HTTP** (port 80) envoie tout en clair. N'importe qui capable d'intercepter le trafic entre le client et le serveur peut lire les données, y compris les mots de passe et les tokens.

**HTTPS** (port 443) ajoute une couche **TLS** (Transport Layer Security) qui chiffre toutes les données échangées. Même si quelqu'un intercepte le trafic, il ne voit que du bruit illisible.

En production, HTTP seul n'est plus acceptable. Tous les navigateurs modernes affichent un avertissement sur les sites non HTTPS. AWS propose des certificats TLS gratuits via **ACM** (AWS Certificate Manager).

---

## 8. DHCP — Comment une machine obtient son IP automatiquement

### Pourquoi c'est important

Tu n'as jamais eu à configurer manuellement l'IP de ton téléphone quand tu te connectes à un WiFi. Tu branches, tu te connectes, et ça marche. C'est le **DHCP** qui fait ça en coulisses. Comprendre DHCP, c'est comprendre ce qui se passe dans les premières secondes où une machine rejoint un réseau — et pourquoi parfois ça échoue.

### Le problème que DHCP résout

Quand une machine arrive sur un réseau, elle a besoin de trois informations pour fonctionner :

- Son **IP privée** (ex: `192.168.1.42`)
- Le **masque** du réseau (ex: `/24`)
- L'adresse du **routeur** (ex: `192.168.1.1` — la Fritz!Box)

Sans ces trois infos, elle ne peut ni parler aux autres machines ni accéder à Internet. DHCP les fournit automatiquement.

### Comment ça fonctionne

Quand ta machine rejoint un réseau, elle ne connaît personne. Elle ne sait même pas quelle IP elle va avoir. Elle utilise alors le **broadcast** pour crier dans le réseau :

```
Nouvelle machine arrive sur le réseau.
Elle n'a pas encore d'IP.

1. DISCOVER — elle envoie en broadcast (192.168.1.255) :
   "Il y a un serveur DHCP ici ?
    J'ai besoin d'une adresse IP !"

2. OFFER — la Fritz!Box répond :
   "Oui, je suis là. Je t'offre 192.168.1.42"

3. REQUEST — la machine confirme :
   "Je prends le 192.168.1.42, merci !"

4. ACK — la Fritz!Box valide :
   "C'est confirmé. Elle est à toi pour 24h."
```

Ce processus en 4 étapes s'appelle **DORA** (Discover, Offer, Request, Acknowledge). Il se déroule en moins d'une seconde.

### Le bail DHCP

L'IP attribuée n'est pas permanente — c'est un **bail** (lease) avec une durée limitée, souvent 24h. Avant l'expiration, la machine renouvelle automatiquement le bail. Si elle ne le renouvelle pas (machine éteinte, déconnectée), l'IP est libérée et peut être donnée à une autre machine.

C'est pourquoi l'IP privée de ton téléphone peut changer d'un jour à l'autre sur ton WiFi domestique.

### DHCP dans le Cloud

Dans un VPC AWS, chaque subnet a son propre serveur DHCP géré automatiquement par AWS. Quand tu crées une EC2, elle reçoit instantanément son IP privée via DHCP — tu n'as rien à configurer.

```
Tu crées une EC2 dans subnet 10.0.10.0/24
       ↓
AWS DHCP attribue automatiquement : 10.0.10.55
       ↓
L'EC2 connaît son IP, son masque, et l'adresse du routeur VPC
       ↓
Elle peut immédiatement communiquer avec les autres ressources
```

> Dans AWS, il est possible de réserver une IP fixe pour une ressource (**IP statique**) pour éviter qu'elle change. C'est ce qu'on appelle une **Elastic IP** pour les IPs publiques, ou simplement assigner une IP privée fixe dans la configuration de l'EC2.

---

## 9. ICMP — Tester si une machine est joignable

### Pourquoi c'est important

Quand quelque chose ne fonctionne pas sur un réseau, la première question est toujours : *"la machine est-elle seulement joignable ?"*. ICMP est l'outil qui répond à cette question. C'est le protocole le plus simple du réseau, et pourtant le plus utilisé pour diagnostiquer les problèmes.

### Ce que c'est

**ICMP** (Internet Control Message Protocol) est un protocole de diagnostic. Contrairement à TCP et UDP qui transportent des données applicatives, ICMP sert uniquement à **envoyer des messages de contrôle** entre machines.

Il ne transporte pas de données métier — pas de fichiers, pas de requêtes SQL, pas de pages web. Juste des messages du type : *"es-tu là ?"*, *"je suis injoignable"*, *"ce réseau n'existe pas"*.

### La commande ping

`ping` est la commande qui utilise ICMP. Elle envoie un message **ICMP Echo Request** à une machine et attend un **ICMP Echo Reply** en retour :

```bash
ping google.com

# Résultat :
PING google.com (142.250.74.46)
64 bytes from 142.250.74.46: icmp_seq=1 ttl=118 time=12.4 ms
64 bytes from 142.250.74.46: icmp_seq=2 ttl=118 time=11.8 ms
64 bytes from 142.250.74.46: icmp_seq=3 ttl=118 time=12.1 ms

--- google.com ping statistics ---
3 packets transmitted, 3 received, 0% packet loss
avg = 12.1 ms
```

Ce que te dit ce résultat :
- La machine `142.250.74.46` est **joignable** ✅
- Chaque aller-retour prend environ **12ms** (c'est la latence)
- **0% de perte** de paquets — la connexion est stable

### Lire un résultat ping

| Résultat | Ce que ça signifie |
|----------|-------------------|
| Réponses normales, latence faible | Machine joignable, connexion saine |
| `Request timeout` | La machine ne répond pas — éteinte, bloquée par firewall, ou ICMP désactivé |
| `Destination host unreachable` | Le réseau lui-même est inaccessible — problème de routage |
| Latence très élevée (>500ms) | Connexion lente ou saturée |
| Perte de paquets > 0% | Connexion instable |

### ICMP et les firewalls

ICMP peut être **bloqué** par un firewall. Dans ce cas, `ping` ne reçoit aucune réponse — mais ça ne veut pas dire que la machine est éteinte. Ça veut juste dire que le firewall bloque ICMP.

C'est un piège classique en Cloud : une EC2 qui ne répond pas au ping n'est pas forcément en panne. Peut-être que son Security Group n'autorise tout simplement pas ICMP.

```
# Sur AWS, pour autoriser le ping vers une EC2,
# il faut ajouter cette règle dans le Security Group :
Type     : ICMP
Protocole: ICMP
Port     : — (ICMP n'utilise pas de ports)
Source   : ton IP ou 0.0.0.0/0
```

### La différence avec TCP/UDP

ICMP n'utilise **pas de ports**. C'est une distinction importante : quand tu configures un Security Group AWS, les règles TCP/UDP spécifient des ports (22, 443, 5432...). Une règle ICMP, elle, s'applique au protocole entier — pas à un port spécifique.

---

## 10. Les couches réseau — OSI et TCP/IP

### Pourquoi c'est important

Quand deux machines s'échangent des données, ce n'est pas un processus monolithique — c'est une série d'étapes organisées en **couches**, chacune avec une responsabilité précise. Comprendre ces couches, c'est comprendre le langage que les ingénieurs réseau et Cloud utilisent tous les jours.

Quand un collègue dit *"le problème est à la couche 4"* ou *"ce Load Balancer travaille à la couche 7"*, il fait référence à ce modèle. Sans cette grille de lecture, ces phrases sont du bruit. Avec elle, elles deviennent précises et exploitables.

### Le modèle OSI — la référence théorique

Le modèle **OSI** (Open Systems Interconnection) est le modèle de référence qui découpe la communication réseau en **7 couches**. Il a été créé pour standardiser la façon dont les équipements et protocoles réseau interagissent.

C'est un modèle **théorique** — il ne correspond pas exactement à ce qui se passe dans les systèmes réels, mais il est universellement utilisé comme référence pour décrire et diagnostiquer les problèmes réseau.

```
┌─────┬──────────────────┬───────────────────────────────────────────────┐
│  7  │  APPLICATION     │ Interface avec l'utilisateur ou l'application  │
│     │                  │ HTTP, FTP, SMTP, DNS, SSH                      │
├─────┼──────────────────┼───────────────────────────────────────────────┤
│  6  │  PRÉSENTATION    │ Formatage, chiffrement, compression            │
│     │                  │ TLS/SSL, encodage UTF-8, JPEG, MP4             │
├─────┼──────────────────┼───────────────────────────────────────────────┤
│  5  │  SESSION         │ Ouverture, gestion et fermeture des sessions   │
│     │                  │ Authentification, reconnexion                  │
├─────┼──────────────────┼───────────────────────────────────────────────┤
│  4  │  TRANSPORT       │ Livraison des données, contrôle de flux        │
│     │                  │ TCP, UDP — ports source et destination         │
├─────┼──────────────────┼───────────────────────────────────────────────┤
│  3  │  RÉSEAU          │ Adressage logique et routage                   │
│     │                  │ IP, ICMP, routeurs                             │
├─────┼──────────────────┼───────────────────────────────────────────────┤
│  2  │  LIAISON         │ Transfert entre deux machines directement      │
│     │                  │ Ethernet, WiFi, adresses MAC                   │
├─────┼──────────────────┼───────────────────────────────────────────────┤
│  1  │  PHYSIQUE        │ Les bits sur le support physique               │
│     │                  │ Câbles, fibres, ondes radio, signaux           │
└─────┴──────────────────┴───────────────────────────────────────────────┘
```

**Astuce mémorisation** : *"Please Do Not Throw Sausage Pizza Away"* → Physical, Data link, Network, Transport, Session, Presentation, Application (de bas en haut).

### Le modèle TCP/IP — la réalité pratique

Le modèle **TCP/IP** est ce qui est réellement implémenté dans les systèmes modernes. Il simplifie les 7 couches OSI en **4 couches** en fusionnant certaines d'entre elles.

```
┌──────────────────────────────────────────────────────────────────────┐
│  COUCHE APPLICATION       OSI 5 + 6 + 7                              │
│  HTTP, HTTPS, DNS, SSH, SMTP, FTP...                                 │
│  → Ce que ton code produit et consomme                               │
├──────────────────────────────────────────────────────────────────────┤
│  COUCHE TRANSPORT         OSI 4                                      │
│  TCP, UDP                                                            │
│  → Segmentation des données, ports, fiabilité                        │
├──────────────────────────────────────────────────────────────────────┤
│  COUCHE INTERNET          OSI 3                                      │
│  IP, ICMP                                                            │
│  → Adressage IP, routage entre réseaux                               │
├──────────────────────────────────────────────────────────────────────┤
│  COUCHE ACCÈS RÉSEAU      OSI 1 + 2                                  │
│  Ethernet, WiFi, câbles, fibres                                      │
│  → Transmission physique des bits                                    │
└──────────────────────────────────────────────────────────────────────┘
```

### Les deux modèles côte à côte

```
    OSI (7 couches)              TCP/IP (4 couches)
┌─────────────────┐           ┌──────────────────────┐
│  7. Application │ ────────► │                      │
│  6. Présentation│ ────────► │    Application        │
│  5. Session     │ ────────► │                      │
├─────────────────┤           ├──────────────────────┤
│  4. Transport   │ ────────► │    Transport          │
├─────────────────┤           ├──────────────────────┤
│  3. Réseau      │ ────────► │    Internet           │
├─────────────────┤           ├──────────────────────┤
│  2. Liaison     │ ────────► │                      │
│  1. Physique    │ ────────► │    Accès réseau       │
└─────────────────┘           └──────────────────────┘
```

### Le voyage d'un paquet — de ton navigateur à Google

Voici ce qui se passe concrètement, couche par couche, quand tu tapes `https://google.com` :

**Côté émetteur (ton navigateur) — on descend les couches :**

```
Couche 7 — Application
  → HTTP construit la requête :
    "GET / HTTP/1.1  Host: google.com"

Couche 6 — Présentation
  → TLS chiffre la requête
    (les données deviennent illisibles pour un observateur)

Couche 4 — Transport
  → TCP encapsule les données
    + ajoute le port source (ex: 52341) et destination (443)
    + numéro de séquence pour garantir l'ordre

Couche 3 — Réseau
  → IP ajoute l'adresse source (ton IP) et destination (IP de Google)
    + TTL (durée de vie du paquet)

Couche 2 — Liaison
  → Ethernet ajoute l'adresse MAC source et destination
    (adresse physique de ta carte réseau → adresse de ta Fritz!Box)

Couche 1 — Physique
  → Les bits partent sur le câble ou en WiFi
```

**Sur le réseau — les routeurs intermédiaires :**

```
Chaque routeur entre toi et Google ne regarde que la Couche 3 (IP)
→ Il lit l'IP de destination
→ Il décide vers quel prochain routeur envoyer le paquet
→ Il ne touche pas aux couches 4, 5, 6, 7
```

**Côté récepteur (serveur Google) — on remonte les couches :**

```
Couche 1 → reçoit les bits
Couche 2 → vérifie l'adresse MAC
Couche 3 → vérifie l'IP de destination (c'est bien lui ?)
Couche 4 → TCP réassemble les segments dans l'ordre, envoie les ACK
Couche 6 → TLS déchiffre les données
Couche 7 → HTTP lit la requête et prépare la réponse
```

### L'encapsulation — l'emballage en couches

À chaque couche, les données sont **encapsulées** : on ajoute un **header** (entête) qui contient les informations nécessaires à cette couche. C'est comme des poupées russes.

```
┌─────────────────────────────────────────────────────┐
│ Header Ethernet │ Header IP │ Header TCP │ Données  │
└─────────────────────────────────────────────────────┘
       ↑                ↑            ↑          ↑
  Couche 2          Couche 3    Couche 4    Couche 7
  (MAC src/dst)   (IP src/dst) (ports,seq) (HTTP,TLS)
```

Chaque couche ne regarde que **son propre header** et ignore les autres. Le routeur (couche 3) ne sait pas que c'est du HTTPS — il voit juste une IP de destination.

### Pourquoi les couches sont importantes dans le Cloud

Dans le Cloud, les outils réseau opèrent à des couches précises — et ça change tout à ce qu'ils peuvent faire :

| Outil AWS | Couche OSI | Ce qu'il voit et fait |
|-----------|-----------|----------------------|
| **Security Group** | 3 et 4 | Filtre par IP source/destination et par port TCP/UDP |
| **Network ACL** | 3 et 4 | Idem, mais au niveau du subnet entier |
| **ALB** (Application Load Balancer) | 7 | Lit le contenu HTTP — peut router selon l'URL, les headers |
| **NLB** (Network Load Balancer) | 4 | Lit uniquement TCP/UDP — plus rapide, moins intelligent |
| **Route Table** | 3 | Décide vers quel gateway envoyer un paquet selon l'IP destination |
| **WAF** (Web Application Firewall) | 7 | Lit le contenu des requêtes HTTP pour bloquer les attaques |

Un **ALB** peut dire *"toutes les requêtes vers `/api/*` vont sur ce groupe de serveurs"* parce qu'il lit la couche 7 (HTTP). Un **NLB** ne peut pas faire ça — il ne voit que des ports et des IPs.

---

## 11. NTP — synchroniser les horloges des machines

### Pourquoi c'est important

Dans une architecture Cloud, tu as potentiellement des dizaines de serveurs qui tournent en parallèle — dans des régions différentes, des fuseaux horaires différents. Chacun génère des logs avec des timestamps. Si les horloges de ces machines ne sont pas synchronisées, déboguer un problème devient un cauchemar : tu essaies de reconstituer une séquence d'événements, mais les timestamps ne correspondent pas.

C'est pour ça que **NTP est l'un des premiers services configurés sur n'importe quel serveur**. AWS l'impose à toutes ses instances EC2 automatiquement.

### Ce que NTP fait

**NTP** (Network Time Protocol) synchronise l'horloge d'une machine avec des serveurs de temps de référence sur Internet. Ce n'est pas une synchronisation ponctuelle — c'est un ajustement **continu et permanent** qui corrige la dérive naturelle de l'horloge.

```
Problème : chaque machine a une horloge interne
           qui dérive de quelques millisecondes par heure.
           Après quelques jours sans synchro → décalage notable.

Solution NTP :
  La machine contacte périodiquement un serveur de temps
  et ajuste son horloge pour rester précise.
```

### Qui déclenche la synchronisation ?

C'est la **machine elle-même** qui initie — jamais le serveur. Au démarrage, le service NTP contacte un serveur de temps et demande l'heure. Ensuite il refait cette synchro automatiquement toutes les quelques minutes pour corriger la dérive.

```
Démarrage de la machine
        │
        ▼
Service NTP démarre automatiquement
(ntpd ou systemd-timesyncd sur Linux)
        │
        │  "quelle heure est-il ?" — UDP port 123
        ▼
Serveur NTP public
(pool.ntp.org, time.aws.com, time.google.com...)
        │
        │  "il est 13:32:05.123456 UTC"
        ▼
Machine ajuste son horloge
        │
        ▼
Répète périodiquement pour corriger la dérive
```

### On synchronise quoi exactement ?

On synchronise **l'heure UTC** — l'heure universelle, sans fuseau horaire. Chaque machine convertit ensuite en heure locale selon son fuseau configuré.

```
Serveur NTP          → 13:32:05 UTC
Ton PC (Berlin)      → affiche 15:32:05 (UTC+2 en été)
EC2 AWS (Irlande)    → affiche 13:32:05 (UTC+0)
EC2 AWS (Virginie)   → affiche 09:32:05 (UTC-4)

Mais dans les LOGS, tout est enregistré en UTC
→ les logs de toutes les machines sont comparables ✅
```

### Pourquoi UTC dans les logs ?

```
Sans NTP (horloges désynchronisées) :

  EC2 App  log : "requête envoyée  à 13:32:05.100"
  EC2 BDD  log : "requête reçue    à 13:32:08.200"

  → Tu crois qu'il y a 3 secondes de latence
  → En réalité c'est juste une horloge mal synchronisée
  → Débogage cauchemardesque ❌

Avec NTP (horloges synchronisées) :

  EC2 App  log : "requête envoyée  à 13:32:05.100 UTC"
  EC2 BDD  log : "requête reçue    à 13:32:05.102 UTC"

  → Latence réelle : 2ms
  → Séquence d'événements claire ✅
```

### NTP dans AWS

AWS fournit son propre serveur NTP accessible depuis toutes les instances EC2 : `169.254.169.123`. C'est une adresse **link-local** (accessible uniquement depuis l'intérieur du VPC) et elle est extrêmement précise.

```
Toutes les EC2 AWS synchronisent automatiquement
leur horloge via : 169.254.169.123 (Amazon Time Sync Service)

→ Tu n'as rien à configurer
→ Précision sub-milliseconde garantie
→ Fonctionne même sans accès Internet (link-local)
```

### Les serveurs NTP publics

Si tu as besoin de NTP en dehors d'AWS (ta machine locale, un serveur on-premise) :

| Serveur | Opérateur |
|---------|-----------|
| `pool.ntp.org` | Communauté mondiale (le plus utilisé) |
| `time.google.com` | Google |
| `time.cloudflare.com` | Cloudflare |
| `time.aws.com` | Amazon (public) |
| `ptbtime1.ptb.de` | PTB — Institut national de métrologie allemand |

---

## 12. NAT — comment les IPs privées accèdent à Internet

### Pourquoi c'est important

On a dit plusieurs fois que les IPs privées ne sont pas routées sur Internet. Mais tes machines à la maison accèdent très bien à Internet malgré leurs IPs en `192.168.x.x`. Et dans AWS, tes EC2 dans des subnets privés peuvent télécharger des mises à jour et appeler des APIs externes.

Le mécanisme qui rend tout ça possible s'appelle **NAT**. C'est l'un des concepts les plus fondamentaux du réseau — et l'un des plus mal compris. Le comprendre en détail, c'est comprendre pourquoi ton architecture Cloud fonctionne comme elle fonctionne.

### Ce que NAT fait exactement

**NAT** (Network Address Translation) traduit les adresses IP dans les paquets qui traversent un routeur. Il remplace l'IP source privée d'une machine par l'IP publique du routeur — et mémorise cette traduction pour faire l'inverse quand la réponse revient.

```
SANS NAT :
  Ta machine (192.168.1.10) → Internet → Google
  Google reçoit source=192.168.1.10
  Google essaie de répondre à 192.168.1.10
  → Impossible : cette IP n'existe pas sur Internet ❌

AVEC NAT :
  Ta machine (192.168.1.10) → Fritz!Box → Internet → Google
  Fritz!Box remplace 192.168.1.10 par 90.45.123.67 (son IP publique)
  Google reçoit source=90.45.123.67
  Google répond à 90.45.123.67
  Fritz!Box sait que c'est pour 192.168.1.10 → retransmet ✅
```

### La table NAT — la mémoire du routeur

Pour savoir à quelle machine interne retransmettre chaque réponse, le routeur maintient une **table NAT**. Elle enregistre chaque connexion active :

```
Table NAT de la Fritz!Box :

IP privée        Port privé   IP publique      Port public   Destination
192.168.1.10     52341   →    90.45.123.67     52341    →    142.250.74.46:443  (Google)
192.168.1.11     48210   →    90.45.123.67     48210    →    151.101.1.140:443  (Reddit)
192.168.1.12     61023   →    90.45.123.67     61023    →    52.84.12.5:443     (Netflix)
```

Trois machines différentes partagent la même IP publique (`90.45.123.67`). Le routeur les distingue grâce aux **numéros de port** — chaque connexion a un port source unique. C'est pour ça qu'on appelle cette technique **PAT** (Port Address Translation) ou **NAT overload** — plusieurs machines partagent une seule IP publique grâce aux ports.

### Les types de NAT

```
NAT statique     → une IP privée = une IP publique fixe
                   Rare, coûteux (une IP publique par machine)
                   Utilisé pour les serveurs qui doivent être joignables
                   depuis Internet (ex: serveur web)

NAT dynamique    → un pool d'IPs publiques partagé entre plusieurs machines
                   Plus flexible mais limité par la taille du pool

PAT / NAT overload → plusieurs machines partagent UNE SEULE IP publique
(le plus courant)   grâce aux ports — c'est ce que fait ta Fritz!Box
                    et la NAT Gateway AWS
```

### NAT dans AWS

Dans un VPC AWS, le NAT fonctionne exactement comme ta Fritz!Box mais à l'échelle Cloud :

```
Subnet privé
  EC2 (10.0.10.42)
       │
       │  "je veux télécharger une mise à jour"
       ▼
  NAT Gateway (10.0.1.5 — dans le subnet public)
       │
       │  remplace 10.0.10.42 par l'IP publique de la NAT Gateway
       ▼
  Internet Gateway → Internet → serveur de mise à jour
       │
       │  réponse revient vers l'IP publique de la NAT Gateway
       ▼
  NAT Gateway consulte sa table NAT → retransmet à 10.0.10.42
       │
       ▼
  EC2 reçoit la réponse ✅
```

**Points clés AWS :**

| | Fritz!Box | NAT Gateway AWS |
|---|---|---|
| Rôle | NAT pour ton réseau domestique | NAT pour un subnet privé VPC |
| IP publique | Attribuée par Telekom | Elastic IP attachée à la NAT GW |
| Sens autorisé | Sortie uniquement | Sortie uniquement |
| Entrée possible ? | Non (sauf port forwarding) | Non — les subnets privés restent inaccessibles |
| Coût | Inclus dans ta box | Payant à l'heure + par Go de données |

> Une NAT Gateway AWS coûte environ **0.045$/heure** + **0.045$/Go** transféré. Dans une architecture multi-AZ, on en crée une par zone de disponibilité — le coût peut vite monter. C'est une décision architecturale importante.

---

## 12. IPv6 — le successeur d'IPv4

### Pourquoi c'est important

IPv4 avec ses ~4 milliards d'adresses est épuisé depuis 2011. IPv6 a été créé pour résoudre ce problème. Dans le Cloud, AWS supporte IPv6 nativement — certaines architectures modernes l'utilisent exclusivement. Tu n'as pas besoin de le maîtriser en profondeur, mais tu dois savoir le reconnaître et comprendre ses différences clés avec IPv4.

### Le format IPv6

Une adresse IPv6 fait **128 bits** — contre 32 pour IPv4. Elle est écrite en hexadécimal, séparée par des deux-points :

```
IPv4 :  192.168.1.42
        4 groupes de 8 bits = 32 bits
        ~4 milliards d'adresses

IPv6 :  2001:0db8:85a3:0000:0000:8a2e:0370:7334
        8 groupes de 16 bits = 128 bits
        340 sextillions d'adresses
        (340 000 000 000 000 000 000 000 000 000 000 000 000)
```

### Simplification de notation

Les adresses IPv6 peuvent être raccourcies :

```
Règle 1 : supprimer les zéros en tête dans chaque groupe
  2001:0db8:0000:0000 → 2001:db8:0:0

Règle 2 : remplacer une séquence de groupes à zéro par ::
  2001:db8:0:0:0:0:0:1 → 2001:db8::1
  (:: ne peut apparaître qu'une seule fois)

Adresses spéciales :
  ::1           → loopback IPv6 (équivalent de 127.0.0.1)
  ::            → adresse non spécifiée (équivalent de 0.0.0.0)
  fe80::/10     → adresses link-local (réseau local uniquement)
```

### Les différences clés avec IPv4

| | IPv4 | IPv6 |
|---|---|---|
| Taille | 32 bits | 128 bits |
| Notation | Décimale (`192.168.1.1`) | Hexadécimale (`2001:db8::1`) |
| NAT nécessaire ? | Oui (adresses épuisées) | Non (assez d'adresses pour tout) |
| DHCP | Oui | Optionnel — auto-configuration possible |
| Broadcast | Oui | Non — remplacé par multicast |
| Sécurité intégrée | Optionnelle (IPsec) | Conçu avec IPsec |

### IPv6 dans AWS

AWS supporte IPv6 sur les VPC. Chaque ressource peut recevoir une adresse IPv6 publique en plus de son IP privée IPv4 :

```
EC2 dans un subnet dual-stack :
  IPv4 privée  : 10.0.1.42        (accès interne VPC)
  IPv6 publique: 2600:1f18::/32   (accessible depuis Internet directement)
```

> Avec IPv6, **plus besoin de NAT** pour accéder à Internet — chaque machine a une adresse publique unique. C'est plus simple architecturalement, mais ça demande une gestion des Security Groups plus rigoureuse (chaque machine est potentiellement joignable depuis Internet).

---

## 13. TLS — comment fonctionne le chiffrement

### Pourquoi c'est important

On a dit qu'HTTPS = HTTP + TLS. Mais qu'est-ce que TLS fait concrètement ? Comment deux machines qui ne se connaissent pas peuvent-elles établir une communication chiffrée en toute confiance ?

Comprendre TLS, c'est comprendre pourquoi les certificats expirent, pourquoi un certificat invalide bloque une connexion, et comment configurer HTTPS sur AWS avec ACM. En tant qu'architecte, tu vas manipuler des certificats régulièrement.

### Le problème que TLS résout

```
Sans TLS :
  Ton navigateur → [réseau] → Serveur Google
  N'importe qui sur le réseau peut lire tes données
  N'importe qui peut se faire passer pour Google

Avec TLS, deux problèmes sont résolus :
  1. Confidentialité  → les données sont chiffrées
  2. Authenticité     → tu es sûr de parler au vrai Google
```

### Les certificats TLS — la carte d'identité d'un serveur

Un **certificat TLS** est un fichier qui contient :
- Le **nom de domaine** qu'il protège (ex: `google.com`)
- La **clé publique** du serveur
- La **signature** d'une autorité de certification qui garantit son authenticité
- Une **date d'expiration**

```
Certificat de google.com :
  Sujet      : google.com
  Émetteur   : Google Trust Services (CA)
  Valide du  : 01/09/2024
  Valide au  : 24/11/2024
  Clé pub    : [données cryptographiques]
  Signature  : [signature de la CA]
```

### Les autorités de certification (CA)

Une **CA** (Certificate Authority) est un organisme de confiance qui signe les certificats. Ton navigateur contient une liste de CAs reconnues (Mozilla, Google, Apple les maintiennent). Si un certificat est signé par une CA reconnue → le navigateur fait confiance.

```
Chaîne de confiance :

  CA Racine (ex: DigiCert Root)       ← ton OS/navigateur lui fait confiance
       │
       │ signe
       ▼
  CA Intermédiaire (ex: Google Trust)
       │
       │ signe
       ▼
  Certificat de google.com            ← le serveur présente ce certificat
```

**CAs courantes :**
- **Let's Encrypt** — gratuit, automatisé, renouvelé tous les 90 jours
- **DigiCert, Comodo, Sectigo** — payants, utilisés en entreprise
- **ACM** (AWS Certificate Manager) — gratuit sur AWS, renouvellement automatique

### Le handshake TLS — établir la connexion chiffrée

Avant d'échanger des données chiffrées, le client et le serveur doivent se mettre d'accord sur comment chiffrer. C'est le **TLS handshake** :

```
Client (navigateur)                    Serveur (google.com)

1. Client Hello  ──────────────────►
   "Voici les algorithmes de chiffrement
    que je supporte. Voici un nombre aléatoire."

2.                ◄──────────────────  Server Hello
                                       "J'ai choisi cet algorithme.
                                        Voici mon certificat.
                                        Voici un nombre aléatoire."

3. Vérification du certificat
   "La signature est valide ? ✅
    Le domaine correspond ? ✅
    Le certificat n'est pas expiré ? ✅"

4. Échange de clé  ────────────────►
   (calcul d'une clé de session partagée
    basée sur les deux nombres aléatoires)

5.                ◄────────────────── Finished
   ════ Données chiffrées ═══════════ avec la clé de session ✅
```

> La clé de session est calculée des deux côtés indépendamment — elle ne transite jamais sur le réseau. Même si quelqu'un capture le trafic, il ne peut pas déchiffrer les données sans cette clé.

### Inspecter un certificat TLS

```bash
# Voir le certificat d'un site avec openssl
openssl s_client -connect google.com:443 -showcerts

# Version plus lisible avec curl
curl -v https://google.com 2>&1 | grep -A 5 "Server certificate"

# Voir la date d'expiration
echo | openssl s_client -connect google.com:443 2>/dev/null \
  | openssl x509 -noout -dates
```

### TLS dans AWS — ACM

**ACM** (AWS Certificate Manager) est le service AWS qui gère les certificats TLS :

```
Tu crées un certificat ACM pour monapp.com
       ↓
AWS vérifie que tu possèdes le domaine
(via un enregistrement DNS ou un email)
       ↓
Certificat émis et stocké dans ACM
       ↓
Tu l'attaches à un ALB, CloudFront, ou API Gateway
       ↓
Renouvellement automatique — tu n'as rien à faire
```

**Avantages ACM :**
- **Gratuit** pour les services AWS intégrés
- **Renouvellement automatique** — plus de certificat expiré en production
- **Intégré** avec ALB, CloudFront, API Gateway

> Les certificats ACM ne peuvent pas être exportés et utilisés sur une EC2 directement. Pour une EC2, tu utilises Let's Encrypt (via Certbot) ou un certificat acheté.

---

## 14. Proxy et Reverse Proxy

### Pourquoi c'est important

Le concept de proxy est partout dans le Cloud — sans forcément utiliser ce mot. Un **ALB** est un reverse proxy. **CloudFront** est un reverse proxy. **nginx** devant ton app est un reverse proxy. Comprendre ce concept, c'est comprendre l'architecture de la quasi-totalité des applications web modernes.

### Le Forward Proxy — parler au nom du client

Un **forward proxy** se place entre le client et Internet. Le client envoie ses requêtes au proxy, qui les transmet à destination. La destination ne voit que l'IP du proxy — pas celle du client.

```
Client → [Forward Proxy] → Internet → Serveur

Utilisation typique :
  - Entreprise : filtrer les sites web accessibles aux employés
  - Anonymisation : masquer l'IP du client (VPN, Tor)
  - Cache : éviter de retélécharger les mêmes ressources
```

### Le Reverse Proxy — parler au nom du serveur

Un **reverse proxy** se place entre Internet et les serveurs. Le client envoie sa requête au proxy, qui la transmet au bon serveur interne. Le client ne voit que l'IP du proxy — pas celle des serveurs.

```
Client → Internet → [Reverse Proxy] → Serveur 1
                                    → Serveur 2
                                    → Serveur 3

Utilisation typique :
  - Load balancing : distribuer le trafic entre plusieurs serveurs
  - SSL termination : déchiffrer TLS une seule fois au niveau du proxy
  - Cache : servir les ressources statiques sans toucher les serveurs
  - Protection : cacher l'architecture interne
```

### Comparatif

```
                 Forward Proxy          Reverse Proxy
Qui il protège : le client              le serveur
Qui le configure: l'utilisateur/IT      l'équipe ops/Cloud
Exemples       : VPN, Squid, Tor        nginx, HAProxy, ALB, CloudFront
Voit l'IP de   : le serveur             le client (parfois)
```

### nginx comme reverse proxy — exemple concret

**nginx** est le reverse proxy le plus utilisé au monde. Voici une configuration typique :

```nginx
# nginx reçoit les requêtes sur le port 443 (HTTPS)
# et les transmet à l'app Node.js sur le port 3000

server {
    listen 443 ssl;
    server_name api.monapp.com;

    # Certificat TLS
    ssl_certificate     /etc/ssl/monapp.crt;
    ssl_certificate_key /etc/ssl/monapp.key;

    location / {
        proxy_pass http://localhost:3000;    # transmet à l'app
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

```
Flux :
Client → HTTPS:443 → [nginx] → HTTP:3000 → [App Node.js]

nginx gère :
  ✅ TLS (déchiffrement)
  ✅ Headers de sécurité
  ✅ Logs
  ✅ Rate limiting

L'app Node.js :
  ✅ Reçoit du HTTP simple (pas besoin de gérer TLS)
  ✅ Protégée derrière nginx
```

### Les reverse proxies dans AWS

Dans AWS, tu n'installes généralement pas nginx manuellement — AWS fournit des services managés qui font la même chose :

| Outil | Type | Ce qu'il fait |
|-------|------|---------------|
| **ALB** | Reverse proxy couche 7 | Load balancing HTTP, routing par URL, SSL termination |
| **NLB** | Reverse proxy couche 4 | Load balancing TCP/UDP ultra-rapide |
| **CloudFront** | Reverse proxy + CDN | Cache global, SSL, protection DDoS |
| **API Gateway** | Reverse proxy | Gestion d'APIs, auth, rate limiting |

```
Architecture typique AWS :

Client
  │
  ▼
CloudFront (reverse proxy + cache global)
  │
  ▼
ALB (reverse proxy + load balancing)
  │
  ├── EC2 App 1
  ├── EC2 App 2
  └── EC2 App 3
```

Chaque couche est un reverse proxy qui ajoute une fonctionnalité. C'est de la composition — chaque composant fait une chose bien.

---

## Récapitulatif du Module 1

| Concept | Ce qu'il faut retenir |
|---------|----------------------|
| **Réseau** | Machines qui s'échangent des données. Dans le Cloud, tout est réseau virtuel (VPC). |
| **IP** | Identifiant d'une machine. IPs privées dans les VPC, publiques pour Internet. |
| **Masque** | Délimite la partie réseau et la partie machine d'une adresse IP. |
| **CIDR** | Notation compacte du masque. `/24` = 256 adresses, `/16` = 65 536. |
| **OSI** | Modèle de référence en 7 couches. Chaque couche a une responsabilité précise. |
| **TCP/IP** | Modèle réel en 4 couches. Application → Transport → Internet → Accès réseau. |
| **Encapsulation** | Chaque couche ajoute son header. Les données voyagent emballées comme des poupées russes. |
| **TCP** | Transport fiable avec handshake et accusés de réception. Pour HTTP, SSH, BDD. |
| **UDP** | Transport rapide sans garantie de livraison. Pour DNS, streaming, jeux. |
| **DHCP** | Attribue automatiquement une IP privée à chaque machine qui rejoint le réseau. |
| **ICMP** | Protocole de diagnostic. `ping` l'utilise pour tester si une machine est joignable. |
| **DNS** | Traduit les noms en IPs. TTL contrôle combien de temps la réponse est mise en cache. |
| **Ports** | "Portes" d'une machine. 80=HTTP, 443=HTTPS, 22=SSH, 5432=Postgres. |
| **HTTPS** | HTTP + TLS = données chiffrées + serveur authentifié. Obligatoire en production. |
| **NAT** | Traduit les IPs privées en IP publique pour accéder à Internet. PAT = plusieurs machines, une IP. |
| **NTP** | Synchronise les horloges de toutes les machines en UTC. Indispensable pour des logs cohérents. |
| **IPv6** | 128 bits, notation hexadécimale. Plus besoin de NAT — chaque machine a une IP publique unique. |
| **TLS** | Chiffrement + authenticité. Certificats signés par une CA. ACM sur AWS = gratuit et automatique. |
| **Proxy** | Se place entre client et serveur. Forward = protège le client. Reverse = protège le serveur. |
| **Couches Cloud** | Security Group = couche 4. ALB = couche 7. NLB = couche 4. WAF = couche 7. |

---

*→ Exercices pratiques : voir `module1-exercices.md`*
*→ Module suivant : Module 2 — Le réseau dans Linux*
