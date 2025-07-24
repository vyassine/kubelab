Pour garder notre environnement de test propre et isolé, nous allons d'abord créer un namespace dédié.

1.  **Créez le fichier `qos-lab-namespace.yaml` :**

    ```yaml
    apiVersion: v1
    kind: Namespace
    metadata:
      name: qos-lab
    ```

2.  **Appliquez le manifeste :**

    ```bash
    kubectl apply -f qos-lab-namespace.yaml
    ```

Toutes les ressources suivantes seront créées dans ce namespace `qos-lab`.

-----

### **Lab 1 : Les Classes de Qualité de Service (QoS) au niveau du Pod**

**Objectif :** Comprendre comment Kubernetes assigne une classe de QoS à un Pod en fonction de ses demandes (`requests`) et limites (`limits`) de ressources, et comprendre la priorité d'éviction de chaque classe. [1]

Nous allons créer trois Pods, un pour chaque classe de QoS.

1.  **Créez le fichier `qos-pods.yaml` :**
    Ce fichier contient les définitions pour les trois classes.

    ```yaml
    # --- Pod de classe "Guaranteed" ---
    # requests et limits sont identiques. C'est la plus haute priorité.
    # Il sera tué en dernier en cas de pénurie de ressources. [2]
    apiVersion: v1
    kind: Pod
    metadata:
      name: qos-guaranteed
      namespace: qos-lab
    spec:
      containers:
      - name: main
        image: nginx
        resources:
          requests:
            memory: "100Mi"
            cpu: "100m"
          limits:
            memory: "100Mi"
            cpu: "100m"
    ---
    # --- Pod de classe "Burstable" ---
    # requests est inférieur à limits. C'est la priorité intermédiaire.
    # Il sera tué après les Pods BestEffort. [1]
    apiVersion: v1
    kind: Pod
    metadata:
      name: qos-burstable
      namespace: qos-lab
    spec:
      containers:
      - name: main
        image: nginx
        resources:
          requests:
            memory: "100Mi"
            cpu: "100m"
          limits:
            memory: "200Mi"
            cpu: "200m"
    ---
    # --- Pod de classe "BestEffort" ---
    # Aucune request ni limit n'est spécifiée. C'est la plus basse priorité.
    # Il sera le premier à être tué en cas de pénurie de ressources. [3]
    apiVersion: v1
    kind: Pod
    metadata:
      name: qos-besteffort
      namespace: qos-lab
    spec:
      containers:
      - name: main
        image: nginx
    ```

2.  **Déployez les Pods :**

    ```bash
    kubectl apply -f qos-pods.yaml
    ```

3.  **Vérifiez la classe de QoS de chaque Pod :**
    Utilisez la commande `kubectl describe` pour inspecter chaque pod et trouver le champ `QoS Class`.

    ```bash
    # Vérifier le pod Guaranteed
    kubectl describe pod qos-guaranteed -n qos-lab | grep "QoS Class"

    # Vérifier le pod Burstable
    kubectl describe pod qos-burstable -n qos-lab | grep "QoS Class"

    # Vérifier le pod BestEffort
    kubectl describe pod qos-besteffort -n qos-lab | grep "QoS Class"
    ```

    **Résultats attendus :**

    ```
    QoS Class:       Guaranteed
    QoS Class:       Burstable
    QoS Class:       BestEffort
    ```

**Conclusion du Lab 1 :** Vous avez vu comment la simple définition des champs `requests` et `limits` dans le manifeste d'un pod détermine sa classe de QoS. [1] Cette classe est cruciale car elle informe le planificateur de la priorité du pod, ce qui a un impact direct sur sa stabilité en cas de contention des ressources sur un nœud. [2, 4]

-----

### **Lab 2 : Limiter les Ressources d'un Namespace avec `ResourceQuota`**

**Objectif :** Démontrer comment un `ResourceQuota` peut imposer des limites sur la consommation totale de ressources (CPU, mémoire) pour un namespace entier, ce qui est essentiel dans les environnements multi-locataires. [5, 6]

1.  **Créez le fichier `quota.yaml` :**
    Nous allons définir un quota strict pour notre namespace `qos-lab`.

    ```yaml
    apiVersion: v1
    kind: ResourceQuota
    metadata:
      name: lab-quota
      namespace: qos-lab
    spec:
      hard:
        requests.cpu: "500m"   # Total des CPU demandés ne peut pas dépasser 0.5 core
        requests.memory: "500Mi"  # Total de la mémoire demandée ne peut pas dépasser 500 MiB
        limits.cpu: "1"         # Total des limites CPU ne peut pas dépasser 1 core
        limits.memory: "1Gi"      # Total des limites mémoire ne peut pas dépasser 1 GiB
        pods: "5"               # Nombre maximum de pods dans le namespace
    ```

2.  **Appliquez le quota au namespace :**

    ```bash
    kubectl apply -f quota.yaml
    ```

