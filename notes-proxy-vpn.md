# Proxy et VPN — Notes personnelles
> Résumé de compréhension — pas un module formel

---

## 1. C'est quoi un Proxy ?

Un proxy est un **intermédiaire intelligent** qui se place entre ton PC et Internet. Il opère à la **couche 7 (Application)** — il comprend HTTP, HTTPS, les URLs.

### Le problème qu'il résout

Sans proxy, chaque machine d'une entreprise parle directement à Internet :
- Impossible de contrôler ce que chacun visite
- Chaque PC expose son IP directement
- Pas de cache centralisé
- Pas de logs centralisés

### Ce que fait un proxy

```
PC employé → [PROXY] → Internet → Site web

Le site web voit l'IP du proxy, pas celle du PC
```

**3 rôles principaux :**
1. **Contrôle d'accès** — bloquer YouTube, Facebook, sites malveillants
2. **Anonymisation** — masquer l'IP réelle du client
3. **Cache** — stocker les réponses populaires en mémoire RAM pour éviter de refaire les requêtes

---

## 2. Proxy vs Default Gateway — pas la même chose

C'est la confusion classique. Les deux semblent être "le point de passage" mais ils opèrent à des couches différentes.

| | Default Gateway (Fritz!Box) | Proxy |
|---|---|---|
| Couche OSI | 3 (IP) | 7 (Application) |
| Voit | Des paquets IP | Des requêtes HTTP |
| Comprend | L'IP destination | L'URL destination |
| Peut bloquer YouTube | ❌ | ✅ |
| Fait le NAT | ✅ | ❌ |
| Toujours présent | ✅ | Optionnel |

**Le proxy utilise lui-même le Default Gateway pour sortir vers Internet — il ne le remplace pas.**

```
TON PC
  │ IP dst = proxy (réseau local → pas besoin du Gateway)
  ▼
PROXY
  │ IP dst = youtube.com (réseau externe → passe par le Gateway)
  ▼
FRITZ!BOX (Default Gateway)
  │ NAT
  ▼
INTERNET → youtube.com
```

---

## 3. Les 3 façons de configurer un Proxy

### Méthode 1 — Configuration dans le navigateur
- Uniquement Chrome/Firefox passe par le proxy
- Les autres apps (Spotify, jeux...) ignorent le proxy
- Config : Paramètres → Proxy → IP:port

### Méthode 2 — Configuration OS (variable d'environnement)
```bash
HTTP_PROXY=http://192.168.1.100:8080
HTTPS_PROXY=http://192.168.1.100:8080
```
- Les apps qui respectent cette variable passent par le proxy
- curl, wget, apt, pip, npm... oui
- Certaines apps non

### Méthode 3 — Proxy transparent
- **Aucune configuration sur le PC**
- Le routeur intercepte tout le trafic via règles iptables
- Redirige vers le proxy automatiquement
- Le PC ne sait même pas qu'il y a un proxy

---

## 4. Ce que le Proxy voit (le problème HTTPS)

```
HTTP (port 80)  → proxy voit TOUT : URL complète, contenu
HTTPS (port 443) → proxy voit seulement le nom de domaine
                   via SNI (Server Name Indication)
                   le contenu reste chiffré → illisible
```

**SNI** = le nom de domaine envoyé en clair avant le handshake TLS pour que le serveur sache quel certificat présenter. Le proxy peut bloquer `youtube.com` même en HTTPS grâce au SNI — mais ne peut pas voir quelle vidéo exactement.

### SSL Inspection — voir même le contenu HTTPS
Certaines entreprises installent un **certificat racine d'entreprise** sur tous les PCs. Le proxy déchiffre le trafic côté PC, lit le contenu, rechiffre côté Internet. C'est techniquement un Man-in-the-Middle autorisé par l'entreprise.

---

## 5. Les agents proxy installés sur le PC

Des applications dédiées s'insèrent entre l'OS et le réseau pour tout intercepter :

- **Zscaler Client Connector** — intercepte tout, redirige vers proxy Cloud
- **Cisco Umbrella** — intercepte les requêtes DNS
- **Pulse Secure / Ivanti** — client VPN d'entreprise

Ces agents installent un **driver réseau virtuel** et modifient les règles iptables automatiquement.

---

## 6. C'est quoi un VPN ?

Un VPN (Virtual Private Network) est un **tunnel chiffré** entre ton PC et un serveur distant. Il opère à la **couche 3 (Réseau)** — il intercepte TOUS les paquets IP, pas seulement le HTTP.

### Comment ça marche concrètement

