# Module 2 — Le réseau dans Linux
### Partie Théorie

> **Cours Networking Fondamentaux · Orienté Cloud**
> Prérequis : Module 1 complété · Durée estimée : 2h

---

## Glossaire

Tous les termes techniques utilisés dans ce module, expliqués en français.

| Terme | Signification | Définition simple |
|-------|--------------|-------------------|
| **ARP** | Address Resolution Protocol | Protocole qui traduit une adresse IP en adresse MAC sur un réseau local |
| **Bastion host** | — | EC2 publique servant de point d'entrée SSH unique vers un VPC privé |
| **Default Gateway** | Passerelle par défaut | L'adresse du routeur que ta machine contacte pour tout ce qui sort de son réseau local |
| **eth0** | Ethernet interface 0 | Nom classique de la première interface réseau filaire sur Linux |
| **Firewall** | Pare-feu | Système qui filtre le trafic réseau selon des règles |
| **hop** | Saut | Chaque routeur traversé par un paquet sur son chemin vers la destination |
| **ifconfig** | Interface Configuration | Ancienne commande Linux pour afficher et configurer les interfaces réseau |
| **iptables** | — | Outil Linux de filtrage réseau bas niveau — le firewall natif du noyau Linux |
| **lo** | Loopback | Interface réseau virtuelle qui représente la machine elle-même (`127.0.0.1`) |
| **MAC** | Media Access Control | Adresse physique unique gravée dans une carte réseau — utilisée pour la communication locale |
| **MTU** | Maximum Transmission Unit | Taille maximale d'un paquet réseau en octets (souvent 1500 sur Ethernet) |
| **mtr** | Matt's Traceroute | Outil qui combine ping et traceroute — affiche latence et perte de paquets en temps réel |
| **netcat (nc)** | — | Outil qui crée des connexions TCP/UDP brutes — utilisé pour tester si un port est ouvert |
| **NIC** | Network Interface Card | Carte réseau — le composant physique ou virtuel qui connecte une machine au réseau |
| **nmap** | Network Mapper | Scanner réseau — découvre les machines actives et les ports ouverts sur un réseau |
| **NTP** | Network Time Protocol | Synchronise l'horloge d'une machine avec des serveurs de temps — UDP port 123 |
| **pcap** | Packet Capture | Format de fichier standard pour sauvegarder des captures réseau |
| **Port éphémère** | — | Port source choisi automatiquement par l'OS pour chaque connexion sortante (32768-60999) |
| **Port forwarding** | Redirection de port | Tunnel SSH qui redirige un port local vers un service distant inaccessible directement |
| **ProxyJump** | — | Option SSH (`-J`) pour se connecter via un serveur intermédiaire (bastion) en un seul saut |
| **resolv.conf** | — | Fichier Linux qui configure les serveurs DNS de la machine |
| **Routage** | — | Mécanisme qui décide par quel chemin un paquet doit voyager |
| **Route Table** | Table de routage | Liste des règles qui indiquent vers quel routeur envoyer un paquet selon sa destination |
| **SOCKS proxy** | — | Proxy réseau générique créé par SSH dynamic forwarding (`-D`) |
| **ss** | Socket Statistics | Commande Linux moderne pour voir les connexions réseau actives (remplace netstat) |
| **SSH Agent** | — | Service local qui mémorise les clés SSH déchiffrées pour éviter de retaper la passphrase |
| **Switch** | Commutateur | Équipement réseau qui connecte plusieurs machines dans le même réseau local |
| **tcpdump** | — | Outil de capture réseau en ligne de commande — le Wireshark du terminal |
| **timedatectl** | — | Commande Linux pour voir et configurer l'heure, le fuseau horaire et NTP |
| **traceroute** | — | Commande qui affiche tous les routeurs traversés entre ta machine et une destination |
| **TTL** | Time To Live | Compteur décrémenté à chaque routeur traversé — quand il atteint 0, le paquet est détruit |
| **ufw** | Uncomplicated Firewall | Interface simplifiée pour gérer iptables sur Ubuntu/Debian |
| **UTC** | Coordinated Universal Time | Heure universelle de référence — utilisée dans tous les logs Cloud |
| **Wireshark** | — | Analyseur de protocoles réseau graphique — capture et affiche les paquets couche par couche |
| **wlan0** | Wireless LAN interface 0 | Nom classique de la première interface WiFi sur Linux |

---

## Sommaire

