# Module 3 — Docker Networking
### Partie Exercices

> Complète la partie théorie (`module3-theorie.md`) avant de faire ces exercices.
> Prérequis : Docker installé (`docker --version`)
> Durée estimée : 2h à 2h30

---

## Exercice 1 — Inspecter le réseau Docker par défaut

**Objectif** : observer l'infrastructure réseau que Docker crée sur ton hôte et comprendre comment elle fonctionne.

```bash
# 1. Voir l'interface docker0 créée par Docker
ip addr show docker0

# 2. Voir tous les réseaux Docker existants
docker network ls

# 3. Lancer un conteneur simple
docker run -d --name test-reseau nginx

# 4. Voir les interfaces veth créées
ip link show | grep veth

# 5. Inspecter le réseau bridge par défaut
docker network inspect bridge

# 6. Voir l'IP attribuée au conteneur
docker inspect test-reseau \
  --format '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}'

# 7. Depuis l'hôte, pinguer le conteneur directement
ping -c 3 $(docker inspect test-reseau \
  --format '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}')

# 8. Voir les règles NAT créées par Docker
sudo iptables -t nat -L DOCKER -n --line-numbers

# 9. Nettoyer
docker stop test-reseau && docker rm test-reseau
```

**Questions :**

1. Quelle est l'adresse IP et le masque CIDR de l'interface `docker0` ?
2. Combien d'interfaces veth vois-tu après avoir lancé le conteneur ? Que représentent-elles ?
3. Quelle IP a été attribuée au conteneur ? Dans quel réseau CIDR est-elle ?
4. Dans `docker network inspect bridge`, combien de conteneurs sont listés sous "Containers" ?
5. Quelle règle iptables Docker a-t-il créée pour ce conteneur ?

<details>
<summary>Voir les réponses attendues</summary>

