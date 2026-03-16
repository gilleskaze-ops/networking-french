# Module 2 — Le réseau dans Linux
### Partie Exercices

> Complète la partie théorie (`module2-theorie.md`) avant de faire ces exercices.
> Durée estimée : 2h30 à 3h
> Machine : Ubuntu 24.04 (ton ThinkPad L15)

---

## Installation des outils

Avant de commencer, installe tous les outils nécessaires :

```bash
sudo apt update
sudo apt install -y \
  wireshark \
  tcpdump \
  mtr \
  netcat-openbsd \
  nmap \
  traceroute \
  iperf3

# Pendant l'installation de Wireshark :
# une fenêtre demande si les non-root peuvent capturer des paquets → OUI

# Ajoute ton utilisateur au groupe wireshark
sudo usermod -aG wireshark $USER

# Applique le changement sans redémarrer
newgrp wireshark
```

---

## Exercice 1 — Observer ARP en action (ip + Wireshark)

**Objectif** : voir concrètement comment ta machine associe les IPs aux adresses MAC — d'abord avec les outils CLI, puis en capturant les paquets ARP en temps réel avec Wireshark.

### Partie A — Avec ip

```bash
# 1. Voir le cache ARP actuel (associations IP → MAC connues)
ip neigh show

# 2. Vider le cache ARP
sudo ip neigh flush all

# 3. Faire un ping vers ta Fritz!Box pour déclencher ARP
ping -c 1 192.168.178.1

# 4. Voir le cache ARP après le ping
ip neigh show

# 5. Voir le cache ARP avec plus de détails
ip neigh show dev wlan0     # ou eth0 selon ton interface
```

### Partie B — Capturer ARP avec Wireshark

```bash
# 1. Vider le cache ARP
sudo ip neigh flush all

# 2. Lancer Wireshark en interface graphique
wireshark &

# Dans Wireshark :
# → Sélectionne ton interface (wlan0 ou eth0)
# → Dans la barre de filtre, tape : arp
# → Clique sur le bouton bleu "Start capturing"

# 3. Dans un autre terminal, déclenche du trafic ARP
ping -c 3 192.168.178.1

# 4. Observe dans Wireshark :
# → Tu vois les paquets "ARP Request" et "ARP Reply"
# → Clique sur un ARP Request pour voir son contenu couche par couche
```

**Ce que tu vas voir dans Wireshark :**

```
Frame 1 — ARP Request
  Ethernet II
    Destination: ff:ff:ff:ff:ff:ff  ← broadcast (tout le monde)
    Source:      aa:bb:cc:dd:ee:ff  ← ta carte réseau
  ARP
    Opcode: request
    Sender IP:  192.168.178.X       ← ton IP
    Target IP:  192.168.178.1       ← Fritz!Box (MAC inconnue = 00:00:00:00:00:00)

Frame 2 — ARP Reply
  Ethernet II
    Destination: aa:bb:cc:dd:ee:ff  ← ta carte réseau (réponse directe)
    Source:      ff:ee:dd:cc:bb:aa  ← MAC de la Fritz!Box
  ARP
    Opcode: reply
    Sender IP:  192.168.178.1       ← Fritz!Box
    Sender MAC: ff:ee:dd:cc:bb:aa   ← voilà la MAC !
```

**Questions :**

1. Dans Wireshark, quelle est l'adresse MAC destination du ARP Request ? Pourquoi cette valeur ?
2. Qui répond au ARP Request ? Avec quelle information ?
3. Après le ARP Reply, que contient ton cache ARP (`ip neigh show`) ?
4. Pourquoi ARP est-il nécessaire alors qu'on connaît déjà l'IP de la Fritz!Box ?

<details>
<summary>Voir les réponses</summary>

1. `ff:ff:ff:ff:ff:ff` — c'est l'adresse MAC de broadcast. Cela signifie "envoyer à toutes les machines du réseau local". Ta machine ne connaît pas encore la MAC de la Fritz!Box, donc elle crie à tout le monde.
2. La Fritz!Box répond avec sa propre adresse MAC — elle dit "c'est moi qui ai l'IP `192.168.178.1`, voici ma MAC".
3. Une entrée `192.168.178.1  →  ff:ee:dd:cc:bb:aa  REACHABLE` — la Fritz!Box est maintenant connue et les paquets suivants iront directement vers elle sans refaire d'ARP.
4. L'IP est une adresse logique (couche 3). Pour envoyer physiquement sur le réseau (couche 2), il faut l'adresse MAC. ARP fait le pont entre les deux couches.

</details>

---

## Exercice 2 — Inspecter ses interfaces réseau

**Objectif** : lire et comprendre toutes les informations d'une interface réseau Linux.

```bash
# 1. Voir toutes les interfaces
ip addr show

# 2. Voir uniquement l'interface loopback
ip addr show lo

# 3. Voir l'interface WiFi ou ethernet active
ip addr show wlan0     # WiFi
ip addr show eth0      # Filaire (ou ens3, enp2s0 selon ta machine)

# 4. Voir uniquement les interfaces actives (UP)
ip link show up

# 5. Voir les statistiques de trafic par interface
ip -s link show
```

**Questions :**

1. Combien d'interfaces réseau ta machine possède-t-elle ? Liste-les avec leur nom.
2. Quelle est l'IP de ton interface loopback ? Et son masque CIDR ?
3. Quelle est l'IP privée de ton interface WiFi ou ethernet ? Sur quel réseau est-elle ?
4. Dans le résultat de `ip -s link show`, que représentent les compteurs RX et TX ?
5. Quelle est la MTU de ton interface principale ? Pourquoi 1500 est la valeur standard ?

<details>
<summary>Voir les réponses</summary>

1. En général : `lo` (loopback), `wlan0` (WiFi) ou `eth0` (filaire), parfois `docker0` si Docker est installé.
2. `127.0.0.1/8` — le `/8` signifie que toute la plage `127.x.x.x` est réservée au loopback.
3. Probablement dans `192.168.178.x/24` (réseau Fritz!Box Telekom).
4. RX = données **reçues** (Receive), TX = données **envoyées** (Transmit). En octets et en nombre de paquets.
5. MTU = 1500 octets. C'est la limite historique d'Ethernet. Un paquet plus grand doit être fragmenté en morceaux de 1500 max.

</details>

---

## Exercice 3 — Lire et comprendre sa table de routage

**Objectif** : maîtriser la table de routage et identifier le Default Gateway.

```bash
# 1. Afficher la table de routage
ip route show

# 2. Version plus détaillée
ip route show table all

# 3. Trouver par quelle route un paquet vers Google passerait
ip route get 8.8.8.8

# 4. Trouver la route vers une IP locale
ip route get 192.168.178.1

# 5. Trouver la route vers ta propre machine
ip route get 127.0.0.1
```

**Questions :**

