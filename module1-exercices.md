# Module 1 — Les bases absolues du réseau
### Partie Exercices

> Complète la partie théorie (`module1-theorie.md`) avant de faire ces exercices.
> Durée estimée : 1h à 1h30

---

## Exercice 1 — Schématiser son application

**Objectif** : prendre conscience de la topologie réseau d'une application avant même d'écrire du code ou de configurer quoi que ce soit.

**Consigne** :
Prends une application que tu connais, ou invente-en une simple (par exemple : un site e-commerce avec un frontend, une API, et une base de données). Sur papier ou dans un fichier texte :

1. Liste tous les composants de l'application (serveur web, API, base de données, cache, etc.)
2. Dessine les connexions entre eux — qui parle à qui ?
3. Pour chaque composant, réponds à la question : **doit-il être accessible depuis Internet, ou uniquement depuis d'autres services internes ?**

**Ce qu'on attend** :
- Le Load Balancer ou l'API Gateway est le seul point d'entrée public
- Les serveurs d'application sont accessibles uniquement depuis le Load Balancer
- La base de données et le cache ne sont **jamais** exposés à Internet

---

## Exercice 2 — Lire des adresses IP

**Objectif** : savoir identifier rapidement si deux machines sont sur le même réseau, à la main.

**Consigne** :
Pour chaque paire d'adresses IP ci-dessous, le masque utilisé est `255.255.255.0` (soit `/24`). Détermine si les deux machines sont sur le **même réseau** ou sur des **réseaux différents**.

```
a) 10.0.1.5    et  10.0.1.200
b) 10.0.1.5    et  10.0.2.5
c) 192.168.0.1 et  192.168.1.1
d) 172.16.5.10 et  172.16.5.254
e) 10.0.10.1   et  10.0.11.1
```

**Astuce** : avec un `/24`, les 3 premiers octets doivent être identiques pour que les machines soient sur le même réseau.

<details>
<summary>Voir les réponses</summary>

- a) ✅ Même réseau — `10.0.1.x` pour les deux
- b) ❌ Réseaux différents — `10.0.1.x` vs `10.0.2.x`
- c) ❌ Réseaux différents — `192.168.0.x` vs `192.168.1.x`
- d) ✅ Même réseau — `172.16.5.x` pour les deux
- e) ❌ Réseaux différents — `10.0.10.x` vs `10.0.11.x`

</details>

---

## Exercice 3 — Calculer des CIDR

**Objectif** : maîtriser la relation entre notation CIDR et nombre d'adresses disponibles.

### Partie A — Calcul mental

Sans calculatrice, détermine le nombre d'adresses totales pour chaque bloc CIDR :

```
a) 10.0.0.0/24
b) 10.0.0.0/16
c) 10.0.0.0/28
d) 192.168.1.0/30
e) 10.0.0.0/8
```

**Rappel** : formule = `2^(32 - préfixe)`

<details>
<summary>Voir les réponses</summary>

- a) `/24` → 2^8 = **256** adresses (251 utilisables dans AWS)
- b) `/16` → 2^16 = **65 536** adresses
- c) `/28` → 2^4 = **16** adresses (11 utilisables dans AWS)
- d) `/30` → 2^2 = **4** adresses (utile pour les liens point à point)
- e) `/8`  → 2^24 = **16 777 216** adresses

</details>

### Partie B — Appartenance à un bloc

L'IP `10.0.1.100` appartient-elle aux blocs suivants ?

```
a) 10.0.1.0/24
b) 10.0.0.0/16
c) 10.0.2.0/24
d) 10.0.0.0/8
```

<details>
<summary>Voir les réponses</summary>

- a) ✅ Oui — `10.0.1.0` à `10.0.1.255` inclut `10.0.1.100`
- b) ✅ Oui — `10.0.0.0` à `10.0.255.255` inclut `10.0.1.100`
- c) ❌ Non — `10.0.2.0` à `10.0.2.255` ne contient pas `10.0.1.100`
- d) ✅ Oui — `10.0.0.0` à `10.255.255.255` inclut tout le reste

</details>

---

## Exercice 4 — Concevoir un plan d'adressage VPC

**Objectif** : appliquer CIDR pour concevoir un vrai plan réseau Cloud.