1. `172.17.0.1/16` — Docker utilise le réseau `172.17.0.0/16` par défaut pour le bridge.
2. Une paire veth par conteneur actif — une extrémité sur l'hôte (visible), l'autre dans le conteneur (nommée `eth0` à l'intérieur).
3. Quelque chose comme `172.17.0.2` — première IP disponible dans le réseau bridge.
4. Un seul conteneur (`test-reseau`) doit apparaître avec son IP et son adresse MAC.
5. Une règle `DNAT` ou `MASQUERADE` pour le réseau `172.17.0.0/16` — elle permet aux conteneurs d'accéder à Internet via l'IP de l'hôte.

</details>

---

## Exercice 2 — Bridge par défaut vs réseau personnalisé

**Objectif** : voir concrètement pourquoi le bridge par défaut ne permet pas la résolution DNS et pourquoi les réseaux personnalisés sont nécessaires.

```bash
# PARTIE A — Bridge par défaut (sans DNS)

# 1. Lancer deux conteneurs sur le bridge par défaut
docker run -d --name conteneur-a alpine sleep 3600
docker run -d --name conteneur-b alpine sleep 3600

# 2. Depuis conteneur-a, essayer de pinguer conteneur-b par NOM
docker exec conteneur-a ping -c 2 conteneur-b
# → Que se passe-t-il ?

# 3. Récupérer l'IP de conteneur-b
IP_B=$(docker inspect conteneur-b \
  --format '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}')
echo "IP de conteneur-b : $IP_B"

# 4. Pinguer par IP — ça doit fonctionner
docker exec conteneur-a ping -c 2 $IP_B

# 5. Voir le /etc/resolv.conf dans conteneur-a
docker exec conteneur-a cat /etc/resolv.conf

# PARTIE B — Réseau personnalisé (avec DNS)

# 6. Créer un réseau personnalisé
docker network create reseau-custom

# 7. Lancer deux nouveaux conteneurs sur ce réseau
docker run -d --name app --network reseau-custom alpine sleep 3600
docker run -d --name db  --network reseau-custom alpine sleep 3600

# 8. Depuis app, pinguer db par NOM
docker exec app ping -c 2 db
# → Que se passe-t-il maintenant ?

# 9. Voir le /etc/resolv.conf dans app
docker exec app cat /etc/resolv.conf
# → Note le serveur DNS 127.0.0.11

# 10. Résoudre le nom db explicitement
docker exec app nslookup db

# 11. Nettoyer
docker stop conteneur-a conteneur-b app db
docker rm conteneur-a conteneur-b app db
docker network rm reseau-custom
```

**Questions :**

1. Qu'est-ce qui se passe quand tu essaies `ping conteneur-b` depuis le bridge par défaut ?
2. Quelle est la différence dans `/etc/resolv.conf` entre un conteneur sur le bridge par défaut et un conteneur sur un réseau personnalisé ?
3. Quelle adresse IP est le serveur DNS Docker sur le réseau personnalisé ?
4. Pourquoi est-il dangereux de se baser sur les IPs Docker plutôt que sur les noms ?

<details>
<summary>Voir les réponses attendues</summary>

1. Erreur `ping: bad address 'conteneur-b'` ou `Name or service not known` — le DNS n'est pas actif sur le bridge par défaut.
2. Sur le bridge par défaut : nameserver pointe vers le DNS de l'hôte. Sur un réseau personnalisé : `nameserver 127.0.0.11` — le DNS Docker embarqué.
3. `127.0.0.11` — c'est le resolver DNS interne de Docker, activé uniquement sur les réseaux personnalisés.
4. Les IPs Docker sont attribuées dynamiquement et changent à chaque redémarrage du conteneur. Si ton app hardcode `172.18.0.3`, elle cassera dès que le conteneur est recréé. Les noms DNS sont stables.

</details>

---

## Exercice 3 — Les modes réseau Docker

**Objectif** : tester les différents modes réseau et observer leurs différences concrètes.

```bash
# MODE BRIDGE (défaut)
docker run --rm --name test-bridge alpine ip addr show
# → eth0 avec une IP en 172.17.x.x

# MODE HOST
docker run --rm --network host alpine ip addr show
# → voit TOUTES les interfaces de l'hôte (eth0, wlan0, docker0...)

# MODE NONE
docker run --rm --network none alpine ip addr show
# → uniquement lo (loopback)

# Tester l'accès Internet dans chaque mode
docker run --rm alpine ping -c 2 8.8.8.8           # bridge → ✅
docker run --rm --network host alpine ping -c 2 8.8.8.8   # host → ✅
docker run --rm --network none alpine ping -c 2 8.8.8.8   # none → ❌

# Comparer les interfaces vues
echo "=== BRIDGE ==="
docker run --rm alpine ip route show

echo "=== HOST ==="
docker run --rm --network host alpine ip route show

echo "=== NONE ==="
docker run --rm --network none alpine ip route show
```

**Questions :**

1. En mode `host`, combien d'interfaces réseau le conteneur voit-il ? Pourquoi ?
2. En mode `none`, quelle interface réseau reste disponible ? À quoi sert-elle ?
3. En mode `bridge`, quelle est la passerelle par défaut du conteneur ?
4. Quel mode utiliserais-tu pour un conteneur qui traite des données sensibles et ne doit jamais accéder à Internet ?
5. Quel mode utiliserais-tu pour un serveur de monitoring qui a besoin d'accéder à toutes les interfaces réseau de l'hôte ?

<details>
<summary>Voir les réponses attendues</summary>

1. En mode `host`, le conteneur voit **toutes** les interfaces de l'hôte — eth0, wlan0, docker0, etc. Il n'y a plus d'isolation réseau, le conteneur partage l'espace réseau de l'hôte.
2. Uniquement `lo` (loopback `127.0.0.1`). Elle permet aux processus à l'intérieur du conteneur de se parler entre eux, mais aucune communication externe n'est possible.
3. La passerelle est `172.17.0.1` — c'est l'interface `docker0` sur l'hôte, qui fait NAT vers Internet.
4. **Mode `none`** — isolation réseau totale, aucun risque d'exfiltration de données.
5. **Mode `host`** — accès direct à toutes les interfaces, idéal pour les outils de monitoring réseau (Prometheus node exporter, Netdata, etc.).

</details>

---

## Exercice 4 — DNS et service discovery

**Objectif** : maîtriser le DNS Docker et les alias réseau pour construire une vraie architecture multi-conteneurs.

```bash
# 1. Créer un réseau pour l'application
docker network create app-network

# 2. Lancer une "base de données" (Redis comme exemple simple)
docker run -d \
  --name ma-redis \
  --network app-network \
  --network-alias cache \
  --network-alias redis \
  redis:alpine

# 3. Lancer un conteneur "application"
docker run -d \
  --name mon-app \
  --network app-network \
  alpine sleep 3600

# 4. Depuis mon-app, tester la résolution DNS avec différents noms
docker exec mon-app nslookup ma-redis
docker exec mon-app nslookup cache
docker exec mon-app nslookup redis

# 5. Tester la connectivité TCP vers Redis (port 6379)
docker exec mon-app nc -zv ma-redis 6379
docker exec mon-app nc -zv cache 6379
docker exec mon-app nc -zv redis 6379

# 6. Voir le serveur DNS utilisé
docker exec mon-app cat /etc/resolv.conf

# 7. Connecter mon-app à un deuxième réseau
docker network create autre-network
docker network connect autre-network mon-app

# Voir les IPs du conteneur sur les deux réseaux
docker inspect mon-app \
  --format '{{range $net, $conf := .NetworkSettings.Networks}}{{$net}}: {{$conf.IPAddress}}{{"\n"}}{{end}}'

# 8. Nettoyer
docker stop ma-redis mon-app
docker rm ma-redis mon-app
docker network rm app-network autre-network
```

**Questions :**

1. Les trois noms `ma-redis`, `cache` et `redis` pointent-ils tous vers la même IP ?
2. Pourquoi Redis écoute-t-il sur le port 6379 et non sur un port standard comme 80 ou 443 ?
3. Quand tu connectes `mon-app` à `autre-network`, combien d'IPs a-t-il maintenant ?
4. Dans un vrai projet, pourquoi utiliserait-on `--network-alias` plutôt que le nom du conteneur ?

<details>
<summary>Voir les réponses attendues</summary>

1. Oui — les trois noms sont des alias du même conteneur et résolvent vers la même IP Docker.
2. Redis utilise le port 6379 par convention — c'est son port par défaut défini dans sa config. Les ports < 1024 sont réservés aux services système (root requis). Les bases de données et caches utilisent des ports dans la plage 1024-65535.
3. Deux IPs — une par réseau. Un conteneur connecté à N réseaux Docker a N interfaces réseau et N adresses IP.
4. `--network-alias` découple le nom du service du nom du conteneur. Si tu renames ou recrées le conteneur, l'alias reste stable. C'est exactement comme un enregistrement DNS CNAME — le nom logique est stable même si l'implémentation change.

</details>

---

## Exercice 5 — Docker Compose networking

**Objectif** : construire une architecture frontend + API + BDD avec Docker Compose et vérifier l'isolation réseau entre les couches.

```bash
# 1. Créer le répertoire du projet
mkdir compose-network-lab && cd compose-network-lab

# 2. Créer le fichier compose.yml
cat > compose.yml << 'EOF'
services:
  frontend:
    image: nginx:alpine
    ports:
      - "8080:80"
    networks:
      - web
    depends_on:
      - api

  api:
    image: python:3.11-alpine
    command: python -m http.server 3000
    networks:
      - web
      - backend
    depends_on:
      - database

  database:
    image: redis:alpine
    networks:
      - backend

networks:
  web:
  backend:
EOF

# 3. Lancer l'architecture
docker compose up -d

# 4. Voir les réseaux créés
docker network ls | grep compose

# 5. Vérifier quels conteneurs sont sur quels réseaux
docker network inspect compose-network-lab_web
docker network inspect compose-network-lab_backend

# 6. Tester la communication entre services

# frontend → api (doit fonctionner — même réseau "web")
docker compose exec frontend wget -qO- http://api:3000

# api → database (doit fonctionner — même réseau "backend")
docker compose exec api nc -zv database 6379

# frontend → database (doit ÉCHOUER — réseaux différents)
docker compose exec frontend nc -zv database 6379

# 7. Voir les IPs de chaque service
docker compose exec frontend cat /etc/hosts
docker compose exec api ip addr show

# 8. Depuis l'hôte — tester l'accès public
curl http://localhost:8080    # ✅ via port publié
curl http://localhost:3000    # ❌ pas de port publié pour api

# 9. Voir les logs réseau
docker compose logs --tail 20

# 10. Nettoyer
docker compose down
cd .. && rm -rf compose-network-lab
```

**Questions :**

1. Quel nom Docker donne-t-il automatiquement aux réseaux créés par Compose ?
2. Pourquoi `frontend → database` échoue-t-il ? Comment le résoudre si c'était nécessaire ?
3. Pourquoi `curl http://localhost:3000` échoue depuis l'hôte même si `api` tourne sur le port 3000 ?
4. Dans une architecture AWS, quelle ressource jouerait le rôle du réseau `web` et laquelle jouerait le rôle du réseau `backend` ?

<details>
<summary>Voir les réponses attendues</summary>

1. `[nom_du_répertoire]_[nom_du_réseau]` — ici `compose-network-lab_web` et `compose-network-lab_backend`.
2. `frontend` et `database` sont sur des réseaux différents (`web` vs `backend`) — Docker les isole. Pour les connecter, il faudrait ajouter `database` au réseau `web` ou `frontend` au réseau `backend` dans le `compose.yml`. En pratique, on ne le ferait PAS — c'est une règle de sécurité volontaire.
3. Le port 3000 de `api` n'est pas publié avec `ports:` dans le compose.yml — il est uniquement accessible depuis les autres conteneurs sur le même réseau Docker. Pour l'exposer, il faudrait ajouter `ports: - "3000:3000"` dans la config de `api`.
4. Le réseau `web` correspond au **subnet public AWS** (accessible depuis Internet via ALB). Le réseau `backend` correspond au **subnet privé AWS** (accessible uniquement depuis les services internes, jamais depuis Internet directement).

</details>

---

## Exercice 6 — Exposition de ports et sécurité

**Objectif** : comprendre les implications sécurité de `-p` et la différence entre binding sur `0.0.0.0` et `127.0.0.1`.

```bash
# 1. Lancer un conteneur avec port publié sur toutes les interfaces
docker run -d --name web-public -p 8080:80 nginx

# 2. Vérifier sur quelle interface il écoute
ss -tlnp | grep 8080
# → 0.0.0.0:8080  (accessible depuis partout)

# 3. Tester l'accès
curl http://localhost:8080           # depuis l'hôte ✅
curl http://192.168.178.X:8080       # depuis le réseau local ✅

# 4. Lancer un conteneur avec port publié UNIQUEMENT sur loopback
docker run -d --name web-local -p 127.0.0.1:9090:80 nginx

# 5. Vérifier
ss -tlnp | grep 9090
# → 127.0.0.1:9090  (loopback uniquement)

# 6. Tester l'accès
curl http://localhost:9090           # depuis l'hôte ✅
curl http://192.168.178.X:9090       # depuis le réseau local ❌

# 7. Scanner les ports ouverts avec nmap
nmap -p 8080,9090 localhost

# 8. Port aléatoire
docker run -d --name web-random -p 80 nginx
docker port web-random
# → quel port l'hôte a-t-il choisi ?
nmap -p $(docker port web-random | cut -d: -f2) localhost

# 9. EXPOSE vs -p — démonstration
# Lancer sans -p
docker run -d --name web-expose nginx
# nginx EXPOSE 80 dans son Dockerfile mais sans -p rien n'est accessible
curl http://localhost:80     # ❌ connexion refusée depuis l'hôte
# Mais accessible depuis d'autres conteneurs Docker
docker run --rm --network container:web-expose alpine \
  wget -qO- http://localhost:80   # ✅ accès depuis un autre conteneur

# 10. Nettoyer
docker stop web-public web-local web-random web-expose
docker rm web-public web-local web-random web-expose
```

**Questions :**

1. Quelle est la différence de sécurité entre `-p 8080:80` et `-p 127.0.0.1:8080:80` ?
2. Un service lancé avec `-p 8080:80` sur une EC2 AWS est-il accessible depuis Internet ? Qu'est-ce qui le protège ?
3. Pourquoi utiliserait-on `-p 127.0.0.1:5432:5432` pour une base de données de développement ?
4. Dans l'étape 9, `web-expose` expose le port 80 mais n'est pas accessible depuis l'hôte. Pourtant un autre conteneur peut y accéder. Explique pourquoi.

<details>
<summary>Voir les réponses attendues</summary>

1. `-p 8080:80` bind sur `0.0.0.0` — accessible depuis n'importe quelle interface (réseau local, Internet si l'hôte est exposé). `-p 127.0.0.1:8080:80` bind uniquement sur loopback — accessible **uniquement** depuis la machine elle-même, jamais depuis l'extérieur.
2. Sur AWS, même avec `-p 8080:80`, c'est le **Security Group de l'EC2** qui décide si le port est accessible depuis Internet. Si le SG n'autorise pas le port 8080 en inbound, il est inaccessible depuis Internet — même si Docker l'a publié.
3. Pour qu'une base de données de dev soit accessible uniquement depuis l'hôte local — aucun risque qu'elle soit accidentellement exposée sur le réseau local ou Internet.
4. `EXPOSE` ne crée pas de règle iptables de redirection vers l'hôte — le port n'est pas accessible depuis l'hôte. Mais à l'intérieur du réseau Docker, les conteneurs communiquent directement par IP Docker sans passer par l'hôte — ils peuvent joindre `172.17.0.X:80` directement.

