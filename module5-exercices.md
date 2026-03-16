# Module 5 — Kubernetes Networking
### Partie Exercices

> Complète la partie théorie (`module5-theorie.md`) avant de faire ces exercices.
> Prérequis : Minikube installé, kubectl configuré
> Durée estimée : 3h à 3h30

---

## Installation et démarrage

```bash
# Démarrer Minikube avec le CNI Calico (supporte les Network Policies)
minikube start --network-plugin=cni --cni=calico

# Vérifier que le cluster est prêt
kubectl cluster-info
kubectl get nodes
kubectl get pods -n kube-system

# Installer l'Ingress Controller nginx
minikube addons enable ingress

# Vérifier l'installation
kubectl get pods -n ingress-nginx
```

---

## Exercice 1 — Explorer le modèle réseau K8s

**Objectif** : observer concrètement les IPs des pods, leur éphémérité, et la communication directe sans NAT.

```bash
# 1. Créer un namespace dédié
kubectl create namespace lab-network

# 2. Déployer deux applications simples
cat << 'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-a
  namespace: lab-network
spec:
  replicas: 2
  selector:
    matchLabels:
      app: app-a
  template:
    metadata:
      labels:
        app: app-a
    spec:
      containers:
      - name: app
        image: nginx:alpine
        ports:
        - containerPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-b
  namespace: lab-network
spec:
  replicas: 2
  selector:
    matchLabels:
      app: app-b
  template:
    metadata:
      labels:
        app: app-b
    spec:
      containers:
      - name: app
        image: nginx:alpine
        ports:
        - containerPort: 80
EOF

# 3. Voir les IPs attribuées aux pods
kubectl get pods -n lab-network -o wide

# 4. Observer que chaque pod a une IP unique
# Notez les IPs — elles sont dans le CIDR du pod network

# 5. Communication directe entre pods (sans Service)
POD_A=$(kubectl get pods -n lab-network -l app=app-a \
  -o jsonpath='{.items[0].metadata.name}')
IP_B=$(kubectl get pods -n lab-network -l app=app-b \
  -o jsonpath='{.items[0].status.podIP}')

echo "Pod A : $POD_A"
echo "IP Pod B : $IP_B"

# Depuis pod-a, pinguer pod-b par IP directement
kubectl exec -n lab-network $POD_A -- ping -c 3 $IP_B

# 6. Observer l'éphémérité des IPs
echo "IP avant suppression : $IP_B"
POD_B=$(kubectl get pods -n lab-network -l app=app-b \
  -o jsonpath='{.items[0].metadata.name}')
kubectl delete pod -n lab-network $POD_B

# Attendre que K8s recrée le pod
kubectl wait --for=condition=ready pod -l app=app-b \
  -n lab-network --timeout=60s

# Voir la nouvelle IP
NEW_IP_B=$(kubectl get pods -n lab-network -l app=app-b \
  -o jsonpath='{.items[0].status.podIP}')
echo "IP après recréation : $NEW_IP_B"
# → IP différente ! C'est le problème que les Services résolvent.

# 7. Voir le CIDR du pod network
kubectl cluster-info dump | grep -m1 "cluster-cidr"

# 8. Nettoyer
kubectl delete namespace lab-network
```

**Questions :**

1. Les IPs des pods sont-elles dans la même plage que les IPs de tes machines locales ?
2. Après suppression et recréation du pod, l'IP a-t-elle changé ?
3. La communication directe pod-to-pod par IP a-t-elle fonctionné ? Pourquoi ?
4. Quelle est la conséquence pour une application qui hardcoderait l'IP d'un pod ?

<details>
<summary>Voir les réponses attendues</summary>

1. Non — les pods ont des IPs dans le CIDR K8s (souvent `10.244.x.x` avec Flannel ou `192.168.x.x` avec Calico), séparé du réseau de la machine hôte.
2. Oui — l'IP change à chaque recréation de pod. C'est la nature éphémère des pods K8s.
3. Oui — K8s garantit que tout pod peut joindre tout autre pod directement, sans NAT. C'est l'une des règles fondamentales du modèle réseau K8s.
4. L'application casserait dès que le pod est recréé (crash, mise à jour, scaling). C'est pourquoi on utilise toujours les Services comme point d'entrée stable.