**Contexte** :
Tu déploies une plateforme SaaS sur AWS avec les contraintes suivantes :

- 2 environnements : `production` et `staging`
- Chaque environnement contient :
  - 1 subnet public (Load Balancer)
  - 1 subnet privé applicatif (serveurs)
  - 1 subnet privé base de données
- Chaque subnet doit pouvoir accueillir au moins 100 machines
- Le VPC de base est `10.0.0.0/16`

**Consigne** :
Propose un plan d'adressage CIDR complet. Justifie le choix de taille pour tes subnets.

<details>
<summary>Voir un exemple de solution</summary>

Un `/24` donne 251 adresses utilisables dans AWS → largement > 100 ✅

```
VPC : 10.0.0.0/16

PRODUCTION
  10.0.1.0/24   subnet-public-prod      (Load Balancer)
  10.0.2.0/24   subnet-app-prod         (Serveurs)
  10.0.3.0/24   subnet-db-prod          (Base de données)

STAGING
  10.0.11.0/24  subnet-public-staging
  10.0.12.0/24  subnet-app-staging
  10.0.13.0/24  subnet-db-staging
```

On laisse des "trous" entre les blocs (ex: 10.0.4 à 10.0.10 libres) pour pouvoir ajouter des subnets plus tard sans tout réorganiser.

</details>

---

## Exercice 5 — TCP ou UDP ?

**Objectif** : savoir choisir le bon protocole de transport selon le cas d'usage.

**Consigne** :
Pour chaque scénario, détermine si TCP ou UDP est le plus adapté, et explique pourquoi en une phrase.

```
a) Un admin se connecte en SSH sur un serveur pour exécuter des commandes
b) Une application résout le nom DNS "api.monapp.com"
c) Un service effectue une requête SQL sur une base PostgreSQL
d) Une app de visioconférence envoie un flux vidéo en direct
e) Un navigateur charge une page HTTPS
f) Un service de monitoring envoie des métriques toutes les secondes
```

<details>
<summary>Voir les réponses</summary>

- a) **TCP** — chaque commande doit arriver intacte et dans l'ordre. Perdre un caractère = commande corrompue.
- b) **UDP** — les requêtes DNS sont très courtes. Pas besoin du handshake TCP. Si la réponse ne revient pas, le client retente simplement.
- c) **TCP** — une requête SQL perdue = résultat faux ou transaction incomplète. Inacceptable.
- d) **UDP** — perdre quelques images est invisible. La latence est bien plus importante que la fiabilité complète.
- e) **TCP** — chaque octet de la page doit arriver intact. Une image corrompue ou un script manquant casse l'expérience.
- f) Ça dépend : **UDP** si une métrique perdue est acceptable (cas courant), **TCP** si chaque point de donnée est critique.

</details>

---

## Exercice 6 — Observer le DNS en ligne de commande

**Objectif** : utiliser `dig` pour inspecter la résolution DNS et comprendre ce qui se passe réellement.

**Prérequis** : avoir `dig` installé (`sudo apt install dnsutils` sur Ubuntu, inclus sur macOS).

### Commandes à exécuter

```bash
# 1. Résolution simple
dig google.com

# 2. Afficher uniquement les IPs (format court)
dig +short google.com

# 3. Interroger un serveur DNS spécifique (8.8.8.8 = Google DNS public)
dig @8.8.8.8 github.com

# 4. Voir les enregistrements MX (serveurs mail)
dig github.com MX

# 5. Tracer toute la chaîne de résolution depuis la racine
dig +trace github.com
```

### Questions

1. Combien d'IPs différentes retourne `dig +short google.com` ? Pourquoi plusieurs IPs pour un seul nom ?
2. Quel est le TTL de l'enregistrement A de `google.com` ? Quelle est l'implication ?
3. Que se passe-t-il si tu interroges `@1.1.1.1` (Cloudflare) au lieu de `@8.8.8.8` — obtiens-tu les mêmes IPs ?
4. Dans le résultat de `dig +trace`, combien d'étapes la résolution prend-elle ?

<details>
<summary>Voir les réponses attendues</summary>