</details>

---

## Exercice 7 — Debug réseau complet

**Objectif** : diagnostiquer et résoudre un problème de connectivité réseau entre conteneurs en suivant une méthode structurée.

```bash
# 1. Créer une architecture intentionnellement cassée
docker network create reseau-a
docker network create reseau-b

docker run -d --name service-a --network reseau-a nginx
docker run -d --name service-b --network reseau-b redis:alpine

# service-a essaie de joindre service-b — ça ne marche pas !
docker exec service-a ping -c 2 service-b 2>&1 || echo "ÉCHEC attendu"

# 2. Diagnostiquer méthodiquement

# Étape 1 : sur quels réseaux est chaque conteneur ?
echo "=== Réseaux de service-a ==="
docker inspect service-a \
  --format '{{range $net, $conf := .NetworkSettings.Networks}}{{$net}}: {{$conf.IPAddress}}{{"\n"}}{{end}}'

echo "=== Réseaux de service-b ==="
docker inspect service-b \
  --format '{{range $net, $conf := .NetworkSettings.Networks}}{{$net}}: {{$conf.IPAddress}}{{"\n"}}{{end}}'

# Étape 2 : les conteneurs se voient-ils par IP ?
IP_B=$(docker inspect service-b \
  --format '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}')
docker exec service-a ping -c 2 $IP_B 2>&1 || echo "Pas joignable par IP non plus"

# Étape 3 : le DNS fonctionne-t-il ?
docker exec service-a nslookup service-b 2>&1

# 3. Identifier le problème
# → Les conteneurs sont sur des réseaux différents et ne peuvent pas se voir

# 4. Résoudre — connecter service-a au réseau de service-b
docker network connect reseau-b service-a

# 5. Retester après correction
docker exec service-a ping -c 2 service-b
docker exec service-a nc -zv service-b 6379

# 6. Utiliser netshoot pour un diagnostic avancé
docker run --rm --network reseau-b nicolaka/netshoot \
  nmap -sV service-b

# 7. Capturer le trafic entre les conteneurs
docker run --rm \
  --network container:service-a \
  nicolaka/netshoot \
  tcpdump -i eth0 -n -c 10 &

# Générer du trafic
docker exec service-a ping -c 3 service-b

# 8. Nettoyer
docker stop service-a service-b
docker rm service-a service-b
docker network rm reseau-a reseau-b
```