```
1. Client VPN crée une interface réseau virtuelle
   ip addr show
   → eth0 : 192.168.178.10  (réseau local réel)
   → tun0 : 10.8.0.5        (interface virtuelle VPN)

2. Modifie la table de routage
   AVANT : default via 192.168.178.1 dev eth0 (Fritz!Box)
   APRÈS : default via 10.8.0.1 dev tun0 (tunnel VPN)

3. TOUT le trafic passe dans le tunnel chiffré
   PC → [chiffrement] → tun0 → eth0 → Fritz!Box → Internet → Serveur VPN → [déchiffrement] → Internet
```

---

## 7. Proxy vs VPN — la différence fondamentale

| | Proxy | VPN |
|---|---|---|
| Couche OSI | 7 (Application) | 3 (Réseau) |
| Intercepte | Requêtes HTTP seulement | TOUS les paquets IP |
| Applications couvertes | Navigateur (ou apps configurées) | TOUTES sans exception |
| Chiffrement | Optionnel | Toujours (IPsec, WireGuard, OpenVPN) |
| Vitesse | Plus rapide | Légèrement plus lent |
| Configuration client | Nécessaire (sauf transparent) | Client VPN installé |

### Visuellement

```
PROXY — seul le navigateur passe par l'intermédiaire :

  Chrome ──────────────► Proxy ──► youtube.com
  Spotify ────────────────────────────────────► Internet direct
  Jeu en ligne ───────────────────────────────► Internet direct


VPN — TOUT passe dans le tunnel :

  Chrome ──────┐
  Spotify ─────┤
  Jeu ─────────┼──► tun0 ──► tunnel chiffré ──► Serveur VPN ──► Internet
  TOUT ────────┘
```

---

## 8. Analogies pour retenir

```
PROXY = traducteur/intermédiaire
  Tu parles à un traducteur
  Le traducteur parle à l'autre personne
  Le traducteur comprend ce que tu dis
  Il peut décider de ne pas traduire certaines choses

VPN = tunnel blindé
  Tu entres dans un tunnel blindé avec ta voiture
  Personne ne voit ce que tu transportes
  A la sortie, ton point de départ apparent
  c'est la sortie du tunnel, pas chez toi

DEFAULT GATEWAY = facteur
  Lit juste l'adresse sur l'enveloppe
  Livre sans regarder le contenu

PROXY = douanier
  Ouvre chaque colis
  Vérifie le contenu
  Autorise ou bloque
  Garde une copie si nécessaire (cache)
```

---

## 9. Quand utiliser quoi

```
PROXY
  ✅ Contrôle d'accès en entreprise
  ✅ Cache pour économiser la bande passante
  ✅ Anonymisation légère et rapide
  ✅ Contourner restrictions géo sur un navigateur
  ❌ Pas de protection pour toutes les apps
  ❌ Pas de chiffrement fort

VPN
  ✅ Accès sécurisé réseau entreprise (télétravail)
  ✅ Chiffrement total — WiFi public dans un café
  ✅ Anonymisation complète toutes les apps
  ✅ Contourner restrictions géo pour tout
  ❌ Légèrement plus lent
  ❌ Le fournisseur VPN voit ton trafic déchiffré
```

---

## 10. Le lien avec AWS (Modules 4 et 6)

```
AWS VPN Gateway
  → tunnel IPsec entre ton datacenter et ton VPC
  → couche 3 — tous les paquets IP
  → exactement le mécanisme VPN

AWS PrivateLink
  → accès privé à un service sans passer par Internet
  → plus proche du proxy transparent
  → couche 7 — trafic applicatif

Proxy d'entreprise → AWS Security Groups
  → les deux filtrent le trafic
  → le proxy filtre par URL/domaine (couche 7)
  → les SG filtrent par IP/port (couche 3/4)

Forward Proxy  = protège les clients (couche 7)
Reverse Proxy  = protège les serveurs (couche 7) → ALB, nginx, CloudFront
VPN Gateway    = connecte deux réseaux (couche 3)
NAT Gateway    = sortie Internet pour subnets privés (couche 3)
```

---

## 11. Les services concrets — qui fait quoi

### nginx

**Créé en 2004** par Igor Sysoev (ingénieur russe) pour résoudre le problème C10K — gérer 10 000 connexions simultanées, ce qu'Apache ne faisait pas bien.

```
Rôle principal : serveur web + reverse proxy

Ce que nginx fait concrètement :

  1. Serveur web (fichiers statiques)
     → sert du HTML, CSS, JS, images
     → ultra-rapide pour les fichiers statiques

  2. Reverse proxy (le plus courant en prod)
     → reçoit les requêtes HTTPS des clients
     → déchiffre le TLS (SSL termination)
     → transmet en HTTP simple à l'app derrière
     → l'app Node.js/Python/Java ne gère pas TLS

  3. Load balancer
     → distribue entre plusieurs instances de l'app
     → retire automatiquement les instances en panne

Lien avec les concepts :
  → C'est un REVERSE PROXY (protège les serveurs)
  → Couche 7 (comprend HTTP/HTTPS)
  → Pas de lien avec le VPN

Exemple config reverse proxy nginx :
  Client → HTTPS:443 → [nginx] → HTTP:3000 → App Node.js
```

