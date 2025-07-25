Absolument. Voici les trois laboratoires sur le placement des Pods, présentés de manière directe avec les étapes et les fichiers YAML, comme les labs précédents.

-----

### **Lab 1 : Placement Simple avec `nodeSelector`**

**Objectif :** Forcer un Pod à s'exécuter sur un nœud spécifique en utilisant un `nodeSelector`.

#### Étape 1 : Étiqueter un Nœud

1.  Choisissez un de vos workers (ex: `multi-noeud-m02`).
2.  Appliquez une étiquette personnalisée :
    ```bash
    kubectl label nodes multi-noeud-m02 disktype=ssd
    ```
3.  Vérifiez que l'étiquette est bien présente avec `kubectl get nodes --show-labels`.

#### Étape 2 : Créer un Pod avec `nodeSelector`

1.  Créez le fichier `lab-nodeselector.yaml` :
    ```yaml
    # lab-nodeselector.yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: nginx-selector
    spec:
      containers:
      - name: nginx
        image: nginx
      nodeSelector:
        disktype: ssd
    ```

#### Étape 3 : Déployer et Vérifier

1.  Appliquez le fichier : `kubectl apply -f lab-nodeselector.yaml`.
2.  Vérifiez le placement du Pod :
    ```bash
    kubectl get pods -o wide
    ```
    **Observation :** Le Pod `nginx-selector` est obligatoirement placé sur le nœud étiqueté `multi-noeud-m02`.

#### Étape 4 : Nettoyage

```bash
kubectl delete -f lab-nodeselector.yaml
kubectl label nodes multi-noeud-m02 disktype-
```

-----

### **Lab 2 : Exclusion et Tolérance avec Taints & Tolerations**

**Objectif :** Rendre un nœud exclusif avec une "Taint", puis déployer un Pod avec une "Toleration" pour l'autoriser à s'y exécuter.

#### Étape 1 : Souiller (Taint) un Nœud

1.  Choisissez un worker (ex: `multi-noeud-m02`).
2.  Appliquez une Taint `NoSchedule` :
    ```bash
    kubectl taint nodes multi-noeud-m02 special-workload=true:NoSchedule
    ```
3.  Vérifiez la Taint avec `kubectl describe node multi-noeud-m02 | grep Taints`.

#### Étape 2 : Déployer un Pod sans Tolérance

1.  Créez le fichier `pod-sans-toleration.yaml` :
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: nginx-standard
    spec:
      containers:
      - name: nginx
        image: nginx
    ```
2.  Appliquez-le (`kubectl apply -f pod-sans-toleration.yaml`) et observez son placement (`kubectl get pods -o wide`).
    **Observation :** Le Pod est placé sur l'autre worker non souillé.

#### Étape 3 : Déployer un Pod avec la Tolérance

1.  Créez le fichier `pod-avec-toleration.yaml` :
    ```yaml
    # pod-avec-toleration.yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: nginx-special
    spec:
      containers:
      - name: nginx
        image: nginx
      tolerations:
      - key: "special-workload"
        operator: "Equal"
        value: "true"
        effect: "NoSchedule"
    ```
2.  Appliquez-le (`kubectl apply -f pod-avec-toleration.yaml`) et observez son placement (`kubectl get pods -o wide`).
    **Observation :** Ce Pod peut maintenant être placé sur le nœud souillé `multi-noeud-m02`.

#### Étape 4 : Nettoyage

```bash
kubectl delete pod nginx-standard nginx-special
kubectl taint nodes multi-noeud-m02 special-workload=true:NoSchedule-
```

-----

### **Lab 3 : Co-location et Séparation avec Pod Affinity / Anti-Affinity**

**Objectif :** Forcer un pod "frontend" à être sur le même nœud qu'un pod "backend" (Affinity), puis forcer un autre service à être sur un nœud différent (Anti-Affinity).

#### Étape 1 : Déployer le Pod de Référence (Backend)

1.  Créez le fichier `pod-backend.yaml` :
    ```yaml
    # pod-backend.yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: backend-pod
      labels:
        app: backend
    spec:
      containers:
      - name: busybox
        image: busybox:1.35
        command: ["sleep", "3600"]
    ```
2.  Appliquez-le (`kubectl apply -f pod-backend.yaml`) et notez sur quel nœud il se trouve avec `kubectl get pods -o wide`.

#### Étape 2 : Utiliser la Pod Affinity (co-location)

1.  Créez le fichier `pod-frontend-affinity.yaml` :
    ```yaml
    # pod-frontend-affinity.yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: frontend-pod
    spec:
      affinity:
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - {key: app, operator: In, values: ["backend"]}
            topologyKey: "kubernetes.io/hostname"
      containers:
      - {name: nginx, image: nginx}
    ```
2.  Appliquez-le (`kubectl apply -f pod-frontend-affinity.yaml`) et vérifiez avec `kubectl get pods -o wide`.
    **Observation :** `frontend-pod` est sur le même nœud que `backend-pod`.

#### Étape 3 : Utiliser la Pod Anti-Affinity (séparation)

1.  Créez le fichier `pod-other-service-anti-affinity.yaml` :
    ```yaml
    # pod-other-service-anti-affinity.yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: other-service-pod
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - {key: app, operator: In, values: ["backend"]}
            topologyKey: "kubernetes.io/hostname"
      containers:
      - {name: redis, image: redis}
    ```
2.  Appliquez-le (`kubectl apply -f pod-other-service-anti-affinity.yaml`) et vérifiez avec `kubectl get pods -o wide`.
    **Observation :** `other-service-pod` est sur un nœud **différent** de `backend-pod`.

#### Étape 4 : Nettoyage

```bash
kubectl delete pod backend-pod frontend-pod other-service-pod
```
