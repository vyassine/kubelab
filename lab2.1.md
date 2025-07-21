## Lab 2.1 : Gérer une Application avec un Deployment
Objectif : Abandonner la gestion manuelle des Pods. Nous allons utiliser un Deployment pour déployer une application, la mettre à l'échelle, effectuer une mise à jour sans interruption de service, et revenir en arrière en cas de problème.

## Étape 1 : Créer le Manifeste du Deployment
Créez un fichier nommé deployment-nginx.yaml avec le contenu suivant. Ce manifeste demande à Kubernetes de s'assurer que 3 répliques de notre Pod Nginx tournent en permanence.

```

# deployment-nginx.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25.3
        ports:
        - containerPort: 80
```
## Étape 2 : Déployer et Observer la Hiérarchie
Appliquez le manifeste pour créer le Deployment :

```Bash

kubectl apply -f deployment-nginx.yaml
```
Vérifiez la création des objets à chaque niveau de la hiérarchie :



1. Vérifiez le Deployment 
```
kubectl get deployment
```
2. Vérifiez le ReplicaSet créé par le Deployment
```
kubectl get rs
```

3. Vérifiez les Pods créés par le ReplicaSet
```
kubectl get pods
```
 Le Deployment a délégué au ReplicaSet la tâche de maintenir 3 Pods.

## Étape 3 : Mettre à l'Échelle (Scaler) l'Application
Nous allons maintenant demander au Deployment de passer de 3 à 5 répliques.

Utilisez la commande scale :

```Bash

kubectl scale deployment nginx-deployment --replicas=5
```
Vérifiez le résultat immédiatement :

```Bash

kubectl get pods
```
Vous devriez voir que Kubernetes a créé 2 nouveaux Pods pour atteindre l'état désiré de 5 répliques.

## Étape 4 : Effectuer une Mise à Jour sans Interruption (Rolling Update)
Déployons une nouvelle version de Nginx (1.26.1) sans que les utilisateurs ne subissent de coupure.

Mettez à jour l'image du conteneur dans le Deployment :

```Bash

kubectl set image deployment/nginx-deployment nginx=nginx:1.26.1
```
Observez la mise à jour en direct. C'est le moment le plus instructif!



Cette commande suivra le statut du déploiement jusqu'à son terme
```
kubectl rollout status deployment/nginx-deployment
```
Pendant ce temps, dans un autre terminal, vous pouvez exécuter kubectl get pods -w pour voir les nouveaux Pods être créés et les anciens être terminés un par un.

Vérifiez l'historique des déploiements :

```Bash

kubectl rollout history deployment/nginx-deployment
```
## Étape 5 : Revenir en Arrière (Rollback)
La nouvelle version a un bug! Nous devons revenir à la version précédente de toute urgence.

Lancez la commande de retour en arrière :

```Bash

kubectl rollout undo deployment/nginx-deployment
```
Vérifiez que le rollback s'est bien passé :

```Bash

kubectl rollout status deployment/nginx-deployment
```
Prouvez que l'ancienne version est de retour. Prenez le nom d'un des Pods et inspectez son image :



# Remplacez <nom-d-un-pod> par un nom de pod obtenu avec 'kubectl get pods'
```
kubectl describe pod <nom-d-un-pod> | grep Image
```
La version de l'image doit être revenue à nginx:1.25.3.