1. [Le voyage d'un paquet — du source à la destination](#1-le-voyage-dun-paquet--du-source-à-la-destination)
2. [Les interfaces réseau](#2-les-interfaces-réseau)
3. [La table de routage](#3-la-table-de-routage)
4. [Les commandes CLI essentielles](#4-les-commandes-cli-essentielles)
5. [Le firewall sous Linux](#5-le-firewall-sous-linux)
6. [Les outils d'analyse réseau avancés](#6-les-outils-danalyse-réseau-avancés)
7. [Configuration réseau locale — DNS, ports éphémères et NTP](#7-configuration-réseau-locale--dns-ports-éphémères-et-ntp)
8. [nmap — scanner et auditer un réseau](#8-nmap--scanner-et-auditer-un-réseau)
9. [SSH avancé — tunneling, port forwarding et bastion](#9-ssh-avancé--tunneling-port-forwarding-et-bastion)

---

## 1. Le voyage d'un paquet — du source à la destination

### Pourquoi c'est important

C'est le cœur de tout ce cours. Comprendre comment un paquet voyage d'une machine à une autre — en passant par des switches, des routeurs, des passerelles — c'est comprendre **pourquoi une connexion fonctionne ou échoue**. Chaque fois qu'un service Cloud ne répond pas, la réponse est quelque part sur ce chemin.

On va suivre un paquet de bout en bout, étape par étape, avec deux scénarios : communication dans le même réseau, puis entre deux réseaux différents.

---

### Scénario 1 — Deux machines sur le même réseau local

**Situation** : ton PC (`192.168.1.10`) veut envoyer un paquet à ton téléphone (`192.168.1.20`). Les deux sont sur le même WiFi, donc le même réseau `192.168.1.0/24`.

#### Étape 1 — Ta machine vérifie si la destination est locale

Avant d'envoyer quoi que ce soit, ta machine pose une question simple :

```
Mon IP        : 192.168.1.10
Masque        : 255.255.255.0  (/24)
Destination   : 192.168.1.20

192.168.1.20 est-il dans mon réseau 192.168.1.0/24 ?
→ Oui — les 3 premiers octets sont identiques.
→ Je peux lui parler directement, sans passer par le routeur.
```

#### Étape 2 — Le problème de la couche 2 : ARP

Ta machine connaît l'IP du téléphone (`192.168.1.20`), mais pour envoyer un paquet sur le réseau local, elle a besoin de son **adresse MAC** — l'adresse physique de sa carte réseau.

C'est le rôle d'**ARP** (Address Resolution Protocol) :

```
Ta machine envoie un broadcast sur tout le réseau local :
"Qui a l'IP 192.168.1.20 ? Réponds-moi avec ton adresse MAC !"

Toutes les machines du réseau reçoivent ce message.
Seul le téléphone (192.168.1.20) répond :
"C'est moi ! Mon adresse MAC est AA:BB:CC:DD:EE:FF"

Ta machine mémorise cette association dans son cache ARP :
192.168.1.20  →  AA:BB:CC:DD:EE:FF
```

Ce cache ARP évite de refaire cette découverte à chaque paquet. Il expire après quelques minutes.

#### Étape 3 — Le Switch fait le reste

Le **switch** est l'équipement (physique ou virtuel) qui relie les machines du même réseau local. Il maintient une **table MAC** : l'association entre chaque adresse MAC et le port physique où la machine est connectée.

```
Switch — table MAC :
  Port 1  →  MAC de ton PC        (AA:11:22:33:44:55)
  Port 2  →  MAC de ton téléphone (AA:BB:CC:DD:EE:FF)
  Port 3  →  MAC de ta TV         (AA:99:88:77:66:55)

Ton PC envoie le paquet avec la MAC du téléphone comme destination.
Le switch lit la MAC destination → envoie uniquement sur le Port 2.
Le téléphone reçoit le paquet.
```

Le switch travaille à la **couche 2** (Liaison) — il ne regarde pas les IPs, uniquement les adresses MAC.

```
PC (192.168.1.10)
      │
      │ [Paquet avec MAC destination = AA:BB:CC:DD:EE:FF]
      ▼
  [ SWITCH ]
      │
      │ (le switch consulte sa table MAC)
      │ "AA:BB:CC:DD:EE:FF → Port 2"
      ▼
Téléphone (192.168.1.20)
```

---

### Scénario 2 — Deux machines sur des réseaux différents

**Situation** : ton PC (`192.168.1.10`) veut accéder à `google.com` (`142.250.74.46`). Google est sur un réseau totalement différent — à l'autre bout d'Internet.

#### Étape 1 — Ta machine détecte que la destination est externe

```
Mon IP        : 192.168.1.10
Masque        : /24
Destination   : 142.250.74.46

142.250.74.46 est-il dans mon réseau 192.168.1.0/24 ?
→ Non — les octets sont complètement différents.
→ Je ne peux pas lui parler directement.
→ Je dois envoyer le paquet à mon Default Gateway.
```

#### Étape 2 — Le Default Gateway : la porte de sortie

Le **Default Gateway** (passerelle par défaut) c'est l'adresse du routeur local — chez toi, c'est ta Fritz!Box (`192.168.1.1`).

Ta machine dit :

```
"Je ne sais pas comment atteindre 142.250.74.46.
 Je ne connais que mon réseau local.
 Je vais donner ce paquet à mon Default Gateway
 et le laisser s'occuper du reste."
```

Pour envoyer le paquet à la Fritz!Box, ta machine fait d'abord un ARP pour trouver sa MAC :

```
ARP : "Qui a 192.168.1.1 ?"
Fritz!Box : "C'est moi, MAC = FF:EE:DD:CC:BB:AA"

Paquet envoyé au switch avec :
  IP destination  : 142.250.74.46   (inchangée — c'est Google)
  MAC destination : FF:EE:DD:CC:BB:AA  (celle de la Fritz!Box)
```

> **Point clé** : L'IP destination ne change jamais pendant le voyage. C'est l'adresse MAC qui change à chaque saut — elle est locale à chaque segment réseau.

#### Étape 3 — La Fritz!Box : NAT et routage

La Fritz!Box reçoit le paquet. Elle fait deux choses :

**1. NAT** — elle remplace ton IP privée par son IP publique :

```
Avant NAT  : source = 192.168.1.10  (ton PC, IP privée)
Après NAT  : source = 90.45.123.67  (IP publique de la Fritz!Box)
```

Sans ça, Google ne saurait pas où renvoyer la réponse — ton IP privée n'existe pas sur Internet.

**2. Routage** — elle consulte sa table de routage pour savoir où envoyer le paquet vers Google, et le transmet au routeur de Telekom.

#### Étape 4 — Le voyage sur Internet (les hops)

Le paquet traverse une série de **routeurs** (hops) entre ta Fritz!Box et les serveurs de Google. Chaque routeur :

1. Reçoit le paquet
2. Lit l'IP destination (`142.250.74.46`)
3. Consulte sa table de routage
4. Envoie le paquet vers le prochain routeur le plus proche de la destination
5. Décrémente le **TTL** de 1 (si TTL atteint 0 → paquet détruit, erreur renvoyée)

```
Ta Fritz!Box  →  Routeur Telekom  →  Routeur Frankfurt  →  ...  →  Google
     hop 1            hop 2               hop 3                      hop N
```

C'est exactement ce que la commande `traceroute` te montre — tous ces sauts, avec leur latence.

#### Étape 5 — La réponse revient

Google répond à l'IP publique de ta Fritz!Box (`90.45.123.67`). La Fritz!Box sait grâce à sa **table NAT** que cette réponse était destinée à ton PC (`192.168.1.10`) — elle lui retransmet.

```
Google → 90.45.123.67 (Fritz!Box)
Fritz!Box consulte table NAT → 192.168.1.10 (ton PC)
Ton PC reçoit la réponse.
```

### Vue d'ensemble — le voyage complet

```
TON PC (192.168.1.10)
  │  ARP → trouve MAC de la Fritz!Box
  │  Envoie paquet : IP dst=142.250.74.46, MAC dst=Fritz!Box
  ▼
SWITCH (couche 2)
  │  Lit la MAC destination → redirige vers Fritz!Box
  ▼
FRITZ!BOX (192.168.1.1)
  │  NAT : remplace 192.168.1.10 par 90.45.123.67
  │  Routage : envoie vers routeur Telekom
  ▼
ROUTEURS INTERNET (plusieurs hops)
  │  Chaque routeur lit l'IP destination et passe au suivant
  ▼
GOOGLE (142.250.74.46)
  │  Reçoit la requête, prépare la réponse
  │  Répond vers 90.45.123.67
  ▼
FRITZ!BOX
  │  NAT inverse : retransmet à 192.168.1.10
  ▼
TON PC
  Reçoit la réponse de Google ✅
```

### Ce que ça donne dans le Cloud AWS

Ce même mécanisme se reproduit dans un VPC AWS :

| Réseau local | Équivalent AWS |
|---|---|
| Switch | Réseau interne du subnet AWS |
| Default Gateway (Fritz!Box) | Route Table du VPC |
| NAT (Fritz!Box) | NAT Gateway AWS |
| IP publique (Fritz!Box) | Elastic IP / IP publique AWS |
| Routeurs Internet | Internet Gateway AWS |

La logique est exactement la même — juste virtualisée.

---

## 2. Les interfaces réseau

### Pourquoi c'est important

Une interface réseau c'est le **point de connexion** entre une machine et un réseau. Sur Linux, tout passe par des interfaces — et savoir les lire, c'est savoir où ta machine est connectée, avec quelle IP, et si la connexion est active. C'est la première chose à vérifier quand un problème réseau survient.

### Ce qu'est une interface réseau

Physiquement, c'est la carte réseau (NIC) de ta machine — le composant qui envoie et reçoit des signaux. Sur Linux, chaque interface a un **nom** et des **propriétés** : adresse IP, adresse MAC, état (up/down), etc.

Mais Linux crée aussi des interfaces **virtuelles** qui n'ont pas de support physique — elles existent uniquement en logiciel.

### Les interfaces courantes sur Linux

```
lo          →  Loopback — la machine elle-même (127.0.0.1)
eth0        →  Première interface filaire (câble ethernet)
eth1        →  Deuxième interface filaire
wlan0       →  Première interface WiFi
ens3, ens4  →  Noms modernes des interfaces filaires (Ubuntu récent)
docker0     →  Interface virtuelle créée par Docker
veth0       →  Interface virtuelle pour les conteneurs
```

### L'interface loopback — lo

L'interface **loopback** (`lo`) est une interface virtuelle spéciale qui représente la machine elle-même. Son adresse est toujours `127.0.0.1` (ou `::1` en IPv6).

```
Quand ton app fait une requête vers 127.0.0.1 ou localhost :
→ Le paquet ne sort jamais sur le réseau physique
→ Il fait un aller-retour interne à la machine
→ Utilisé pour qu'un service parle à un autre service sur la même machine
```

Exemple concret : ton app Node.js tourne sur le port 3000 et veut parler à Redis sur le port 6379, tous les deux sur la même machine. Ils communiquent via `127.0.0.1` — le réseau physique n'est pas impliqué.

### Lire une interface réseau sur Linux

```bash
ip addr show eth0

# Résultat typique :
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500
    link/ether aa:bb:cc:dd:ee:ff brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.10/24 brd 192.168.1.255 scope global eth0
    inet6 fe80::1/64 scope link

# Ce que ça veut dire :
# UP            → interface active
# mtu 1500      → taille max d'un paquet = 1500 octets
# link/ether    → adresse MAC de la carte réseau
# inet          → adresse IPv4 et masque CIDR (/24)
# inet6         → adresse IPv6
# brd           → adresse broadcast du réseau
```

### Les interfaces réseau dans le Cloud

Dans AWS, chaque EC2 possède au moins une interface réseau virtuelle appelée **ENI** (Elastic Network Interface). Elle fonctionne exactement comme `eth0` sur une machine physique — elle a une IP privée, une adresse MAC, et elle est attachée à un subnet.

```
EC2 instance
  └── ENI (eth0)
        ├── IP privée  : 10.0.1.42
        ├── IP publique: 54.12.34.56 (optionnelle)
        ├── MAC        : 0a:1b:2c:3d:4e:5f
        └── Subnet     : subnet-public-a (10.0.1.0/24)
```

---

## 3. La table de routage

### Pourquoi c'est important

La table de routage c'est le **GPS de ta machine**. Quand elle reçoit un paquet à envoyer, elle consulte cette table pour savoir par où le faire passer. Sans table de routage correcte, les paquets ne savent pas où aller — et les connexions échouent silencieusement.

Dans AWS, la **Route Table** d'un VPC est exactement ce concept — c'est elle qui définit si un subnet est public ou privé.

### Comment fonctionne une table de routage

Une table de routage est une liste de règles. Chaque règle dit :

```
"Pour atteindre ce réseau (destination),
 envoie le paquet via cette interface ou ce routeur (via)."
```

Linux parcourt ces règles du plus spécifique au plus général, et utilise la première qui correspond.

### Exemple de table de routage

```bash
ip route show

# Résultat typique :
default via 192.168.1.1 dev eth0
192.168.1.0/24 dev eth0 proto kernel scope link src 192.168.1.10
127.0.0.0/8 dev lo scope host
```

Décryptage ligne par ligne :

```
default via 192.168.1.1 dev eth0
→ "Pour TOUT ce que je ne reconnais pas
   envoie via 192.168.1.1 (Fritz!Box) par l'interface eth0"
→ C'est la ligne du Default Gateway

192.168.1.0/24 dev eth0
→ "Pour atteindre n'importe quelle machine en 192.168.1.X
   parle-leur directement via eth0 (pas besoin de routeur)"
→ C'est le réseau local direct

127.0.0.0/8 dev lo
→ "Pour tout ce qui commence par 127.X.X.X
   utilise l'interface loopback (lo)"
→ C'est le trafic local à la machine
```

### La règle "default" — le Default Gateway

La ligne `default` est la plus importante. C'est la règle de dernier recours : si aucune autre règle ne correspond, le paquet part vers le Default Gateway.

```
Ma machine reçoit un paquet à envoyer vers 142.250.74.46

→ Est-ce que 142.250.74.46 est dans 192.168.1.0/24 ? Non
→ Est-ce que 142.250.74.46 est dans 127.0.0.0/8 ? Non
→ Aucune règle spécifique → utiliser "default"
→ Envoyer à 192.168.1.1 (Fritz!Box) via eth0
```

### Tables de routage dans AWS

Dans un VPC AWS, chaque subnet a sa propre Route Table. C'est elle qui définit si un subnet est **public** ou **privé** :

```
Route Table — subnet PUBLIC
  10.0.0.0/16   →  local           (trafic interne au VPC)
  0.0.0.0/0     →  igw-xxxxxxxx    (Internet Gateway → accès Internet)

Route Table — subnet PRIVÉ
  10.0.0.0/16   →  local           (trafic interne au VPC)
  0.0.0.0/0     →  nat-xxxxxxxx    (NAT Gateway → sortie vers Internet uniquement)

Route Table — subnet BASE DE DONNÉES
  10.0.0.0/16   →  local           (trafic interne au VPC uniquement)
  (pas de règle 0.0.0.0/0 → aucun accès Internet dans aucune direction)
```

La règle `0.0.0.0/0` c'est l'équivalent Cloud de `default` — elle dit "tout ce qui n'est pas local, envoie-le là". Pointer vers une Internet Gateway = accès public. Pointer vers une NAT Gateway = sortie uniquement, pas d'entrée.

---

## 4. Les commandes CLI essentielles

### Pourquoi c'est important

Sur Linux, le réseau se diagnostique et se configure en ligne de commande. Ces outils sont disponibles sur n'importe quel serveur Linux — dans le Cloud, sur une EC2, dans un conteneur. Maîtriser ces commandes, c'est pouvoir débugger n'importe quel problème réseau sans interface graphique.

---

### ip — la commande centrale

`ip` est la commande moderne pour tout ce qui concerne les interfaces et le routage. Elle remplace les anciennes commandes `ifconfig` et `route`.

```bash
# Voir toutes les interfaces et leurs IPs
ip addr show
ip a              # version courte

# Voir une interface spécifique
ip addr show eth0

# Voir la table de routage
ip route show
ip r              # version courte

# Voir le cache ARP (association IP → MAC)
ip neigh show

# Activer / désactiver une interface
sudo ip link set eth0 up
sudo ip link set eth0 down

# Ajouter une IP temporaire à une interface
sudo ip addr add 10.0.0.5/24 dev eth0

# Ajouter une route
sudo ip route add 10.0.2.0/24 via 10.0.1.1
```

---

### ping — tester la connectivité

`ping` envoie des paquets ICMP Echo Request et attend les Echo Reply. C'est le premier outil à utiliser pour diagnostiquer un problème réseau.

```bash
# Ping basique
ping google.com

# Nombre fixe de paquets
ping -c 4 google.com

# Ping avec intervalle personnalisé (en secondes)
ping -i 0.5 google.com

# Ping avec taille de paquet personnalisée
ping -s 1000 google.com

# Ping vers l'interface loopback (toujours disponible)
ping 127.0.0.1
```

**Stratégie de diagnostic avec ping :**

```
Problème : "je n'arrive pas à joindre mon serveur AWS"

Étape 1 : ping 127.0.0.1
  → Si ça échoue : problème sur ta machine elle-même (rarissime)

Étape 2 : ping 192.168.1.1  (ta Fritz!Box)
  → Si ça échoue : problème sur ton réseau local

Étape 3 : ping 8.8.8.8  (Google DNS, IP connue)
  → Si ça échoue : pas d'accès Internet

Étape 4 : ping nom-de-domaine
  → Si ça échoue mais l'étape 3 marche : problème DNS

Étape 5 : ping IP-de-ton-serveur
  → Si ça échoue : serveur éteint, ou ICMP bloqué par Security Group
```

---

### traceroute — voir le chemin des paquets

`traceroute` affiche tous les routeurs (hops) traversés entre ta machine et une destination. C'est le moyen de voir concrètement le voyage d'un paquet dont on a parlé en section 1.

```bash
# Installation si nécessaire
sudo apt install traceroute

# Utilisation
traceroute google.com
traceroute 8.8.8.8
```

**Lire un résultat traceroute :**

```
traceroute to google.com (142.250.74.46)

 1  192.168.1.1 (192.168.1.1)       1.2 ms    ← ta Fritz!Box
 2  10.45.0.1 (10.45.0.1)           8.4 ms    ← routeur Telekom
 3  87.234.12.1 (87.234.12.1)      10.1 ms    ← backbone Telekom
 4  72.14.232.1 (72.14.232.1)      11.8 ms    ← réseau Google
 5  142.250.74.46 (142.250.74.46)  12.3 ms    ← destination ✅

# * * *  signifie que ce routeur ne répond pas à ICMP (firewall)
# mais le paquet passe quand même
```

Chaque ligne c'est un routeur avec sa latence. Plus tu t'éloignes géographiquement, plus la latence augmente.

---

### ss — voir les connexions actives

`ss` (Socket Statistics) affiche toutes les connexions réseau actives sur ta machine. C'est l'outil pour savoir **qui parle à qui** et sur **quel port** en ce moment.

```bash
# Toutes les connexions TCP actives
ss -tn

# Avec les processus associés (nécessite sudo)
sudo ss -tnp

# Voir les ports en écoute (services qui attendent des connexions)
ss -tlnp

# Filtrer par port
ss -tnp sport = :443
ss -tnp dport = :5432
```

**Lire un résultat ss :**

```bash
sudo ss -tnp

State    Recv-Q  Send-Q  Local Address:Port   Peer Address:Port  Process
ESTAB    0       0       192.168.1.10:52341   142.250.74.46:443  chrome
ESTAB    0       0       192.168.1.10:22      10.0.1.5:54231     sshd
LISTEN   0       128     0.0.0.0:80           0.0.0.0:*          nginx

# ESTAB  = connexion établie (active)
# LISTEN = service en écoute, attend des connexions entrantes
# Local  = ton IP:port
# Peer   = IP:port de l'autre machine
```

---

### curl — tester des requêtes HTTP

`curl` permet d'envoyer des requêtes HTTP depuis le terminal. C'est l'outil de référence pour tester des APIs, vérifier qu'un service répond, et inspecter les headers HTTP.

```bash
# Requête GET simple
curl https://httpbin.org/get

# Voir les headers de réponse
curl -I https://httpbin.org/get

# Voir tout : connexion, headers, corps
curl -v https://httpbin.org/get

# Requête POST avec JSON
curl -X POST https://httpbin.org/post \
  -H "Content-Type: application/json" \
  -d '{"key": "value"}'

# Tester avec un timeout (utile pour détecter les services lents)
curl --max-time 5 https://monservice.com

# Télécharger un fichier
curl -O https://example.com/fichier.zip

# Suivre les redirections
curl -L http://example.com
```

**Cas d'usage Cloud typique :**

```bash
# Vérifier qu'un service derrière un Load Balancer répond
curl -v https://api.monapp.com/health

# Tester depuis une EC2 si elle peut joindre Internet
curl ifconfig.me   # → affiche l'IP publique vue de l'extérieur

# Tester si une EC2 peut atteindre un autre service interne
curl http://10.0.10.55:3000/health
```

---

### netstat — vue d'ensemble des connexions (ancienne méthode)

`netstat` est l'ancienne version de `ss`. Elle est encore très présente dans la documentation et sur les anciens systèmes, donc utile à connaître.

```bash
# Connexions TCP actives
netstat -tn

# Ports en écoute avec processus
netstat -tlnp

# Statistiques réseau globales
netstat -s
```

> Sur les systèmes modernes, préfère `ss` — c'est plus rapide et plus précis. Mais si tu atterris sur un vieux serveur sans `ss`, `netstat` sera là.

---

### Récapitulatif des commandes

| Commande | Ce qu'elle fait | Cas d'usage |
|----------|----------------|-------------|
| `ip a` | Affiche interfaces et IPs | Vérifier son IP, son masque |
| `ip r` | Affiche la table de routage | Vérifier le Default Gateway |
| `ip neigh` | Cache ARP (IP → MAC) | Voir les machines connues sur le réseau local |
| `ping` | Teste si une machine est joignable | Premier diagnostic réseau |
| `traceroute` | Affiche le chemin des paquets | Identifier où ça bloque |
| `ss -tnp` | Connexions TCP actives | Voir qui parle à qui |
| `ss -tlnp` | Ports en écoute | Voir quels services tournent |
| `curl -v` | Requête HTTP détaillée | Tester une API, inspecter les headers |
| `netstat -tlnp` | Ports en écoute (ancienne méthode) | Compatibilité anciens systèmes |

---

## 5. Le firewall sous Linux

### Pourquoi c'est important

Un firewall contrôle quel trafic est autorisé à entrer et sortir d'une machine. C'est la dernière ligne de défense avant qu'un paquet atteigne ton application. Sur Linux, le firewall est intégré dans le noyau — il est toujours présent, qu'il soit configuré ou non.

Comprendre le firewall Linux, c'est comprendre les bases de la sécurité réseau qui se retrouvent dans les Security Groups AWS, les Network ACLs, et tous les systèmes de filtrage Cloud.

### iptables — le firewall du noyau Linux

**iptables** est l'outil qui parle directement au système de filtrage du noyau Linux (Netfilter). Il est puissant mais complexe — c'est pourquoi des outils plus simples comme `ufw` ont été créés par-dessus.

iptables organise ses règles en **chaînes** :

```
INPUT   → trafic qui ENTRE sur la machine (connexions entrantes)
OUTPUT  → trafic qui SORT de la machine (connexions sortantes)
FORWARD → trafic qui TRAVERSE la machine (si elle fait office de routeur)
```

Et chaque règle finit par une **action** :

```
ACCEPT  → autoriser le paquet
DROP    → bloquer silencieusement (l'expéditeur ne sait pas)
REJECT  → bloquer et notifier l'expéditeur
```

```bash
# Voir les règles actuelles
sudo iptables -L -n -v

# Autoriser le trafic SSH entrant (port 22)
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# Autoriser le trafic HTTP entrant (port 80)
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT

# Bloquer tout le reste en entrée
sudo iptables -A INPUT -j DROP

# Supprimer toutes les règles (attention !)
sudo iptables -F
```

### ufw — la version simplifiée

**ufw** (Uncomplicated Firewall) est une interface plus humaine pour iptables. C'est ce qu'on utilise sur Ubuntu pour configurer le firewall simplement.

```bash
# Voir le statut
sudo ufw status

# Activer ufw
sudo ufw enable

# Autoriser SSH (sinon tu te coupes l'accès !)
sudo ufw allow ssh
sudo ufw allow 22/tcp

# Autoriser HTTP et HTTPS
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Autoriser un port spécifique
sudo ufw allow 5432/tcp

# Bloquer un port
sudo ufw deny 3306/tcp

# Autoriser depuis une IP spécifique uniquement
sudo ufw allow from 192.168.1.5 to any port 22

# Voir les règles numérotées
sudo ufw status numbered

# Supprimer une règle par numéro
sudo ufw delete 3
```

### Lien direct avec les Security Groups AWS

Le Security Group AWS est conceptuellement identique à ufw/iptables, mais géré par AWS au niveau du réseau virtuel :

| Concept Linux | Équivalent AWS |
|---|---|
| `ufw allow 443/tcp` | Règle SG : port 443, TCP, Inbound |
| `ufw allow from 10.0.1.0/24` | Règle SG : source = 10.0.1.0/24 |
| `ufw deny` (DROP) | Absence de règle dans le SG (deny par défaut) |
| Chaîne INPUT | Règles Inbound du Security Group |
| Chaîne OUTPUT | Règles Outbound du Security Group |

La différence principale : iptables/ufw sont sur la machine elle-même. Les Security Groups AWS filtrent **avant** que le trafic atteigne l'instance — au niveau de l'hyperviseur réseau AWS.

```
Internet
    │
    │  [Security Group AWS]  ← filtre ici, avant l'EC2
    │
    ▼
  EC2
    │
    │  [ufw / iptables]  ← filtre ici, sur l'OS de l'EC2
    ▼
  Application
```

En pratique sur AWS, on configure principalement les Security Groups. ufw sur l'EC2 est une couche de défense supplémentaire (defense in depth).

---

## 6. Les outils d'analyse réseau avancés

### Pourquoi c'est important

Les commandes vues en section 4 (`ping`, `ss`, `curl`) répondent à des questions simples : "est-ce joignable ?", "qui écoute ?", "est-ce que l'API répond ?". Mais parfois le problème est plus subtil — un paquet qui se perd, un handshake qui n'aboutit pas, une latence anormale sur un routeur précis.

Les outils de cette section te donnent une **vision radiographique** du réseau. Tu ne te contentes plus de savoir si ça marche ou pas — tu vois **exactement ce qui circule**, octet par octet, paquet par paquet. C'est la différence entre un médecin qui demande "tu as mal ?" et un médecin qui fait une radio.

Sur un serveur Cloud en production, ces outils sont souvent la seule façon de comprendre un problème réseau intermittent ou inexplicable.

---

### Wireshark — voir les paquets en temps réel

**Wireshark** est l'analyseur de protocoles réseau le plus utilisé au monde. Il capture tout le trafic qui passe par une interface réseau et te permet de l'inspecter couche par couche, en temps réel ou après coup.

C'est l'outil qui rend **visible** tout ce qu'on a étudié en théorie — le handshake TCP, les requêtes ARP, le chiffrement TLS, les requêtes DNS. Avec Wireshark, ces concepts ne sont plus abstraits.

#### Installation

```bash
sudo apt install wireshark

# Autoriser la capture sans root
sudo usermod -aG wireshark $USER
newgrp wireshark
```

#### Les filtres d'affichage — le cœur de Wireshark

Wireshark capture tout — sans filtre, tu es noyé sous des milliers de paquets. Les filtres te permettent de voir uniquement ce qui t'intéresse :

```
# Filtres par protocole
arp                     → uniquement les paquets ARP
icmp                    → uniquement ping
dns                     → uniquement DNS
tcp                     → tout le trafic TCP
http                    → requêtes HTTP en clair (port 80)
tls                     → handshakes TLS

# Filtres par port
tcp.port == 443         → HTTPS
tcp.port == 22          → SSH
tcp.port == 5432        → PostgreSQL

# Filtres par IP
ip.addr == 8.8.8.8      → trafic vers/depuis cette IP
ip.src == 192.168.1.10  → trafic sortant de cette machine
ip.dst == 142.250.74.46 → trafic entrant vers cette IP

# Filtres TCP avancés
tcp.flags.syn == 1      → paquets SYN (début de connexions)
tcp.flags.reset == 1    → connexions réinitialisées (erreurs)

# Combiner les filtres
tcp.port == 443 and ip.addr == 8.8.8.8
dns or arp
```

#### Exemple concret — observer un TCP handshake

```
# Tu lances la capture avec le filtre : tcp.port == 443
# Puis dans un terminal : curl https://google.com

# Wireshark affiche :

N°   Time    Source           Destination      Info
1    0.000   192.168.178.X    142.250.74.46    TCP  → SYN
             [ton PC]         [Google]         "je veux me connecter"

2    0.012   142.250.74.46    192.168.178.X    TCP  → SYN-ACK
             [Google]         [ton PC]         "d'accord, je suis prêt"

3    0.012   192.168.178.X    142.250.74.46    TCP  → ACK
             [ton PC]         [Google]         "compris, on commence"

4    0.012   192.168.178.X    142.250.74.46    TLS  → Client Hello
             [ton PC]         [Google]         "voici mes paramètres TLS"

5    0.025   142.250.74.46    192.168.178.X    TLS  → Server Hello + Certificate
             [Google]         [ton PC]         "voici mon certificat"

6+          ...données chiffrées...            impossible à lire → c'est HTTPS ✅
```

Clique sur n'importe quel paquet → Wireshark l'ouvre couche par couche :

```
▼ Frame 1 (couche 1 — physique)
▼ Ethernet II (couche 2 — MAC source et destination)
▼ Internet Protocol (couche 3 — IP source et destination)
▼ Transmission Control Protocol (couche 4 — ports, flags, numéros de séquence)
```

#### Sauvegarder et partager une capture

```bash
# Dans Wireshark : File → Save As → capture.pcap
# Ou depuis le terminal avec tcpdump (voir ci-dessous)
# Le format .pcap est standard — lisible par Wireshark, tcpdump, etc.
```

---

### tcpdump — Wireshark en ligne de commande

**tcpdump** fait la même chose que Wireshark mais entièrement en terminal. C'est l'outil indispensable sur un **serveur Linux sans interface graphique** — une EC2 AWS, un VPS, un conteneur. Tu ne peux pas lancer Wireshark sur une EC2, mais tu peux toujours lancer tcpdump.

#### Syntaxe générale

```bash
sudo tcpdump [options] [filtre]

# Options essentielles :
# -i wlan0    → interface à écouter
# -n          → ne pas résoudre les noms (plus rapide, plus lisible)
# -v          → verbose (plus de détails)
# -c 10       → s'arrêter après 10 paquets
# -w fichier  → sauvegarder dans un fichier .pcap
# -r fichier  → lire un fichier .pcap sauvegardé
# -A          → afficher le contenu en ASCII (pour voir le texte)
```

#### Exemples concrets avec output

```bash
# Capturer tout le trafic (limité à 5 paquets)
sudo tcpdump -i wlan0 -n -c 5

# Output :
13:42:01 IP 192.168.178.X.52341 > 142.250.74.46.443: TCP Flags [P.] seq 1:100
13:42:01 IP 142.250.74.46.443 > 192.168.178.X.52341: TCP Flags [.] ack 100
13:42:01 IP 192.168.178.X.43210 > 8.8.8.8.53: UDP, length 32       ← requête DNS
13:42:01 IP 8.8.8.8.53 > 192.168.178.X.43210: UDP, length 48       ← réponse DNS
13:42:01 IP 192.168.178.X.52341 > 142.250.74.46.443: TCP Flags [F.] ← fermeture
```

```bash
# Capturer uniquement ICMP (ping)
sudo tcpdump -i wlan0 -n icmp

# Puis dans un autre terminal : ping -c 2 google.com
# Output :
13:43:10 IP 192.168.178.X > 142.250.74.46: ICMP echo request, id 1, seq 1
13:43:10 IP 142.250.74.46 > 192.168.178.X: ICMP echo reply, id 1, seq 1
13:43:11 IP 192.168.178.X > 142.250.74.46: ICMP echo request, id 1, seq 2
13:43:11 IP 142.250.74.46 > 192.168.178.X: ICMP echo reply, id 1, seq 2
# Chaque ping = 1 request + 1 reply
```

```bash
# Capturer DNS et voir les requêtes/réponses
sudo tcpdump -i wlan0 -n port 53

# Puis : dig google.com
# Output :
13:44:05 IP 192.168.178.X.54321 > 192.168.178.1.53: UDP "google.com A?"
13:44:05 IP 192.168.178.1.53 > 192.168.178.X.54321: UDP "142.250.74.46"
# 1 requête vers le resolver (Fritz!Box) → 1 réponse avec l'IP
```

```bash
# Voir le TCP handshake complet
sudo tcpdump -i wlan0 -n "host google.com and tcp"

# Puis : curl https://google.com
# Output :
IP 192.168.178.X.52500 > 142.250.74.46.443: Flags [S]        ← SYN
IP 142.250.74.46.443 > 192.168.178.X.52500: Flags [S.]       ← SYN-ACK
IP 192.168.178.X.52500 > 142.250.74.46.443: Flags [.]        ← ACK
IP 192.168.178.X.52500 > 142.250.74.46.443: Flags [P.] TLS   ← données
IP 142.250.74.46.443 > 192.168.178.X.52500: Flags [P.] TLS   ← réponse
IP 192.168.178.X.52500 > 142.250.74.46.443: Flags [F.]       ← fermeture

# Flags TCP :
# [S]  = SYN       (connexion initiée)
# [S.] = SYN-ACK   (connexion acceptée)
# [.]  = ACK       (confirmation)
# [P.] = PUSH-ACK  (données envoyées)
# [F.] = FIN       (fermeture propre)
# [R]  = RESET     (fermeture brutale / erreur)
```

```bash
# Sauvegarder pour analyser dans Wireshark plus tard
sudo tcpdump -i wlan0 -n -w /tmp/capture.pcap

# Lire le fichier sauvegardé
sudo tcpdump -r /tmp/capture.pcap
# Ou ouvrir directement dans Wireshark : File → Open → capture.pcap
```

#### Cas d'usage Cloud typique

```bash
# Sur une EC2 — débugger pourquoi l'app ne reçoit pas les requêtes
sudo tcpdump -i eth0 -n port 3000

# Si tu vois les paquets arriver → problème dans l'app
# Si tu ne vois rien → problème réseau (Security Group, Route Table)

# Capturer le trafic entre EC2 et RDS
sudo tcpdump -i eth0 -n host 10.0.20.5 and port 5432
```

---

### mtr — traceroute en temps réel avec statistiques

**mtr** (Matt's Traceroute) combine `ping` et `traceroute` en un seul outil interactif. Là où `traceroute` fait une seule passe et s'arrête, `mtr` envoie des paquets **en continu** et calcule des statistiques sur la durée — latence moyenne, perte de paquets, stabilité.

C'est l'outil de référence pour diagnostiquer des problèmes de performance réseau ou des pertes de paquets intermittentes.

#### Lancer mtr

```bash
# Interface interactive (temps réel)
mtr google.com

# Mode rapport — envoie 10 cycles et affiche un résumé
mtr --report --report-cycles 10 google.com

# Utiliser TCP au lieu d'ICMP (si ICMP est bloqué par un firewall)
sudo mtr --tcp --port 443 google.com
```

#### Lire le résultat

```
# mtr --report -c 10 google.com

Host                         Loss%   Snt  Last   Avg  Best  Wrst StDev
 1. 192.168.178.1             0.0%    10   1.1   1.0   0.9   1.3   0.1  ← Fritz!Box
 2. 10.45.0.1                 0.0%    10   8.2   8.0   7.8   8.5   0.2  ← Telekom hop 1
 3. 87.234.12.5               0.0%    10   9.1   9.0   8.8   9.4   0.2  ← Telekom hop 2
 4. 72.14.232.1               0.0%    10  10.5  10.4  10.1  10.9   0.3  ← réseau Google
 5. 142.250.74.46             0.0%    10  12.1  12.0  11.8  12.4   0.2  ← Google ✅

# Colonnes :
# Loss%  → % de paquets perdus sur ce hop
# Snt    → nombre de paquets envoyés
# Last   → latence du dernier paquet (ms)
# Avg    → latence moyenne
# Best   → meilleure latence mesurée
# Wrst   → pire latence mesurée
# StDev  → écart-type (faible = stable, élevé = fluctuant)
```

#### Interpréter les résultats

```
Cas 1 — Tout à 0% de perte, latences stables
→ Réseau sain ✅

Cas 2 — Un hop à 20% de perte, les suivants à 0%
→ Ce routeur déprioritise les réponses ICMP/mtr (firewall)
→ Le trafic normal passe bien — pas un vrai problème ✅

Cas 3 — Un hop à 20% de perte ET les suivants aussi à 20%
→ Vraie perte de paquets à partir de ce hop ❌
→ Problème réseau réel entre ce routeur et la destination

Cas 4 — Latence qui explose à partir d'un hop précis
→ Congestion ou lien lent à ce niveau
→ Si c'est sur le backbone Internet → rien à faire
→ Si c'est dans ton réseau → investiguer
```

#### Cas d'usage Cloud

```bash
# Mesurer la latence vers une région AWS depuis ta machine
mtr --report -c 10 ec2.eu-west-1.amazonaws.com

# Comparer deux régions AWS pour choisir la plus proche
mtr --report -c 5 ec2.eu-west-1.amazonaws.com    # Irlande
mtr --report -c 5 ec2.eu-central-1.amazonaws.com # Frankfurt

# Frankfurt sera généralement plus rapide depuis l'Allemagne
```

---

### netcat — créer des connexions TCP/UDP manuellement

**netcat** (ou `nc`) est l'outil le plus simple et le plus polyvalent du réseau. Il peut créer une connexion TCP ou UDP vers n'importe quel port, écouter sur n'importe quel port, et transférer des données. On l'appelle parfois le "couteau suisse du réseau".

Son utilité principale dans le Cloud : **vérifier si un port est accessible** avant même d'avoir une application qui tourne dessus.

#### Syntaxe générale

```bash
nc [options] [hôte] [port]

# Options essentielles :
# -l        → mode écoute (serveur)
# -z        → mode scan (tester un port sans envoyer de données)
# -v        → verbose (afficher le résultat)
# -n        → ne pas résoudre les noms DNS
# -w 3      → timeout de 3 secondes
# -u        → utiliser UDP au lieu de TCP
```

#### Exemples concrets

**Tester si un port est ouvert :**

```bash
nc -zv google.com 443
# Output si ouvert :
Connection to google.com (142.250.74.46) 443 port [tcp/https] succeeded!

nc -zv google.com 23
# Output si fermé :
nc: connect to google.com port 23 (tcp) failed: Connection refused

nc -zv -w 3 10.0.20.5 5432
# Output si bloqué par firewall (timeout) :
nc: connect to 10.0.20.5 port 5432 (tcp) timed out: Operation now in progress

# La différence est importante :
# "Connection refused" → le serveur est joignable, rien n'écoute sur ce port
# "timed out"          → le firewall bloque silencieusement (DROP)
```

**Créer un serveur TCP et s'y connecter :**

```bash
# Terminal 1 — serveur qui écoute sur le port 9000
nc -l 9000

# Terminal 2 — client qui se connecte
nc localhost 9000

# Tape du texte dans Terminal 2 → apparaît dans Terminal 1
# Tape du texte dans Terminal 1 → apparaît dans Terminal 2
# C'est du TCP brut bidirectionnel — aucun protocole applicatif
```

**Envoyer une requête HTTP à la main :**

```bash
# Connexion directe au port 80 (HTTP non chiffré)
nc httpbin.org 80

# Une fois connecté, tape exactement ceci puis deux fois Entrée :
GET /get HTTP/1.1
Host: httpbin.org

# Réponse brute du serveur :
HTTP/1.1 200 OK
Content-Type: application/json
...
{"url": "http://httpbin.org/get", ...}

# Tu viens de faire une requête HTTP sans navigateur ni curl
# Tu vois exactement ce que curl fait en coulisses
```

**Transférer des données entre deux machines :**

```bash
# Machine réceptrice — écoute et écrit dans un fichier
nc -l 9001 > recu.txt

# Machine émettrice — envoie un fichier
cat fichier.txt | nc IP-destinataire 9001

# Ou envoyer un message simple
echo "hello depuis netcat" | nc localhost 9001
```

#### Cas d'usage Cloud — débugger une connexion EC2 → RDS

```bash
# Depuis l'EC2 app, tester si RDS est accessible
nc -zv 10.0.20.5 5432

# Résultat possible 1 : succeeded!
# → Le réseau est OK, le Security Group autorise la connexion
# → Si l'app ne se connecte pas quand même → problème de credentials ou config

# Résultat possible 2 : Connection refused
# → RDS est joignable mais PostgreSQL n'écoute pas (service arrêté ?)

# Résultat possible 3 : timed out
# → Le Security Group de RDS bloque le port 5432
# → Vérifier la règle Inbound de la RDS
```

---

### Récapitulatif des outils d'analyse

| Outil | Interface | Cas d'usage principal | Disponible sans GUI |
|-------|-----------|----------------------|---------------------|
| **Wireshark** | Graphique | Analyse visuelle des paquets, exploration des protocoles | ❌ |
| **tcpdump** | Terminal | Capture sur serveur distant, sauvegarde `.pcap` | ✅ |
| **mtr** | Terminal | Diagnostic de latence et perte de paquets sur le trajet | ✅ |
| **netcat** | Terminal | Tester si un port est ouvert, connexions TCP/UDP brutes | ✅ |

> Sur une EC2 AWS en production, tu n'as jamais Wireshark. Tu as tcpdump, mtr et netcat. Ces trois outils couvrent 95% des besoins de diagnostic réseau en Cloud.

---

## 7. Configuration réseau locale — DNS, ports éphémères et NTP

### Pourquoi c'est important

On a vu les concepts en théorie — DNS, ports éphémères, synchronisation des horloges. Dans cette section on descend au niveau de l'OS Linux pour voir comment tout ça est configuré concrètement, où les fichiers se trouvent, et comment les modifier. C'est ce que tu feras sur chaque EC2 que tu administreras.

---

### /etc/hosts — le DNS local de la machine

**`/etc/hosts`** est le premier endroit que Linux consulte avant de faire une requête DNS. C'est une liste statique de correspondances nom → IP, directement sur la machine.

```bash
cat /etc/hosts

# Contenu typique :
127.0.0.1       localhost
127.0.1.1       mon-pc
::1             localhost ip6-localhost

# Tu peux ajouter tes propres entrées :
192.168.178.20  nas.local
10.0.10.55      api.prod.internal
```

**Ordre de résolution DNS sur Linux :**

```
1. /etc/hosts          ← consulté en premier, toujours
2. Cache DNS local     ← systemd-resolved ou nscd
3. Serveurs DNS        ← configurés dans /etc/resolv.conf
```

Si un nom est dans `/etc/hosts`, Linux ne fait jamais de requête DNS pour ce nom — il utilise directement l'IP définie. C'est comme ça qu'on peut forcer `monapp.local` à pointer vers `127.0.0.1` en dev.

```bash
# Ajouter une entrée (nécessite sudo)
echo "127.0.0.1 monapp.local" | sudo tee -a /etc/hosts

# Tester
ping monapp.local
curl http://monapp.local:3000
```

**Cas d'usage Cloud :**

```
Sur une EC2, /etc/hosts est souvent utilisé pour :
- Forcer la résolution d'un nom interne sans DNS
- Override temporaire pendant une migration
- Tests locaux sans toucher le DNS réel
```

---

### /etc/resolv.conf — configurer les serveurs DNS

**`/etc/resolv.conf`** indique à Linux quels serveurs DNS contacter pour résoudre les noms.

```bash
cat /etc/resolv.conf

# Contenu typique sur Ubuntu avec systemd-resolved :
nameserver 127.0.0.53       ← resolver local (systemd-resolved)
options edns0 trust-ad

# Sur une EC2 AWS :
nameserver 10.0.0.2         ← serveur DNS AWS (toujours VPC_CIDR + 2)
search eu-west-1.compute.internal
```

```bash
# Voir quel serveur DNS ta machine utilise réellement
resolvectl status

# Voir le cache DNS local (entrées mémorisées)
resolvectl statistics

# Vider le cache DNS local
sudo resolvectl flush-caches

# Tester la résolution via un serveur DNS spécifique
dig @8.8.8.8 google.com        # via Google DNS
dig @192.168.178.1 google.com  # via ta Fritz!Box
```

**Le serveur DNS AWS — une règle à connaître :**

Dans tout VPC AWS, le serveur DNS est toujours accessible à l'adresse **`VPC_CIDR + 2`** :

```
VPC CIDR : 10.0.0.0/16
DNS AWS  : 10.0.0.2     (toujours)

VPC CIDR : 172.16.0.0/12
DNS AWS  : 172.16.0.2   (toujours)
```

C'est cette adresse qui apparaît dans `/etc/resolv.conf` sur toutes tes EC2.

---

### Les ports éphémères — comment l'OS les gère

On a vu en théorie que l'OS choisit automatiquement le port source de chaque connexion. Voici comment ça fonctionne concrètement sous Linux.

**La plage des ports éphémères :**

```bash
# Voir la plage configurée sur ta machine
cat /proc/sys/net/ipv4/ip_local_port_range

# Résultat typique :
32768   60999
# → Linux utilise les ports 32768 à 60999 comme ports source
```

**Comment l'OS choisit un port :**

```
Tu ouvres une connexion vers google.com:443

Linux cherche un port libre dans [32768, 60999]
→ vérifie que le port n'est pas déjà utilisé
→ assigne le premier disponible (ex: 52341)
→ crée l'entrée dans la table des connexions

Connexion : 192.168.178.10:52341 → 142.250.74.46:443
```

**Voir les ports éphémères en action :**

```bash
# Ouvre plusieurs connexions simultanées
curl https://google.com &
curl https://github.com &
curl https://aws.amazon.com &

# Voir les ports source utilisés
ss -tn | grep ESTAB

# Résultat :
ESTAB  192.168.178.10:54231  142.250.74.46:443    ← google
ESTAB  192.168.178.10:48102  140.82.121.4:443     ← github
ESTAB  192.168.178.10:51876  54.239.28.85:443     ← aws
# Chaque connexion a son propre port source ✅
```

**Conflit de ports — comment la Fritz!Box gère :**

```
Machine A (192.168.178.10) choisit port 5555 → google.com
Machine B (192.168.178.11) choisit port 5555 → google.com

Les deux arrivent à la Fritz!Box avec port src = 5555
→ Pas de conflit côté local (IPs privées différentes)
→ Conflit potentiel côté Internet (même IP publique)

Fritz!Box résout :
  192.168.178.10:5555 → 90.45.123.67:5555  (inchangé)
  192.168.178.11:5555 → 90.45.123.67:5556  (renommé)

Table NAT :
  port public 5555 → 192.168.178.10:5555
  port public 5556 → 192.168.178.11:5555
```

---

### NTP en pratique sous Linux

On a vu le concept de NTP dans le Module 1. Voici comment le vérifier et le configurer sur Linux.

**Vérifier l'état de la synchronisation :**

```bash
# Voir l'heure actuelle et l'état de synchronisation
timedatectl

# Résultat :
               Local time: Mon 2024-10-14 15:32:05 CEST
           Universal time: Mon 2024-10-14 13:32:05 UTC
                 RTC time: Mon 2024-10-14 13:32:05
                Time zone: Europe/Berlin (CEST, +0200)
System clock synchronized: yes          ← ✅ synchronisé
              NTP service: active        ← ✅ service actif
          RTC in local TZ: no
```

```bash
# Voir les serveurs NTP utilisés et leur état
timedatectl show-timesync --all

# Ou avec systemd-timesyncd
systemctl status systemd-timesyncd

# Voir les statistiques de synchro (offset, délai)
ntpq -p    # si ntpd est installé
chronyc tracking   # si chrony est installé
```

**Configurer les serveurs NTP :**

```bash
# Fichier de configuration systemd-timesyncd
sudo nano /etc/systemd/timesyncd.conf

# Contenu à modifier :
[Time]
NTP=pool.ntp.org time.google.com
FallbackNTP=ntp.ubuntu.com

# Redémarrer le service
sudo systemctl restart systemd-timesyncd

# Vérifier
timedatectl
```

**Sur une EC2 AWS :**

```bash
# AWS fournit son propre serveur NTP link-local
# Il est configuré automatiquement sur toutes les EC2

cat /etc/systemd/timesyncd.conf
# NTP=169.254.169.123   ← Amazon Time Sync Service

# Vérifier que la synchro est active
timedatectl | grep synchronized
# System clock synchronized: yes  ✅
```

**Pourquoi UTC dans les logs Cloud :**

```bash
# Voir les logs système avec timestamps UTC
journalctl --utc -n 20

# Sur AWS CloudWatch, tous les logs sont en UTC
# Si ton app loggue en heure locale → problèmes de corrélation

# Bonne pratique : toujours configurer tes apps pour logger en UTC
# En Node.js  : new Date().toISOString()       → UTC automatique
# En Python   : datetime.utcnow().isoformat()  → UTC
# En Java     : Instant.now().toString()        → UTC
```

---

## 8. nmap — scanner et auditer un réseau

### Pourquoi c'est important

En tant qu'architecte Cloud, tu seras régulièrement confronté à cette question : *"quels ports sont réellement ouverts sur cette EC2 ?"*. La configuration théorique de ton Security Group dit une chose — la réalité du réseau peut dire autre chose. **nmap** te permet de vérifier la réalité.

C'est aussi l'outil qu'utilisent les auditeurs de sécurité pour tester la surface d'attaque d'une infrastructure. Comprendre nmap, c'est comprendre ce qu'un attaquant voit de l'extérieur de ton réseau.

### Ce que nmap fait

**nmap** (Network Mapper) est un scanner réseau. Il envoie des paquets vers une ou plusieurs machines et analyse les réponses pour déterminer :
- Quelles machines sont actives sur un réseau
- Quels ports sont ouverts sur une machine
- Quels services tournent sur ces ports
- Quel OS tourne sur la machine cible

```bash
# Installation
sudo apt install nmap -y
```

### Scanner les ports d'une machine

```bash
# Scan de base — ports les plus courants (1000 ports)
nmap 192.168.178.1

# Scanner une IP distante
nmap scanme.nmap.org    # site de test officiel nmap

# Output typique :
PORT     STATE  SERVICE
22/tcp   open   ssh
80/tcp   open   http
443/tcp  open   https
3306/tcp closed mysql
8080/tcp filtered http-proxy

# États possibles :
# open     → port ouvert, un service écoute
# closed   → port fermé, la machine répond mais rien n'écoute
# filtered → firewall bloque — pas de réponse (comme ufw DROP)
```

### Types de scans essentiels

```bash
# Scanner tous les ports (1-65535) — plus lent mais complet
nmap -p- 192.168.178.1

# Scanner des ports spécifiques
nmap -p 22,80,443,5432 192.168.178.1

# Scanner une plage de ports
nmap -p 1-1000 192.168.178.1

# Détection de version des services (-sV)
nmap -sV 192.168.178.1
# Output :
# 22/tcp  open  ssh     OpenSSH 8.9p1
# 80/tcp  open  http    nginx 1.24.0

# Détection de l'OS (-O) — nécessite sudo
sudo nmap -O 192.168.178.1

# Scan "silencieux" SYN — plus rapide, moins visible dans les logs
sudo nmap -sS 192.168.178.1

# Scan UDP (DNS, NTP, etc.)
sudo nmap -sU -p 53,123,161 192.168.178.1
```

### Scanner un réseau entier

```bash
# Découvrir toutes les machines actives sur ton réseau local
nmap -sn 192.168.178.0/24

# Output :
# Nmap scan report for 192.168.178.1   (Fritz!Box)
# Nmap scan report for 192.168.178.10  (ton PC)
# Nmap scan report for 192.168.178.11  (téléphone)
# Nmap scan report for 192.168.178.20  (TV)

# Scanner tous les ports de toutes les machines du réseau
nmap -p 22,80,443 192.168.178.0/24
```

### Lire un rapport nmap complet

```bash
# Scan complet avec détails
sudo nmap -sS -sV -O -p- --open 192.168.178.1

# --open → afficher uniquement les ports ouverts
# -sS    → scan SYN (rapide, discret)
# -sV    → détecter les versions des services
# -O     → détecter l'OS
# -p-    → tous les ports

# Output :
PORT    STATE SERVICE VERSION
22/tcp  open  ssh     OpenSSH 8.2p1
80/tcp  open  http    lighttpd 1.4.59
443/tcp open  https   lighttpd 1.4.59
OS details: Linux 4.15 - 5.6
```

### nmap dans le Cloud AWS

```bash
# Depuis ton laptop — vérifier ce qui est visible depuis Internet
# (ce que voit un attaquant externe)
nmap -sV monapp.com

# Depuis une EC2 — auditer un subnet interne
# (ce que voit une machine compromise à l'intérieur)
nmap -sn 10.0.0.0/24      # quelles EC2 sont actives ?
nmap -p 5432 10.0.20.0/24  # quelles machines exposent PostgreSQL ?

# Vérifier qu'une RDS n'est PAS accessible depuis Internet
nmap -p 5432 <IP-publique-de-ta-RDS>
# → filtered = Security Group bloque ✅
# → open     = problème de sécurité ❌
```

> **Note importante** : nmap génère du trafic réseau notable. Sur un réseau d'entreprise ou dans le Cloud, scanner des machines sans autorisation est illégal. En dev/test sur tes propres ressources AWS — c'est autorisé et recommandé.

---

## 9. SSH avancé — tunneling, port forwarding et bastion

### Pourquoi c'est important

Tu utilises SSH tous les jours pour te connecter à des serveurs. Mais SSH fait bien plus que ouvrir un terminal distant — c'est un **tunnel chiffré polyvalent** qui peut transporter n'importe quel type de trafic réseau.

Comprendre SSH avancé, c'est comprendre comment accéder à des ressources privées dans un VPC AWS (RDS, ElastiCache, services internes) sans les exposer à Internet, comment mettre en place un bastion host, et comment débugger des connexions complexes.

### Rappel — clés vs mots de passe

Avant d'aller plus loin, un point fondamental sur l'authentification SSH :

```
Authentification par mot de passe :
  Client → "je suis alice, mot de passe: xxxx"
  Serveur → vérifie dans /etc/shadow
  ❌ Vulnérable au brute-force
  ❌ Désactivé par défaut sur AWS EC2

Authentification par clé (recommandée) :
  Tu génères une paire de clés :
    → clé privée  : reste sur ta machine (JAMAIS partagée)
    → clé publique: copiée sur le serveur (~/.ssh/authorized_keys)

  Connexion :
    Client prouve qu'il possède la clé privée
    (sans jamais l'envoyer sur le réseau)
    Serveur vérifie avec la clé publique
    ✅ Impossible à brute-forcer
    ✅ Standard AWS EC2
```

```bash
# Générer une paire de clés
ssh-keygen -t ed25519 -C "mon-commentaire"
# → crée ~/.ssh/id_ed25519       (privée — chmod 600 obligatoire)
# → crée ~/.ssh/id_ed25519.pub   (publique — peut être partagée)

# Copier la clé publique sur un serveur
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@serveur

# Se connecter avec une clé spécifique
ssh -i ~/.ssh/ma-cle.pem ec2-user@54.12.34.56
```

### Le fichier ~/.ssh/config — simplifier les connexions

Au lieu de retaper les options SSH à chaque fois, on les stocke dans un fichier de config :

```bash
# ~/.ssh/config

Host mon-serveur
    HostName 54.12.34.56
    User ec2-user
    IdentityFile ~/.ssh/ma-cle.pem
    Port 22

Host ec2-prod
    HostName 10.0.10.42
    User ubuntu
    IdentityFile ~/.ssh/prod.pem

# Connexion simplifiée :
ssh mon-serveur      # au lieu de : ssh -i ~/.ssh/ma-cle.pem ec2-user@54.12.34.56
ssh ec2-prod
```

### Local Port Forwarding — accéder à un service distant en local

Le **local port forwarding** crée un tunnel entre un port local et un port distant. Tu peux accéder à un service qui tourne sur un serveur distant comme s'il tournait sur ta machine.

```
Cas d'usage typique : accéder à une RDS PostgreSQL
qui est dans un subnet privé AWS (pas d'accès direct)

SANS tunnel :
  Ton laptop → RDS (10.0.20.5:5432)  ← impossible, subnet privé

AVEC tunnel SSH via une EC2 publique :
  Ton laptop:5432 → [tunnel SSH] → EC2 publique → RDS:5432

Commande :
ssh -L 5432:10.0.20.5:5432 ec2-user@54.12.34.56

  -L = Local port forwarding
  5432 = port local (sur ton laptop)
  10.0.20.5:5432 = destination finale (la RDS)
  ec2-user@54.12.34.56 = le serveur SSH intermédiaire (EC2 publique)
```

```bash
# Syntaxe générale
ssh -L [port_local]:[host_distant]:[port_distant] [user@serveur_ssh]

# Exemples concrets :

# Accéder à une RDS PostgreSQL privée
ssh -L 5432:10.0.20.5:5432 ec2-user@54.12.34.56
# Puis dans un autre terminal :
psql -h localhost -p 5432 -U postgres  # ← se connecte via le tunnel

# Accéder à une interface web interne (Kibana, Grafana...)
ssh -L 5601:10.0.30.10:5601 ec2-user@54.12.34.56
# Puis ouvrir : http://localhost:5601

# Mode background (-N = pas de shell, -f = background)
ssh -fNL 5432:10.0.20.5:5432 ec2-user@54.12.34.56
```

### Remote Port Forwarding — exposer un service local vers l'extérieur

Le **remote port forwarding** fait l'inverse : expose un port local sur le serveur distant.

```bash
# Syntaxe
ssh -R [port_distant]:[host_local]:[port_local] [user@serveur]

# Exemple : exposer ton app locale (port 3000) sur le serveur distant
ssh -R 8080:localhost:3000 ec2-user@54.12.34.56
# → http://54.12.34.56:8080 affiche ton app locale

# Utile pour :
# - Démos rapides sans déployer
# - Tests de webhooks (Stripe, GitHub...) en dev local
```

### Dynamic Port Forwarding — SOCKS proxy

Le **dynamic port forwarding** transforme SSH en proxy SOCKS — tout le trafic réseau peut être routé via le tunnel.

```bash
# Créer un proxy SOCKS sur le port local 1080
ssh -D 1080 ec2-user@54.12.34.56

# Configurer ton navigateur pour utiliser ce proxy :
# SOCKS5 → localhost:1080

# Tout le trafic du navigateur passe par l'EC2
# → Tu "vois" Internet depuis la perspective de l'EC2
# → Utile pour tester l'accès à des ressources internes AWS
```

### Le Bastion Host (Jump Host) — le gardien du VPC

Un **bastion host** (ou jump host) est une EC2 dans un subnet public dont le seul rôle est de servir de point d'entrée SSH vers le reste du VPC privé. C'est le pattern de sécurité standard AWS.

```
Internet
    │
    │  SSH port 22
    ▼
[Bastion EC2]          ← subnet public, IP publique
(54.12.34.56)            Security Group : port 22 depuis ton IP uniquement
    │
    │  SSH interne
    ▼
[EC2 App]              ← subnet privé, pas d'IP publique
(10.0.10.42)             Security Group : port 22 depuis le bastion uniquement
    │
    ▼
[RDS PostgreSQL]       ← subnet privé BDD
(10.0.20.5)              Security Group : port 5432 depuis EC2 App uniquement
```

```bash
# Connexion en deux sauts manuels
ssh ec2-user@54.12.34.56           # hop 1 : bastion
ssh ec2-user@10.0.10.42            # hop 2 : depuis le bastion vers l'EC2 privée

# Connexion en un seul saut avec ProxyJump (-J)
ssh -J ec2-user@54.12.34.56 ec2-user@10.0.10.42

# Configurer dans ~/.ssh/config (méthode recommandée)
Host bastion
    HostName 54.12.34.56
    User ec2-user
    IdentityFile ~/.ssh/prod.pem

Host ec2-app
    HostName 10.0.10.42
    User ec2-user
    IdentityFile ~/.ssh/prod.pem
    ProxyJump bastion

# Connexion directe en une commande
ssh ec2-app    # SSH traverse automatiquement le bastion ✅

# Tunnel vers RDS via le bastion en un seul saut
ssh -J ec2-user@54.12.34.56 -L 5432:10.0.20.5:5432 ec2-user@10.0.10.42
```

### SSH Agent Forwarding — ne pas copier les clés

Quand tu te connectes via un bastion, tu dois pouvoir te connecter aux EC2 privées sans copier ta clé privée sur le bastion (mauvaise pratique de sécurité). L'**agent forwarding** résout ça :

```bash
# Ajouter ta clé à l'agent SSH local
ssh-add ~/.ssh/prod.pem

# Se connecter avec agent forwarding activé (-A)
ssh -A ec2-user@54.12.34.56

# Depuis le bastion, tu peux maintenant te connecter
# aux EC2 privées en utilisant ta clé locale
# sans qu'elle soit physiquement sur le bastion
ssh ec2-user@10.0.10.42   ← fonctionne grâce à l'agent forwarding ✅

# Dans ~/.ssh/config
Host bastion
    HostName 54.12.34.56
    User ec2-user
    IdentityFile ~/.ssh/prod.pem
    ForwardAgent yes
```

---

## Récapitulatif du Module 2

| Concept | Ce qu'il faut retenir |
|---------|----------------------|
| **ARP** | Traduit une IP locale en adresse MAC. Nécessaire pour tout envoi sur le réseau local. |
| **Switch** | Connecte les machines du même réseau. Travaille à la couche 2 (MAC). |
| **Default Gateway** | La "porte de sortie" du réseau local. Tout ce qui n'est pas local lui est envoyé. |
| **NAT / PAT** | La Fritz!Box renomme les ports en cas de conflit. IP privée + port = identifiant unique. |
| **Interface réseau** | Point de connexion entre une machine et un réseau. `eth0`, `wlan0`, `lo` sur Linux. |
| **Loopback (lo)** | Interface virtuelle = la machine elle-même. Adresse `127.0.0.1`. |
| **Table de routage** | La liste des règles qui dit à un paquet par où passer. `ip route show`. |
| **ip** | Commande Linux centrale pour interfaces, routage, ARP. |
| **ping** | Test de connectivité basique via ICMP. Premier réflexe de diagnostic. |
| **traceroute** | Affiche tous les routeurs traversés. Permet de localiser où ça bloque. |
| **ss** | Affiche les connexions TCP actives et les ports en écoute. |
| **curl** | Envoie des requêtes HTTP depuis le terminal. Essentiel pour tester des APIs. |
| **iptables / ufw** | Firewall Linux. Même logique que les Security Groups AWS. |
| **Wireshark** | Capture et analyse les paquets visuellement. Rend visible tout ce qu'on a étudié. |
| **tcpdump** | Wireshark en terminal. Indispensable sur les serveurs Cloud sans GUI. |
| **mtr** | Traceroute continu avec statistiques. Détecte les pertes de paquets intermittentes. |
| **netcat** | Tester si un port est ouvert. `nc -zv IP port` = premier réflexe de debug Cloud. |
| **/etc/hosts** | DNS local statique. Consulté avant tout serveur DNS. |
| **/etc/resolv.conf** | Serveurs DNS de la machine. Sur AWS : toujours `VPC_CIDR + 2`. |
| **Ports éphémères** | L'OS choisit automatiquement le port source (32768-60999) pour chaque connexion. |
| **NTP / timedatectl** | Synchronisation des horloges. Indispensable pour des logs Cloud cohérents. |
| **nmap** | Scanner les ports d'une machine ou d'un réseau. Auditer la surface d'attaque. |
| **SSH clés** | Clé privée sur ta machine, clé publique sur le serveur. Jamais de mot de passe en prod. |
| **Port forwarding** | Tunnel SSH pour accéder à des ressources privées (RDS, services internes). |
| **Bastion host** | EC2 publique servant de point d'entrée SSH vers le VPC privé. Pattern AWS standard. |
| **ProxyJump** | Connexion SSH en un saut via un bastion. `ssh -J bastion ec2-privee`. |

---

*→ Exercices pratiques : voir `module2-exercices.md`*
*→ Module suivant : Module 3 — Networking et conteneurs*
