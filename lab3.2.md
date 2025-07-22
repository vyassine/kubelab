# Lab 3.2 : Sécuriser une Application avec les Network Policies
Objectif : Isoler un Pod "backend" et n'autoriser que le Pod "frontend" à communiquer avec lui.

## Étape 1 : Déployer un Client et un Serveur
Déployez un pod "serveur" Nginx avec le label app=backend :

```Bash

kubectl run backend --image=nginx --labels=app=backend
```
Déployez un pod "client" BusyBox avec le label app=frontend :

```Bash

kubectl run frontend --image=busybox --labels=app=frontend -- sleep 3600
```
## Étape 2 : Vérifier la Communication (Avant la Politique)
Obtenez l'adresse IP du pod backend :

```Bash

BACKEND_IP=$(kubectl get pod backend -o jsonpath='{.status.podIP}')
```
Essayez de contacter le backend depuis le frontend :

```Bash

kubectl exec frontend -- wget -O- $BACKEND_IP
```
Résultat attendu : La page HTML de Nginx s'affiche. La communication est ouverte.

## Étape 3 : Appliquer une Politique "Deny-All"

Créez le fichier netpol-deny-all.yaml pour bloquer tout le trafic entrant par défaut.

```YAML

# netpol-deny-all.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
spec:
  podSelector: {}
  policyTypes:
  - Ingress

```
Appliquez-le et revérifiez la communication :

```Bash

kubectl apply -f netpol-deny-all.yaml
kubectl exec frontend -- wget -O- $BACKEND_IP --timeout=5
```
Résultat attendu : La commande échoue avec un "timeout". Le trafic est maintenant bloqué.

## Étape 4 : Autoriser le Flux Spécifique
Créez le fichier netpol-allow-frontend.yaml pour créer une exception.

```YAML

# netpol-allow-frontend.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-traffic-to-backend
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
```
Appliquez-le et vérifiez une dernière fois :

```Bash

kubectl apply -f netpol-allow-frontend.yaml
kubectl exec frontend -- wget -O- $BACKEND_IP
```
Résultat attendu : La page HTML s'affiche à nouveau. Nous avons créé une règle de pare-feu ciblée.