**Questions :**

1. Quelle était la cause du problème de connectivité initial ?
2. Après `docker network connect reseau-b service-a` — combien de réseaux et d'IPs a maintenant `service-a` ?
3. `nc -zv service-b 6379` retourne `succeeded` — que confirme exactement cette commande ?
4. Dans une architecture AWS, quelle situation ressemble à "deux conteneurs sur des réseaux différents qui ne peuvent pas se parler" ?

<details>
<summary>Voir les réponses attendues</summary>

1. `service-a` et `service-b` étaient sur des réseaux Docker distincts (`reseau-a` et `reseau-b`). Docker isole les réseaux — les conteneurs sur des réseaux différents ne peuvent pas communiquer, même sur le même hôte.
2. Deux réseaux (`reseau-a` et `reseau-b`) et **deux IPs** — une par réseau. Un conteneur peut être membre de plusieurs réseaux Docker simultanément.
3. Elle confirme que : le nom `service-b` se résout correctement (DNS OK), la machine est joignable (routing OK), et quelque chose écoute sur le port 6379 (service Redis démarré). C'est le test de connectivité complet en 3 en 1.
4. Deux EC2 dans des **subnets différents sans route entre eux** dans la Route Table, ou avec des Security Groups qui bloquent la communication. La solution AWS équivalente serait d'ajouter une route ou d'ouvrir le Security Group — exactement comme `docker network connect`.