</details>

---

## Exercice 2 — Services : ClusterIP, NodePort, LoadBalancer

**Objectif** : créer les trois types de Services et comprendre leurs différences d'accessibilité.

```bash
kubectl create namespace lab-services

# 1. Déployer une application web
cat << 'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  namespace: lab-services
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: web
        image: nginx:alpine
        ports:
        - containerPort: 80
EOF

# Attendre que les pods soient prêts
kubectl wait --for=condition=ready pod -l app=webapp \
  -n lab-services --timeout=60s

# 2. SERVICE CLUSTERIP
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: webapp-clusterip
  namespace: lab-services
spec:
  type: ClusterIP
  selector:
    app: webapp
  ports:
  - port: 80
    targetPort: 80
EOF

# Voir la ClusterIP attribuée
kubectl get service webapp-clusterip -n lab-services

# Tester depuis un pod dans le cluster
kubectl run test-pod --image=nicolaka/netshoot \
  --rm -it -n lab-services -- \
  curl http://webapp-clusterip

# Tester depuis l'hôte → doit ÉCHOUER (ClusterIP non accessible depuis l'extérieur)
CLUSTER_IP=$(kubectl get svc webapp-clusterip -n lab-services \
  -o jsonpath='{.spec.clusterIP}')
curl --connect-timeout 3 http://$CLUSTER_IP || echo "❌ Non accessible depuis l'hôte"

# 3. SERVICE NODEPORT
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: webapp-nodeport
  namespace: lab-services
spec:
  type: NodePort
  selector:
    app: webapp
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
EOF

# Voir le NodePort
kubectl get service webapp-nodeport -n lab-services

# Tester via le NodePort de Minikube
minikube service webapp-nodeport -n lab-services --url
# → affiche l'URL accessible depuis l'hôte
curl $(minikube service webapp-nodeport -n lab-services --url)

# 4. VOIR LES ENDPOINTS (pods derrière le service)
kubectl get endpoints -n lab-services

# Supprimer un pod et voir les endpoints se mettre à jour
POD=$(kubectl get pods -n lab-services -l app=webapp \
  -o jsonpath='{.items[0].metadata.name}')
kubectl delete pod -n lab-services $POD &
kubectl get endpoints -n lab-services -w
# → observer l'endpoint disparaître puis réapparaître

# 5. DEBUG — voir kube-proxy et iptables
minikube ssh "sudo iptables -t nat -L KUBE-SERVICES -n | grep lab"

# 6. Nettoyer
kubectl delete namespace lab-services
```

**Questions :**

1. La ClusterIP est-elle accessible depuis ton laptop (en dehors du cluster) ?
2. Quelle est la plage de ports autorisée pour NodePort ?
3. Quand tu supprimes un pod, que se passe-t-il dans les Endpoints du Service ?
4. Dans `iptables -t nat -L KUBE-SERVICES`, que représentent les règles K8s ?

<details>
<summary>Voir les réponses attendues</summary>

1. **Non** — la ClusterIP est uniquement accessible depuis l'intérieur du cluster. C'est son but : communication interne uniquement. Pour accéder depuis l'extérieur, il faut NodePort, LoadBalancer ou Ingress.
2. **30000-32767** — plage réservée aux NodePorts dans K8s. Les ports en dessous sont réservés aux services système.
3. K8s retire immédiatement l'IP du pod supprimé des Endpoints, puis ajoute l'IP du nouveau pod créé. Le Service ne route plus vers le pod supprimé — le changement est quasi-instantané.
4. Ces règles iptables sont gérées par **kube-proxy**. Elles interceptent le trafic vers les ClusterIPs (IPs virtuelles) et le redirigent vers les IPs réelles des pods via DNAT. Le ClusterIP n'existe pas physiquement — c'est une règle iptables.

</details>

---

## Exercice 3 — DNS Kubernetes avec CoreDNS

**Objectif** : comprendre le système DNS K8s et le format de nommage des Services.

