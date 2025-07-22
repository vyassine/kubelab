# Lab 3.1 : Routage HTTP avec Ingress

Objectif : Déployer deux applications distinctes et les exposer sur le même nom de domaine mais avec des chemins différents, en utilisant un Ingress.

## Étape 1 : Activer l'Addon Ingress sur Minikube
Minikube nécessite que l'on active manuellement le contrôleur Ingress.

```bash

minikube addons enable ingress
```
Attendez quelques secondes que les pods du contrôleur Ingress démarrent dans le namespace ingress-nginx.

## Étape 2 : Déployer les Applications
Nous allons déployer deux applications : une pour voter et une pour voir les résultats. Chacune aura son propre Deployment et son propre Service de type ClusterIP.

Créez le fichier vote-app.YAML :

```YAML

# vote-app.YAML
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vote-deployment
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
      - name: vote-app
        image: kodekloud/examplevotingapp_vote:v1
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: vote-service
spec:
  type: ClusterIP
  selector:
    app: vote
  ports:
  - port: 80
    targetPort: 80
```
Créez le fichier result-app.YAML :

```YAML

# result-app.```YAML
apiVersion: apps/v1
kind: Deployment
metadata:
  name: result-deployment
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
      - name: result-app
        image: kodekloud/examplevotingapp_result:v1
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: result-service
spec:
  type: ClusterIP
  selector:
    app: result
  ports:
  - port: 80
    targetPort: 80

```
Appliquez les deux manifestes :

```bash

kubectl apply -f vote-app.YAML
kubectl apply -f result-app.YAML
```
## Étape 3 : Configurer le DNS Local
Pour que notre navigateur sache où trouver vote.local, nous devons le faire pointer vers l'IP de notre cluster Minikube.

Obtenez l'adresse IP de Minikube :

```bash

minikube ip
```
Modifiez votre fichier hosts local. Sur votre propre machine (pas la VM), ouvrez le fichier /etc/hosts (Linux/macOS) ou C:\Windows\System32\drivers\etc\hosts (Windows) avec des droits d'administrateur et ajoutez la ligne suivante, en remplaçant <IP_DE_MINIKUBE> par l'IP obtenue à l'étape précédente :

<IP_DE_MINIKUBE> vote.local
## Étape 4 : Déployer la Ressource Ingress

Créez le fichier ingress-example.AML qui contient les règles de routage.

```YAML

# ingress-example.YAML
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: webapp-ingress
spec:
  rules:
  - host: "vote.local"
    http:
      paths:
      - path: /result
        pathType: Prefix
        backend:
          service:
            name: result-service
            port:
              number: 80
      - path: /
        pathType: Prefix
        backend:
          service:
            name: vote-service
            port:
              number: 80
```

Appliquez ce fichier :

```bash

kubectl apply -f ingress-example.YAML
```
## Étape 5 : Tester le Routage
Ouvrez votre navigateur web local.

Allez sur http://vote.local → Vous devriez voir l'application de vote.

Allez sur http://vote.local/result → Vous devriez voir la page des résultats.

Succès! L'Ingress Controller a bien routé le trafic vers le bon service en fonction de l'URL, le tout sur le port 80 standard.