---

### Apache HTTP Server

**Créé en 1995** — le premier grand serveur web open source. A fait tourner plus de 50% d'Internet pendant des années.

```
Rôle : même que nginx mais architecture différente

  nginx   → asynchrone, event-driven
            gère des milliers de connexions avec peu de RAM
            meilleur pour les fichiers statiques et le proxy

  Apache  → multi-process/multi-thread
            un processus/thread par connexion
            meilleur pour les langages comme PHP
            via le module mod_php intégré

Lien avec les concepts :
  → aussi un REVERSE PROXY (mod_proxy)
  → Couche 7
  → Pas de lien avec le VPN
  → Aujourd'hui souvent remplacé par nginx en prod

Quand tu vois Apache aujourd'hui :
  → hébergements web partagés (OVH, Hostinger...)
  → applications PHP legacy (WordPress, Drupal)
  → environnements LAMP (Linux Apache MySQL PHP)
```

---

### Squid

**Créé en 1996** — le proxy forward open source de référence.

```
Rôle : FORWARD PROXY (protège les clients)
       C'est le "vrai" proxy d'entreprise

Ce que Squid fait :
  → contrôle d'accès (bloquer sites par URL/catégorie)
  → cache des réponses HTTP (économie bande passante)
  → authentification des utilisateurs
  → logging de tout le trafic
  → SSL inspection (voir le contenu HTTPS)

Lien avec les concepts :
  → C'est un FORWARD PROXY (côté clients)
  → Couche 7
  → Pas de lien avec le VPN
  → Typiquement installé sur un serveur Linux
    dans le réseau de l'entreprise

Tu le rencontres dans :
  → réseaux d'entreprise, écoles, universités
  → souvent couplé avec Cisco ou Fortinet
  → pour filtrer Internet des employés
```

---

### HAProxy

**Créé en 2000** — le load balancer/proxy haute performance de référence.

```
Rôle : load balancer + reverse proxy ultra-performant

Différence avec nginx :
  nginx   → serveur web + reverse proxy (polyvalent)
  HAProxy → UNIQUEMENT load balancer/proxy
            pas de serveur web
            mais encore plus performant pour le load balancing

Ce que HAProxy fait :
  → distribue le trafic entre des dizaines de serveurs
  → health checks en temps réel
  → routing avancé (par URL, par header, par cookie)
  → supporte TCP et HTTP (couches 4 ET 7)
  → millions de connexions simultanées

Lien avec les concepts :
  → REVERSE PROXY (couche 4 et 7)
  → Pas de lien avec le VPN
  → Utilisé devant nginx ou directement devant les apps

Tu le rencontres dans :
  → grosses infrastructures web (Twitter, GitHub l'utilisaient)
  → devant des clusters de serveurs
  → souvent en combinaison avec nginx
```

---

### Traefik

**Créé en 2015** — le reverse proxy moderne pensé pour les conteneurs.

```
Rôle : reverse proxy + load balancer pour Docker/Kubernetes

Ce qui le différencie de nginx/HAProxy :
  → auto-découverte des services Docker/K8s
  → pas besoin de modifier la config manuellement
    quand tu ajoutes un nouveau conteneur
  → il détecte automatiquement les nouveaux services
  → gère Let's Encrypt automatiquement (certificats TLS)

Lien avec les concepts :
  → REVERSE PROXY (couche 7)
  → Ingress Controller dans Kubernetes (Module 5)
  → Pas de lien avec le VPN

Tu le rencontres dans :
  → environnements Docker Compose
  → clusters Kubernetes (comme Ingress Controller)
  → startups et équipes DevOps modernes
```

---

### WireGuard

**Créé en 2016** par Jason Donenfeld — le VPN moderne qui a révolutionné le domaine.

```
Rôle : protocole VPN ultra-simple et ultra-rapide

Pourquoi c'est une révolution :
  OpenVPN (ancien) : 400 000 lignes de code
  WireGuard        :   4 000 lignes de code
  → 100x moins de code = 100x moins de bugs potentiels
  → intégré directement dans le noyau Linux depuis 2020

Ce que WireGuard fait :
  → crée une interface réseau virtuelle (wg0)
  → chiffrement moderne (ChaCha20, Curve25519)
  → ultra-rapide — comparable à pas de VPN
  → connexion quasi-instantanée (vs OpenVPN = plusieurs secondes)

Lien avec les concepts :
  → VPN pur (couche 3)
  → Crée une interface tun/wg sur le PC
  → Modifie la table de routage
  → Tout le trafic passe dans le tunnel

Tu le rencontres dans :
  → VPNs personnels (Mullvad, ProtonVPN utilisent WireGuard)
  → connexions site-to-site
  → remplacement moderne d'OpenVPN
  → AWS VPN supporte WireGuard indirectement
```