1. Souvent 4 à 8 IPs — Google utilise le **DNS load balancing** pour distribuer le trafic entre ses serveurs dans le monde.
2. TTL souvent court (60-300s) — Google change ses IPs régulièrement pour le routage. Un TTL court permet une propagation rapide des changements.
3. Les IPs peuvent être légèrement différentes — chaque resolver DNS peut recevoir des réponses différentes selon sa localisation (anycast).
4. En général 3-4 étapes : racine → TLD → serveur authoritative → réponse finale.

</details>

---

## Exercice 7 — Inspecter du trafic HTTP avec curl

**Objectif** : observer concrètement ce qui circule dans une requête HTTP.

**Prérequis** : avoir `curl` installé (disponible par défaut sur Linux et macOS).

### Commandes à exécuter

```bash
# 1. Requête GET simple
curl https://httpbin.org/get

# 2. Voir uniquement les headers de réponse
curl -I https://httpbin.org/get

# 3. Voir TOUT : headers envoyés + headers reçus + corps (très utile pour débugger)
curl -v https://httpbin.org/get

# 4. Envoyer un POST avec du JSON
curl -X POST https://httpbin.org/post \
  -H "Content-Type: application/json" \
  -d '{"nom": "Alice", "role": "dev"}'

# 5. Provoquer différents codes de statut
curl -v https://httpbin.org/status/200
curl -v https://httpbin.org/status/404
curl -v https://httpbin.org/status/500

# 6. Suivre les redirections
curl -v https://httpbin.org/redirect/2
curl -L https://httpbin.org/redirect/2   # avec suivi automatique
```

### Questions

1. Dans le résultat de `curl -v`, identifie les lignes qui représentent le **handshake TLS**. Que se passe-t-il avant l'envoi de la requête HTTP ?
2. Quelle est la différence entre `curl -I` et `curl -v` ?
3. Que contient le header `Content-Type` dans la réponse du POST ?
4. Sans le flag `-L`, que retourne `curl` sur `/redirect/2` ? Quel code de statut ?
5. Comment `curl` indique-t-il la différence entre un `404` et un `500` dans sa sortie ?

<details>
<summary>Voir les réponses attendues</summary>

1. Les lignes `* TLSv1.3 ...` dans la sortie de `-v` montrent le handshake TLS. Avant même d'envoyer `GET /`, le client et le serveur négocient le chiffrement.
2. `-I` envoie une requête HEAD (pas de corps) et affiche uniquement les headers. `-v` montre tout : connexion, handshake, headers envoyés et reçus, et le corps.
3. `application/json` — le serveur indique qu'il répond en JSON.
4. Code `302 Found` avec un header `Location` pointant vers la prochaine URL. Sans `-L`, curl s'arrête là et n'suit pas.
5. Dans les deux cas curl affiche le code dans la première ligne de réponse (`HTTP/2 404` ou `HTTP/2 500`). La différence est sémantique : 404 = ressource inconnue, 500 = le serveur a planté.

</details>

---

## Exercice 8 — Concevoir des règles de Security Group

**Objectif** : traduire une architecture réseau en règles de firewall concrètes.

**Contexte** :
Tu déploies cette architecture sur AWS :

```
Internet  →  [Application Load Balancer]  →  [EC2 App]  →  [RDS PostgreSQL]
```

- Le Load Balancer doit accepter le trafic HTTP et HTTPS depuis Internet
- L'EC2 doit accepter uniquement le trafic venant du Load Balancer (port 3000, ton app Node.js)
- La RDS doit accepter uniquement le trafic venant de l'EC2 (port 5432)
- Un admin doit pouvoir se connecter en SSH à l'EC2 depuis une IP fixe : `203.0.113.5`

**Consigne** :
Complète le tableau de règles pour chaque ressource.

| Ressource | Direction | Port | Source | Raison |
|-----------|-----------|------|--------|--------|
| ALB | Inbound | ? | ? | ? |
| ALB | Inbound | ? | ? | ? |
| EC2 App | Inbound | ? | ? | ? |
| EC2 App | Inbound | ? | ? | ? |
| RDS | Inbound | ? | ? | ? |

<details>
<summary>Voir la solution</summary>