```bash
kubectl create namespace lab-dns-a
kubectl create namespace lab-dns-b

# 1. Créer des services dans deux namespaces différents
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: api-service
  namespace: lab-dns-a
spec:
  selector:
    app: api
  ports:
  - port: 80
---
apiVersion: v1
kind: Service
metadata:
  name: database
  namespace: lab-dns-b
spec:
  selector:
    app: db
  ports:
  - port: 5432
EOF

# 2. Lancer un pod de debug dans lab-dns-a
kubectl run debug-pod \
  --image=nicolaka/netshoot \
  --namespace=lab-dns-a \
  -it --rm -- bash

# Dans le pod debug :

# 3. Voir la config DNS du pod
cat /etc/resolv.conf
# → nameserver: IP de CoreDNS
# → search: lab-dns-a.svc.cluster.local svc.cluster.local cluster.local

# 4. Résoudre le service dans le MÊME namespace
nslookup api-service
nslookup api-service.lab-dns-a
nslookup api-service.lab-dns-a.svc.cluster.local

# 5. Résoudre le service dans un AUTRE namespace
# Le nom court NE FONCTIONNE PAS cross-namespace
nslookup database
# → FAIL (pas dans le même namespace)

nslookup database.lab-dns-b
# → OK (namespace explicite)

nslookup database.lab-dns-b.svc.cluster.local
# → OK (nom complet)

# 6. Voir CoreDNS lui-même
nslookup kubernetes.default.svc.cluster.local
# → IP du Service kubernetes (API server)

# 7. Tester la résolution d'un service externe
nslookup google.com
# → CoreDNS transmet les requêtes externes vers les DNS upstream

# 8. Quitter le pod debug (exit)
exit

# 9. Voir la config de CoreDNS
kubectl get configmap coredns -n kube-system -o yaml

# 10. Nettoyer
kubectl delete namespace lab-dns-a lab-dns-b
```

**Questions :**

1. Depuis un pod dans `lab-dns-a`, `nslookup api-service` fonctionne-t-il ? Et `nslookup database` ?
2. Quel est le nom DNS complet du Service `database` dans le namespace `lab-dns-b` ?
3. Que contient le champ `search` dans `/etc/resolv.conf` du pod ? À quoi sert-il ?
4. CoreDNS peut-il résoudre `google.com` ? Comment ?

<details>
<summary>Voir les réponses attendues</summary>

1. `nslookup api-service` **fonctionne** — même namespace, K8s ajoute automatiquement `.lab-dns-a.svc.cluster.local`. `nslookup database` **échoue** — service dans un autre namespace, le nom court ne suffit pas.
2. `database.lab-dns-b.svc.cluster.local` — format complet : `<service>.<namespace>.svc.cluster.local`.
3. Le champ `search` liste les suffixes DNS à essayer automatiquement. Quand tu tapes `api-service`, K8s essaie `api-service.lab-dns-a.svc.cluster.local`, puis `api-service.svc.cluster.local`, etc. C'est ce qui permet les noms courts dans le même namespace.
4. Oui — CoreDNS transmet les requêtes pour les noms non-K8s vers les serveurs DNS upstream (configurés dans le ConfigMap CoreDNS). Ces serveurs résolvent les noms externes normalement.

</details>

---

## Exercice 4 — Ingress avec nginx

**Objectif** : configurer un Ingress pour router le trafic vers plusieurs services selon l'URL.