1. Quelle est la ligne `default` dans ta table de routage ? Quelle IP est ton Default Gateway ?
2. Quelle est la différence entre `ip route get 8.8.8.8` et `ip route get 192.168.178.1` ?
3. Si tu supprimes la route `default`, que se passe-t-il pour les connexions vers Internet ? (ne pas le faire, juste raisonner)
4. Dans AWS, quelle est l'équivalent de la ligne `default via 192.168.178.1` dans une Route Table de subnet public ?

<details>
<summary>Voir les réponses</summary>

1. Quelque chose comme `default via 192.168.178.1 dev wlan0` — l'IP est celle de ta Fritz!Box.
2. Pour `8.8.8.8` (externe) : la route utilisée est `default via 192.168.178.1` — le paquet passe par la Fritz!Box. Pour `192.168.178.1` (local) : la route utilisée est directe via l'interface, sans passer par un routeur.
3. Plus aucune connexion vers Internet ne serait possible. Les paquets vers des IPs externes n'auraient nulle part où aller et seraient abandonnés.
4. La ligne `0.0.0.0/0 → igw-xxxxxxxx` (Internet Gateway) — c'est exactement le même rôle : "tout ce qui n'est pas local, envoie vers cette passerelle".

</details>

---

## Exercice 4 — Tracer le chemin d'un paquet avec traceroute

**Objectif** : observer concrètement le voyage d'un paquet à travers les routeurs Internet.

```bash
# Installation
sudo apt install traceroute

# 1. Tracer le chemin vers Google
traceroute google.com

# 2. Tracer vers un serveur AWS (région Europe)
traceroute ec2.eu-west-1.amazonaws.com

# 3. Tracer vers un serveur géographiquement lointain
traceroute yahoo.co.jp

# 4. Version avec ICMP au lieu d'UDP (parfois plus lisible)
traceroute -I google.com

# 5. Limiter le nombre de hops maximum
traceroute -m 10 google.com
```

**Questions :**