| Ressource | Direction | Port | Source | Raison |
|-----------|-----------|------|--------|--------|
| ALB | Inbound | 80 (TCP) | `0.0.0.0/0` | HTTP depuis n'importe où sur Internet |
| ALB | Inbound | 443 (TCP) | `0.0.0.0/0` | HTTPS depuis n'importe où sur Internet |
| EC2 App | Inbound | 3000 (TCP) | Security Group de l'ALB | Trafic venant uniquement du Load Balancer |
| EC2 App | Inbound | 22 (TCP) | `203.0.113.5/32` | SSH depuis l'IP fixe de l'admin uniquement |
| RDS | Inbound | 5432 (TCP) | Security Group de l'EC2 | Trafic venant uniquement du serveur app |

**Points importants** :
- La RDS n'a **aucune règle** autorisant Internet → elle est inaccessible depuis l'extérieur. ✅
- Utiliser le **Security Group comme source** (et non une plage IP) est la bonne pratique AWS : si l'EC2 change d'IP, la règle reste valide.
- Le SSH est restreint à `203.0.113.5/32` (une seule IP) — jamais `0.0.0.0/0` pour SSH en production.

</details>

---

## Exercice 11 — Identifier les couches OSI

**Objectif** : ancrer le modèle OSI en associant protocoles, outils et problèmes à leurs couches respectives.

### Partie A — À quelle couche OSI appartient chaque élément ?

```
a) Une adresse IP
b) Le protocole HTTP
c) Un câble ethernet
d) Le protocole TCP
e) Le chiffrement TLS
f) Une adresse MAC (adresse physique d'une carte réseau)
g) Le protocole DNS
h) Un routeur
i) Le protocole UDP
j) Le port 443
```

<details>
<summary>Voir les réponses</summary>

- a) **Couche 3** — les IPs sont l'adressage logique (Réseau)
- b) **Couche 7** — HTTP est un protocole applicatif (Application)
- c) **Couche 1** — le support de transmission (Physique)
- d) **Couche 4** — TCP gère la fiabilité et les ports (Transport)
- e) **Couche 6** — TLS chiffre les données (Présentation)
- f) **Couche 2** — les adresses MAC sont l'adressage physique (Liaison)
- g) **Couche 7** — DNS est un protocole applicatif (Application)
- h) **Couche 3** — les routeurs lisent et traitent les IPs (Réseau)
- i) **Couche 4** — UDP est aussi un protocole de transport (Transport)
- j) **Couche 4** — les ports appartiennent à la couche Transport

</details>

### Partie B — À quelle couche se situe le problème ?

Pour chaque scénario, identifie à quelle couche OSI se situe la cause probable.

```
a) Ton câble ethernet est débranché — tu n'as aucune connectivité
b) Ton app reçoit des données corrompues à cause de paquets perdus
c) Un Security Group AWS bloque le port 5432
d) Le certificat TLS de ton site est expiré
e) La route vers un subnet n'existe pas dans la Route Table AWS
f) Une requête arrive au mauvais serveur à cause d'une mauvaise config de Load Balancer HTTP
```

<details>
<summary>Voir les réponses</summary>

- a) **Couche 1** (Physique) — problème de support physique
- b) **Couche 4** (Transport) — UDP ne garantit pas la livraison
- c) **Couche 4** (Transport) — les Security Groups filtrent par port
- d) **Couche 6** (Présentation) — TLS est la couche de chiffrement
- e) **Couche 3** (Réseau) — le routage est une responsabilité couche 3
- f) **Couche 7** (Application) — le Load Balancer applicatif lit les URLs et headers HTTP

</details>

### Partie C — Remettre les couches dans l'ordre

Tu envoies une requête HTTPS depuis ton navigateur. Remets dans l'ordre ces événements :

```
A. Les bits sont envoyés sur le WiFi
B. TCP établit la connexion (handshake SYN/SYN-ACK/ACK)
C. HTTP construit la requête "GET /index.html HTTP/1.1"
D. IP ajoute les adresses source et destination
E. TLS chiffre la requête HTTP
F. Ethernet ajoute les adresses MAC source et destination
```

<details>
<summary>Voir la réponse</summary>

Ordre correct — on descend les couches de 7 vers 1 :

```
C  →  HTTP construit la requête         (Couche 7 — Application)
E  →  TLS chiffre                       (Couche 6 — Présentation)
B  →  TCP établit la connexion          (Couche 4 — Transport)
D  →  IP ajoute les adresses            (Couche 3 — Réseau)
F  →  Ethernet ajoute les adresses MAC  (Couche 2 — Liaison)
A  →  Les bits partent sur le WiFi      (Couche 1 — Physique)
```