3.  **Vérifiez le quota :**
    Inspectez le quota pour voir les limites appliquées et l'utilisation actuelle (qui devrait être de 0).

    ```bash
    kubectl describe resourcequota lab-quota -n qos-lab
    ```

4.  **Testez le quota en créant un pod qui le respecte :**
    Créez un fichier `pod-respecte-quota.yaml`. Ce pod demande 250m de CPU et 250Mi de mémoire, ce qui est bien en dessous des limites du quota.

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: pod-respecte-quota
      namespace: qos-lab
    spec:
      containers:
      - name: main
        image: nginx
        resources:
          requests:
            memory: "250Mi"
            cpu: "250m"
          limits:
            memory: "500Mi"
            cpu: "500m"
    ```

    Appliquez-le. La création devrait réussir.

    ```bash
    kubectl apply -f pod-respecte-quota.yaml
    # Résultat attendu : pod/pod-respecte-quota created
    ```

5.  **Testez le quota en essayant de créer un pod qui le viole :**
    Maintenant, créez un fichier `pod-viole-quota.yaml`. Ce pod demande 400Mi de mémoire. Comme le pod précédent en utilise déjà 250Mi, le total (650Mi) dépasserait la limite de 500Mi du quota.

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: pod-viole-quota
      namespace: qos-lab
    spec:
      containers:
      - name: main
        image: nginx
        resources:
          requests:
            memory: "400Mi"
            cpu: "100m"
    ```

    Essayez de l'appliquer. La création devrait échouer.

    ```bash
    kubectl apply -f pod-viole-quota.yaml
    ```

    **Résultat attendu :** Vous recevrez une erreur claire indiquant que le quota a été dépassé.
    `Error from server (Forbidden): error when creating "pod-viole-quota.yaml": pods "pod-viole-quota" is forbidden: exceeded quota: lab-quota, requested: requests.memory=400Mi, used: requests.memory=250Mi, limited: requests.memory=500Mi`

**Conclusion du Lab 2 :** Les `ResourceQuotas` sont un outil puissant pour les administrateurs afin de garantir un partage équitable des ressources du cluster entre différentes équipes ou projets, empêchant un seul namespace de monopoliser les ressources. [7, 8]

-----

### **Lab 3 : Démonstration d'un `OOMKilled` (Out of Memory)**

**Objectif :** Provoquer intentionnellement une erreur `OOMKilled` pour voir comment Kubernetes protège un nœud en tuant un conteneur qui dépasse sa limite de mémoire. [9, 10]

1.  **Créez le fichier `oom-pod.yaml` :**
    Nous allons déployer un pod avec une limite de mémoire très basse et utiliser une image de test de stress pour consommer plus de mémoire que ce qui est autorisé. L'image `polinux/stress` est parfaite pour cela. [11, 12]

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: oom-pod
      namespace: qos-lab
    spec:
      containers:
      - name: stress-container
        image: polinux/stress
        resources:
          requests:
            memory: "50Mi"
          limits:
            memory: "100Mi" # Limite stricte de 100 MiB
        # La commande demande au conteneur d'allouer 200 Mo de mémoire,
        # ce qui est bien au-dessus de notre limite de 100 MiB.
        command: ["stress"]
        args: ["--vm", "1", "--vm-bytes", "200M", "--vm-hang", "1"]
    ```

2.  **Déployez le pod et observez son cycle de vie :**
    Utilisez l'option `-w` (watch) pour voir les changements de statut en temps réel.

    ```bash
    kubectl get pod oom-pod -n qos-lab -w
    ```

3.  **Observation du comportement :**
    Vous observerez la séquence suivante :

      * Le pod passe à l'état `ContainerCreating`, puis `Running`.
      * Après quelques secondes, le statut changera brusquement en `OOMKilled`.
      * Kubernetes tentera de le redémarrer, et le pod entrera dans une boucle `CrashLoopBackOff` car il sera tué à chaque fois qu'il tentera de dépasser sa limite de mémoire.

4.  **Inspectez les détails du pod pour confirmer la cause :**
    Arrêtez la commande `watch` (Ctrl+C) et décrivez le pod.

    ```bash
    kubectl describe pod oom-pod -n qos-lab
    ```

    Faites défiler jusqu'en bas de la sortie. Dans la section `Containers` \> `Last State`, vous verrez clairement la raison de la dernière terminaison :

    ```
    Last State:     Terminated
      Reason:       OOMKilled
      Exit Code:    137
    ```

    L'**Exit Code 137** est le signal standard envoyé par le noyau Linux pour indiquer un "Out of Memory Kill". [9, 10]

**Conclusion du Lab 3 :** Vous avez vu en action le mécanisme de protection de Kubernetes. En tuant un conteneur qui ne respecte pas sa limite de mémoire, Kubernetes empêche une seule application défaillante de consommer toute la mémoire d'un nœud et de provoquer une instabilité pour toutes les autres applications qui y sont hébergées. [13, 14]

-----

### **Nettoyage**

Pour supprimer toutes les ressources créées pendant ces ateliers, supprimez simplement le namespace.

```bash
kubectl delete namespace qos-lab
```
