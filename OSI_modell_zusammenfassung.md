# Le Modèle OSI — Résumé Complet
**OSI = Open Systems Interconnection**

---

## Principe fondamental : Encapsulation

Chaque couche ajoute son propre **header** en envoi et le retire en réception.

```
ENVOI (7→1) :
Couche 7 : [données HTTP]
Couche 6 : [chiffrement][données HTTP]
Couche 5 : [session ID][chiffrement][données HTTP]
Couche 4 : [header TCP/ports][session][chiffrement][données HTTP]
Couche 3 : [header IP][header TCP][session][chiffrement][données HTTP]
Couche 2 : [header MAC][header IP][header TCP][session][chiffrement][données HTTP]
Couche 1 : signaux électriques/radio/optiques

RÉCEPTION (1→7) :
Chaque couche lit et retire son propre header, passe le reste à la couche supérieure.
```

---

## Moyen mnémotechnique

**De 1 → 7 :** **P**lease **D**o **N**ot **T**hrow **S**ausage **P**izza **A**way
**De 7 → 1 :** **A**ll **P**eople **S**eem **T**o **N**eed **D**ata **P**rocessing

---

## Couche 1 — Physical

**Rôle :** Transmettre les bits bruts sous forme de signaux.

**En envoi (7→1) :**
- Convertit les bits en signaux électriques (câble), optiques (fibre) ou radio (Wi-Fi)

**En réception (1→2) :**
- Reçoit les signaux physiques et les convertit en bits `0` et `1`

**Module qui bosse :** Le **transceiver** de la carte réseau (NIC)

**Protocoles/Technologies :** Ethernet (câble), Wi-Fi (802.11), Fibre optique, Bluetooth, USB

**Analogie :** La route sur laquelle roulent les camions — elle transporte sans interpréter.

---

## Couche 2 — Data Link

**Rôle :** Communication dans le réseau local via adresses MAC.

**En envoi (3→2) :**
- Reçoit les données + MAC destination de la couche 3 (via ARP)
- Ajoute header MAC (source + destination)
- Vérifie l'intégrité via CRC
- Passe à la couche 1

**En réception (1→3) :**
- Reçoit la trame de la couche 1
- Lit l'adresse MAC destination
- MAC = la mienne ✅ → retire header MAC, passe à couche 3
- MAC ≠ la mienne ❌ → trame ignorée et jetée

**Module qui bosse :** La **NIC (Network Interface Card)** + son driver

**Protocoles/Technologies :** Ethernet, Wi-Fi (802.11), ARP (fait le lien avec couche 3)

**Analogie :** Le facteur dans ton immeuble — il connaît tous les appartements (MAC) mais ne peut pas sortir de l'immeuble.

**Note importante :** Le **switch** travaille en couche 2. Il maintient une table MAC et envoie chaque trame uniquement au bon port. En mode **promiscuous** (Wireshark), la NIC accepte toutes les trames même celles qui ne lui sont pas destinées.

---

## Couche 3 — Network

**Rôle :** Routage entre réseaux via adresses IP.

**En envoi (4→3) :**
- Reçoit données + ports de la couche 4
- Consulte la table de routage (`ip route`)
- IP destination dans mon réseau local ? → accès direct
- IP destination hors réseau ? → passe par la passerelle (ex: Fritzbox)
- Utilise **ARP** pour trouver le MAC correspondant à l'IP
- Ajoute header IP (source + destination)
- Passe à la couche 2

**En réception (2→4) :**
- Reçoit paquet de la couche 2
- Lit l'IP destination
- IP = la mienne ✅ → retire header IP, passe à couche 4
- IP ≠ la mienne → suis-je un routeur ? Si oui route, sinon jette

**Modules qui bossent :**
- **Pile IP** (noyau Linux) → routage, vérification IP
- **ARP** → résolution IP → MAC (uniquement ça !)
- **ICMP** → diagnostics (`ping`, `traceroute`)

**Commandes utiles :**
```bash
ip route        # table de routage
arp -n          # table ARP (IP → MAC)
ping 8.8.8.8    # test ICMP
```

**Analogie :** La poste nationale — elle route les colis entre villes (IP). Le routeur travaille ici, pas le switch.

**Note :** Le **routeur** travaille en couche 3. Ta Fritzbox contient un modem + switch (L2) + routeur (L3) dans un seul boîtier.

---

## Couche 4 — Transport

**Rôle :** Segmentation, réassemblage, ports et fiabilité.

**En envoi (5→4) :**
- Reçoit les données de la couche 5
- Découpe en segments si données trop grandes
- Ajoute port source et port destination
- Choisit TCP (fiable) ou UDP (rapide)
- Passe à la couche 3

**En réception (3→5) :**
- Reçoit les segments de la couche 3
- Lit les ports → quelle application ?
- Réassemble les segments dans le bon ordre (TCP)
- Vérifie que tous les segments sont arrivés (TCP)
- Retire header TCP/UDP
- Passe à la couche 5

**Module qui bosse :** **Pile TCP/IP** du noyau Linux

**Commandes utiles :**
```bash
ss -tulnp    # voir les ports en écoute
```

**Protocoles :** TCP, UDP

**Analogie :** Le service de livraison — il découpe les gros colis en paquets numérotés et s'assure qu'ils arrivent tous dans le bon ordre.

