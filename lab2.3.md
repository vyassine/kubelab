# Lab 2.3 : Isoler une Application dans un Namespace
Objectif : Apprendre à utiliser les Namespaces pour créer des "clusters virtuels" et isoler des environnements.

## Étape 1 : Nettoyage
Supprimons d'abord les ressources créées dans le namespace default.

```Bash

kubectl delete deployment nginx-deployment
kubectl delete service nginx-service
```
##  Étape 2 : Créer un Namespace
Créons un nouvel espace de travail nommé production.

```Bash

kubectl create namespace production
```
## Étape 3 : Déployer les Ressources dans le Nouveau Namespace
Modifiez deployment-nginx.yaml pour y ajouter namespace: production :

```YAML

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: production # <-- AJOUTER CETTE LIGNE
spec:
  #... le reste du fichier est identique
```
Modifiez service-nginx.yaml de la même manière :

```YAML

apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: production # <-- AJOUTER CETTE LIGNE
spec:
  #... le reste du fichier est identique
```
Appliquez les fichiers modifiés :

```Bash

kubectl apply -f deployment-nginx.yaml
kubectl apply -f service-nginx.yaml
```
## Étape 4 : Vérifier l'Isolation
Listez les pods dans le namespace par défaut :

```Bash

kubectl get pods
```
Résultat attendu : No resources found in default namespace.

Listez toutes les ressources dans le namespace production :

```Bash

kubectl get all -n production
```
Vous verrez maintenant toutes vos ressources, bien isolées dans le namespace production.

## Étape 5 : Nettoyage Final
Supprimez le namespace pour nettoyer toutes les ressources du projet en une seule commande.

```Bash

kubectl delete namespace production
```