1. Combien de hops entre ta machine et Google ? Quel est le premier hop (c'est qui ?) ?
2. À quel hop la latence commence-t-elle à augmenter significativement ? Qu'est-ce que ça indique ?
3. Certains hops affichent `* * *` — que signifie ça ? Est-ce que le paquet passe quand même ?
4. Compare la latence vers `google.com` et vers `yahoo.co.jp` — quelle différence et pourquoi ?
5. Dans un contexte AWS, si tu fais un `traceroute` depuis une EC2 vers Internet et qu'il s'arrête au deuxième hop, qu'est-ce que ça suggère ?

<details>
<summary>Voir les réponses</summary>

1. En général 5 à 15 hops depuis l'Allemagne. Le premier hop est toujours ta Fritz!Box (`192.168.178.1`).
2. La latence augmente quand le paquet quitte l'Allemagne ou l'Europe — signal que le câble physique est plus long (fibre transatlantique par exemple).
3. `* * *` signifie que ce routeur ne répond pas aux requêtes ICMP traceroute (firewall). Le paquet passe quand même — traceroute ne le voit juste pas.
4. `yahoo.co.jp` est au Japon — la latence sera bien plus élevée (100-200ms vs 10-20ms pour Google EU) à cause de la distance physique des câbles sous-marins.
5. Le Security Group ou la Route Table de l'EC2 bloque le trafic sortant au-delà du VPC — probablement pas de NAT Gateway ou d'Internet Gateway configurée.

</details>

---

## Exercice 5 — Observer les connexions avec ss

**Objectif** : utiliser `ss` pour voir en temps réel ce qui se passe sur le réseau de ta machine.

```bash
# 1. Voir toutes les connexions TCP établies
ss -tn

# 2. Avec les processus (qui utilise quoi)
sudo ss -tnp

# 3. Voir les ports en écoute
sudo ss -tlnp

# 4. Dans un terminal, ouvre un navigateur sur un site quelconque
# puis dans un autre terminal :
sudo ss -tnp | grep ESTAB

# 5. Voir les connexions UDP actives
ss -un

# 6. Filtrer sur un port spécifique
ss -tnp dport = :443
```

**Questions :**

1. Quels ports sont en écoute sur ta machine (`LISTEN`) ? Qu'est-ce que ça révèle sur les services actifs ?
2. Quelles connexions TCP établies (`ESTAB`) vois-tu ? Vers quelles IPs ?
3. Ouvre un onglet sur un nouveau site web, puis rerun `ss -tnp` — quelles nouvelles connexions apparaissent ?
4. Quelle est la différence entre `Local Address:Port` et `Peer Address:Port` ?
5. Pourquoi `0.0.0.0:22` dans la colonne Local Address signifie-t-il que SSH écoute sur toutes les interfaces ?

<details>
<summary>Voir les réponses</summary>

1. Typiquement port 22 (SSH si activé), 631 (CUPS impression), parfois 53 (DNS local). Chaque port en écoute = un service qui attend des connexions.
2. Les connexions de ton navigateur vers les sites visités (port 443), éventuellement des connexions SSH si tu en as d'actives.
3. De nouvelles connexions `ESTAB` vers l'IP du serveur du site sur le port 443.
4. `Local` = l'IP et port de ta machine. `Peer` = l'IP et port de l'autre machine avec qui tu es connecté.
5. `0.0.0.0` signifie "toutes les interfaces réseau". SSH acceptera les connexions arrivant sur n'importe quelle interface (WiFi, ethernet, loopback). Si c'était `127.0.0.1:22`, SSH n'accepterait que les connexions locales.

</details>

---

## Exercice 6 — Diagnostic réseau complet

**Objectif** : appliquer la méthode de diagnostic étape par étape face à un scénario de panne.

**Scénario** : tu essaies d'accéder à `https://api.monservice.com` depuis ton terminal et tu n'obtiens aucune réponse.

Exécute les commandes suivantes dans l'ordre et interprète chaque résultat :

```bash
# Étape 1 — Ma machine fonctionne-t-elle ?
ping -c 2 127.0.0.1

# Étape 2 — Mon réseau local fonctionne-t-il ?
ping -c 2 192.168.178.1

# Étape 3 — J'ai accès à Internet ?
ping -c 2 8.8.8.8

# Étape 4 — Le DNS fonctionne-t-il ?
dig +short api.monservice.com

# Étape 5 — Le serveur est-il joignable ?
ping -c 2 $(dig +short api.monservice.com | head -1)

# Étape 6 — Le port HTTPS répond-il ?
curl -v --max-time 5 https://api.monservice.com

# Étape 7 — Où ça bloque exactement ?
traceroute api.monservice.com
```

**Questions :**

Pour chaque étape, décris ce que tu fais si le test **échoue** :

1. Étape 1 échoue → ?
2. Étape 2 échoue → ?
3. Étape 3 échoue → ?
4. Étape 4 échoue → ?
5. Étape 5 échoue mais étape 3 réussit → ?
6. Étape 6 échoue avec `Connection refused` → ?
7. Étape 6 échoue avec `Connection timed out` → ?

<details>
<summary>Voir les réponses</summary>

1. **Étape 1 échoue** → Problème sur la machine elle-même (stack réseau du noyau). Rarissime — redémarre la machine.
2. **Étape 2 échoue** → Problème sur le réseau local : câble débranché, WiFi déconnecté, Fritz!Box HS.
3. **Étape 3 échoue** → Pas d'accès Internet : vérifie la connexion WAN de ta Fritz!Box, appelle Telekom si nécessaire.
4. **Étape 4 échoue** → Problème DNS : le nom ne se résout pas. Essaie `dig @8.8.8.8 api.monservice.com` pour bypasser ton DNS local. Si ça marche, c'est ton resolver DNS qui est HS.
5. **Étape 5 échoue** → Le serveur ne répond pas au ping : il est éteint, ou ICMP est bloqué par son firewall/Security Group. Passe à l'étape 6 pour tester TCP directement.
6. **Connection refused** → Le serveur est joignable mais rien n'écoute sur le port 443. L'application n'est probablement pas démarrée, ou écoute sur un autre port.
7. **Connection timed out** → Le paquet n'arrive pas à destination ou la réponse ne revient pas. Suspect : firewall, Security Group qui bloque le port 443, ou mauvaise Route Table.

</details>

---

## Exercice 7 — Configurer ufw

**Objectif** : mettre en place des règles de firewall sur ta machine Linux.

> ⚠️ **Attention** : si tu travailles à distance via SSH, autorise TOUJOURS le port 22 avant d'activer ufw — sinon tu te coupes l'accès.

```bash
# 1. Vérifier le statut actuel
sudo ufw status verbose

# 2. Autoriser SSH en premier (sécurité)
sudo ufw allow ssh

# 3. Activer ufw
sudo ufw enable

# 4. Voir les règles actives
sudo ufw status numbered

# 5. Autoriser HTTP et HTTPS
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# 6. Autoriser un port depuis une IP spécifique seulement
sudo ufw allow from 192.168.178.0/24 to any port 5432

# 7. Bloquer un port
sudo ufw deny 23/tcp    # Telnet — toujours bloquer ça

# 8. Tester — essaie de te connecter sur un port bloqué
curl --max-time 3 http://localhost:23

# 9. Voir les règles avec leur numéro pour les supprimer
sudo ufw status numbered

# 10. Supprimer une règle par numéro
sudo ufw delete 3    # adapte le numéro

# 11. Désactiver ufw si besoin
sudo ufw disable
```

**Questions :**

1. Quelle est la politique par défaut de ufw pour le trafic entrant ? Et sortant ?
2. Quelle est la différence entre `ufw deny` et `ufw reject` ?
3. La règle `from 192.168.178.0/24 to any port 5432` — qu'est-ce qu'elle autorise exactement ?
4. Traduis cette règle ufw en règle AWS Security Group : `ufw allow from 10.0.1.0/24 to any port 3000`

<details>
<summary>Voir les réponses</summary>

1. Par défaut ufw : **entrant = DENY** (tout bloqué sauf ce qu'on autorise), **sortant = ALLOW** (tout autorisé). C'est le comportement sécurisé par défaut.
2. `deny` abandonne le paquet silencieusement (l'expéditeur attend un timeout). `reject` abandonne le paquet **et envoie un message d'erreur** à l'expéditeur (connexion refusée immédiatement). `reject` est plus propre pour l'expérience utilisateur, `deny` est plus discret.
3. Elle autorise toutes les machines du réseau `192.168.178.x` (ton réseau local Fritz!Box) à se connecter sur le port 5432 (PostgreSQL). Les machines externes ne peuvent pas.
4. Security Group Inbound : `Port 3000 / TCP / Source: 10.0.1.0/24` — seules les machines du subnet `10.0.1.0/24` peuvent atteindre ce service sur le port 3000.

</details>

---

## Exercice 8 — Relier tout : simuler une architecture Cloud en local

**Objectif** : reproduire localement la logique d'une architecture Cloud avec les outils Linux vus dans ce module.

**Scénario** : tu simules une EC2 qui fait tourner une app web. Tu vas démarrer un serveur HTTP minimaliste, configurer le firewall, et tester la connectivité.

```bash
# 1. Démarre un serveur HTTP simple sur le port 8080
#    (Python est installé par défaut sur Ubuntu)
python3 -m http.server 8080 &

# 2. Vérifie qu'il écoute bien
sudo ss -tlnp | grep 8080

# 3. Teste depuis ta propre machine (loopback)
curl http://127.0.0.1:8080

# 4. Teste depuis l'IP locale de ta machine
curl http://$(ip route get 8.8.8.8 | grep src | awk '{print $7}'):8080

# 5. Configure ufw pour autoriser uniquement le port 8080
sudo ufw allow 8080/tcp
sudo ufw status

# 6. Simule un accès "depuis Internet" (depuis ton réseau local)
#    Sur un autre appareil du même réseau (téléphone, autre PC),
#    accède à : http://[IP-de-ta-machine]:8080

# 7. Maintenant bloque le port 8080
sudo ufw deny 8080/tcp

# 8. Reteste depuis le loopback — toujours accessible ?
curl http://127.0.0.1:8080

# 9. Reteste depuis l'IP locale — accessible ?
curl http://[ton-IP]:8080

# 10. Arrête le serveur
kill %1
```

**Questions :**

1. À l'étape 8, pourquoi le loopback est-il encore accessible malgré la règle `deny 8080` ?
2. Quelle est l'analogie entre ce serveur Python et une EC2 AWS avec une app déployée ?
3. Si c'était une vraie EC2 AWS, qu'est-ce qui remplacerait la règle ufw pour contrôler l'accès au port 8080 ?
4. L'IP que tu as utilisée à l'étape 4 (`ip route get 8.8.8.8 | grep src | awk '{print $7}'`) — qu'est-ce que cette commande fait exactement ?

<details>
<summary>Voir les réponses</summary>

1. ufw par défaut ne filtre pas le trafic loopback (`lo`). L'interface loopback est toujours exclue des règles firewall — sinon les services locaux se casseraient mutuellement.
2. Le serveur Python = l'application qui tourne sur l'EC2. Le port 8080 = le port exposé par l'app. ufw = le Security Group. Ta machine locale = l'EC2 dans son subnet.
3. Le **Security Group AWS** remplacerait ufw. On y créerait une règle Inbound : `Port 8080 / TCP / Source: 0.0.0.0/0` pour autoriser tout Internet, ou une source restreinte pour limiter l'accès.
4. Cette commande demande à Linux "quelle route utiliserais-tu pour joindre 8.8.8.8 ?" et extrait l'IP source que Linux utiliserait — c'est-à-dire ton IP locale sur l'interface active. C'est une astuce pour trouver son IP principale sans chercher dans `ip addr show`.

</details>

---

## Exercice 9 — tcpdump : capturer le réseau en ligne de commande

**Objectif** : utiliser `tcpdump` pour capturer et analyser le trafic réseau directement depuis le terminal — sans interface graphique. C'est l'outil indispensable sur un serveur Linux distant (EC2, VPS) où Wireshark n'est pas disponible.

### Partie A — Premières captures

```bash
# 1. Capturer tout le trafic sur l'interface principale
sudo tcpdump -i wlan0          # Ctrl+C pour arrêter

# 2. Capturer avec noms résolus désactivés (plus lisible)
sudo tcpdump -i wlan0 -n

# 3. Limiter à 10 paquets
sudo tcpdump -i wlan0 -n -c 10

# 4. Capturer uniquement le trafic ICMP (ping)
# Dans terminal 1 — lance la capture
sudo tcpdump -i wlan0 -n icmp

# Dans terminal 2 — génère du trafic ICMP
ping -c 3 google.com

# Observe les requêtes et réponses ICMP dans terminal 1
```

### Partie B — Filtres utiles

```bash
# Capturer uniquement DNS (port 53)
sudo tcpdump -i wlan0 -n port 53

# Dans un autre terminal, déclenche une résolution DNS
dig google.com

# Capturer uniquement HTTP/HTTPS
sudo tcpdump -i wlan0 -n port 443

# Capturer le trafic vers/depuis une IP spécifique
sudo tcpdump -i wlan0 -n host 8.8.8.8

# Capturer uniquement le trafic entrant
sudo tcpdump -i wlan0 -n dst host $(ip route get 8.8.8.8 | grep src | awk '{print $7}')

# Voir le contenu des paquets (ASCII)
sudo tcpdump -i wlan0 -n -A port 80

# Sauvegarder une capture dans un fichier (lisible par Wireshark)
sudo tcpdump -i wlan0 -n -w /tmp/capture.pcap

# Lire le fichier capturé
sudo tcpdump -r /tmp/capture.pcap
```

### Partie C — Observer le TCP handshake

```bash
# Terminal 1 — capturer le handshake TCP vers google.com
sudo tcpdump -i wlan0 -n "host google.com and tcp"

# Terminal 2 — déclencher une connexion TCP
curl -v https://google.com 2>&1 | head -20
```

**Ce que tu vas voir :**

```
# Le handshake en 3 étapes :
IP 192.168.178.X.XXXXX > 142.250.X.X.443: Flags [S]       ← SYN
IP 142.250.X.X.443 > 192.168.178.X.XXXXX: Flags [S.]      ← SYN-ACK
IP 192.168.178.X.XXXXX > 142.250.X.X.443: Flags [.]       ← ACK

# Flags TCP :
# [S]  = SYN
# [S.] = SYN-ACK
# [.]  = ACK
# [P.] = PUSH-ACK (données)
# [F.] = FIN (fermeture)
```

**Questions :**

1. Dans la capture DNS, combien de paquets vois-tu pour une seule requête `dig` ? Quel protocole (UDP ou TCP) ?
2. Dans le handshake TCP, quel est le port source de ta machine ? Est-il fixe ou aléatoire ?
3. Comment sauvegarder une capture tcpdump pour l'ouvrir ensuite dans Wireshark ?
4. Sur une EC2 AWS sans interface graphique, quel outil utilises-tu pour analyser le trafic réseau ?

<details>
<summary>Voir les réponses</summary>

1. En général 2 paquets : 1 requête UDP sortante + 1 réponse UDP entrante. DNS utilise UDP par défaut pour les requêtes courtes.
2. Le port source est **aléatoire** (éphémère) — typiquement entre 32768 et 60999. Seul le port destination (443) est fixe. C'est Linux qui choisit un port libre pour identifier la connexion.
3. `sudo tcpdump -i wlan0 -w /tmp/capture.pcap` puis ouvrir le fichier `.pcap` dans Wireshark avec `File → Open`.
4. **tcpdump** — c'est exactement pour ça qu'il existe. Sur un serveur distant sans GUI, tcpdump est ton Wireshark en ligne de commande.

</details>

---

## Exercice 10 — mtr : traceroute en temps réel

**Objectif** : utiliser `mtr` pour analyser le chemin des paquets de façon continue et détecter des problèmes de latence ou de perte de paquets sur le trajet.

`mtr` combine `ping` et `traceroute` en un seul outil interactif — il envoie des paquets en continu et met à jour les statistiques en temps réel.

```bash
# 1. Lancer mtr vers Google (interface interactive)
mtr google.com

# Navigation dans mtr :
# q      → quitter
# d      → changer le mode d'affichage
# p      → pause/resume
# ?      → aide

# 2. Mode rapport (non interactif, utile pour les scripts)
mtr --report --report-cycles 10 google.com

# 3. Comparer deux destinations
mtr --report -c 5 google.com
mtr --report -c 5 amazon.com

# 4. Tracer vers un serveur AWS Europe
mtr --report -c 5 ec2.eu-west-1.amazonaws.com

# 5. Utiliser TCP au lieu d'ICMP (utile si ICMP est bloqué)
sudo mtr --tcp --port 443 google.com
```

**Lire le résultat de mtr :**

```
Host                      Loss%   Snt   Last   Avg  Best  Wrst StDev
1. 192.168.178.1           0.0%    10    1.2   1.1   0.9   1.5   0.2  ← Fritz!Box
2. 10.45.0.1               0.0%    10    8.4   8.1   7.9   9.2   0.4  ← Telekom
3. 87.234.12.1            10.0%    10   11.2  10.8   9.1  15.3   1.8  ← backbone (perte!)
4. 142.250.74.46           0.0%    10   12.3  12.1  11.8  12.9   0.3  ← Google ✅

# Colonnes :
# Loss%  = pourcentage de paquets perdus
# Snt    = nombre de paquets envoyés
# Last   = latence du dernier paquet (ms)
# Avg    = latence moyenne
# Best   = meilleure latence
# Wrst   = pire latence
# StDev  = stabilité (faible = stable, élevé = fluctuant)
```

**Questions :**

1. Sur ton réseau, quel hop a la latence la plus faible ? Et la plus élevée ?
2. Si un hop affiche `10.0%` de perte mais les hops suivants affichent `0.0%` — y a-t-il vraiment un problème ?
3. Quelle est la différence entre `mtr` et `traceroute` en termes d'utilisation ?
4. Tu analyses une EC2 lente depuis ton laptop. `mtr` montre une latence élevée à partir du hop 4 (backbone d'AWS). Où se situe le problème ?

<details>
<summary>Voir les réponses</summary>

1. Le hop 1 (Fritz!Box) a la latence la plus faible (souvent < 2ms — c'est ton réseau local). La latence augmente progressivement avec la distance.
2. **Non, pas de vrai problème.** Certains routeurs limitent délibérément leurs réponses ICMP pour ne pas surcharger leur CPU avec les sondes traceroute/mtr. Le trafic normal passe bien — c'est juste que ce routeur déprioritise les réponses aux outils de diagnostic.
3. `traceroute` fait **une seule passe** et s'arrête. `mtr` envoie des paquets **en continu** et calcule des statistiques sur la durée — bien plus utile pour détecter des problèmes intermittents.
4. Le problème est **entre Internet et AWS** (réseau de transit), pas sur ta machine ni sur l'EC2. Tu n'as pas la main dessus — c'est l'infrastructure réseau d'AWS ou de l'opérateur intermédiaire. Si ça persiste, changer de région AWS peut aider.

</details>

---

## Exercice 11 — netcat : tester les connexions TCP/UDP manuellement

**Objectif** : utiliser `netcat` (nc) pour créer et tester des connexions réseau brutes — sans application, sans protocole applicatif. C'est l'outil parfait pour vérifier qu'un port est ouvert et que deux machines peuvent communiquer.

### Partie A — Client et serveur TCP sur la même machine

```bash
# Terminal 1 — démarrer un serveur TCP sur le port 9000
nc -l 9000
# nc écoute maintenant sur le port 9000 et attend une connexion

# Terminal 2 — se connecter en client
nc localhost 9000

# Tape du texte dans Terminal 2 → il apparaît dans Terminal 1
# Tape du texte dans Terminal 1 → il apparaît dans Terminal 2
# C'est un flux TCP brut bidirectionnel

# Ctrl+C pour fermer
```

### Partie B — Vérifier si un port est ouvert

```bash
# Tester si le port 443 (HTTPS) est ouvert sur google.com
nc -zv google.com 443

# Tester le port 80
nc -zv google.com 80

# Tester un port fermé
nc -zv google.com 23

# Tester un port local
nc -zv localhost 22

# Tester plusieurs ports d'un coup
nc -zv google.com 80 443 8080

# Options :
# -z = scan mode (ne pas envoyer de données, juste tester la connexion)
# -v = verbose (afficher le résultat)
# -w 3 = timeout de 3 secondes
```

### Partie C — Simuler une requête HTTP manuellement

```bash
# Se connecter directement au port 80 d'un serveur
nc httpbin.org 80

# Une fois connecté, tape exactement ces lignes
# (appuie sur Entrée deux fois à la fin) :
GET /get HTTP/1.1
Host: httpbin.org

# Tu vas voir la réponse HTTP brute du serveur
```

### Partie D — Transférer un fichier entre deux machines

```bash
# Sur la machine réceptrice (Terminal 1)
nc -l 9001 > fichier_recu.txt

# Sur la machine émettrice (Terminal 2)
echo "Bonjour depuis netcat !" | nc localhost 9001

# Vérifie que le fichier a bien été reçu
cat fichier_recu.txt
```

**Questions :**

1. Dans la Partie A, que se passe-t-il si tu fermes le Terminal 1 (serveur) sans fermer le Terminal 2 ?
2. `nc -zv google.com 23` retourne `Connection refused` ou `timed out` — quelle est la différence ?
3. Dans la Partie C, pourquoi voit-on la réponse HTTP brute sans TLS ? Que faudrait-il faire pour HTTPS ?
4. Dans un contexte Cloud, comment utiliserais-tu `nc -zv` pour débugger une connexion entre une EC2 app et une RDS ?

<details>
<summary>Voir les réponses</summary>

1. Le Terminal 2 (client) reçoit une erreur de connexion fermée (`broken pipe` ou la session se termine). TCP détecte que l'autre extrémité a fermé la connexion et notifie le client.
2. `Connection refused` = le serveur est joignable mais rien n'écoute sur ce port (le système répond "non"). `Timed out` = le paquet n'arrive pas à destination ou aucune réponse n'est reçue — souvent un firewall qui `DROP` silencieusement.
3. On se connecte en HTTP pur (port 80, pas de chiffrement). Pour HTTPS, il faudrait d'abord établir une session TLS — `netcat` ne gère pas TLS. On utiliserait plutôt `openssl s_client -connect google.com:443`.
4. Depuis l'EC2 app : `nc -zv <IP-RDS> 5432` — si ça retourne `Connection refused` ou `timed out`, le Security Group de la RDS bloque la connexion. Si ça retourne `succeeded`, le réseau est OK et le problème est ailleurs (credentials, configuration app).

</details>

---

## Exercice 12 — Wireshark : capturer et analyser le TCP handshake

**Objectif** : voir avec Wireshark le handshake TCP et le chiffrement TLS d'une vraie connexion HTTPS — et comprendre visuellement ce qu'on a étudié en théorie.

```bash
# 1. Lance Wireshark
wireshark &

# Dans Wireshark :
# → Sélectionne ton interface (wlan0 ou eth0)
# → Filtre : tcp.port == 443
# → Démarre la capture

# 2. Dans un terminal, génère une connexion HTTPS
curl https://httpbin.org/get

# 3. Arrête la capture dans Wireshark (bouton rouge)
```

**Ce que tu vas voir et analyser :**

```
# Étape 1 — Cherche le TCP handshake
# Filtre Wireshark : tcp.flags.syn == 1
# Tu verras les paquets SYN, SYN-ACK, ACK

# Étape 2 — Cherche le TLS handshake
# Filtre Wireshark : tls
# Tu verras : Client Hello, Server Hello, Certificate, Finished

# Étape 3 — Voir les données chiffrées
# Après le TLS handshake, tout est chiffré → illisible
# C'est exactement le but de HTTPS
```

**Filtres Wireshark utiles à connaître :**

```
arp                    → uniquement ARP
icmp                   → uniquement ping
dns                    → uniquement DNS
tcp.port == 443        → uniquement HTTPS
tcp.port == 80         → uniquement HTTP
tcp.flags.syn == 1     → paquets SYN (début de connexion)
ip.addr == 8.8.8.8     → trafic vers/depuis cette IP
http                   → requêtes HTTP (déchiffrées, port 80)
```

**Questions :**

1. Dans le TCP handshake, quel paquet vient en premier — SYN ou SYN-ACK ? Qui l'envoie ?
2. Après le TLS handshake, peux-tu lire le contenu de la requête HTTP (`GET /get`) dans Wireshark ? Pourquoi ?
3. Ouvre le premier paquet SYN et développe les couches dans Wireshark. Quelles couches OSI vois-tu ?
4. Compare une capture sur le port 80 (HTTP) et une sur le port 443 (HTTPS) — quelle différence dans la lisibilité des données ?

<details>
<summary>Voir les réponses</summary>

1. **SYN** vient en premier — c'est **ton client** (curl) qui initie la connexion. Le serveur répond avec SYN-ACK, puis le client confirme avec ACK.
2. **Non** — les données sont chiffrées par TLS. Wireshark voit des octets illisibles après le handshake TLS. C'est exactement le but de HTTPS : même quelqu'un qui capture le trafic ne peut pas lire le contenu.
3. Tu vois : `Frame` (couche 1), `Ethernet II` (couche 2), `Internet Protocol` (couche 3), `Transmission Control Protocol` (couche 4). Les couches 5-7 ne sont pas visibles dans le SYN lui-même — elles apparaissent dans les paquets de données.
4. En HTTP (port 80), Wireshark affiche clairement les requêtes et réponses en texte lisible (`GET /page HTTP/1.1`, headers, corps). En HTTPS (port 443), après le TLS handshake, tout est du bruit chiffré — impossible à lire. C'est la preuve concrète de l'utilité de HTTPS.

</details>

---

## Quiz final — Module 2

Réponds sans regarder le cours.

1. Ta machine veut envoyer un paquet à `192.168.1.50` (même réseau `/24`). Passe-t-elle par le Default Gateway ?
2. Quel protocole permet de trouver l'adresse MAC associée à une IP sur le réseau local ?
3. Tu fais `traceroute google.com` et le premier hop est `192.168.178.1`. C'est quoi ?
4. Quelle commande utilises-tu pour voir quels ports sont en écoute sur ta machine ?
5. Quelle est la différence entre `ss -tn` et `ss -tlnp` ?
6. Tu vois `* * *` à la ligne 3 d'un traceroute. Est-ce que le paquet continue quand même son chemin ?
7. Dans une Route Table AWS, que signifie la règle `0.0.0.0/0 → nat-xxxxxxxx` ?
8. Quelle est la politique par défaut de ufw pour le trafic entrant ?
9. Quelle est la différence entre un Security Group AWS et ufw sur une EC2 — où chacun filtre-t-il ?
10. Tu ajoutes la règle `ufw allow from 10.0.1.0/24 to any port 5432`. Qui peut accéder à PostgreSQL ?
11. Sur une EC2 sans interface graphique, quel outil utilises-tu pour capturer le trafic réseau ?
12. Quelle commande netcat utilises-tu pour vérifier si le port 5432 d'une RDS est accessible ?
13. Quelle est la différence principale entre `mtr` et `traceroute` ?
14. Dans Wireshark, tu filtres `tcp.flags.syn == 1` — quels paquets vois-tu ?
15. `nc -zv 10.0.20.5 5432` retourne `timed out` au lieu de `Connection refused`. Qu'est-ce que ça indique ?

<details>
<summary>Voir les réponses</summary>

1. **Non** — destination dans le même réseau local → communication directe via ARP + switch, sans routeur.
2. **ARP** — broadcast "qui a cette IP ?" puis réponse avec la MAC.
3. Ta **Fritz!Box** — le Default Gateway, premier routeur traversé.
4. `sudo ss -tlnp` — `t` TCP, `l` listen, `n` numérique, `p` processus.
5. `ss -tn` = connexions TCP **établies** (ESTAB). `ss -tlnp` = ports **en écoute** (LISTEN) avec processus.
6. **Oui** — ce routeur bloque les sondes traceroute (firewall) mais laisse passer le trafic normal.
7. Subnet privé : sortie vers Internet via NAT Gateway autorisée, entrée depuis Internet impossible.
8. **DENY** — tout bloqué par défaut, on autorise explicitement.
9. **Security Group** filtre au niveau réseau AWS avant que le paquet atteigne l'EC2. **ufw** filtre sur l'OS après réception. Les deux peuvent coexister.
10. Uniquement les machines du subnet `10.0.1.0/24`.
11. **tcpdump** — `sudo tcpdump -i eth0 -n`. Peut sauvegarder en `.pcap` pour Wireshark.
12. `nc -zv 10.0.20.5 5432` — `-z` scan sans données, `-v` verbose.
13. `traceroute` fait une seule passe. `mtr` envoie en continu et affiche des statistiques (latence, perte) — bien plus utile pour les problèmes intermittents.
14. Les paquets **SYN** — le début de chaque connexion TCP, première étape du handshake.
15. Le firewall **DROP** silencieusement les paquets — Security Group ou ufw bloque sans répondre. `Connection refused` viendrait du serveur lui-même (port fermé mais joignable).

</details>

---

## Exercice 13 — DNS local : /etc/hosts et /etc/resolv.conf

**Objectif** : comprendre et manipuler la configuration DNS locale de Linux — les deux fichiers que tu retrouveras sur chaque EC2 AWS.

```bash
# 1. Voir le contenu actuel de /etc/hosts
cat /etc/hosts

# 2. Voir la configuration DNS actuelle
cat /etc/resolv.conf

# 3. Voir quel serveur DNS ta machine utilise réellement
resolvectl status | grep "DNS Servers"

# 4. Voir le cache DNS local
resolvectl statistics

# 5. Tester la résolution normale
dig +short google.com

# 6. Ajouter une entrée dans /etc/hosts
echo "127.0.0.1 monapp.local" | sudo tee -a /etc/hosts

# 7. Tester — la résolution passe par /etc/hosts (pas par DNS)
ping -c 1 monapp.local
dig +short monapp.local    # dig ignore /etc/hosts — retourne rien
curl http://monapp.local   # curl utilise /etc/hosts ✅

# 8. Comprendre la différence
# getent hosts utilise /etc/hosts comme le système
getent hosts monapp.local
getent hosts google.com

# 9. Nettoyer
sudo sed -i '/monapp.local/d' /etc/hosts

# 10. Vider le cache DNS local
sudo resolvectl flush-caches
resolvectl statistics    # comparer avant/après
```

**Questions :**

1. Dans `/etc/resolv.conf`, quelle est l'adresse du nameserver configuré ?
2. Pourquoi `dig +short monapp.local` ne retourne rien alors que `ping monapp.local` fonctionne ?
3. Sur une EC2 AWS dans un VPC `10.0.0.0/16`, quelle adresse DNS verras-tu dans `/etc/resolv.conf` ?
4. Quel est l'ordre de résolution DNS sur Linux — dans quel ordre les sources sont-elles consultées ?

<details>
<summary>Voir les réponses</summary>

1. Sur Ubuntu moderne : `127.0.0.53` (systemd-resolved qui fait le proxy DNS local). Sur une EC2 AWS : `10.0.0.2` (VPC CIDR + 2).
2. `dig` bypasse `/etc/hosts` et contacte directement les serveurs DNS — il ne trouve rien car `monapp.local` n'existe pas dans le DNS. `ping` utilise la résolution système complète (qui inclut `/etc/hosts`) — il trouve l'entrée et l'utilise.
3. `10.0.0.2` — sur tout VPC AWS, le serveur DNS est toujours à l'adresse `VPC_CIDR + 2`.
4. Ordre : **1.** `/etc/hosts` → **2.** cache DNS local (systemd-resolved) → **3.** serveurs DNS configurés dans `/etc/resolv.conf`. Ce comportement est défini dans `/etc/nsswitch.conf`.

</details>

---

## Exercice 14 — Observer les ports éphémères

**Objectif** : voir concrètement comment l'OS gère les ports source et comprendre le mécanisme derrière la table NAT.

```bash
# 1. Voir la plage des ports éphémères configurée
cat /proc/sys/net/ipv4/ip_local_port_range

# 2. Ouvrir plusieurs connexions simultanées
curl https://google.com > /dev/null &
curl https://github.com > /dev/null &
curl https://cloudflare.com > /dev/null &

# 3. Voir les ports source utilisés
ss -tn | grep ESTAB

# 4. Observer qu'ils sont tous différents et dans la plage éphémère
# Format : IP_locale:PORT_SOURCE  →  IP_distante:PORT_DEST

# 5. Voir combien de connexions sont actives en ce moment
ss -tn | grep ESTAB | wc -l

# 6. Voir les ports source utilisés par un processus spécifique
sudo ss -tnp | grep chrome    # ou firefox, curl, etc.

# 7. Simuler deux connexions vers le même serveur
nc -zv google.com 443 &
nc -zv google.com 443 &
ss -tn | grep "google"
# → deux connexions vers le même serveur avec deux ports source différents
```

**Questions :**

1. Quelle est la plage de ports éphémères sur ta machine ?
2. Si tu ouvres 10 onglets dans ton navigateur, combien de ports source différents vois-tu dans `ss -tn` ?
3. Deux machines sur ton réseau local choisissent toutes les deux le port source 5555 pour contacter Google. Comment la Fritz!Box résout-elle ce conflit dans sa table NAT ?
4. Pourquoi les ports 0-1023 ne sont-ils jamais utilisés comme ports éphémères ?

<details>
<summary>Voir les réponses</summary>

1. En général `32768 60999` sur Linux — soit 28231 ports disponibles simultanément.
2. En général 1 port par connexion TCP active — 10 onglets = jusqu'à 10 ports différents (parfois moins si certaines connexions sont réutilisées via HTTP keep-alive).
3. La Fritz!Box **renomme le port public** d'une des deux connexions. Par exemple : Machine A garde le port 5555, Machine B est remappée sur le port 5556 côté Internet. Sa table NAT mémorise les deux associations séparément.
4. Les ports 0-1023 sont les **ports bien connus** (well-known ports) — réservés aux services système (SSH=22, HTTP=80, HTTPS=443...). Les utiliser comme ports source provoquerait des conflits avec les services en écoute.

</details>

---

## Exercice 15 — NTP : vérifier et configurer la synchronisation

**Objectif** : vérifier l'état NTP de ta machine, comprendre ce qu'il affiche, et savoir le configurer.

```bash
# 1. Voir l'état complet de l'heure et NTP
timedatectl

# 2. Voir les serveurs NTP utilisés
timedatectl show-timesync --all 2>/dev/null || \
  systemctl status systemd-timesyncd | grep -A5 "Status:"

# 3. Voir l'offset actuel (décalage entre ta machine et le serveur NTP)
# Installe chrony si nécessaire
sudo apt install chrony -y
chronyc tracking

# Output de chronyc tracking :
# Reference ID    : A29F2301 (time.cloudflare.com)
# Stratum         : 4
# Ref time (UTC)  : Mon Oct 14 13:32:05 2024
# System time     : 0.000123456 seconds fast of NTP time  ← offset
# Last offset     : +0.000001234 seconds
# RMS offset      : 0.000045678 seconds
# Frequency       : 12.345 ppm slow
# Residual freq   : +0.001 ppm
# Skew            : 0.123 ppm
# Root delay      : 0.012345678 seconds
# Root dispersion : 0.001234567 seconds

# 4. Voir les sources NTP et leur qualité
chronyc sources -v

# 5. Forcer une synchronisation immédiate
sudo chronyc makestep

# 6. Comparer l'heure locale et UTC
date                    # heure locale
date -u                 # heure UTC
date +"%Y-%m-%dT%H:%M:%SZ"   # format ISO 8601 UTC (format des logs Cloud)

# 7. Voir les logs système avec timestamps UTC
journalctl --utc -n 10
```

**Questions :**

1. Ta machine est-elle synchronisée ? (`System clock synchronized: yes` ?)
2. Quel est le serveur NTP utilisé par ta machine ?
3. Que signifie "Stratum 4" dans le résultat de `chronyc tracking` ?
4. Pourquoi les logs Cloud sont-ils toujours en UTC et pas en heure locale ?
5. Sur une EC2 AWS, quel est le serveur NTP configuré automatiquement ? Pourquoi cette adresse est-elle spéciale ?

<details>
<summary>Voir les réponses</summary>

1. Normalement oui sur Ubuntu — `systemd-timesyncd` est actif par défaut.
2. Souvent `ntp.ubuntu.com` ou les serveurs de ton opérateur (Telekom). Visible dans `timedatectl show-timesync`.
3. Le **stratum** indique la distance par rapport à l'horloge de référence. Stratum 1 = directement connecté à une horloge atomique. Stratum 4 = 4 niveaux de délégation. Plus le stratum est bas, plus l'heure est précise.
4. UTC est universel — pas de fuseaux horaires, pas d'heure d'été. Si deux EC2 dans des régions différentes loggent en heure locale, il est impossible de corréler les événements sans conversion. En UTC, tous les logs sont directement comparables.
5. `169.254.169.123` — c'est l'**Amazon Time Sync Service**. C'est une adresse **link-local** (préfixe `169.254.x.x`) accessible uniquement depuis l'intérieur du VPC, sans passer par Internet. Ultra-précise (synchronisée sur des horloges atomiques AWS) et toujours disponible même si l'instance n'a pas accès à Internet.

</details>

</details>

---

## Exercice 16 — nmap : scanner et auditer un réseau

**Objectif** : utiliser nmap pour découvrir les machines et les ports ouverts sur ton réseau local — et comprendre ce qu'un attaquant verrait de l'extérieur.

```bash
# Installation
sudo apt install nmap -y

# 1. Scanner le site de test officiel nmap (autorisé)
nmap scanme.nmap.org

# 2. Scanner ta Fritz!Box
nmap 192.168.178.1

# 3. Découvrir toutes les machines actives sur ton réseau local
nmap -sn 192.168.178.0/24

# 4. Scanner les ports courants d'une machine avec détection de version
nmap -sV 192.168.178.1

# 5. Scanner tous les ports de ta propre machine
nmap -p- localhost

# 6. Scan SYN rapide sur ta machine (nécessite sudo)
sudo nmap -sS -p 1-1000 localhost

# 7. Comparer open vs filtered vs closed
# Lance d'abord ufw sur un port
sudo ufw deny 9999/tcp
# Puis scanne
nmap -p 9999 localhost        # → filtered (DROP silencieux)
sudo ufw reject 9999/tcp
nmap -p 9999 localhost        # → closed (REJECT avec réponse)
sudo ufw delete deny 9999/tcp
sudo ufw delete reject 9999/tcp

# 8. Scanner UDP (DNS sur ta Fritz!Box)
sudo nmap -sU -p 53 192.168.178.1
```

**Questions :**

1. Combien de machines actives trouves-tu sur ton réseau `192.168.178.0/24` ?
2. Quels ports sont ouverts sur ta Fritz!Box ? Qu'est-ce que ça révèle sur ses services ?
3. Quelle est la différence entre `filtered` et `closed` dans un résultat nmap ?
4. Pourquoi `nmap -sS` nécessite-t-il `sudo` alors que `nmap` sans option ne le nécessite pas ?
5. Dans un contexte AWS, depuis quelle position ferais-tu un scan nmap pour vérifier la sécurité de ton architecture ?

<details>
<summary>Voir les réponses</summary>

1. En général 3-6 machines : Fritz!Box, ton PC, téléphone, TV, éventuellement NAS ou autres appareils connectés.
2. Souvent ports 53 (DNS), 80 (interface admin HTTP), 443 (interface admin HTTPS), 5060 (VoIP). Ça révèle que la Fritz!Box fait routeur + DNS + serveur web d'administration + VoIP.
3. `filtered` = un firewall bloque silencieusement (DROP) — pas de réponse du tout. `closed` = la machine est joignable mais rien n'écoute sur ce port — elle répond avec un RST/ACK TCP. `filtered` est plus sécurisé car il ne confirme même pas l'existence de la machine.
4. `nmap -sS` (SYN scan) envoie des paquets TCP bruts et nécessite des privilèges root pour manipuler les sockets réseau à ce niveau. `nmap` sans option utilise un scan TCP complet via les appels système normaux — pas besoin de root.
5. **Deux positions complémentaires** : depuis Internet (ton laptop) pour voir ce qu'un attaquant externe verrait — seuls les ports publics devraient apparaître. Depuis une EC2 dans le VPC pour voir la surface d'attaque interne — vérifie que la RDS et les services privés ne sont pas accessibles depuis un subnet qui ne devrait pas les voir.

</details>

---

## Exercice 17 — SSH avancé : tunneling et bastion

**Objectif** : maîtriser le port forwarding SSH et la connexion via bastion — les deux techniques les plus utilisées pour accéder aux ressources privées dans AWS.

### Partie A — Local port forwarding (simulation locale)

```bash
# 1. Démarrer un "service" sur le port 8888 (simule une RDS ou service interne)
python3 -m http.server 8888 &

# 2. Vérifier qu'il écoute
ss -tlnp | grep 8888

# 3. Créer un tunnel SSH local → local (sur ta propre machine)
# Redirige le port local 9999 vers localhost:8888 via SSH loopback
ssh -L 9999:localhost:8888 localhost

# Dans un autre terminal :
curl http://localhost:9999   # ← passe par le tunnel SSH
curl http://localhost:8888   # ← accès direct (comparaison)

# 4. Mode silencieux (pas de shell, juste le tunnel)
ssh -fNL 9999:localhost:8888 localhost

# 5. Tuer le tunnel
pkill -f "ssh -fNL"

# 6. Arrêter le service
kill %1
```

### Partie B — Configurer ~/.ssh/config

```bash
# 1. Voir ta config SSH actuelle
cat ~/.ssh/config 2>/dev/null || echo "Pas de config SSH"

# 2. Créer une config SSH propre
cat >> ~/.ssh/config << 'EOF'

# Serveur de test local
Host localhost-test
    HostName localhost
    User $USER
    Port 22
    ServerAliveInterval 60
    ServerAliveCountMax 3
EOF

# 3. Tester la connexion avec l'alias
ssh localhost-test

# 4. Voir les options utiles
# ServerAliveInterval → envoie un keepalive toutes les 60s
# ServerAliveCountMax → déconnecte après 3 keepalives sans réponse
# Ces options évitent les connexions SSH qui se ferment en idle
```

### Partie C — Simuler un bastion avec deux machines locales

```bash
# Si tu as ta VM Ubuntu (glkaze) disponible, on simule l'architecture bastion :

# Architecture :
# Ton ThinkPad (laptop) → VM Ubuntu (bastion) → service sur VM (cible)

# 1. Démarrer un service sur ta VM (côté VM)
python3 -m http.server 7777

# 2. Depuis ton ThinkPad — connexion directe (devrait échouer si le port est filtré)
curl http://IP-VM:7777

# 3. Tunnel SSH via la VM comme bastion
ssh -L 7777:localhost:7777 glkaze@IP-VM

# Dans un autre terminal sur ton ThinkPad :
curl http://localhost:7777  # ← passe par le tunnel SSH via la VM ✅

# 4. ProxyJump — se connecter à une machine via une autre
# (remplace IP-VM-2 par une deuxième VM si tu en as une)
ssh -J glkaze@IP-VM glkaze@IP-VM-2

# 5. Ajouter dans ~/.ssh/config
cat >> ~/.ssh/config << 'EOF'

Host vm-bastion
    HostName IP-VM
    User glkaze
    IdentityFile ~/.ssh/id_ed25519

Host vm-cible
    HostName IP-VM-2
    User glkaze
    IdentityFile ~/.ssh/id_ed25519
    ProxyJump vm-bastion
EOF

# Connexion directe via bastion en une commande
ssh vm-cible
```

**Questions :**

1. Dans la Partie A, quelle est la différence entre `ssh -L` et `ssh -fNL` ?
2. À quoi sert `ServerAliveInterval` dans `~/.ssh/config` ? Quand est-ce utile en Cloud ?
3. Dans une architecture AWS avec bastion, pourquoi ne copie-t-on PAS la clé privée sur le bastion ?
4. Quelle est la différence entre `ssh -J bastion cible` et se connecter manuellement en deux étapes ?
5. Tu veux accéder à une RDS PostgreSQL privée (`10.0.20.5:5432`) via un bastion (`54.12.34.56`). Écris la commande SSH complète.

<details>
<summary>Voir les réponses</summary>

1. `ssh -L` ouvre le tunnel ET un shell interactif — quand tu fermes le terminal, le tunnel se ferme. `ssh -fNL` lance le tunnel en **arrière-plan** (`-f`) sans ouvrir de shell (`-N`) — le tunnel reste actif même après avoir fermé le terminal.
2. `ServerAliveInterval 60` envoie un paquet keepalive toutes les 60 secondes pour maintenir la connexion active. Sur AWS, les Security Groups et les NAT Gateways ferment les connexions TCP inactives après un timeout (souvent 350 secondes) — sans keepalive, ta session SSH se coupe silencieusement pendant une longue commande.
3. **Principe du moindre privilège et SSH Agent Forwarding** — si le bastion est compromis, l'attaquant ne peut pas récupérer ta clé privée. Avec l'agent forwarding (`ForwardAgent yes`), ta clé reste sur ton laptop mais le bastion peut l'utiliser pour rebondir vers les EC2 privées.
4. `ssh -J bastion cible` est **une seule connexion chiffrée de bout en bout** — ta clé n'est jamais exposée sur le bastion. Deux étapes manuelles nécessitent ta clé sur le bastion ou l'agent forwarding. ProxyJump est plus sécurisé et plus pratique.
5. `ssh -fNL 5432:10.0.20.5:5432 ec2-user@54.12.34.56` — puis se connecter avec `psql -h localhost -p 5432 -U postgres`.

</details>

---

*→ Retour au cours : `module2-theorie.md`*
*→ Module suivant : Module 3 — Networking et conteneurs*