</details>

---

## Exercice 8 — Architecture complète avec monitoring réseau

**Objectif** : construire une architecture réaliste avec Compose, vérifier l'isolation, et utiliser les outils de monitoring pour observer le trafic.

```bash
mkdir app-complete && cd app-complete

cat > compose.yml << 'EOF'
services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    networks:
      - public
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf

  app:
    image: python:3.11-alpine
    command: python -m http.server 8000
    networks:
      - public
      - private

  redis:
    image: redis:alpine
    networks:
      - private

  debug:
    image: nicolaka/netshoot
    command: sleep 3600
    networks:
      - public
      - private

networks:
  public:
  private:
EOF

cat > nginx.conf << 'EOF'
server {
    listen 80;
    location / {
        proxy_pass http://app:8000;
    }
}
EOF

# 1. Lancer l'architecture
docker compose up -d

# 2. Vérifier l'isolation réseau
echo "=== Test public network ==="
docker compose exec debug ping -c 2 nginx
docker compose exec debug ping -c 2 app

echo "=== Test private network ==="
docker compose exec debug ping -c 2 redis
docker compose exec debug nc -zv redis 6379

echo "=== Test isolation ==="
# nginx ne peut pas joindre redis (réseaux différents)
docker compose exec nginx ping -c 2 redis 2>&1 | tail -2

# 3. Observer le trafic réseau
# Capturer le trafic HTTP qui passe par nginx → app
docker compose exec debug tcpdump -i eth0 -n -c 20 port 8000 &
curl http://localhost:80    # générer du trafic
wait

# 4. Scanner les ports de chaque service depuis debug
docker compose exec debug nmap -p 80,8000,6379 nginx
docker compose exec debug nmap -p 80,8000,6379 app
docker compose exec debug nmap -p 80,8000,6379 redis

# 5. Voir les tables ARP et routes dans debug
docker compose exec debug arp -n
docker compose exec debug ip route show

# 6. Inspecter les réseaux Compose
docker network inspect app-complete_public
docker network inspect app-complete_private

# 7. Nettoyer
docker compose down
cd .. && rm -rf app-complete
```