```bash
# Vérifier que l'Ingress Controller nginx est installé
kubectl get pods -n ingress-nginx

kubectl create namespace lab-ingress

# 1. Déployer deux applications
cat << 'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: lab-ingress
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: web
        image: nginx:alpine
        ports:
        - containerPort: 80
        volumeMounts:
        - name: html
          mountPath: /usr/share/nginx/html
      volumes:
      - name: html
        configMap:
          name: frontend-html
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: frontend-html
  namespace: lab-ingress
data:
  index.html: |
    <h1>Frontend Service</h1>
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: lab-ingress
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - name: api
        image: nginx:alpine
        ports:
        - containerPort: 80
        volumeMounts:
        - name: html
          mountPath: /usr/share/nginx/html
      volumes:
      - name: html
        configMap:
          name: api-html
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-html
  namespace: lab-ingress
data:
  index.html: |
    {"service": "API", "status": "ok"}
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-svc
  namespace: lab-ingress
spec:
  selector:
    app: frontend
  ports:
  - port: 80
---
apiVersion: v1
kind: Service
metadata:
  name: api-svc
  namespace: lab-ingress
spec:
  selector:
    app: api
  ports:
  - port: 80
EOF

# 2. Créer l'Ingress avec routing par path
cat << 'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  namespace: lab-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: monapp.local
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-svc
            port:
              number: 80
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-svc
            port:
              number: 80
EOF

# 3. Voir l'Ingress créé
kubectl get ingress -n lab-ingress

# 4. Obtenir l'IP de Minikube
MINIKUBE_IP=$(minikube ip)
echo "Minikube IP : $MINIKUBE_IP"

# 5. Ajouter le hostname dans /etc/hosts
echo "$MINIKUBE_IP monapp.local" | sudo tee -a /etc/hosts

# 6. Tester le routing
curl http://monapp.local/        # → Frontend
curl http://monapp.local/api     # → API

# 7. Voir les logs de l'Ingress Controller
kubectl logs -n ingress-nginx \
  $(kubectl get pods -n ingress-nginx -o jsonpath='{.items[0].metadata.name}') \
  --tail=20

# 8. Ajouter le routing par hostname
cat << 'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress-host
  namespace: lab-ingress
spec:
  rules:
  - host: api.monapp.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-svc
            port:
              number: 80
  - host: www.monapp.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-svc
            port:
              number: 80
EOF

# Ajouter les nouveaux hostnames
echo "$MINIKUBE_IP api.monapp.local" | sudo tee -a /etc/hosts
echo "$MINIKUBE_IP www.monapp.local" | sudo tee -a /etc/hosts

curl http://api.monapp.local     # → API
curl http://www.monapp.local     # → Frontend

# 9. Nettoyer
kubectl delete namespace lab-ingress
sudo sed -i '/monapp.local/d' /etc/hosts
```

**Questions :**

1. Quelle est la différence entre le routing par path et le routing par hostname dans Ingress ?
2. Pourquoi l'annotation `rewrite-target: /` est-elle nécessaire dans l'étape 2 ?
3. Sans Ingress, combien de LoadBalancers faudrait-il pour exposer les deux services ?
4. Sur AWS EKS, quel composant remplacerait l'Ingress Controller nginx ?

<details>
<summary>Voir les réponses attendues</summary>

1. **Path routing** : même domaine (`monapp.local`), différents chemins (`/api`, `/`). **Host routing** : domaines différents (`api.monapp.local`, `www.monapp.local`). Les deux peuvent être combinés. Le host routing est plus propre pour les microservices avec des domaines dédiés.
2. Sans `rewrite-target: /`, la requête vers `/api/users` est transmise telle quelle au service backend — qui ne connaît pas le préfixe `/api`. Le rewrite supprime le préfixe avant de transmettre, donc le backend reçoit `/users` comme attendu.
3. **Deux LoadBalancers** — un par Service de type LoadBalancer. Avec Ingress, un seul suffit pour les deux. Sur AWS, chaque ALB coûte ~$16/mois minimum — économie directe.
4. **AWS Load Balancer Controller** — il traduit les ressources Ingress K8s en ALB AWS natifs avec toutes les fonctionnalités AWS (ACM, WAF, Security Groups, etc.).

</details>

---

## Exercice 5 — Network Policies

**Objectif** : implémenter une architecture 3-tiers avec Network Policies et vérifier que l'isolation fonctionne.