---

### OpenVPN

**Créé en 2001** — le VPN open source historique.

```
Rôle : protocole VPN mature et très répandu

  → fonctionne sur TCP ou UDP
  → port 1194 UDP (ou 443 TCP pour contourner les firewalls)
  → très configurable mais complexe
  → plus lent que WireGuard
  → encore très utilisé en entreprise

Lien avec les concepts :
  → VPN pur (couche 3)
  → Même mécanisme que WireGuard (tun0, table de routage)
  → Pulse Secure / Cisco AnyConnect sont basés
    sur des principes similaires

Tu le rencontres dans :
  → VPNs d'entreprise (avec serveur OpenVPN interne)
  → solutions comme ProtonVPN, NordVPN
  → accès remote aux réseaux d'entreprise
```

---

### Zscaler

**Fondé en 2007** — le proxy d'entreprise Cloud moderne.

```
Rôle : proxy forward as a Service (SaaS)

La différence avec Squid (proxy local) :
  Squid    → installé sur un serveur dans l'entreprise
             les employés au bureau passent par lui

  Zscaler  → service Cloud hébergé par Zscaler
             l'agent sur le PC envoie TOUT le trafic
             vers les serveurs Zscaler dans le Cloud
             même depuis chez toi (télétravail)
             même sur un réseau 4G

Ce que Zscaler fait :
  → filtre les URLs malveillantes
  → inspecte le trafic HTTPS (SSL inspection)
  → détecte les malwares
  → log tout le trafic de tous les employés
  → fonctionne partout dans le monde

Lien avec les concepts :
  → FORWARD PROXY (couche 7) dans le Cloud
  → Agent installé sur le PC (comme on a vu)
  → Pas un VPN — mais l'agent ressemble à un client VPN
    car il redirige tout le trafic

Tu le rencontres dans :
  → grandes entreprises (banques, assurances, CAC40)
  → politique "Zero Trust Network Access"
  → souvent imposé sur les laptops d'entreprise
```

---

### Cloudflare

**Fondé en 2010** — le réseau de proxy/CDN mondial le plus utilisé.

```
Rôle : reverse proxy + CDN + protection DDoS as a Service

Ce que Cloudflare fait :
  → se place devant ton site web
  → masque l'IP réelle de ton serveur
  → cache ton contenu dans 300+ datacenters mondiaux
  → protège contre les attaques DDoS
  → gère les certificats TLS gratuitement
  → optimise les performances (HTTP/3, compression)

Lien avec les concepts :
  → REVERSE PROXY géant (couche 7)
  → CDN (Content Delivery Network) = cache mondial
  → Pas un VPN

  Cloudflare a aussi des produits VPN-like :
  → Cloudflare WARP = VPN grand public (WireGuard)
  → Cloudflare Access = Zero Trust (comme Zscaler)

Tu le rencontres partout :
  → ~20% d'Internet passe par Cloudflare
  → sites qui ont le logo "Protected by Cloudflare"
  → AWS CloudFront est l'équivalent AWS de Cloudflare CDN
```

---

### Tableau récapitulatif de tous les services

| Service | Type | Couche | Côté | Usage |
|---------|------|--------|------|-------|
| **nginx** | Reverse proxy + serveur web | 7 | Serveur | App web, load balancing |
| **Apache** | Reverse proxy + serveur web | 7 | Serveur | App PHP, hébergement |
| **HAProxy** | Load balancer + reverse proxy | 4 et 7 | Serveur | Haute perf, gros trafic |
| **Traefik** | Reverse proxy | 7 | Serveur | Docker/Kubernetes |
| **Squid** | Forward proxy | 7 | Client | Contrôle accès entreprise |
| **Zscaler** | Forward proxy Cloud | 7 | Client | Zero Trust entreprise |
| **Cloudflare** | Reverse proxy + CDN | 7 | Serveur | Protection DDoS, cache |
| **WireGuard** | VPN | 3 | Client+Serveur | VPN moderne, rapide |
| **OpenVPN** | VPN | 3 | Client+Serveur | VPN entreprise historique |
| **Pulse Secure** | VPN client | 3 | Client | Accès réseau entreprise |
| **ALB AWS** | Reverse proxy managé | 7 | Serveur | Load balancing sur AWS |
| **CloudFront** | Reverse proxy + CDN | 7 | Serveur | CDN AWS |
| **AWS VPN GW** | VPN | 3 | Réseau | Connexion datacenter↔AWS |

---

*Notes personnelles — discussion interactive*