À destination, le serveur fait exactement l'inverse — il remonte les couches de 1 à 7.

</details>

---

## Quiz final — Module 1

Réponds à ces questions sans regarder le cours.

1. Quelle est la différence entre une IP privée et une IP publique ?
2. Un subnet `/24` dans AWS — combien d'IPs sont disponibles pour tes machines ?
3. Pourquoi DNS utilise UDP plutôt que TCP pour la majorité des requêtes ?
4. `10.0.5.0` et `10.0.6.0` sont-elles dans le même `/24` ? Et dans le même `/16` ?
5. Tu changes l'IP de ton serveur mais les utilisateurs continuent d'arriver sur l'ancienne. Quelle est la cause probable, et comment l'aurais-tu évité ?
6. Un utilisateur reçoit une erreur `403 Forbidden` sur ton API. Qui est responsable — le client ou le serveur ?
7. Quel port ouvrir dans un Security Group pour autoriser SSH ?
8. Quelle est la différence entre HTTP et HTTPS sur le plan de la sécurité ?
9. Tu veux nommer tes services internes dans AWS (ex: `database.prod.internal`). Quel service AWS utilises-tu ?
10. Un service de streaming vidéo perd 1% de ses paquets UDP. Est-ce un problème grave ?

<details>
<summary>Voir les réponses</summary>

1. Une IP publique est unique mondiale et routable sur Internet. Une IP privée n'existe que dans un réseau local ou un VPC — deux machines sur des réseaux différents peuvent avoir la même IP privée sans conflit.
2. **251** — AWS réserve 5 adresses par subnet (réseau, routeur, DNS, usage futur, broadcast).
3. Les requêtes DNS sont courtes et il est acceptable d'en perdre une (le client retente simplement). UDP évite l'overhead du handshake TCP, ce qui rend la résolution bien plus rapide.
4. Non pour `/24` (réseaux différents : `10.0.5.x` ≠ `10.0.6.x`). Oui pour `/16` (même réseau `10.0.x.x`).
5. Le TTL DNS est encore actif — les resolvers ont l'ancienne réponse en cache. Pour l'éviter : baisser le TTL à 60s la veille de la migration.
6. Le **client** — les erreurs `4xx` indiquent une erreur côté client (ici, permissions insuffisantes).
7. **Port 22** (TCP).
8. HTTP envoie tout en clair — n'importe qui peut intercepter et lire les données. HTTPS chiffre tout via TLS et authentifie le serveur via son certificat.
9. **Route 53 Private Hosted Zone**.
10. Non, en général. UDP est choisi précisément parce que perdre quelques paquets est acceptable pour le streaming. L'impact visuel est quasi invisible, et la latence reste faible.

</details>

---

## Exercice 9 — Observer DHCP en action

**Objectif** : voir concrètement comment ta machine a obtenu son IP privée.

### Commandes à exécuter

```bash
# 1. Voir l'IP privée attribuée par DHCP et la durée du bail
ip a

# Sur Linux — voir les détails du bail DHCP actif
cat /var/lib/dhcp/dhclient.leases

# Ou selon ta distribution
cat /var/lib/dhclient/dhclient.leases

# 2. Voir l'adresse de ton routeur (ta Fritz!Box)
ip route show
# La ligne "default via X.X.X.X" = adresse de ta Fritz!Box

# 3. Forcer le renouvellement du bail DHCP (libère et redemande une IP)
sudo dhclient -r     # libère le bail
sudo dhclient        # redemande une IP
```

### Questions

1. Quelle IP privée ta machine a-t-elle reçu ? Sur quelle plage (`192.168.x.x` ou `10.x.x.x`) ?
2. Quelle est l'adresse IP de ta Fritz!Box (le "default gateway") ?
3. Après avoir forcé le renouvellement du bail, as-tu obtenu la même IP ou une différente ?
4. Quelle est la durée de ton bail DHCP ?

<details>
<summary>Voir les réponses attendues</summary>