```bash
kubectl create namespace lab-netpol

# 1. Déployer l'architecture frontend + api + database
cat << 'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: lab-netpol
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: nicolaka/netshoot
        command: ["sleep", "3600"]
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: lab-netpol
spec:
  replicas: 1
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - name: api
        image: nginx:alpine
        ports:
        - containerPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database
  namespace: lab-netpol
spec:
  replicas: 1
  selector:
    matchLabels:
      app: database
  template:
    metadata:
      labels:
        app: database
    spec:
      containers:
      - name: db
        image: nginx:alpine
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: api
  namespace: lab-netpol
spec:
  selector:
    app: api
  ports:
  - port: 80
---
apiVersion: v1
kind: Service
metadata:
  name: database
  namespace: lab-netpol
spec:
  selector:
    app: database
  ports:
  - port: 80
EOF

kubectl wait --for=condition=ready pod -l app=frontend \
  -n lab-netpol --timeout=60s

# 2. AVANT Network Policies — vérifier que tout est accessible
FRONTEND_POD=$(kubectl get pods -n lab-netpol -l app=frontend \
  -o jsonpath='{.items[0].metadata.name}')

echo "=== AVANT Network Policies ==="
kubectl exec -n lab-netpol $FRONTEND_POD -- \
  curl -s --connect-timeout 3 http://api && echo "frontend→api : ✅"
kubectl exec -n lab-netpol $FRONTEND_POD -- \
  curl -s --connect-timeout 3 http://database && echo "frontend→database : ✅"

# 3. Appliquer les Network Policies
cat << 'EOF' | kubectl apply -f -
# Deny all par défaut dans ce namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: lab-netpol
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
---
# Frontend peut contacter api uniquement
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-policy
  namespace: lab-netpol
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
    - protocol: TCP
      port: 80
  # Autoriser DNS
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - protocol: UDP
      port: 53
---
# API reçoit depuis frontend, contacte database
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-policy
  namespace: lab-netpol
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
    ports:
    - protocol: TCP
      port: 80
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - protocol: TCP
      port: 80
---
# Database reçoit uniquement depuis api
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-policy
  namespace: lab-netpol
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
    - protocol: TCP
      port: 80
EOF

# 4. APRÈS Network Policies — vérifier l'isolation
echo "=== APRÈS Network Policies ==="

kubectl exec -n lab-netpol $FRONTEND_POD -- \
  curl -s --connect-timeout 3 http://api \
  && echo "frontend→api : ✅" \
  || echo "frontend→api : ❌"

kubectl exec -n lab-netpol $FRONTEND_POD -- \
  curl -s --connect-timeout 3 http://database \
  && echo "frontend→database : ✅ (PROBLÈME!)" \
  || echo "frontend→database : ❌ (isolation OK ✅)"

# 5. Voir les Network Policies
kubectl get networkpolicy -n lab-netpol
kubectl describe networkpolicy api-policy -n lab-netpol

# 6. Nettoyer
kubectl delete namespace lab-netpol
```

**Questions :**

1. Avant les Network Policies, `frontend → database` était-il possible ?
2. Après les Network Policies, `frontend → database` est-il bloqué ? Pourquoi ?
3. Pourquoi faut-il autoriser le trafic UDP port 53 pour le pod frontend ?
4. Quelle Network Policy ressemble le plus à un Security Group AWS ?

<details>
<summary>Voir les réponses attendues</summary>

1. **Oui** — par défaut K8s n'a aucune isolation. Tout pod peut joindre tout autre pod. C'est le comportement "flat network" de K8s.
2. **Oui, bloqué** — la policy `frontend-policy` limite l'egress du frontend uniquement vers les pods `app: api`. La database n'est pas dans les destinations autorisées.
3. Sans autoriser DNS (UDP 53), le frontend ne peut pas résoudre les noms des Services (`api`, `database`). Le `default-deny-all` bloque tout egress — y compris les requêtes DNS vers CoreDNS dans `kube-system`.
4. La policy `database-policy` ressemble le plus à un Security Group AWS : elle définit exactement **qui peut accéder** en Ingress sur quels ports — exactement comme `SG-DB : inbound 5432 depuis SG-App`.

</details>

---

## Exercice 6 — Debug réseau complet dans K8s

**Objectif** : utiliser les outils de diagnostic réseau K8s pour investiguer et résoudre des problèmes de connectivité.

