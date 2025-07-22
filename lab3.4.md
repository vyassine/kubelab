# Lab 3.4 : Déployer une Application Stateful
Objectif : Utiliser un StatefulSet pour automatiser la création de PVCs pour chaque réplica d'une application.

## Étape 1 : Créer le Service Headless
Un StatefulSet a besoin d'un Service "headless" (sans tête) pour gérer les identités réseau de ses Pods. Créez le fichier headless-service.yaml :

```YAML

# headless-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-headless
spec:
  clusterIP: None
  selector:
    app: nginx
  ports:
    - port: 80
```
## Étape 2 : Créer le StatefulSet
Créez le fichier statefulset-example.yaml. Notez la section volumeClaimTemplates qui sert de modèle pour créer une PVC pour chaque Pod.

```YAML

# statefulset-example.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  selector:
    matchLabels:
      app: nginx
  serviceName: "nginx-headless"
  replicas: 3
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes:
      storageClassName: "standard"
      resources:
        requests:
          storage: 1Gi

```
## Étape 3 : Déployer et Observer
Appliquez les deux manifestes :

```bash

kubectl apply -f headless-service.yaml
kubectl apply -f statefulset-example.yaml
```
Observez la création ordonnée des Pods :

```bash

kubectl get pods -w
```
Vous verrez web-0 être créé et devenir Running AVANT que web-1 ne commence, et ainsi de suite.

Observez la création automatique des PVCs :

```bash

kubectl get pvc
```
Vous verrez 3 PVCs créées automatiquement : www-web-0, www-web-1, et www-web-2.
