# Lab du Projet de Synthèse : Application Multi-Tiers

Objectif : Mettre en pratique tous les concepts de la semaine pour déployer une application de vote complète et robuste. Nous allons transformer des manifestes de base en manifestes prêts pour la production en ajoutant des sondes de santé, des gestions de ressources, un stockage persistant et un routage Ingress.

Prérequis : Avoir un cluster Minikube fonctionnel avec l'addon Ingress activé (minikube addons enable ingress).

##  Étape 1 : Préparation de l'Environnement
Créez un namespace dédié pour isoler complètement notre projet :

```Bash

kubectl create namespace vote
```
Configurez votre DNS local. Obtenez d'abord l'IP de votre Minikube :

```Bash

minikube ip
```
Ensuite, sur votre machine locale (pas la VM), modifiez votre fichier hosts (/etc/hosts ou C:\Windows\System32\drivers\etc\hosts) pour y ajouter les lignes suivantes, en remplaçant  <IP_PUBLIQUE_DE_LA_VM> par l'adresse de votre Droplet :e obtenue :
```
<IP_PUBLIQUE_DE_LA_VM> vote.votre-domaine.local
<IP_PUBLIQUE_DE_LA_VM> result.votre-domaine.local
```

##  Étape 2 : Déploiement des Composants
Créez les fichiers YAML suivants et appliquez-les tous dans le namespace vote avec la commande 
```
kubectl apply -f <nom-du-fichier> -n vote.
```
1. Redis (Cache)
Redis est un cache en mémoire, il peut donc être un Deployment simple.

Fichier redis-depl.yaml :

```YAML

apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-deployment
  labels:
    app: redis
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:alpine
        ports:
        - containerPort: 6379
        resources:
          requests:
            cpu: "100m"
            memory: "100Mi"
          limits:
            cpu: "200m"
            memory: "200Mi"
---
apiVersion: v1
kind: Service
metadata:
  name: redis
  labels:
    app: redis
spec:
  type: ClusterIP
  ports:
  - port: 6379
    targetPort: 6379
  selector:
    app: redis
```

2. PostgreSQL (Base de Données)
PostgreSQL stocke l'état de l'application, nous devons donc utiliser un StatefulSet avec un stockage persistant.

Fichier postgres-statefulset.yaml :

```YAML

apiVersion: v1
kind: Service
metadata:
  name: db # Le worker s'attend à ce nom de service
  labels:
    app: postgres
spec:
  type: ClusterIP
  ports:
  - port: 5432
    targetPort: 5432
  selector:
    app: postgres
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres-statefulset
spec:
  serviceName: "db"
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:9.4
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_USER
          value: "postgres"
        - name: POSTGRES_PASSWORD
          value: "postgres"
        resources:
          requests:
            cpu: "250m"
            memory: "256Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
        volumeMounts:
        - name: postgres-data
          mountPath: /var/lib/postgresql/data
  # Modèle pour créer une PVC pour chaque pod du StatefulSet
  volumeClaimTemplates:
  - metadata:
      name: postgres-data
    spec:
      accessModes:
        - ReadWriteOnce
      storageClassName: "standard" # Utilise la StorageClass par défaut de Minikube
      resources:
        requests:
          storage: 1Gi

```

3. Worker (Application de Traitement)
Le worker est une application sans état qui traite les votes.

Fichier worker-depl.yaml :

```YAML

apiVersion: apps/v1
kind: Deployment
metadata:
  name: worker-deployment
  labels:
    app: worker
spec:
  replicas: 1
  selector:
    matchLabels:
      app: worker
  template:
    metadata:
      labels:
        app: worker
    spec:
      containers:
      - name: worker
        image: kodekloud/examplevotingapp_worker:v1
        resources:
          requests:
            cpu: "100m"
            memory: "100Mi"
          limits:
            cpu: "200m"
            memory: "200Mi"
```

4. Vote (Frontend de Vote)
C'est une application web, nous ajoutons donc des sondes de santé.

Fichier vote-depl.yaml :

```YAML

apiVersion: apps/v1
kind: Deployment
metadata:
  name: vote-deployment
  labels:
    app: vote
spec:
  replicas: 2
  selector:
    matchLabels:
      app: vote
  template:
    metadata:
      labels:
        app: vote
    spec:
      containers:
      - name: vote
        image: kodekloud/example-voting-app-vote:v1
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "250m"
            memory: "256Mi"
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: vote
  labels:
    app: vote
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: vote

```

5. Result (Frontend des Résultats)
C'est également une application web avec des sondes de santé.

Fichier result-depl.yaml :

```YAML

apiVersion: apps/v1
kind: Deployment
metadata:
  name: result-deployment
  labels:
    app: result
spec:
  replicas: 1
  selector:
    matchLabels:
      app: result
  template:
    metadata:
      labels:
        app: result
    spec:
      containers:
      - name: result
        image: kodekloud/example-voting-app-result:v1
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "250m"
            memory: "256Mi"
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: result
  labels:
    app: result
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: result
```

6. Ingress (Routage Externe)
C'est le point d'entrée qui route le trafic vers nos deux applications web.

Fichier ingress.yaml :

```YAML

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: vote-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: "vote.votre-domaine.local"
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: vote
            port:
              number: 80
  - host: "result.votre-domaine.local"
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: result
            port:
              number: 80

```

##  Étape 3 : Déploiement et Vérification
Sauvegardez tous les fichiers ci-dessus dans un répertoire.

Appliquez tous les manifestes dans le namespace vote :

```Bash

kubectl apply -f. -n vote
```
Vérifiez que tout démarre correctement :

```Bash

# Attendez que tous les pods soient 'Running' et 'Ready'
kubectl get pods -n vote -w

# Vérifiez que la PVC pour postgres a bien été créée et est 'Bound'
kubectl get pvc -n vote

# Vérifiez que l'Ingress a bien reçu une adresse
kubectl get ingress -n vote

```

##  Étape 4 :  utiliser un reverse proxy Nginx sur ta VM Ubuntu

Minikube tourne dans la VM avec IP privée interne (ex: 192.168.49.2) — services exposés sur NodePort ou ClusterIP.

Nginx sur la VM écoute sur l’IP publique ur port 80 ou 443.

Nginx redirige le trafic HTTP vers ton service Kubernetes via l’IP Minikube + port NodePort.

### Étapes pour configurer Nginx comme reverse proxy
1. Installer Nginx :
```
sudo apt update
sudo apt install nginx -y
```
2.Vérifie que Nginx est bien lancé :
```
sudo systemctl status nginx
```
3. Crée un fichier de configuration pour ton app (ex: /etc/nginx/sites-available/vote):
```
server {
    listen 80;
    server_name vote.votre-domaine.local result.votre-domaine.local;;

    location / {
        proxy_pass http://192.168.49.2:30721;  # IP minikube + port NodePort du service vote
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}

```
Active cette config :

```bash
sudo ln -s /etc/nginx/sites-available/vote /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx

```

Testez l'application :

Ouvrez http://vote.votre-domaine.local dans votre navigateur et votez.

Ouvrez http://result.votre-domaine.local pour voir les résultats s'afficher après quelques secondes.

Félicitations! Vous avez déployé une application multi-tiers complète en utilisant les meilleures pratiques de Kubernetes.