```bash
kubectl create namespace lab-debug

# 1. Déployer une architecture avec un problème intentionnel
cat << 'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: client
  namespace: lab-debug
spec:
  replicas: 1
  selector:
    matchLabels:
      app: client
  template:
    metadata:
      labels:
        app: client
    spec:
      containers:
      - name: client
        image: nicolaka/netshoot
        command: ["sleep", "3600"]
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: server
  namespace: lab-debug
spec:
  replicas: 2
  selector:
    matchLabels:
      app: server
  template:
    metadata:
      labels:
        app: server
    spec:
      containers:
      - name: server
        image: nginx:alpine
        ports:
        - containerPort: 80
---
# Service avec un bug intentionnel (mauvais selector)
apiVersion: v1
kind: Service
metadata:
  name: server-svc
  namespace: lab-debug
spec:
  selector:
    app: serveur   # ← ERREUR : "serveur" au lieu de "server"
  ports:
  - port: 80
    targetPort: 80
EOF

kubectl wait --for=condition=ready pod -l app=client \
  -n lab-debug --timeout=60s

CLIENT_POD=$(kubectl get pods -n lab-debug -l app=client \
  -o jsonpath='{.items[0].metadata.name}')

# 2. Reproduire le problème
kubectl exec -n lab-debug $CLIENT_POD -- \
  curl --connect-timeout 3 http://server-svc \
  || echo "❌ Connexion impossible"

# 3. MÉTHODE DE DIAGNOSTIC

# Étape 1 : Le Service existe-t-il ?
kubectl get service server-svc -n lab-debug

# Étape 2 : Le DNS résout-il le Service ?
kubectl exec -n lab-debug $CLIENT_POD -- \
  nslookup server-svc

# Étape 3 : Y a-t-il des Endpoints ? (pods sélectionnés)
kubectl get endpoints server-svc -n lab-debug
# → Si vide : le selector ne correspond à aucun pod ← PROBLÈME TROUVÉ

# Étape 4 : Voir les labels des pods
kubectl get pods -n lab-debug --show-labels

# Étape 5 : Comparer selector du Service avec labels des pods
kubectl describe service server-svc -n lab-debug | grep Selector

# 4. IDENTIFIER LE BUG
echo "Selector du Service : app=serveur"
echo "Labels des pods     : app=server"
echo "→ Typo dans le selector !"

# 5. CORRIGER le Service
kubectl patch service server-svc -n lab-debug \
  --type='json' \
  -p='[{"op": "replace", "path": "/spec/selector/app", "value": "server"}]'

# 6. Vérifier la correction
kubectl get endpoints server-svc -n lab-debug
# → Doit maintenant afficher les IPs des pods

# 7. Tester à nouveau
kubectl exec -n lab-debug $CLIENT_POD -- \
  curl --connect-timeout 3 http://server-svc \
  && echo "✅ Connexion réussie !"

# 8. Autres commandes de debug utiles
# Voir tous les événements du namespace
kubectl get events -n lab-debug --sort-by='.lastTimestamp'

# Voir les logs d'un pod
kubectl logs -n lab-debug -l app=server --tail=20

# Port-forward pour accéder depuis l'hôte
kubectl port-forward -n lab-debug svc/server-svc 8080:80 &
curl http://localhost:8080
kill %1

# Exécuter des outils réseau dans un pod existant
kubectl exec -n lab-debug $CLIENT_POD -- ss -tnp
kubectl exec -n lab-debug $CLIENT_POD -- ip route show
kubectl exec -n lab-debug $CLIENT_POD -- ip addr show

# 9. Nettoyer
kubectl delete namespace lab-debug
```

**Questions :**

1. Quelle commande t'a permis d'identifier le problème ?
2. Que signifie un Service avec une liste d'Endpoints vide ?
3. `kubectl port-forward` — à quoi sert-il et quand l'utilise-t-on ?
4. Quelle est la méthode de diagnostic en 5 étapes pour un Service inaccessible ?

<details>
<summary>Voir les réponses attendues</summary>

1. `kubectl get endpoints server-svc` — elle a montré une liste vide, ce qui indique que le selector du Service ne correspond à aucun pod. Combiné avec `kubectl get pods --show-labels`, le problème devient évident.
2. Un Service avec Endpoints vides signifie que son **selector ne correspond à aucun pod** en cours d'exécution. Le Service existe, le DNS le résout, mais il n'y a rien vers quoi router le trafic — les connexions échouent.
3. `kubectl port-forward` crée un tunnel entre un port local de ta machine et un port d'un Service ou Pod dans le cluster. Utilisé pour accéder à des services non exposés depuis l'hôte — debug, accès temporaire à une UI (Grafana, Kibana), test d'une API sans Ingress.
4. Méthode de diagnostic : **1.** `kubectl get service` (le service existe ?) **2.** `nslookup service-name` depuis un pod (DNS résout ?) **3.** `kubectl get endpoints` (des pods sont sélectionnés ?) **4.** `kubectl get pods --show-labels` (les labels correspondent au selector ?) **5.** `kubectl describe pod` + `kubectl logs` (le pod est healthy et le service répond ?).

