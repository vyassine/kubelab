# Lab 2.2 : Le Deployment et les Init Containers 
Objectif : Déployer une application de manière résiliente, la mettre à l'échelle, et utiliser un Init Container pour une tâche de préparation simple (comme une vérification ou une temporisation).

Contexte : Nous allons déployer une application web Nginx. Avant que le serveur web ne démarre, un Init Container va simuler une tâche de préparation (par exemple, attendre qu'un service externe soit prêt). Nous allons simuler cela avec une simple temporisation et un message dans les logs.

Instructions Pas à Pas :

##   Création du Deployment  :
Créez un fichier nommé lab2.2-deployment-corrige.yaml. Notez qu'il n'y a aucune section volumes ou volumeMounts.

```YAML

# lab2.2-deployment-corrige.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-deployment
spec:
  replicas: 2 # Nous démarrons avec 2 instances
  selector:
    matchLabels:
      app: webapp # Le sélecteur pour lier ce Deployment à ses Pods
  template:
    metadata:
      labels:
        app: webapp # Le label que les Pods porteront
    spec:
      initContainers:
      - name: init-preparation # Nom du conteneur d'initialisation
        image: busybox:1.35
        # Cette commande simule une tâche qui prend 5 secondes
        command: ['sh', '-c', 'echo "Tache de préparation en cours..."; sleep 5; echo "Préparation terminée !"']
      containers:
      - name: nginx-container
        image: nginx:1.21
        ports:
        - containerPort: 80
```
## Déploiement et Inspection :

```Bash

kubectl apply -f lab2.2-deployment-corrige.yaml
```
Vérifiez le statut du Deployment et des Pods :

```Bash

kubectl get deployment,replicaset,pods
```
Ce qu'il faut observer : Vous devez voir 1 Deployment, 1 ReplicaSet et 2 Pods passer à l'état Running. Cela peut prendre quelques secondes de plus à cause de la temporisation de 5 secondes de l'Init Container.

Inspectez un Pod pour voir l'état de l'Init Container :

```Bash

# Remplacez <nom-du-pod> par un des noms de vos Pods
kubectl describe pod <nom-du-pod>
```
Ce qu'il faut observer : Dans la section Init Containers, vous verrez que init-preparation s'est terminé avec succès (State: Terminated, Reason: Completed). Ceci prouve qu'il a bien tourné avant le conteneur nginx-container.

Consultez les logs de l'Init Container :
Pour voir la sortie de notre commande, nous devons cibler spécifiquement les logs du conteneur d'initialisation avec l'option -c.

```Bash

# Remplacez <nom-du-pod> par le même nom de Pod
kubectl logs <nom-du-pod> -c init-preparation
```
Ce qu'il faut observer : Vous devriez voir les messages :

Tache de préparation en cours...
Préparation terminée !
Cela confirme que notre tâche simulée a bien été exécutée.
