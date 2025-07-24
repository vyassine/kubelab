Absolument. Voici le laboratoire complet, réorganisé étape par étape de manière très détaillée pour créer les permissions RBAC et le fichier `kubeconfig` associé.

-----

### **Laboratoire : Créer un Accès Restreint avec RBAC et `kubeconfig`**

**Objectif :** Fournir un accès sécurisé et limité à un utilisateur (représenté par un `ServiceAccount`) pour qu'il puisse créer des pods uniquement dans un namespace spécifique, en utilisant son propre fichier `kubeconfig`.

-----

### **Étape 1 : Préparation de l'Environnement**

Dans cette première étape, nous créons le namespace de travail et le compte de service qui représentera notre utilisateur.

1.  **Créez un fichier** nommé `1-setup.yaml` avec le contenu suivant :

    ```yaml
    # 1-setup.yaml
    apiVersion: v1
    kind: Namespace
    metadata:
      name: dev
    ---
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: dev-sa
      namespace: dev
    ```

2.  **Appliquez ce fichier** pour créer les ressources dans votre cluster :

    ```bash
    kubectl apply -f 1-setup.yaml
    ```

    **Résultat :** Le namespace `dev` et le `ServiceAccount` `dev-sa` sont maintenant créés.

-----

### **Étape 2 : Définition des Permissions avec un `Role`**

Nous définissons maintenant ce que le `ServiceAccount` aura le droit de faire. Ce `Role` n'existera que dans le namespace `dev`.

1.  **Créez un fichier** nommé `2-role.yaml` :

    ```yaml
    # 2-role.yaml
    apiVersion: rbac.authorization.k8s.io/v1
    kind: Role
    metadata:
      name: pod-creator-role
      namespace: dev
    rules:
    - apiGroups: [""] # "" représente l'API group "core" de Kubernetes
      resources: ["pods"]
      verbs: ["create", "get", "list"] # Uniquement le droit de créer, lister et voir les pods
    ```

2.  **Appliquez ce fichier** pour créer le rôle :

    ```bash
    kubectl apply -f 2-role.yaml
    ```

    **Résultat :** Un rôle nommé `pod-creator-role` existe maintenant, mais il n'est encore assigné à personne.

-----

### **Étape 3 : Attribution des Permissions avec un `RoleBinding`**

C'est l'étape cruciale où nous lions le `ServiceAccount` (`dev-sa`) au `Role` (`pod-creator-role`).

1.  **Créez un fichier** nommé `3-binding.yaml` :

    ```yaml
    # 3-binding.yaml
    apiVersion: rbac.authorization.k8s.io/v1
    kind: RoleBinding
    metadata:
      name: dev-can-create-pods
      namespace: dev
    subjects:
    - kind: ServiceAccount
      name: dev-sa
      namespace: dev
    roleRef:
      kind: Role
      name: pod-creator-role
      apiGroup: rbac.authorization.k8s.io
    ```

2.  **Appliquez ce fichier** pour lier le rôle à l'utilisateur :

    ```bash
    kubectl apply -f 3-binding.yaml
    ```

    **Résultat :** Le `ServiceAccount` `dev-sa` a maintenant les permissions définies dans `pod-creator-role`, et ce, uniquement à l'intérieur du namespace `dev`.

-----

### **Étape 4 : Création du Fichier `kubeconfig`**

Nous allons maintenant générer le fichier de connexion portable pour notre `dev-sa`.

1.  **Ouvrez votre terminal.** Nous allons exécuter une série de commandes pour collecter les informations nécessaires.

2.  **Générez un token d'authentification** pour `dev-sa` et stockez-le dans une variable :

    ```bash
    export DEV_SA_TOKEN=$(kubectl create token dev-sa --namespace dev)
    ```

3.  **Récupérez les informations de votre cluster** et stockez-les dans des variables :

    ```bash
    export CLUSTER_SERVER=$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}')
    export CLUSTER_CA_DATA=$(cat $(kubectl config view --raw -o jsonpath='{.clusters[0].cluster.certificate-authority}') | base64 -w 0)
    export CLUSTER_NAME=$(kubectl config view --minify -o jsonpath='{.clusters[0].name}')
    ```

4.  **Assemblez le fichier `kubeconfig`** en utilisant les variables que vous venez de créer. Exécutez cette commande complète :

    ```bash
    cat <<EOF > dev-kubeconfig.yaml
    apiVersion: v1
    kind: Config
    preferences: {}
    clusters:
    - name: ${CLUSTER_NAME}
      cluster:
        server: ${CLUSTER_SERVER}
        certificate-authority-data: ${CLUSTER_CA_DATA}
    users:
    - name: dev-sa
      user:
        token: ${DEV_SA_TOKEN}
    contexts:
    - name: dev-context
      context:
        cluster: ${CLUSTER_NAME}
        user: dev-sa
        namespace: dev
    current-context: dev-context
    EOF
    ```

    **Résultat :** Un fichier nommé `dev-kubeconfig.yaml` a été créé dans votre répertoire. Il contient toutes les informations nécessaires pour que `dev-sa` puisse se connecter.

-----

### **Étape 5 : Test du `kubeconfig` et des Permissions**

Vérifions que notre fichier fonctionne et que les permissions sont correctement appliquées.

1.  **Testez la création de pod dans le namespace `dev` (doit réussir) :**

    ```bash
    kubectl --kubeconfig=dev-kubeconfig.yaml run nginx-test --image=nginx
    ```

      * **Vérification :**
        ```bash
        kubectl --kubeconfig=dev-kubeconfig.yaml get pods
        ```
        Vous devez voir le pod `nginx-test` en cours d'exécution.

2.  **Testez la création de pod dans le namespace `default` (doit échouer) :**

    ```bash
    kubectl --kubeconfig=dev-kubeconfig.yaml run nginx-fail --image=nginx --namespace=default
    ```

    **Résultat attendu :** Une erreur `Forbidden`, car les droits ne s'appliquent pas au namespace `default`.

3.  **Testez la suppression de pod dans le namespace `dev` (doit échouer) :**

    ```bash
    kubectl --kubeconfig=dev-kubeconfig.yaml delete pod nginx-test
    ```

    **Résultat attendu :** Une erreur `Forbidden`, car le rôle n'inclut pas le verbe `delete`.

-----

### **Étape 6 : Nettoyage**

Pour supprimer toutes les ressources créées durant ce laboratoire :

1.  **Supprimez le namespace `dev`**, ce qui supprimera également le `ServiceAccount`, le `Role`, le `RoleBinding` et le pod de test :

    ```bash
    kubectl delete namespace dev
    ```

2.  **Supprimez les fichiers YAML locaux** que vous avez créés :

    ```bash
    rm 1-setup.yaml 2-role.yaml 3-binding.yaml dev-kubeconfig.yaml
    ```
