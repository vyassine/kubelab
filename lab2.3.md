# Lab 2.2 : Exposer une Application avec un Service
Objectif : Rendre notre application Nginx, gérée par le Deployment, accessible depuis l'extérieur de notre cluster.

## Étape 1 : Créer le Manifeste du Service
Créez un fichier nommé service-nginx.yaml. Nous utilisons un type NodePort pour exposer un port sur la VM.

```YAML

# service-nginx.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```
## Étape 2 : Déployer le Service et y Accéder
Appliquez le manifeste du Service :

```Bash

kubectl apply -f service-nginx.yaml
```
Obtenez l'URL d'accès. Minikube fournit une commande très pratique pour cela :

```Bash

minikube service nginx-service --url
```
Cette commande va vous donner une URL, par exemple http://192.168.49.2:30007.

Ouvrez cette URL dans le navigateur de votre machine locale.

Résultat attendu : La page d'accueil "Welcome to nginx!" s'affiche.

