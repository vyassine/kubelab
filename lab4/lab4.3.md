#  Lab 4.3 : Configurer des Permissions RBAC
Objectif : Créer un compte de service app-monitoring avec des droits limités, lui permettant uniquement de lire les Pods et les Services.

##  Étape 1 : Créer le ServiceAccount
C'est l'identité que nous allons restreindre.

```bash

kubectl create serviceaccount app-monitoring
```

##  Étape 2 : Créer le Role
Créez un fichier monitoring-role.yaml pour définir les permissions.

```YAML

# monitoring-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: monitoring-role
rules:
- apiGroups: [""] # "" indique l'API core de Kubernetes
  resources: ["pods", "services"]
  verbs: ["get", "list", "watch"]
```

Appliquez-le :

```bash

kubectl apply -f monitoring-role.yaml
```

##  Étape 3 : Créer le RoleBinding
Créez un fichier monitoring-binding.yaml pour lier le ServiceAccount au Role.

```YAML

# monitoring-binding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: monitoring-binding
subjects:
- kind: ServiceAccount
  name: app-monitoring
  namespace: default
roleRef:
  kind: Role
  name: monitoring-role
  apiGroup: rbac.authorization.k8s.io
```
Appliquez-le :

```bash

kubectl apply -f monitoring-binding.yaml
```

##  Étape 4 : Tester les Permissions
Utilisez kubectl auth can-i pour vérifier ce que notre ServiceAccount a le droit de faire.

Vérifiez une permission autorisée :

```bash

# En tant que 'app-monitoring', puis-je lister les pods?
kubectl auth can-i list pods --as=system:serviceaccount:default:app-monitoring
```
Résultat attendu : yes

Vérifiez une permission non autorisée :

```bash

# En tant que 'app-monitoring', puis-je lister les secrets?
kubectl auth can-i list secrets --as=system:serviceaccount:default:app-monitoring
```
Résultat attendu : no
