# 🔐 Sécurité Linux & Réseau
## Guide complet du SysAdmin — Firewalls, Auth, Autorisations, Proxies

> **Niveau** : Intermédiaire  
> **Cible** : SysAdmin / DevOps / Cloud Engineer  
> **Périmètre** : Firewalls, Authentification, Autorisation, DAC, MAC, RBAC, Proxies, et bonnes pratiques

---

## Table des matières

1. [Les concepts fondamentaux de la sécurité](#1-les-concepts-fondamentaux-de-la-sécurité)
2. [Authentification — Qui es-tu ?](#2-authentification--qui-es-tu-)
3. [Autorisation — Qu'as-tu le droit de faire ?](#3-autorisation--quas-tu-le-droit-de-faire-)
4. [DAC — Discretionary Access Control](#4-dac--discretionary-access-control)
5. [MAC — Mandatory Access Control](#5-mac--mandatory-access-control)
6. [RBAC — Role-Based Access Control](#6-rbac--role-based-access-control)
7. [Les Firewalls — Théorie](#7-les-firewalls--théorie)
8. [iptables — Le firewall historique Linux](#8-iptables--le-firewall-historique-linux)
9. [nftables — Le successeur moderne](#9-nftables--le-successeur-moderne)
10. [UFW — Interface simplifiée (Ubuntu)](#10-ufw--interface-simplifiée-ubuntu)
11. [Firewalls applicatifs (WAF)](#11-firewalls-applicatifs-waf)
12. [Proxies — Théorie et types](#12-proxies--théorie-et-types)
13. [Proxy Firewall](#13-proxy-firewall)
14. [Reverse Proxy](#14-reverse-proxy)
15. [SSH — Sécurité en profondeur](#15-ssh--sécurité-en-profondeur)
16. [Chiffrement — Les bases indispensables](#16-chiffrement--les-bases-indispensables)
17. [Surveillance et détection](#17-surveillance-et-détection)
18. [Checklist sécurité d'un nouveau serveur](#18-checklist-sécurité-dun-nouveau-serveur)
19. [Glossaire](#19-glossaire)

---

## 1. Les concepts fondamentaux de la sécurité

### La triade CIA

Toute la sécurité informatique repose sur trois piliers fondamentaux appelés **CIA** (pas l'agence — l'acronyme anglais) :

```
┌─────────────────────────────────────────────────┐
│                                                 │
│         C — Confidentiality (Confidentialité)  │
│         I — Integrity       (Intégrité)         │
│         A — Availability    (Disponibilité)     │
│                                                 │
└─────────────────────────────────────────────────┘
```

| Pilier | Définition | Exemple de menace | Exemple de protection |
|--------|-----------|-------------------|-----------------------|
| **Confidentialité** | Seules les personnes autorisées accèdent aux données | Interception réseau (eavesdropping) | Chiffrement TLS, VPN |
| **Intégrité** | Les données ne peuvent pas être altérées sans détection | Modification d'un fichier par un attaquant | Checksums, signatures GPG |
| **Disponibilité** | Le système est accessible quand on en a besoin | Attaque DDoS, panne | Redondance, rate limiting, firewall |

> 💡 Un firewall protège principalement la **confidentialité** et la **disponibilité**. Un système de permissions protège la **confidentialité** et l'**intégrité**.

### Les trois questions de sécurité

Toute décision de sécurité répond à ces trois questions dans cet ordre :

```
1. AUTHENTIFICATION  → Qui es-tu ?         (Prouver son identité)
        ↓
2. AUTORISATION      → Qu'as-tu le droit ? (Ce que l'identité peut faire)
        ↓
3. AUDIT (ACCOUNTING)→ Qu'as-tu fait ?     (Traçabilité des actions)
```

Ces trois étapes forment l'acronyme **AAA** (Triple-A), fondamental en sécurité réseau (utilisé notamment dans RADIUS, Kerberos, AWS IAM).

### Principe du moindre privilège (Least Privilege)

> **Règle d'or** : Toute entité (utilisateur, service, processus) ne doit avoir que les permissions **strictement nécessaires** à son fonctionnement, rien de plus.

Exemples concrets :
- Un service web `nginx` n'a pas besoin d'accès root → le faire tourner en utilisateur `www-data`
- Un développeur n'a pas besoin d'accès à la base de données de production
- Un script de backup n'a besoin que de la lecture, pas de l'écriture

### Défense en profondeur (Defense in Depth)

Ne jamais reposer la sécurité sur une seule couche. Si une couche est compromise, la suivante doit tenir :

```
Internet
    │
    ▼
[Firewall réseau]          ← couche 1 : filtrage des connexions
    │
    ▼
[Proxy / Load Balancer]    ← couche 2 : filtrage applicatif
    │
    ▼
[Firewall applicatif WAF]  ← couche 3 : analyse du contenu HTTP
    │
    ▼
[Authentification forte]   ← couche 4 : vérification d'identité
    │
    ▼
[Autorisations RBAC]       ← couche 5 : contrôle des actions
    │
    ▼
[Chiffrement des données]  ← couche 6 : protection des données au repos
    │
    ▼
[Audit et monitoring]      ← couche 7 : détection des anomalies
```

---

## 2. Authentification — Qui es-tu ?

### Les facteurs d'authentification

L'authentification repose sur un ou plusieurs des trois facteurs suivants :

| Facteur | Définition | Exemples |
|---------|-----------|---------|
| **Quelque chose que tu sais** | Un secret mémorisé | Mot de passe, PIN, phrase secrète |
| **Quelque chose que tu as** | Un objet physique | Carte à puce, YubiKey, téléphone (TOTP) |
| **Quelque chose que tu es** | Une caractéristique biologique | Empreinte, reconnaissance faciale |

**MFA (Multi-Factor Authentication)** : combiner au moins deux facteurs différents. Un mot de passe + un code SMS = MFA (2FA). C'est le minimum en production.

### Authentification par mot de passe — Ce qu'il faut savoir

Les mots de passe ne sont **jamais** stockés en clair. Sur Linux, les hashés sont dans `/etc/shadow` :

```bash
cat /etc/shadow
# gilles:$6$rounds=5000$sel$hashlong...:19800:0:99999:7:::
#   |       |  |         |   |
#   |       |  |         |   └─ Hash du mot de passe
#   |       |  |         └─ Sel (salt) aléatoire
#   |       |  └─ Nombre d'itérations
#   |       └─ Algorithme : $6$ = SHA-512
#   └─ Nom d'utilisateur
```

**Algorithmes de hashage courants sur Linux :**

| Préfixe | Algorithme | Statut |
|---------|-----------|--------|
| `$1$` | MD5 | ❌ Obsolète, ne pas utiliser |
| `$5$` | SHA-256 | ⚠️ Acceptable |
| `$6$` | SHA-512 | ✅ Standard actuel |
| `$y$` | yescrypt | ✅ Recommandé (Ubuntu 22.04+) |

> ⚠️ **Pourquoi le salt ?** Sans sel, deux utilisateurs avec le même mot de passe auraient le même hash, et les attaques par rainbow table (tables précalculées) seraient efficaces. Le sel est aléatoire et unique par utilisateur, rendant ces attaques impossibles.

### Authentification par clé SSH — Le standard Cloud

C'est le mécanisme d'authentification le plus utilisé en Cloud/DevOps. Il repose sur la cryptographie asymétrique :

```
┌──────────────────────────────────────────────────────────────┐
│                    Paire de clés SSH                         │
│                                                              │
│  Clé PRIVÉE  ←──── garde sur ta machine, ne partage JAMAIS  │
│  Clé PUBLIQUE ───→ dépose sur le serveur (authorized_keys)  │
│                                                              │
│  Ce que la clé publique chiffre, seule la clé privée peut   │
│  déchiffrer. Et vice versa.                                  │
└──────────────────────────────────────────────────────────────┘
```

**Le processus d'authentification SSH :**

```
Client (toi)                        Serveur
    │                                   │
    │──── 1. "Je veux me connecter" ───▶│
    │                                   │ Génère un challenge aléatoire
    │◀─── 2. Challenge chiffré ─────────│ chiffré avec ta clé publique
    │                                   │
    │ Déchiffre avec ta clé privée      │
    │──── 3. Répond au challenge ──────▶│
    │                                   │ Vérifie la réponse
    │◀─── 4. "Accès accordé" ───────────│
```

**Générer une paire de clés :**

```bash
# Algorithme recommandé en 2024 : Ed25519
ssh-keygen -t ed25519 -C "gilles@monserveur"

# RSA 4096 bits (compatible mais moins moderne)
ssh-keygen -t rsa -b 4096 -C "gilles@monserveur"

# TOUJOURS protéger la clé privée par une passphrase !
```

**Déployer la clé publique sur un serveur :**

```bash
# Méthode simple
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@serveur

# Méthode manuelle
cat ~/.ssh/id_ed25519.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
chmod 700 ~/.ssh/
```

### PAM — Pluggable Authentication Modules

PAM est la couche d'abstraction d'authentification de Linux. Il permet de changer la méthode d'authentification sans modifier les applications.

```
Application (sshd, login, sudo...)
        │
        ▼
      [PAM]  ← point central
        │
   ┌────┴────────────────────┐
   ▼         ▼               ▼
pam_unix  pam_ldap      pam_google_auth
(shadow)  (Active Dir)  (2FA TOTP)
```

**Fichiers de config PAM :**

```bash
ls /etc/pam.d/          # un fichier par service
cat /etc/pam.d/sshd     # config PAM pour SSH
cat /etc/pam.d/sudo     # config PAM pour sudo
```

**Modules PAM importants :**

| Module | Rôle |
|--------|------|
| `pam_unix` | Authentification locale (shadow) |
| `pam_ldap` | Authentification LDAP/Active Directory |
| `pam_google_authenticator` | TOTP 2FA (Google Authenticator) |
| `pam_faillock` | Verrouillage après X tentatives échouées |
| `pam_pwquality` | Politique de complexité des mots de passe |
| `pam_limits` | Limites de ressources par utilisateur |

---

## 3. Autorisation — Qu'as-tu le droit de faire ?

### Authentification ≠ Autorisation

C'est une distinction fondamentale que beaucoup confondent :

- **Authentification** : "C'est bien toi, Gilles." (identité prouvée)
- **Autorisation** : "Mais toi, Gilles, tu n'as pas le droit de modifier ce fichier." (droits vérifiés)

On peut être authentifié sans être autorisé à faire quelque chose. C'est pour ça qu'on a des systèmes de contrôle d'accès distincts.

### Les modèles de contrôle d'accès

Il existe trois grandes familles de modèles, qu'on va explorer en détail dans les sections suivantes :

| Modèle | Nom complet | Qui décide des droits ? | Exemple |
|--------|-------------|------------------------|---------|
| **DAC** | Discretionary Access Control | Le propriétaire de la ressource | Permissions Unix (`chmod`) |
| **MAC** | Mandatory Access Control | Le système (politique centrale) | SELinux, AppArmor |
| **RBAC** | Role-Based Access Control | L'administrateur via des rôles | AWS IAM, sudo rules |

---

## 4. DAC — Discretionary Access Control

### Concept

Dans le DAC, **le propriétaire d'une ressource décide qui peut y accéder**. Il a la discrétion (d'où le nom) d'accorder ou révoquer les permissions. C'est le modèle natif de Linux.

> **Analogie** : tu es propriétaire d'un appartement. Tu décides qui a une clé. Tu peux donner une clé à ton ami, à ton voisin, ou à personne. C'est toi qui choisis, pas le gouvernement.

### Les permissions Unix (DAC classique)

Chaque fichier et répertoire Linux a :
- Un **propriétaire** (user)
- Un **groupe** (group)
- Des **permissions** pour : propriétaire / groupe / tous les autres

```bash
ls -la /etc/passwd
# -rw-r--r-- 1 root root 2847 mars  15 10:23 /etc/passwd
#  │││││││││   │    │
#  │││││││││   │    └─ Groupe propriétaire
#  │││││││││   └─ Utilisateur propriétaire
#  ││││││││└─ Autres (other) : r--
#  │││││└──── Groupe (group) : r--
#  ││└──────── Propriétaire (user) : rw-
#  └─ Type : - = fichier, d = dossier, l = lien
```

### La table des permissions

| Permission | Octal | Sur un fichier | Sur un répertoire |
|-----------|-------|---------------|-------------------|
| `r` (read) | 4 | Lire le contenu | Lister les fichiers (`ls`) |
| `w` (write) | 2 | Modifier le fichier | Créer/supprimer des fichiers |
| `x` (execute) | 1 | Exécuter le fichier | Entrer dans le dossier (`cd`) |
| `-` | 0 | Aucun droit | Aucun droit |

**Calcul octal :**

```
rwx = 4+2+1 = 7
rw- = 4+2+0 = 6
r-x = 4+0+1 = 5
r-- = 4+0+0 = 4
--- = 0+0+0 = 0

chmod 755 fichier  →  rwxr-xr-x
chmod 644 fichier  →  rw-r--r--
chmod 600 fichier  →  rw-------
chmod 700 dossier  →  rwx------
```

**Permissions types et leurs usages :**

| Valeur | Pattern | Usage typique |
|--------|---------|---------------|
| `777` | `rwxrwxrwx` | ❌ DANGER — jamais en production |
| `755` | `rwxr-xr-x` | Répertoires, scripts exécutables |
| `644` | `rw-r--r--` | Fichiers de config lisibles par tous |
| `640` | `rw-r-----` | Fichiers de config sensibles (groupe seulement) |
| `600` | `rw-------` | Clés privées SSH, fichiers secrets |
| `700` | `rwx------` | Répertoires privés (ex: ~/.ssh) |
| `400` | `r--------` | Fichiers en lecture seule stricte |

### Commandes de gestion DAC

```bash
# Changer les permissions
chmod 644 fichier.txt
chmod u+x script.sh        # ajouter exécution au propriétaire
chmod g-w fichier.txt      # retirer écriture au groupe
chmod o= fichier.txt       # supprimer tous les droits aux autres

# Changer le propriétaire
chown gilles fichier.txt
chown gilles:devops fichier.txt    # propriétaire + groupe

# Changer le groupe
chgrp devops fichier.txt

# Récursif
chmod -R 755 /var/www/html/
chown -R www-data:www-data /var/www/
```

### Les bits spéciaux : SUID, SGID, Sticky Bit

En plus des permissions classiques, Linux dispose de trois bits spéciaux souvent méconnus mais importants.

#### SUID (Set User ID) — bit 4

```bash
chmod u+s fichier
# ou
chmod 4755 fichier
# Affichage : -rwsr-xr-x (s à la place du x du propriétaire)
```

**Effet** : quand ce fichier est exécuté, il tourne avec les permissions du **propriétaire** (pas de l'exécutant).

**Exemple réel :** `/usr/bin/passwd` a le SUID root. Quand un utilisateur normal change son mot de passe, `passwd` a besoin de modifier `/etc/shadow` qui appartient à root. Le SUID lui permet d'avoir temporairement les droits root.

```bash
ls -la /usr/bin/passwd
# -rwsr-xr-x 1 root root 68208 ... /usr/bin/passwd
#      ^ s = SUID actif
```

> ⚠️ **Danger** : un fichier SUID root mal sécurisé est une faille d'escalade de privilèges classique. Lister tous les SUID sur un système :

```bash
find / -perm /4000 -type f 2>/dev/null
```

#### SGID (Set Group ID) — bit 2

```bash
chmod g+s repertoire/
# Affichage : drwxr-sr-x
```

**Effet sur un répertoire** : tous les fichiers créés dans ce répertoire hériteront du groupe du répertoire (pas du groupe de l'utilisateur qui crée). Très utile pour les répertoires partagés entre une équipe.

```bash
# Exemple : répertoire partagé pour l'équipe devops
sudo mkdir /srv/devops-shared
sudo chgrp devops /srv/devops-shared
sudo chmod 2775 /srv/devops-shared
# Maintenant tout fichier créé ici appartient au groupe devops
```

#### Sticky Bit — bit 1

```bash
chmod +t repertoire/
chmod 1777 /tmp
# Affichage : drwxrwxrwt
```

**Effet** : dans un répertoire avec le sticky bit, un utilisateur ne peut supprimer que **ses propres fichiers**, même si le répertoire est accessible à tous en écriture.

**Exemple :** `/tmp` est accessible à tous (777) mais avec le sticky bit (1777). Chaque utilisateur peut créer des fichiers dans `/tmp` mais ne peut pas supprimer ceux des autres.

```bash
ls -ld /tmp
# drwxrwxrwt 10 root root 4096 ... /tmp
#          ^ t = sticky bit actif
```

### Les ACL (Access Control Lists) — DAC étendu

Les permissions Unix standards ne permettent que trois entités (owner/group/other). Les ACL permettent de définir des permissions pour des utilisateurs ou groupes **spécifiques** supplémentaires.

```bash
# Installer les outils ACL
sudo apt install acl

# Voir les ACL d'un fichier
getfacl fichier.txt

# Donner des droits de lecture à un utilisateur spécifique
setfacl -m u:alice:r fichier.txt

# Donner des droits d'écriture à un groupe spécifique
setfacl -m g:devops:rw fichier.txt

# Supprimer une ACL
setfacl -x u:alice fichier.txt

# Supprimer toutes les ACL
setfacl -b fichier.txt
```

**Sortie de `getfacl` :**

```
# file: fichier.txt
# owner: gilles
# group: gilles
user::rw-
user:alice:r--        ← ACL spécifique pour alice
group::r--
group:devops:rw-      ← ACL spécifique pour le groupe devops
mask::rw-
other::r--
```

> 💡 Si un fichier a des ACL, `ls -la` affiche un `+` à la fin des permissions : `-rw-rw-r--+`

### Limites du DAC

Le DAC est souple mais a une faiblesse fondamentale : **si un processus est compromis, il hérite des droits de son propriétaire**. Si tu lances une application en root et qu'elle est exploitée, l'attaquant a les droits root. C'est là qu'intervient le MAC.

---

## 5. MAC — Mandatory Access Control

### Concept

Dans le MAC, **c'est le système (une politique centrale) qui décide des droits**, pas le propriétaire de la ressource. Même root peut être contraint par le MAC.

> **Analogie** : tu travailles dans une agence gouvernementale. Même si tu es propriétaire d'un document classifié "Top Secret", tu ne peux pas le transmettre à quelqu'un qui n'a pas l'habilitation correspondante. Ce n'est plus toi qui décides — c'est la politique de sécurité de l'État.

**Différence clé avec le DAC :**
- En DAC : le propriétaire peut donner ses droits à qui il veut
- En MAC : même le propriétaire ne peut pas outrepasser la politique centrale

### SELinux (Security-Enhanced Linux)

SELinux est développé par la NSA et intégré au kernel Linux. Il est le standard sur Red Hat, CentOS, Fedora. Il associe des **labels (contextes)** à tous les processus, fichiers et ressources.

**Concept central : le contexte SELinux**

```bash
ls -Z /var/www/html/index.html
# -rw-r--r--. root root unconfined_u:object_r:httpd_sys_content_t:s0 index.html
#                                    |          |                  |   |
#                                    |          |                  |   └─ Niveau de sensibilité
#                                    |          |                  └─ Type (le plus important)
#                                    |          └─ Rôle
#                                    └─ Utilisateur SELinux
```

**Le type est l'élément clé :** une règle SELinux dit par exemple "le processus de type `httpd_t` (Apache) peut lire les fichiers de type `httpd_sys_content_t`". Si Apache est compromis et tente de lire `/etc/shadow`, SELinux bloque — même si les permissions Unix le permettraient.

**Les trois modes SELinux :**

```bash
getenforce           # voir le mode actuel
setenforce 1         # Enforcing (bloque et log)
setenforce 0         # Permissive (log sans bloquer) — pour déboguer

# Mode permanent dans /etc/selinux/config
SELINUX=enforcing    # Recommandé en production
SELINUX=permissive   # Pour le debug
SELINUX=disabled     # ❌ Jamais en production
```

**Commandes SELinux essentielles :**

```bash
# Voir le contexte d'un processus
ps -eZ | grep httpd

# Corriger le contexte d'un fichier
restorecon -v /var/www/html/monsite.html

# Récursif
restorecon -Rv /var/www/html/

# Voir les refus SELinux récents
ausearch -m avc -ts recent
sealert -a /var/log/audit/audit.log
```

### AppArmor — L'alternative Ubuntu/Debian

AppArmor est le pendant de SELinux sur Ubuntu/Debian. Il est plus simple à configurer car il utilise des **profils par chemin de fichier** plutôt que des labels sur chaque inode.

**Vérifier le statut AppArmor :**

```bash
sudo apparmor_status
# ou
sudo aa-status
```

Sortie typique :

```
apparmor module is loaded.
26 profiles are loaded.
24 profiles are in enforce mode.
   /usr/bin/man
   /usr/sbin/mysqld
   ...
2 profiles are in complain mode.
```

**Les modes AppArmor :**

| Mode | Comportement |
|------|-------------|
| `enforce` | La politique est appliquée, les violations sont bloquées et logguées |
| `complain` | Les violations sont logguées mais pas bloquées (debug) |
| `disabled` | Profil désactivé |

**Commandes AppArmor :**

```bash
# Voir les profils disponibles
ls /etc/apparmor.d/

# Mettre un profil en mode complain (pour debug)
sudo aa-complain /etc/apparmor.d/usr.sbin.nginx

# Remettre en enforce
sudo aa-enforce /etc/apparmor.d/usr.sbin.nginx

# Recharger un profil après modification
sudo apparmor_parser -r /etc/apparmor.d/usr.sbin.nginx

# Voir les logs AppArmor
sudo dmesg | grep apparmor
sudo journalctl | grep apparmor
```

**Anatomie d'un profil AppArmor :**

```
/etc/apparmor.d/usr.sbin.nginx

#include <tunables/global>

/usr/sbin/nginx {
  #include <abstractions/base>
  #include <abstractions/nameservice>

  capability net_bind_service,     ← peut binder des ports < 1024
  capability setuid,               ← peut changer d'UID
  capability setgid,

  /var/log/nginx/*.log w,          ← peut écrire les logs
  /var/www/html/** r,              ← peut lire le contenu web
  /etc/nginx/** r,                 ← peut lire sa config
  deny /etc/shadow r,              ← ne peut PAS lire shadow
}
```

> 💡 **SELinux vs AppArmor :** SELinux est plus puissant mais plus complexe. AppArmor est plus accessible pour commencer. Sur Ubuntu, AppArmor est actif par défaut — apprends à lire ses logs et à manipuler les profils.

---

## 6. RBAC — Role-Based Access Control

### Concept

Dans le RBAC, on n'assigne pas les permissions **directement aux utilisateurs** mais à des **rôles**. Les utilisateurs reçoivent ensuite un ou plusieurs rôles.

```
Sans RBAC                         Avec RBAC
─────────────────────             ──────────────────────────────
alice → lire DB                   ROLE "DBA" → lire DB
alice → écrire DB                             → écrire DB
alice → sauvegarder DB                        → sauvegarder DB

bob   → lire DB                   ROLE "Dev" → lire DB
bob   → sauvegarder DB                        → lire logs

carol → lire DB                   alice → rôle DBA
carol → lire logs                 bob   → rôle DBA + Dev
                                  carol → rôle Dev
```

**Avantages du RBAC :**
- Gestion centralisée : modifier un rôle = modifier les droits de tous ses membres
- Auditabilité : on sait exactement pourquoi un utilisateur a un accès
- Scalabilité : ajouter 100 utilisateurs = leur assigner un rôle, pas 100 × N permissions

### RBAC sous Linux : `sudo` et les sudoers

`sudo` est l'implémentation RBAC la plus courante sous Linux. Il permet à des utilisateurs non-root d'exécuter des commandes avec les privilèges d'un autre utilisateur (souvent root).

**Le fichier `/etc/sudoers` :**

```bash
# Toujours éditer avec visudo (valide la syntaxe avant de sauvegarder)
sudo visudo
```

**Syntaxe des règles sudoers :**

```
utilisateur  hôte=(utilisateur_cible)  commandes

# Exemples :
root         ALL=(ALL:ALL)  ALL        ← root peut tout faire partout
gilles       ALL=(ALL:ALL)  ALL        ← même chose pour gilles

# Permettre à alice de redémarrer nginx sans mot de passe
alice        ALL=(root)  NOPASSWD: /usr/bin/systemctl restart nginx

# Permettre au groupe devops de gérer les services
%devops      ALL=(root)  /usr/bin/systemctl start *, \
                         /usr/bin/systemctl stop *, \
                         /usr/bin/systemctl restart *, \
                         /usr/bin/systemctl status *

# Permettre à bob de gérer APT sans mot de passe
bob          ALL=(root)  NOPASSWD: /usr/bin/apt update, /usr/bin/apt upgrade
```

**Bonnes pratiques sudoers :**

```bash
# Utiliser des fichiers séparés dans /etc/sudoers.d/ (plus propre)
sudo visudo -f /etc/sudoers.d/devops-team

# Contenu du fichier :
%devops ALL=(root) /usr/bin/systemctl restart nginx, \
                   /usr/bin/systemctl restart postgresql

# Vérifier les droits sudo d'un utilisateur
sudo -l -U alice
sudo -l               # ses propres droits

# Exécuter une commande en tant qu'un autre utilisateur
sudo -u postgres psql
```

> ⚠️ **Règle d'or** : évite `ALL=(ALL:ALL) ALL` pour les comptes non-administrateurs. Soit précis sur les commandes autorisées. Un `NOPASSWD: ALL` pour un développeur est une faille de sécurité majeure.

### RBAC dans le Cloud : AWS IAM

AWS IAM est l'implémentation RBAC du Cloud AWS. Le concept est identique mais l'échelle est différente :

```
Utilisateur IAM / Service (EC2, Lambda...)
        │
        └─ appartient à un Groupe IAM
                │
                └─ Groupe a des Politiques (Policies)
                        │
                        └─ Politique définit :
                              - Actions autorisées (s3:GetObject, ec2:StartInstances...)
                              - Ressources concernées (ARN)
                              - Conditions éventuelles (IP, heure, MFA requis...)
```

**Exemple de politique IAM (JSON) :**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::mon-bucket",
        "arn:aws:s3:::mon-bucket/*"
      ]
    },
    {
      "Effect": "Deny",
      "Action": "s3:DeleteObject",
      "Resource": "*"
    }
  ]
}
```

> 💡 **AWS IAM = RBAC à grande échelle.** Les mêmes principes s'appliquent : moindre privilège, rôles plutôt que permissions directes, séparation des responsabilités.

---

## 7. Les Firewalls — Théorie

### Qu'est-ce qu'un firewall ?

Un firewall (pare-feu) est un système qui **filtre le trafic réseau** selon des règles prédéfinies. Il inspecte les paquets qui entrent, sortent, ou transitent et décide : autoriser ou bloquer.

> **Analogie** : le firewall est le vigile à l'entrée d'une boîte de nuit. Il a une liste de règles ("pas de chaussures de sport", "interdit aux mineurs"). Il vérifie chaque paquet (visiteur) et décide d'autoriser ou non l'entrée.

### Les générations de firewalls

#### Génération 1 — Filtrage de paquets (Packet Filtering)

Opère au niveau des couches 3 (réseau) et 4 (transport) du modèle OSI. Inspecte :
- Adresse IP source/destination
- Port source/destination
- Protocole (TCP, UDP, ICMP)

**Caractéristiques :**
- ✅ Rapide et peu coûteux en ressources
- ❌ Stateless : chaque paquet est analysé indépendamment (ne connaît pas le contexte)
- ❌ Ne comprend pas le contenu applicatif
- Exemples : `iptables` en mode simple, ACL de routeur

#### Génération 2 — Filtrage stateful (Stateful Inspection)

Maintient une **table d'état des connexions actives**. Un paquet est évalué dans le contexte de sa connexion.

```
Table d'état (conntrack) :
192.168.1.10:52341 → 93.184.216.34:443  ESTABLISHED
192.168.1.10:52342 → 8.8.8.8:53         TIME_WAIT
```

**Avantage :** peut autoriser le retour des paquets d'une connexion initiée de l'intérieur sans ouvrir le port explicitement. C'est ainsi que `iptables` avec `conntrack` fonctionne.

#### Génération 3 — Filtrage applicatif (Deep Packet Inspection)

Inspecte le **contenu** des paquets jusqu'à la couche 7 (application). Comprend les protocoles HTTP, DNS, FTP, etc.

**Peut détecter :**
- Un tunnel HTTP qui transporte autre chose que du HTTP
- Des malwares dans le trafic réseau
- Des violations de politique (ex: streaming YouTube pendant les heures de travail)

**Exemples :** Palo Alto, Check Point, Fortinet, pfSense avec plugins

#### Génération 4 — NGFW (Next Generation Firewall)

Combine filtrage stateful + DPI + fonctionnalités supplémentaires :
- **IPS** (Intrusion Prevention System) : détecte et bloque les attaques connues
- **Filtrage d'URL** par catégorie
- **Contrôle des applications** (bloquer Skype, autoriser Teams)
- **Identification utilisateur** (règles basées sur l'identité AD, pas juste l'IP)
- **SSL Inspection** : déchiffre le TLS pour inspecter le contenu

### Les zones réseau

Un firewall segmente le réseau en zones avec des niveaux de confiance différents :

```
Internet (non fiable)
        │
   [Firewall périmétrique]
        │
   ┌────┴──────────────────────────┐
   │                               │
   ▼                               ▼
 DMZ (semi-fiable)            LAN (fiable)
  Serveurs publics             Postes de travail
  (web, mail, DNS)             Serveurs internes
                               Bases de données
```

**DMZ (DeMilitarized Zone)** : zone intermédiaire où sont placés les serveurs accessibles depuis Internet (web, mail). Ils sont isolés du réseau interne : même si un serveur DMZ est compromis, l'attaquant ne peut pas directement accéder au LAN.

### Les politiques de firewall : Whitelist vs Blacklist

| Politique | Principe | Sécurité | Complexité |
|-----------|---------|----------|-----------|
| **Whitelist** (allowlist) | Tout est interdit sauf ce qui est explicitement autorisé | ✅ Maximale | ⚠️ Plus de travail |
| **Blacklist** (denylist) | Tout est autorisé sauf ce qui est explicitement interdit | ❌ Minimale | ✅ Plus simple |

> 🔑 **Règle absolue** : sur tout serveur de production, utilise la politique **whitelist** (default DENY). Tu ouvres explicitement les ports dont tu as besoin, et tout le reste est bloqué.

### Sens du trafic : INPUT, OUTPUT, FORWARD

```
                    ┌────────────────────────────┐
Paquets entrants    │                            │  Paquets sortants
────────────────▶  INPUT                      OUTPUT  ──────────────▶
                    │         [Système]          │
                    │                            │
Paquets en transit  │        FORWARD             │
────────────────────────────────────────────────────────────────────▶
                    └────────────────────────────┘
```

| Chaîne | Trafic concerné |
|--------|----------------|
| `INPUT` | Paquets **destinés** à cette machine |
| `OUTPUT` | Paquets **générés par** cette machine |
| `FORWARD` | Paquets qui **traversent** cette machine (routage) |

---

## 8. iptables — Le firewall historique Linux

### Architecture d'iptables

iptables organise ses règles en **tables** → **chaînes** → **règles** :

```
Tables :
├── filter    ← la table principale (INPUT, OUTPUT, FORWARD)
├── nat       ← traduction d'adresses (PREROUTING, POSTROUTING)
├── mangle    ← modification des paquets
└── raw       ← connexion tracking

Pour chaque table, des chaînes :
└── filter
    ├── INPUT     ← paquets entrants
    ├── OUTPUT    ← paquets sortants
    └── FORWARD   ← paquets routés
```

### Commandes de base iptables

```bash
# Voir les règles actuelles
sudo iptables -L -v -n
sudo iptables -L -v -n --line-numbers   # avec numéros de ligne

# Table nat
sudo iptables -t nat -L -v -n
```

### Politique par défaut (Default Policy)

```bash
# Politique whitelist : tout bloquer par défaut
sudo iptables -P INPUT DROP
sudo iptables -P FORWARD DROP
sudo iptables -P OUTPUT ACCEPT    # on fait confiance aux sorties
```

> ⚠️ **Attention** : si tu appliques `iptables -P INPUT DROP` en SSH sans avoir d'abord autorisé le port 22, tu te déconnectes et tu perds l'accès au serveur. Toujours autoriser SSH avant de mettre le DROP par défaut !

### Règles essentielles

```bash
# Autoriser le loopback (interface lo) — toujours nécessaire
sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -A OUTPUT -o lo -j ACCEPT

# Autoriser les connexions déjà établies (stateful)
sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# Autoriser SSH (port 22)
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# Autoriser HTTP et HTTPS
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# Autoriser ICMP (ping) — pour le diagnostic
sudo iptables -A INPUT -p icmp -j ACCEPT

# Tout bloquer par défaut (doit venir après les règles d'autorisation)
sudo iptables -A INPUT -j DROP
```

### Syntaxe complète d'une règle iptables

```bash
iptables  -A INPUT  -p tcp  -s 192.168.1.0/24  --dport 22  -j ACCEPT
    │       │         │       │                   │            │
    │       │         │       │                   │            └─ Action (ACCEPT, DROP, REJECT, LOG)
    │       │         │       │                   └─ Port destination
    │       │         │       └─ IP/réseau source
    │       │         └─ Protocole (tcp, udp, icmp, all)
    │       └─ Chaîne (INPUT, OUTPUT, FORWARD)
    └─ Commande (-A append, -I insert, -D delete, -R replace)
```

### Règles avancées

```bash
# Limiter les tentatives de connexion SSH (brute force protection)
sudo iptables -A INPUT -p tcp --dport 22 -m state --state NEW \
  -m recent --set --name SSH
sudo iptables -A INPUT -p tcp --dport 22 -m state --state NEW \
  -m recent --update --seconds 60 --hitcount 4 --name SSH -j DROP

# Bloquer une IP spécifique
sudo iptables -A INPUT -s 192.168.1.100 -j DROP

# Autoriser SSH depuis une IP spécifique seulement
sudo iptables -A INPUT -p tcp --dport 22 -s 10.0.0.5 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 22 -j DROP     # bloquer tous les autres

# Logger les paquets bloqués (avant le DROP)
sudo iptables -A INPUT -j LOG --log-prefix "IPT_DROP: " --log-level 4
sudo iptables -A INPUT -j DROP

# Supprimer une règle
sudo iptables -D INPUT -p tcp --dport 80 -j ACCEPT

# Vider toutes les règles (flush)
sudo iptables -F
```

### Persister les règles iptables

Les règles iptables sont **perdues au redémarrage** sans persistance :

```bash
# Sauvegarder les règles
sudo iptables-save > /etc/iptables/rules.v4
sudo ip6tables-save > /etc/iptables/rules.v6

# Restaurer au démarrage avec iptables-persistent
sudo apt install iptables-persistent
# Les règles actuelles sont automatiquement sauvegardées

# Recharger manuellement
sudo iptables-restore < /etc/iptables/rules.v4
```

---

## 9. nftables — Le successeur moderne

### Pourquoi nftables remplace iptables ?

`nftables` est le successeur d'`iptables` intégré au kernel depuis Linux 3.13. Il est le backend par défaut sur Debian 10+ et Ubuntu 20.04+.

**Avantages de nftables :**
- Syntaxe plus claire et cohérente
- Une seule commande (`nft`) au lieu de `iptables`, `ip6tables`, `arptables`, `ebtables`
- Meilleures performances (moins d'appels système)
- Support natif IPv4 et IPv6

```bash
# Voir les règles actuelles
sudo nft list ruleset

# Vérifier si nftables est actif
sudo systemctl status nftables
```

### Syntaxe nftables

```bash
# Créer une table
sudo nft add table inet filter

# Créer des chaînes
sudo nft add chain inet filter input \
  { type filter hook input priority 0 \; policy drop \; }

sudo nft add chain inet filter output \
  { type filter hook output priority 0 \; policy accept \; }

# Ajouter des règles
sudo nft add rule inet filter input iif lo accept
sudo nft add rule inet filter input ct state established,related accept
sudo nft add rule inet filter input tcp dport 22 accept
sudo nft add rule inet filter input tcp dport { 80, 443 } accept
```

**Fichier de configuration :**

```bash
# /etc/nftables.conf
table inet filter {
  chain input {
    type filter hook input priority 0; policy drop;

    iif lo accept
    ct state established,related accept
    tcp dport ssh accept
    tcp dport { http, https } accept
    icmp type echo-request accept
    icmpv6 type echo-request accept
  }

  chain output {
    type filter hook output priority 0; policy accept;
  }

  chain forward {
    type filter hook forward priority 0; policy drop;
  }
}

# Activer
sudo nft -f /etc/nftables.conf
sudo systemctl enable nftables
```

---

## 10. UFW — Interface simplifiée (Ubuntu)

### Qu'est-ce qu'UFW ?

UFW (*Uncomplicated Firewall*) est une interface de configuration simplifiée pour `iptables`/`nftables`. Il est installé par défaut sur Ubuntu mais **désactivé par défaut**.

> 💡 UFW ne remplace pas iptables/nftables — il les configure à ta place avec une syntaxe plus lisible. C'est une couche d'abstraction.

### Commandes UFW essentielles

```bash
# Activer/désactiver UFW
sudo ufw enable
sudo ufw disable

# Voir le statut
sudo ufw status
sudo ufw status verbose
sudo ufw status numbered     # avec numéros pour supprimer facilement

# Politique par défaut (whitelist)
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

### Gérer les règles UFW

```bash
# Autoriser un port
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Autoriser un service par nom (utilise /etc/services)
sudo ufw allow ssh
sudo ufw allow http
sudo ufw allow https

# Autoriser depuis une IP spécifique
sudo ufw allow from 10.0.0.5 to any port 22

# Autoriser un sous-réseau
sudo ufw allow from 192.168.1.0/24

# Autoriser un plage de ports
sudo ufw allow 6000:6007/tcp

# Bloquer
sudo ufw deny 23/tcp          # bloquer telnet

# Supprimer une règle
sudo ufw delete allow 80/tcp
sudo ufw delete 3              # supprimer la règle numéro 3

# Voir les profils d'applications disponibles
sudo ufw app list
sudo ufw allow "Nginx Full"    # HTTP + HTTPS
sudo ufw allow "OpenSSH"
```

### Exemple de configuration UFW complète pour un serveur web

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from 10.0.0.5 to any port 22    # SSH depuis ton IP admin seulement
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
sudo ufw status verbose
```

---

## 11. Firewalls applicatifs (WAF)

### Concept du WAF

Un WAF (*Web Application Firewall*) opère à la **couche 7 (application)** et comprend le protocole HTTP/HTTPS. Il peut détecter et bloquer des attaques applicatives que les firewalls réseau ne voient pas.

**Ce qu'un WAF détecte :**

| Attaque | Description |
|---------|------------|
| **SQL Injection** | `' OR 1=1 --` dans un formulaire |
| **XSS** | `<script>alert('hack')</script>` dans une URL |
| **CSRF** | Requêtes forgées depuis un autre site |
| **Path Traversal** | `../../etc/passwd` dans une URL |
| **RCE** | Tentative d'exécution de commandes via l'application |
| **DDoS applicatif** | Flood de requêtes HTTP valides |

### ModSecurity — WAF open source

ModSecurity est le WAF open source le plus utilisé. Il s'intègre à Apache et Nginx.

```bash
# Installation avec Nginx
sudo apt install libmodsecurity3 libnginx-mod-http-modsecurity

# Activer dans nginx.conf
modsecurity on;
modsecurity_rules_file /etc/nginx/modsec/main.conf;
```

**ModSecurity fonctionne avec des règles OWASP CRS (Core Rule Set) :**

```bash
# Télécharger les règles OWASP CRS
git clone https://github.com/coreruleset/coreruleset.git /etc/nginx/modsec/crs

# Configuration de base
cp /etc/nginx/modsec/crs/crs-setup.conf.example /etc/nginx/modsec/crs/crs-setup.conf
```

> 💡 **En Cloud AWS**, le WAF natif est **AWS WAF** qui s'intègre à CloudFront, ALB, et API Gateway. Il fonctionne sur le même principe mais est managé.

---

## 12. Proxies — Théorie et types

### Qu'est-ce qu'un proxy ?

Un proxy est un **intermédiaire** entre un client et un serveur. Au lieu de communiquer directement, le client envoie ses requêtes au proxy, qui les transmet au serveur (et vice versa).

```
Sans proxy :
Client ──────────────────────────────▶ Serveur

Avec proxy :
Client ──────▶ [Proxy] ──────────────▶ Serveur
Client ◀──────           ◀────────────  Serveur
```

### Proxy Forward (proxy classique)

Le proxy forward est côté **client**. Le client est configuré pour envoyer ses requêtes au proxy, qui les transmet vers Internet.

```
LAN (clients)               Internet
┌──────────┐
│ Client A │──┐
└──────────┘  │    ┌───────────────┐     ┌──────────────┐
              ├───▶│  Proxy Forward │────▶│  Serveur Web │
┌──────────┐  │    └───────────────┘     └──────────────┘
│ Client B │──┘
└──────────┘
```

**Usages :**
- Contrôle d'accès (filtrer les sites Web dans une entreprise)
- Cache (stocker les pages fréquemment demandées)
- Anonymisation (masquer l'IP des clients)
- Compression du trafic

**Exemples :** Squid, Privoxy

```bash
# Configuration Squid (proxy forward)
sudo apt install squid

# /etc/squid/squid.conf (extrait)
http_port 3128
acl localnet src 192.168.1.0/24
http_access allow localnet
http_access deny all

# Sites bloqués
acl sites_bloques dstdomain .facebook.com .youtube.com
http_access deny sites_bloques
```

### Proxy Transparent

Un proxy transparent intercepte le trafic **sans que le client le sache** (pas de configuration côté client). Le routeur redirige le trafic HTTP vers le proxy.

```bash
# Rediriger le trafic HTTP vers Squid via iptables
sudo iptables -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to-port 3128
```

**Usage :** filtrage de contenu dans les entreprises, FAI, hôtels.

---

## 13. Proxy Firewall

### Concept

Le proxy firewall combine les fonctions de proxy et de firewall applicatif. Il ne laisse **jamais** le trafic passer directement entre le client et le serveur — il reçoit la connexion, l'inspecte, et ouvre une nouvelle connexion vers la destination.

```
                              [Proxy Firewall]
                         ┌────────────────────────┐
Client ──connexion 1──▶  │  ① Inspecte la requête │
                         │  ② Applique les règles  │  ──connexion 2──▶ Serveur
                         │  ③ Si OK, crée nouvelle│
                         │     connexion           │
                         └────────────────────────┘
```

**Avantage clé** : comme il ouvre une nouvelle connexion, il peut :
- Inspecter le contenu complet (même chiffré avec SSL inspection)
- Bloquer les commandes dangereuses (ex: bloquer `DELETE` en HTTP mais autoriser `GET`)
- Masquer les erreurs du serveur réel

**Exemple avec Nginx en mode proxy firewall :**

```nginx
server {
    listen 80;
    server_name monapp.com;

    # Bloquer les méthodes HTTP dangereuses
    if ($request_method !~ ^(GET|HEAD|POST)$) {
        return 405;
    }

    # Bloquer les User-Agents suspects
    if ($http_user_agent ~* (sqlmap|nikto|nmap)) {
        return 403;
    }

    # Limiter le rate (anti-DDoS)
    limit_req zone=api burst=20 nodelay;

    location / {
        proxy_pass http://backend:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

### Fail2Ban — Protection dynamique

Fail2Ban surveille les logs et **bannit automatiquement** les IPs qui effectuent trop de tentatives échouées. Il modifie les règles iptables/nftables en temps réel.

```bash
sudo apt install fail2ban

# Configurer (ne jamais modifier jail.conf directement)
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo nano /etc/fail2ban/jail.local
```

**Configuration jail.local :**

```ini
[DEFAULT]
# Durée du ban (en secondes)
bantime  = 3600     # 1 heure
# Fenêtre d'observation
findtime = 600      # 10 minutes
# Nombre de tentatives avant ban
maxretry = 5

[sshd]
enabled = true
port    = ssh
logpath = /var/log/auth.log

[nginx-http-auth]
enabled  = true
logpath  = /var/log/nginx/error.log
maxretry = 5
```

**Commandes Fail2Ban :**

```bash
# Voir le statut
sudo fail2ban-client status
sudo fail2ban-client status sshd

# Débanner une IP manuellement
sudo fail2ban-client set sshd unbanip 192.168.1.100

# Voir les IPs bannies
sudo fail2ban-client get sshd banip
```

---

## 14. Reverse Proxy

### Concept

Un reverse proxy est côté **serveur**. Les clients ne communiquent jamais directement avec les serveurs applicatifs — ils passent toujours par le reverse proxy.

```
                              [Reverse Proxy]
Internet        ┌───────────────────────────────────┐
Clients ───────▶│  Reçoit toutes les connexions     │────▶ Serveur App 1
                │  Distribue selon les règles       │────▶ Serveur App 2
                │  (nom de domaine, URL, etc.)      │────▶ Serveur App 3
                └───────────────────────────────────┘
```

**Ce que fait un reverse proxy :**

| Fonction | Explication |
|----------|-------------|
| **Load Balancing** | Distribue le trafic entre plusieurs serveurs |
| **SSL Termination** | Gère le TLS/SSL, les serveurs backend reçoivent du HTTP |
| **Caching** | Met en cache les réponses statiques |
| **Compression** | Compresse les réponses (gzip) |
| **Sécurité** | Masque l'architecture interne, filtre les requêtes |
| **VirtualHost** | Un seul IP, plusieurs sites selon le nom de domaine |

### Nginx comme reverse proxy

```nginx
# /etc/nginx/sites-available/monapp.conf

# Groupe de serveurs backend (load balancing)
upstream backend_api {
    server 10.0.0.10:8080 weight=3;    # reçoit 3x plus de trafic
    server 10.0.0.11:8080 weight=1;
    server 10.0.0.12:8080 backup;     # utilisé seulement si les autres tombent
}

server {
    listen 443 ssl;
    server_name api.monapp.com;

    ssl_certificate /etc/ssl/certs/monapp.crt;
    ssl_certificate_key /etc/ssl/private/monapp.key;

    # Hardening SSL
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    # Headers de sécurité
    add_header Strict-Transport-Security "max-age=31536000" always;
    add_header X-Frame-Options DENY;
    add_header X-Content-Type-Options nosniff;

    location / {
        proxy_pass http://backend_api;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

# Rediriger HTTP vers HTTPS
server {
    listen 80;
    server_name api.monapp.com;
    return 301 https://$host$request_uri;
}
```

---

## 15. SSH — Sécurité en profondeur

### Sécuriser le daemon SSH (`/etc/ssh/sshd_config`)

SSH est la porte d'entrée principale de tes serveurs. Sa configuration mérite une attention particulière.

```bash
sudo nano /etc/ssh/sshd_config
```

**Configuration sécurisée complète :**

```ini
# ── Port ──────────────────────────────────────────────────────
# Changer le port par défaut (sécurité par obscurité, mais ça réduit le bruit)
Port 2222

# ── Authentification ──────────────────────────────────────────
# Désactiver l'authentification par mot de passe (clé SSH uniquement)
PasswordAuthentication no
PubkeyAuthentication yes

# Désactiver root login
PermitRootLogin no

# Désactiver les méthodes d'auth faibles
ChallengeResponseAuthentication no
UsePAM yes

# Nombre de tentatives d'auth avant déconnexion
MaxAuthTries 3

# Temps maximum pour s'authentifier
LoginGraceTime 20

# ── Sessions ──────────────────────────────────────────────────
# Déconnecter les sessions inactives après 5 minutes
ClientAliveInterval 300
ClientAliveCountMax 2

# Nombre de connexions simultanées non authentifiées
MaxStartups 10:30:60

# Désactiver le forwarding inutile
AllowAgentForwarding no
X11Forwarding no
AllowTcpForwarding no     # Mettre à 'yes' si tu utilises des tunnels SSH

# ── Restriction d'accès ───────────────────────────────────────
# Autoriser seulement certains utilisateurs
AllowUsers gilles deploy-user

# Ou certains groupes
AllowGroups sysadmin devops

# ── Crypto ────────────────────────────────────────────────────
# N'utiliser que des algorithmes modernes
KexAlgorithms curve25519-sha256,diffie-hellman-group16-sha512
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com
MACs hmac-sha2-256-etm@openssh.com,hmac-sha2-512-etm@openssh.com
```

**Après modification :**

```bash
# Valider la configuration avant de recharger (évite de se bloquer)
sudo sshd -t

# Recharger
sudo systemctl reload sshd
```

> ⚠️ **Règle critique** : avant d'appliquer `PasswordAuthentication no`, vérifie que ta clé SSH fonctionne dans une **session séparée**. Si tu fermes ta session avant de vérifier et que la clé ne fonctionne pas, tu perds l'accès au serveur.

### Sécuriser `~/.ssh/authorized_keys`

```bash
# Options disponibles dans authorized_keys
# Format : options clé-publique commentaire

# Restreindre à une IP source
from="10.0.0.5" ssh-ed25519 AAAA... gilles@workstation

# Interdire les tunnels et le forwarding
no-port-forwarding,no-X11-forwarding,no-agent-forwarding ssh-ed25519 AAAA...

# Forcer l'exécution d'une commande spécifique (pour les clés de déploiement)
command="/usr/bin/rsync --server ..." ssh-ed25519 AAAA... deploy-key
```

---

## 16. Chiffrement — Les bases indispensables

### Chiffrement symétrique vs asymétrique

| | Symétrique | Asymétrique |
|-|-----------|-------------|
| **Clés** | Une seule clé partagée | Paire clé publique / clé privée |
| **Vitesse** | ✅ Rapide | ❌ Lent |
| **Problème** | Comment partager la clé secrètement ? | Résout le problème d'échange de clé |
| **Usages** | Chiffrement bulk des données | Échange de clé, signatures |
| **Exemples** | AES-256, ChaCha20 | RSA, ECDSA, Ed25519 |

**TLS combine les deux :**

```
Phase 1 (asymétrique) : échange de clé via RSA/ECDH
Phase 2 (symétrique)  : chiffrement du trafic avec AES (plus rapide)
```

### TLS/SSL — Le fondement de HTTPS

TLS (*Transport Layer Security*) chiffre les communications réseau. "SSL" est l'ancien nom, souvent encore utilisé par habitude.

**Versions :**

| Version | Statut |
|---------|--------|
| SSL 2.0 | ❌ Obsolète, vulnérable |
| SSL 3.0 | ❌ Obsolète (POODLE) |
| TLS 1.0 | ❌ Déprécié |
| TLS 1.1 | ❌ Déprécié |
| TLS 1.2 | ✅ Encore utilisé |
| TLS 1.3 | ✅ Standard actuel, recommandé |

```bash
# Vérifier le TLS d'un serveur
openssl s_client -connect monsite.com:443 -tls1_3

# Ou avec nmap
nmap --script ssl-enum-ciphers -p 443 monsite.com
```

### Certificats — La chaîne de confiance

```
Root CA (Autorité de Certification Racine)
    └── Intermediate CA
            └── Ton certificat (leaf certificate)
                    ├── Domaine : monsite.com
                    ├── Valide jusqu'au : 2025-12-31
                    └── Signé par : Intermediate CA
```

**Let's Encrypt + Certbot (certificats gratuits) :**

```bash
sudo apt install certbot python3-certbot-nginx

# Obtenir un certificat
sudo certbot --nginx -d monsite.com -d www.monsite.com

# Renouvellement automatique (déjà configuré par certbot)
sudo certbot renew --dry-run

# Vérifier les certificats installés
sudo certbot certificates
```

### Chiffrer des fichiers avec GPG

```bash
# Générer une paire de clés GPG
gpg --gen-key

# Chiffrer un fichier (pour un destinataire)
gpg --encrypt --recipient alice@example.com fichier.txt

# Déchiffrer
gpg --decrypt fichier.txt.gpg > fichier.txt

# Chiffrer avec un mot de passe symétrique
gpg --symmetric fichier_secret.txt

# Signer un fichier (intégrité)
gpg --sign fichier.txt
gpg --verify fichier.txt.gpg
```

---

## 17. Surveillance et détection

### Les logs de sécurité à surveiller

```bash
/var/log/auth.log          # Authentifications SSH, sudo, PAM
/var/log/syslog            # Logs système généraux
/var/log/kern.log          # Logs kernel (iptables LOG)
/var/log/nginx/access.log  # Accès web
/var/log/nginx/error.log   # Erreurs web
/var/log/fail2ban.log      # Bans Fail2Ban
/var/log/audit/audit.log   # Logs SELinux/auditd
```

### Surveiller les connexions SSH en temps réel

```bash
# Voir les tentatives de connexion SSH
sudo tail -f /var/log/auth.log | grep sshd

# Compter les tentatives par IP
sudo grep "Failed password" /var/log/auth.log | \
  awk '{print $11}' | sort | uniq -c | sort -rn | head -20

# Voir qui est connecté en ce moment
who
w
last        # historique des connexions
lastb       # tentatives échouées
```

### `auditd` — Audit système complet

```bash
sudo apt install auditd

# Surveiller l'accès à un fichier sensible
sudo auditctl -w /etc/passwd -p rwa -k passwd_changes
sudo auditctl -w /etc/sudoers -p rwa -k sudoers_changes

# Lire les logs d'audit
sudo ausearch -k passwd_changes
sudo aureport --auth          # rapport d'authentification
sudo aureport --failed        # actions échouées
```

### `netstat` / `ss` — Surveiller les connexions réseau

```bash
# Ports en écoute
sudo ss -tlnp
sudo netstat -tlnp

# Toutes les connexions établies
sudo ss -anp
sudo ss -anp | grep ESTABLISHED

# Connexions par IP
sudo ss -anp | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -rn

# Processus qui écoute sur un port
sudo ss -tlnp | grep :80
sudo fuser 80/tcp
```

### Détecter les intrusions : outils clés

```bash
# rkhunter : détection de rootkits
sudo apt install rkhunter
sudo rkhunter --update
sudo rkhunter --check

# chkrootkit : autre outil de détection
sudo apt install chkrootkit
sudo chkrootkit

# Lynis : audit de sécurité complet
sudo apt install lynis
sudo lynis audit system
```

---

## 18. Checklist sécurité d'un nouveau serveur

### ✅ Étape 1 — Accès et authentification

```
□ Créer un utilisateur admin non-root
□ Lui donner accès sudo (groupe sudo ou règle sudoers)
□ Copier la clé SSH publique sur le serveur
□ Tester la connexion SSH par clé depuis un autre terminal
□ Désactiver PasswordAuthentication dans sshd_config
□ Désactiver PermitRootLogin dans sshd_config
□ Recharger sshd (sudo systemctl reload sshd)
□ Vérifier que la connexion fonctionne toujours
```

### ✅ Étape 2 — Mise à jour système

```
□ sudo apt update && sudo apt upgrade --dry-run
□ Analyser les paquets à mettre à jour
□ sudo apt upgrade
□ Redémarrer si le kernel a changé
```

### ✅ Étape 3 — Firewall

```
□ sudo ufw default deny incoming
□ sudo ufw default allow outgoing
□ sudo ufw allow from [TON_IP_ADMIN] to any port 22
□ Ajouter les autres ports nécessaires (80, 443...)
□ sudo ufw enable
□ sudo ufw status verbose — vérifier les règles
```

### ✅ Étape 4 — Fail2Ban

```
□ sudo apt install fail2ban
□ Créer /etc/fail2ban/jail.local
□ Configurer la jail sshd
□ sudo systemctl enable --now fail2ban
□ sudo fail2ban-client status sshd
```

### ✅ Étape 5 — Hardening système

```
□ Désactiver les services inutiles (sudo systemctl list-units --type=service)
□ Vérifier les ports en écoute (sudo ss -tlnp)
□ Activer les mises à jour de sécurité automatiques (unattended-upgrades)
□ Configurer logrotate pour les logs
□ Installer et configurer auditd
□ Installer rkhunter, lancer un premier scan de référence
```

### ✅ Étape 6 — Monitoring

```
□ Configurer les alertes email pour les connexions root/sudo
□ Surveiller /var/log/auth.log régulièrement
□ Planifier des scans Lynis réguliers (cron)
□ Mettre en place un système de centralisation de logs (si disponible)
```

### ✅ Questions à se poser avant d'ouvrir un port

```
1. Ce service a-t-il vraiment besoin d'être accessible depuis Internet ?
2. Peut-on restreindre à des IPs spécifiques ?
3. Le service tourne-t-il avec le moindre privilège possible ?
4. Le service est-il à jour (pas de CVE connues) ?
5. Y a-t-il un log des accès ?
6. Y a-t-il une protection anti-brute-force ?
```

---

## 19. Glossaire

| Terme | Définition |
|-------|-----------|
| **AAA** | Authentication, Authorization, Accounting — les trois piliers du contrôle d'accès |
| **ACL** | Access Control List — liste définissant qui peut accéder à quoi |
| **AppArmor** | Système MAC basé sur des profils par chemin de fichier (Ubuntu/Debian) |
| **Audit** | Enregistrement des actions pour la traçabilité et la détection d'incidents |
| **Authentification** | Processus de vérification de l'identité d'un utilisateur ou système |
| **Autorisation** | Processus de vérification des droits d'accès après authentification |
| **CA** | Certificate Authority — entité qui émet et signe des certificats numériques |
| **CIA** | Confidentiality, Integrity, Availability — les trois piliers de la sécurité |
| **Conntrack** | Connection Tracking — suivi de l'état des connexions dans iptables |
| **CVE** | Common Vulnerabilities and Exposures — base de données des vulnérabilités connues |
| **DAC** | Discretionary Access Control — le propriétaire décide des permissions |
| **DDoS** | Distributed Denial of Service — attaque par saturation distribuée |
| **DMZ** | DeMilitarized Zone — zone réseau semi-protégée pour les serveurs publics |
| **DPI** | Deep Packet Inspection — inspection du contenu applicatif des paquets |
| **Ed25519** | Algorithme de cryptographie asymétrique recommandé pour SSH |
| **Fail2Ban** | Outil qui bannit automatiquement les IPs malveillantes via les logs |
| **GPG** | GNU Privacy Guard — outil de chiffrement et signature basé sur OpenPGP |
| **IPS** | Intrusion Prevention System — système de détection et blocage d'intrusions |
| **iptables** | Firewall historique Linux (interface userspace pour Netfilter) |
| **MAC** | Mandatory Access Control — le système impose les permissions, pas le propriétaire |
| **MFA** | Multi-Factor Authentication — authentification à plusieurs facteurs |
| **ModSecurity** | WAF open source intégrable à Nginx/Apache |
| **NGFW** | Next Generation Firewall — firewall avec DPI, IPS, et contrôle applicatif |
| **nftables** | Successeur moderne d'iptables (Linux 3.13+) |
| **PAM** | Pluggable Authentication Modules — couche d'abstraction de l'authentification Linux |
| **PPA** | Personal Package Archive — dépôt non-officiel Ubuntu |
| **Proxy** | Intermédiaire entre un client et un serveur |
| **RBAC** | Role-Based Access Control — permissions assignées à des rôles, pas aux individus |
| **Reverse Proxy** | Proxy côté serveur — reçoit les connexions à la place des serveurs applicatifs |
| **Salt** | Valeur aléatoire ajoutée au mot de passe avant hashage pour prévenir les rainbow tables |
| **SELinux** | Security-Enhanced Linux — système MAC développé par la NSA |
| **SGID** | Set Group ID — bit spécial qui fait hériter le groupe aux fichiers créés |
| **SUID** | Set User ID — bit spécial qui exécute un binaire avec les droits de son propriétaire |
| **Stateful** | Inspection de paquets tenant compte de l'état de la connexion |
| **Sticky Bit** | Bit spécial qui empêche les utilisateurs de supprimer les fichiers des autres |
| **TLS** | Transport Layer Security — protocole de chiffrement des communications réseau |
| **UFW** | Uncomplicated Firewall — interface simplifiée pour iptables/nftables |
| **WAF** | Web Application Firewall — firewall applicatif spécialisé HTTP/HTTPS |
| **Whitelist** | Politique "tout interdit sauf ce qui est explicitement autorisé" |
| **XSS** | Cross-Site Scripting — injection de scripts malveillants dans des pages web |

---

*Document généré pour la formation Cloud/AWS — Phase Linux SysAdmin*  
*Périmètre : Sécurité Linux & Réseau — Fondamentaux à Intermédiaire*
