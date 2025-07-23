# Lab 4.1 : Gérer la Configuration et les Secrets
Objectif : Apprendre à externaliser la configuration et les données sensibles d'une application en utilisant des ConfigMaps et des Secrets, puis à les injecter dans un Pod.

## Étape 1 : Créer un ConfigMap et un Secret
Nous allons créer les objets de manière impérative, ce qui est rapide et efficace pour des données simples.

Créez un ConfigMap pour stocker la configuration non sensible :

```bash

kubectl create configmap app-config \
  --from-literal=LOG_LEVEL=info \
  --from-literal=API_ENDPOINT=http://api.service.local

```
Créez un Secret pour stocker les données sensibles. kubectl s'occupe de l'encodage Base64 pour nous.

```bash

kubectl create secret generic db-credentials \
  --from-literal=user=admin \
  --from-literal=password='S3cr3tP@ssw0rd'
```

## Étape 2 : Créer un Pod qui Consomme la Configuration
Créez un fichier nommé pod-config-lab.yaml. Ce Pod va injecter les données de deux manières : en tant que variables d'environnement et en tant que fichiers dans un volume.

```YAML

# pod-config-lab.yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod-config
spec:
  containers:
  - name: app
    image: busybox
    # Commande pour garder le conteneur en vie
    command: ["sleep", "3600"]
    # Méthode 1: Injection en tant que variables d'environnement
    env:
      # Injecter une clé spécifique du ConfigMap
      - name: LOG_LEVEL_FROM_CM
        valueFrom:
          configMapKeyRef:
            name: app-config
            key: LOG_LEVEL
      # Injecter une clé spécifique du Secret
      - name: DB_USER_FROM_SECRET
        valueFrom:
          secretKeyRef:
            name: db-credentials
            key: user
    # Méthode 2: Injection en tant que volume monté
    volumeMounts:
    - name: secret-volume
      mountPath: "/etc/secrets"
      readOnly: true
  volumes:
  - name: secret-volume
    secret:
      secretName: db-credentials

```


## Étape 3 : Déployer et Vérifier
Appliquez le manifeste du Pod :

```bash

kubectl apply -f pod-config-lab.yaml
```
Vérifiez les variables d'environnement injectées :

```bash

# Vérifier la valeur venant du ConfigMap
kubectl exec test-pod-config -- env | grep LOG_LEVEL_FROM_CM


# Vérifier la valeur venant du Secret
kubectl exec test-pod-config -- env | grep DB_USER_FROM_SECRET
```
Vérifiez le volume monté à partir du Secret :

```bash

# Lister les fichiers dans le volume
kubectl exec test-pod-config -- ls /etc/secrets

# Lire le contenu du fichier 'password'
kubectl exec test-pod-config -- cat /etc/secrets/password
```