</details>

---

## Quiz final — Module 5

Réponds sans regarder le cours.

1. Quelle est la règle fondamentale du modèle réseau K8s concernant la communication entre pods ?
2. Pourquoi ne faut-il jamais hardcoder l'IP d'un pod dans une application K8s ?
3. Quelle est la différence entre ClusterIP et NodePort ?
4. Tu as 5 microservices à exposer sur Internet. Combien de Load Balancers utilises-tu avec des Services LoadBalancer vs avec Ingress ?
5. Le format DNS complet d'un Service `database` dans le namespace `production` ?
6. Depuis un pod dans le namespace `default`, peut-on contacter un service `api` dans le namespace `staging` avec `curl http://api` ?
7. Par défaut, un pod peut-il joindre n'importe quel autre pod dans le cluster ?
8. Quelle commande utilises-tu pour voir les pods derrière un Service ?
9. Sur EKS, quelle est la différence entre un cluster K8s standard et EKS avec VPC CNI pour les IPs des pods ?
10. Tu appliques une Network Policy `default-deny-all` mais oublies d'autoriser DNS. Quel problème survient ?
11. `kubectl get endpoints mon-service` retourne une liste vide. Que suspectes-tu ?
12. Quelle est la différence entre un Ingress Controller nginx et AWS Load Balancer Controller sur EKS ?

<details>
<summary>Voir les réponses</summary>

1. Tout pod peut communiquer avec tout autre pod **directement, sans NAT**. L'IP source n'est jamais modifiée. C'est la règle fondamentale qui distingue K8s de Docker.
2. Les IPs de pods sont **éphémères** — elles changent à chaque recréation du pod (crash, mise à jour, scaling). Un hardcode casse immédiatement. On utilise les Services qui ont des IPs stables.
3. **ClusterIP** : IP virtuelle accessible uniquement dans le cluster. **NodePort** : expose le service sur un port (30000-32767) de chaque nœud, accessible depuis l'extérieur via `<IP-noeud>:<nodePort>`.
4. **5 Load Balancers** avec Services LoadBalancer. **1 Load Balancer** avec Ingress. L'Ingress route le trafic vers les bons services selon l'URL/hostname — économie de 4 LB.
5. `database.production.svc.cluster.local`
6. **Non** — `curl http://api` cherche `api` dans le namespace `default`. Pour le namespace `staging`, il faut `curl http://api.staging` ou la forme complète `api.staging.svc.cluster.local`.
7. **Oui** — par défaut K8s n'a aucune isolation réseau entre pods. Tous les pods peuvent se joindre. Il faut des Network Policies pour restreindre.
8. `kubectl get endpoints <nom-du-service>` — liste les IPs:ports des pods actuellement sélectionnés par le Service.
9. Sur un cluster standard, les pods ont des IPs dans le **pod CIDR** (réseau overlay séparé). Sur EKS avec VPC CNI, les pods ont des **IPs directement du VPC** — ce sont des hôtes à part entière du réseau AWS, visibles dans les Security Groups, Flow Logs, etc.
10. Les pods avec la policy appliquée ne peuvent plus résoudre les noms DNS → `curl http://api` échoue avec "Name or service not known". Il faut toujours autoriser explicitement l'egress UDP 53 vers CoreDNS quand on utilise `default-deny-all`.
11. Le **selector du Service ne correspond à aucun pod** en cours d'exécution — soit les pods n'existent pas, soit il y a une typo dans les labels, soit les pods sont dans un namespace différent.
12. **nginx Ingress Controller** : crée un pod nginx dans le cluster, gère le routing lui-même, portable (fonctionne sur n'importe quel K8s). **AWS Load Balancer Controller** : crée de vrais ALB AWS natifs dans le VPC, intégration complète avec ACM, WAF, Security Groups — mais uniquement sur EKS.

</details>

---

*→ Retour au cours : `module5-theorie.md`*
*→ Module suivant : Module 6 — Networking as Code avec Terraform*