**Questions :**

1. `nginx` peut-il joindre `redis` ? Pourquoi ?
2. Quel conteneur peut joindre à la fois `nginx` ET `redis` ? Pourquoi est-il dans cette position ?
3. Dans la capture tcpdump, quelle IP source vois-tu dans les requêtes de `nginx` vers `app` ?
4. Cette architecture reflète quel pattern AWS exactement ?

<details>
<summary>Voir les réponses attendues</summary>

1. **Non** — `nginx` est uniquement sur le réseau `public`, `redis` est uniquement sur le réseau `private`. Ils sont isolés. C'est voulu — nginx n'a pas à accéder directement à Redis.
2. **`app`** — il est sur les deux réseaux (`public` et `private`). Il joue le rôle de l'API qui reçoit les requêtes du frontend (via le réseau public) et accède à la base de données (via le réseau privé). C'est exactement le rôle d'un serveur d'application dans une architecture 3-tiers.
3. L'IP Docker de `nginx` sur le réseau `public` (quelque chose comme `172.18.0.2`). Le trafic circule directement entre IPs Docker — sans passer par l'hôte ni NAT.
4. C'est l'équivalent d'une architecture **3-tiers AWS** : réseau `public` = subnet public (ALB + serveur web), réseau `private` = subnet privé (app servers + BDD). `nginx` = ALB, `app` = EC2 en subnet privé applicatif, `redis` = ElastiCache en subnet privé BDD.