**Note :** Combinaison IP (couche 3) + Port (couche 4) identifie précisément chaque flux :
```
192.168.178.144:52341 ↔ 142.250.185.46:443  → Google
192.168.178.144:52342 ↔ 208.65.153.238:443  → YouTube
```

---

## Couche 5 — Session

**Rôle :** Ouvrir, maintenir et fermer les conversations entre machines.

**En envoi (6→5) :**
- Ouvre une session si elle n'existe pas encore
- Tague le message avec le contexte de session
- Passe à la couche 6

**En réception (4→6) :**
- Identifie à quelle session appartient ce message
- Vérifie que la session est toujours valide
- Maintient ou ferme la session si nécessaire
- Passe à la couche 6

**Modules qui bossent :**
- **OpenSSL** → sessions TLS/HTTPS
- **libpam** → sessions d'authentification (visible dans `auth.log` !)
- Les applications elles-mêmes (SSH, browser...)

**Protocoles :** NetBIOS, RPC, PPTP, sessions HTTP via cookies

**Analogie :** Le standard téléphonique — il ouvre la ligne, maintient la communication, raccroche quand c'est terminé et peut reprendre si la ligne coupe.

**Exemple concret :**
```
Browser avec 3 onglets ouverts :
Onglet 1 → google.com   → Session A
Onglet 2 → youtube.com  → Session B
Onglet 3 → github.com   → Session C
```
Chaque onglet a sa propre session — pas de mélange !

**Note :** Les couches 5, 6, 7 sont souvent fusionnées dans les applications modernes (HTTP gère ses sessions via cookies sans protocole séparé).

---

## Couche 6 — Presentation

**Rôle :** Chiffrement, encodage et compression des données.

**En envoi (7→6) :**
- Reçoit données brutes de la couche 7
- Compresse (moins de bande passante)
- Encode (UTF-8, ASCII, Base64...)
- Chiffre (AES, RSA, TLS...)
- Passe à la couche 5

**En réception (5→7) :**
- Reçoit données de la couche 5
- Déchiffre
- Décode
- Décompresse
- Passe données lisibles à la couche 7

**Modules qui bossent :**
- **OpenSSL** → chiffrement TLS/SSL
- **Codecs** → compression vidéo/audio (ffmpeg)
- **iconv** → conversion d'encodages

**Protocoles/Formats :** TLS/SSL, UTF-8, ASCII, Base64, JPEG, PNG, MP4

**Analogie :** Le traducteur et l'emballeur — il traduit dans une langue commune, compresse pour économiser de la place et scelle l'enveloppe pour la sécurité.

---

## Couche 7 — Application

**Rôle :** Interface directe avec l'utilisateur et les applications.

**En envoi (utilisateur→réseau) :**
- L'utilisateur génère une action (clic, requête, email...)
- L'application formate selon son protocole
- Passe à la couche 6

**En réception (réseau→utilisateur) :**
- Reçoit données de la couche 6
- Les interprète et affiche à l'utilisateur

**Module qui bosse :** L'**application elle-même** — browser, client SSH, client email...

**Protocoles :**

| Protocole | Usage |
|---|---|
| HTTP/HTTPS | Navigation web |
| SMTP/IMAP/POP3 | Email |
| FTP/SFTP | Transfert de fichiers |
| DNS | Résolution de noms de domaine |
| DHCP | Attribution d'adresses IP |
| SSH | Connexion distante sécurisée |

**Analogie :** La lettre elle-même — c'est le contenu, ce que l'utilisateur voit et utilise directement.

---

## Vue d'ensemble complète

```
┌─────────────────────────────────────────────────────────┐
│  7  Application  │ HTTP, SSH, DNS, SMTP    │ Application │
│  6  Presentation │ TLS, UTF-8, JPEG        │ OpenSSL     │
│  5  Session      │ Sessions, cookies       │ libpam      │
│  4  Transport    │ TCP, UDP, Ports         │ Pile TCP/IP │
│  3  Network      │ IP, ARP, ICMP           │ Pile IP     │
│  2  Data Link    │ MAC, Ethernet, WiFi     │ NIC         │
│  1  Physical     │ Câbles, signaux, ondes  │ Transceiver │
└─────────────────────────────────────────────────────────┘
```

---

## Analogie complète — La livraison d'un colis

```
Couche 7 : Le contenu de la lettre (message)
Couche 6 : La lettre chiffrée et compressée
Couche 5 : Le numéro de conversation (contexte)
Couche 4 : L'enveloppe avec "Bureau 302" (port)
Couche 3 : Le carton avec "Paris, France" (IP)
Couche 2 : L'étiquette transporteur avec code barre (MAC)
Couche 1 : Le camion qui roule sur la route (signaux)
```

---

## Pourquoi le modèle OSI est important ?

- **Diagnostic réseau** → si un problème survient, on sait exactement à quelle couche chercher
- **Interopérabilité** → des machines différentes peuvent communiquer car elles suivent les mêmes règles
- **Modularité** → chaque couche peut évoluer indépendamment

**En Cloud (AWS) :**
- Couche 2 → VPC, subnets, ENI
- Couche 3 → Route Tables, NAT Gateway
- Couche 4 → Security Groups (règles de ports)
- Couche 7 → Application Load Balancer, API Gateway
