#  Lab 1 : Mettre en Place des Sondes de Santé (Health Probes)
Objectif : Déployer un Pod avec des sondes livenessProbe et readinessProbe pour observer comment Kubernetes gère la santé et la disponibilité d'une application.

## Étape 1 : Créer le Manifeste du Pod avec Sondes
Créez un fichier nommé pod-probes-lab.yaml. Ce Pod utilise une image de test qui expose des endpoints pour simuler des pannes.

```YAML

# pod-probes-lab.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-probes
spec:
  containers:
  - name: app
    image: kodekloud/simple-webapp-health
    ports:
    - containerPort: 8080
    # Sonde de Vivacité : si elle échoue, le conteneur est redémarré.
    livenessProbe:
      httpGet:
        path: /api/health
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
    # Sonde de Disponibilité : si elle échoue, le pod ne reçoit plus de trafic.
    readinessProbe:
      httpGet:
        path: /api/ready
        port: 8080
      initialDelaySeconds: 10
      periodSeconds: 5
```

##  Étape 2 : Déployer et Observer le Scénario "Tout va Bien"
Appliquez le manifeste :

```Bash

kubectl apply -f pod-probes-lab.yaml
```
Observez le statut du Pod. Il va passer par plusieurs états.

```Bash

kubectl get pods -w
```
Vous verrez d'abord le Pod Running mais 0/1 READY. Après le délai de la readinessProbe, il passera à 1/1 READY.

## Étape 3 : Simuler une Panne de Vivacité (Liveness)
Nous allons maintenant simuler une condition où l'application est bloquée (deadlock).

Exécutez une commande dans le Pod pour corrompre son état de santé :

```Bash

kubectl exec pod-probes -- curl -X POST http://localhost:8080/api/health/corrupt
```
Observez le Pod attentivement :

```Bash

kubectl get pods -w
```
Après quelques secondes, vous verrez le statut du Pod changer, et le compteur RESTARTS passera à 1. La sonde livenessProbe a échoué, et Kubernetes a redémarré le conteneur pour le "réparer".

## Étape 4 : Simuler une Indisponibilité (Readiness)
Simulons maintenant un cas où l'application est surchargée et ne peut plus accepter de nouvelles requêtes.

Attendez que le Pod soit à nouveau Running et 1/1 READY.

Rendez l'application "non prête" :

```Bash

kubectl exec pod-probes -- curl -X POST http://localhost:8080/api/ready/false
```
Observez à nouveau le statut du Pod :

```Bash

kubectl get pods -w
```
Le statut READY va passer de 1/1 à 0/1. Le Pod est toujours Running (le compteur RESTARTS n'augmente pas), mais Kubernetes sait qu'il ne faut plus lui envoyer de trafic.

## Étape 5 : Nettoyage
```Bash

kubectl delete -f pod-probes-lab.yaml
```