</details>

---

## Quiz final — Module 3

Réponds sans regarder le cours.

1. Quelle interface réseau Docker crée-t-il automatiquement sur l'hôte lors de l'installation ?
2. Qu'est-ce qu'une paire veth et à quoi sert-elle ?
3. Pourquoi deux conteneurs sur le bridge par défaut ne peuvent-ils pas se parler par nom ?
4. Quelle adresse IP est le serveur DNS Docker sur un réseau personnalisé ?
5. En mode `host`, le conteneur a-t-il sa propre adresse IP ?
6. Quelle est la différence entre `EXPOSE 80` dans un Dockerfile et `-p 80:80` dans `docker run` ?
7. `-p 8080:80` vs `-p 127.0.0.1:8080:80` — quelle différence de sécurité ?
8. Dans Docker Compose, comment les services se trouvent-ils mutuellement ?
9. Tu as deux services Compose sur des réseaux différents qui ne peuvent pas communiquer. Comment règles-tu ça ?
10. Quelle image Docker contient tous les outils de debug réseau (ping, nmap, tcpdump, etc.) ?
11. `nc -zv mon-service 5432` retourne `timed out`. Que suspectes-tu ?
12. `nc -zv mon-service 5432` retourne `Connection refused`. Que suspectes-tu ?

<details>
<summary>Voir les réponses</summary>

1. **docker0** — un bridge réseau virtuel sur lequel tous les conteneurs se connectent par défaut.
2. Une paire veth est un câble réseau virtuel en deux parties — une extrémité sur l'hôte (connectée au bridge docker0), l'autre à l'intérieur du conteneur (nommée `eth0`). Ce qui entre d'un côté ressort de l'autre.
3. Le réseau bridge par défaut n'a pas de serveur DNS embarqué. La résolution par nom nécessite un réseau **personnalisé** qui active `127.0.0.11`.
4. **`127.0.0.11`** — visible dans `/etc/resolv.conf` à l'intérieur du conteneur.
5. **Non** — en mode `host`, le conteneur partage l'espace réseau de l'hôte. Il utilise les interfaces et IPs de l'hôte directement.
6. `EXPOSE 80` est documentation uniquement — ne publie rien. `-p 80:80` crée réellement une règle iptables et rend le port accessible depuis l'hôte.
7. `-p 8080:80` bind sur `0.0.0.0` — accessible depuis le réseau local et Internet. `-p 127.0.0.1:8080:80` bind sur loopback — accessible uniquement depuis l'hôte lui-même.
8. Par leur **nom de service** défini dans `compose.yml`. Docker Compose crée automatiquement un réseau avec DNS pour le projet — chaque service est résolvable par son nom.
9. Ajouter les deux services au même réseau dans `compose.yml`, ou utiliser `docker network connect` manuellement.
10. **nicolaka/netshoot** — une image de debug réseau avec tous les outils préinstallés.
11. Un **firewall bloque** silencieusement (DROP) — le paquet ne passe pas. Suspect : règle ufw, Security Group, ou les conteneurs sont sur des réseaux Docker différents.
12. Le service est **joignable** mais rien n'écoute sur ce port — le service n'est probablement pas démarré, ou écoute sur un port différent.

</details>

---

*→ Retour au cours : `module3-theorie.md`*
*→ Module suivant : Module 4 — AWS Networking + Kubernetes*
