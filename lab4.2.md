# Lab 4.2 : Utiliser une Image d'un Registre Privé
Objectif : Déployer une application en utilisant une image stockée dans un référentiel privé sur Docker Hub.

##  Étape 0 : Prérequis (à faire une seule fois)
Créez un token d'accès sur Docker Hub avec des permissions de lecture (Read-only).

Créez un référentiel (repository) privé sur votre compte Docker Hub (ex: mon-repo-prive).

Sur votre VM Ubuntu, préparez et poussez une image vers ce référentiel :

```bash

# 1. Connectez-vous à Docker Hub (utilisez votre token comme mot de passe)
docker login --username <votre-user-dockerhub>

# 2. Tirez une image simple pour la démo
docker pull nginx:alpine

# 3. Retaguez-la pour votre repo privé
docker tag nginx:alpine <votre-user-dockerhub>/mon-repo-prive:v1

# 4. Poussez-la vers votre repo privé
docker push <votre-user-dockerhub>/mon-repo-prive:v1

```

## Étape 1 : Créer le imagePullSecret dans Kubernetes
Créez le Secret qui contient vos identifiants Docker Hub.

```bash

kubectl create secret docker-registry mon-secret-dockerhub \
  --docker-username='<votre-user-dockerhub>' \
  --docker-password='<votre-token-d-acces>'

```
## Étape 2 : Créer le Pod qui utilise l'Image Privée
Créez un fichier pod-prive.yaml.

```YAML

# pod-prive.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-prive-test
spec:
  containers:
  - name: app
    # Remplacez par le chemin de VOTRE image privée
    image: <votre-user-dockerhub>/mon-repo-prive:v1
  # On indique à Kubernetes quel secret utiliser pour l'authentification
  imagePullSecrets:
  - name: mon-secret-dockerhub

```
## Étape 3 : Déployer et Vérifier
Appliquez le manifeste du Pod :

```bash

kubectl apply -f pod-prive.yaml
```
Attendez quelques secondes, puis décrivez le Pod pour vérifier les événements :

```bash

kubectl describe pod pod-prive-test
```
Regardez dans la section Events. Vous devriez voir un message Successfully pulled image..., confirmant que l'authentification a fonctionné