1. Probablement dans `192.168.178.X` (plage par défaut des Fritz!Box de Telekom en Allemagne) ou `192.168.1.X` selon la config.
2. Souvent `192.168.178.1` pour une Fritz!Box Telekom — c'est elle qui gère le DHCP de ton réseau domestique.
3. En général la même — le serveur DHCP tend à donner la même IP à la même machine (basé sur l'adresse MAC). Mais ce n'est pas garanti.
4. Souvent 24h (86400 secondes) pour un réseau domestique.

</details>

---

## Exercice 10 — Utiliser ping et diagnostiquer avec ICMP

**Objectif** : utiliser `ping` pour tester la connectivité réseau à différents niveaux.

### Commandes à exécuter

```bash
# 1. Tester si une machine externe est joignable
ping google.com

# 2. Tester avec un nombre fixe de paquets (ici 4)
ping -c 4 google.com

# 3. Tester ta propre machine (loopback — toujours disponible)
ping 127.0.0.1

# 4. Tester ta Fritz!Box (remplace par ton adresse gateway)
ping 192.168.178.1

# 5. Tester une IP qui n'existe pas
ping 10.0.0.254

# 6. Comparer la latence vers différentes destinations
ping -c 4 google.com
ping -c 4 1.1.1.1        # Cloudflare DNS
ping -c 4 amazon.de
```

### Questions

1. Quelle est la latence moyenne vers `google.com` depuis ta machine ? Est-ce normal ?
2. Que se passe-t-il quand tu ping `127.0.0.1` ? Pourquoi cette adresse répond-elle toujours ?
3. Que retourne `ping 10.0.0.254` ? Quelle est la différence avec un ping qui reçoit une réponse ?
4. Quelle destination a la latence la plus faible ? Pourquoi selon toi ?
5. Si tu pinges une EC2 AWS et qu'elle ne répond pas, est-ce forcément qu'elle est en panne ?

<details>
<summary>Voir les réponses attendues</summary>

1. Depuis l'Allemagne, `google.com` répond généralement en 10-25ms. C'est normal — les serveurs Google ont des points de présence proches en Europe.
2. `127.0.0.1` est l'adresse de **loopback** — elle représente ta propre machine. Le paquet ne sort jamais sur le réseau, il fait un aller-retour interne. Elle répond toujours, même sans connexion réseau.
3. `ping 10.0.0.254` retourne `Destination Host Unreachable` ou timeout — cette IP n'existe pas sur ton réseau. Le routeur ne sait pas où envoyer le paquet.
4. `1.1.1.1` (Cloudflare) est souvent la plus rapide — c'est un serveur DNS optimisé pour la vitesse avec des points de présence partout dans le monde.
5. **Non** — l'EC2 peut très bien tourner normalement mais avoir son Security Group configuré pour bloquer ICMP. C'est même la configuration par défaut sur AWS. Un ping qui échoue ne suffit pas à conclure qu'une machine est en panne.

</details>

---

## Exercice 12 — Observer NAT en action

**Objectif** : voir concrètement la traduction d'adresse en comparant l'IP vue depuis l'intérieur et depuis l'extérieur du réseau.

```bash
# 1. Voir ton IP privée
ip addr show | grep "inet "

# 2. Voir ton IP publique (ce que le monde extérieur voit)
curl -s ifconfig.me

# 3. Voir plus d'infos sur ton IP publique
curl -s ipinfo.io/json

# 4. Vérifier que ton téléphone (en WiFi) partage la même IP publique
# Sur ton téléphone : https://whatismyip.com
# Compare avec curl ifconfig.me sur ton PC

# 5. Observer la table NAT de ta Fritz!Box
# http://192.168.178.1 → Internet → Verbindungen actives
```

**Questions :**

1. Quelle est la différence entre l'IP retournée par `ip addr show` et `curl ifconfig.me` ?
2. Ton téléphone en WiFi a-t-il la même IP publique que ton PC ? Pourquoi ?
3. Si tu ouvres 3 onglets sur 3 sites différents, comment la Fritz!Box sait-elle à quel onglet retransmettre chaque réponse ?
4. Dans AWS, quel service joue exactement le rôle de la Fritz!Box pour les subnets privés ?

<details>
<summary>Voir les réponses</summary>

1. `ip addr show` retourne ton IP **privée** (`192.168.178.X`). `curl ifconfig.me` retourne l'IP **publique** de ta Fritz!Box — ce que les serveurs Internet voient.
2. **Oui, la même** — les deux appareils sont derrière la même Fritz!Box. Tous les appareils du réseau partagent la même IP publique.
3. Grâce aux **numéros de port source** — chaque connexion a un port unique. La Fritz!Box maintient une table NAT qui associe chaque port à la connexion correspondante.
4. La **NAT Gateway AWS** — déployée dans un subnet public, elle reçoit le trafic des EC2 privées et remplace leurs IPs par son Elastic IP.

</details>

---

## Exercice 13 — Inspecter des certificats TLS

**Objectif** : voir le contenu d'un certificat TLS et comprendre la chaîne de confiance.

```bash
# 1. Voir les infos essentielles du certificat de google.com
echo | openssl s_client -connect google.com:443 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates

# 2. Faire pareil avec github.com
echo | openssl s_client -connect github.com:443 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates

# 3. Voir le certificat via curl
curl -v https://google.com 2>&1 | grep -E "subject|issuer|SSL"

# 4. Tester un certificat expiré (site de test maintenu exprès)
curl -v https://expired.badssl.com 2>&1 | tail -10

# 5. Forcer la connexion malgré le certificat invalide
curl --insecure https://expired.badssl.com
```

**Questions :**

1. Quel est l'émetteur (issuer) du certificat de `google.com` ?
2. Que se passe-t-il quand tu fais `curl https://expired.badssl.com` sans `--insecure` ?
3. Quelle est la différence entre `--insecure` dans curl et ce qu'un navigateur fait ?
4. Sur AWS, quel service renouvelle automatiquement les certificats ?

<details>
<summary>Voir les réponses</summary>

1. **GTS CA 1C3** (Google Trust Services) — CA intermédiaire de Google, signée par une CA racine reconnue.
2. curl retourne `SSL certificate problem: certificate has expired` et refuse la connexion.
3. `--insecure` bypasse **silencieusement** la vérification. Un navigateur affiche une page d'avertissement visible et demande une confirmation explicite.
4. **ACM** (AWS Certificate Manager) — renouvellement automatique, intégré avec ALB, CloudFront et API Gateway.

</details>

---

## Exercice 14 — Configurer un reverse proxy avec nginx

**Objectif** : mettre en place nginx comme reverse proxy devant une application — exactement comme un ALB AWS transmet le trafic à des EC2.

```bash
# 1. Installer nginx
sudo apt install nginx -y

# 2. Démarrer une app simple sur le port 3000
python3 -m http.server 3000 &

# 3. Vérifier que l'app répond directement
curl http://localhost:3000

# 4. Créer la config nginx reverse proxy
sudo nano /etc/nginx/sites-available/monapp

# Contenu à coller :
server {
    listen 8080;
    server_name localhost;

    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}

# 5. Activer et recharger
sudo ln -s /etc/nginx/sites-available/monapp /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx

# 6. Tester via nginx (port 8080)
curl http://localhost:8080

# 7. Comparer les headers
curl -v http://localhost:3000   # accès direct
curl -v http://localhost:8080   # via nginx

# 8. Voir les logs nginx
sudo tail -f /var/log/nginx/access.log

# 9. Arrêter l'app
kill %1
```

**Questions :**

1. Du point de vue du client, quelle est la différence entre accéder au port 3000 et au port 8080 ?
2. À quoi servent les headers `X-Real-IP` et `X-Forwarded-For` qu'nginx ajoute ?
3. Si tu bloques le port 3000 avec ufw mais laisses le 8080 ouvert — qu'est-ce que ça simule dans une architecture Cloud ?
4. Dans AWS, quel service remplace nginx ? Qu'est-ce qu'il ajoute ?

<details>
<summary>Voir les réponses</summary>

1. **Aucune différence visible** pour le client — il reçoit la même réponse. En coulisses, nginx intercepte, transmet à l'app, retransmet la réponse. L'app voit uniquement des requêtes venant de `127.0.0.1`.
2. Sans ces headers, l'app ne connaîtrait que l'IP de nginx (`127.0.0.1`). `X-Real-IP` transmet l'IP réelle du client original. `X-Forwarded-For` contient toute la chaîne d'IPs traversées.
3. Ça simule un **subnet privé AWS** : l'app (EC2 privée) n'est pas accessible directement. Seul nginx (ALB en subnet public) est exposé. Le client ne peut atteindre l'app qu'en passant par le proxy.
4. L'**ALB** (Application Load Balancer). Il ajoute : load balancing, health checks automatiques, SSL termination avec ACM, routing par URL, intégration WAF, haute disponibilité multi-AZ — tout managé par AWS.

</details>

---

## Quiz final — Module 1

Réponds à ces questions sans regarder le cours.

1. Quelle est la différence entre une IP privée et une IP publique ?
2. Un subnet `/24` dans AWS — combien d'IPs sont disponibles pour tes machines ?
3. Pourquoi DNS utilise UDP plutôt que TCP pour la majorité des requêtes ?
4. `10.0.5.0` et `10.0.6.0` sont-elles dans le même `/24` ? Et dans le même `/16` ?
5. Tu changes l'IP de ton serveur mais les utilisateurs continuent d'arriver sur l'ancienne. Quelle est la cause probable, et comment l'aurais-tu évité ?
6. Un utilisateur reçoit une erreur `403 Forbidden` sur ton API. Qui est responsable — le client ou le serveur ?
7. Quel port ouvrir dans un Security Group pour autoriser SSH ?
8. Quelle est la différence entre HTTP et HTTPS sur le plan de la sécurité ?
9. Tu veux nommer tes services internes dans AWS (ex: `database.prod.internal`). Quel service AWS utilises-tu ?
10. Un service de streaming vidéo perd 1% de ses paquets UDP. Est-ce un problème grave ?
11. Ta machine vient de rejoindre un réseau WiFi. Quel protocole lui attribue automatiquement une IP privée ?
12. Tu pingues une EC2 AWS et elle ne répond pas. Est-ce qu'elle est forcément en panne ?
13. Trois machines avec des IPs privées différentes partagent la même IP publique. Quel mécanisme permet ça ?
14. Quelle est la différence entre un forward proxy et un reverse proxy ?
15. Un certificat TLS expire demain sur ton app en production. Quel service AWS t'évite ce problème ?
16. Pourquoi IPv6 n'a pas besoin de NAT alors qu'IPv4 en a besoin ?

<details>
<summary>Voir les réponses</summary>

1. Une IP publique est unique mondiale et routable sur Internet. Une IP privée n'existe que dans un réseau local ou VPC — deux machines sur des réseaux différents peuvent avoir la même sans conflit.
2. **251** — AWS réserve 5 adresses par subnet.
3. Les requêtes DNS sont courtes, perdre une est acceptable (le client retente). UDP évite l'overhead du handshake TCP.
4. Non pour `/24` (`10.0.5.x` ≠ `10.0.6.x`). Oui pour `/16` (même réseau `10.0.x.x`).
5. Le TTL DNS est encore actif — les resolvers ont l'ancienne réponse en cache. Baisser le TTL à 60s la veille de la migration l'aurait évité.
6. Le **client** — les `4xx` sont des erreurs côté client.
7. **Port 22** (TCP).
8. HTTP envoie en clair — n'importe qui peut intercepter. HTTPS chiffre via TLS et authentifie le serveur.
9. **Route 53 Private Hosted Zone**.
10. Non — UDP est choisi précisément parce que perdre quelques paquets est acceptable pour le streaming.
11. **DHCP** — broadcast → offer → request → ACK (DORA).
12. **Non** — le Security Group bloque peut-être ICMP. C'est la config par défaut sur AWS.
13. **PAT / NAT overload** — le routeur (Fritz!Box ou NAT Gateway AWS) distingue les machines grâce aux numéros de port source.
14. **Forward proxy** = se place côté client, parle au nom du client (VPN, filtrage entreprise). **Reverse proxy** = se place côté serveur, parle au nom du serveur (ALB, nginx, CloudFront).
15. **ACM** (AWS Certificate Manager) — il renouvelle automatiquement les certificats attachés aux ALB, CloudFront, et API Gateway.
16. IPv4 a ~4 milliards d'adresses — épuisées, d'où le besoin de NAT pour partager des IPs publiques. IPv6 a 340 sextillions d'adresses — assez pour donner une IP publique unique à chaque machine, NAT inutile.

</details>

---

*→ Retour au cours : `module1-theorie.md`*
*→ Module suivant : Module 2 — Le réseau dans Linux*
