Le **Horizontal Pod Autoscaler (HPA)** est l'un des mécanismes les plus puissants de Kubernetes pour gérer la scalabilité de vos applications de manière automatique. Il permet à votre cluster de réagir dynamiquement aux variations de charge, en ajoutant ou en supprimant des répliques de pods pour correspondre à la demande.

Voici un atelier pratique complet pour vous faire manipuler et observer le HPA en action.

-----

### **Atelier Pratique : L'Autoscaling avec le Horizontal Pod Autoscaler (HPA)**

**Objectif :** Comprendre comment le HPA ajuste automatiquement le nombre de répliques d'un `Deployment` en fonction de la consommation de CPU, en simulant une augmentation de charge puis en observant le retour à la normale.

#### **Prérequis : Installation du Metrics Server**

Le HPA a besoin d'une source de métriques pour prendre ses décisions. Il ne peut pas deviner la charge de vos pods. Le composant standard pour cela est le **Metrics Server**, un service qui collecte les métriques de consommation de ressources (CPU, mémoire) de vos pods et nœuds.

Sur la plupart des environnements locaux comme Minikube, vous pouvez l'installer en tant qu'add-on :

```bash
minikube addons enable metrics-server
```

Après une minute ou deux, vérifiez que le Metrics Server est bien en cours d'exécution :

```bash
# Cette commande devrait afficher l'utilisation CPU/Mémoire de vos nœuds
kubectl top nodes
```

Si cette commande retourne des valeurs, vous êtes prêt. Si elle retourne une erreur, attendez encore un peu que le Metrics Server s'initialise.

#### **Étape 1 : Déployer une Application Cible**

Nous avons besoin d'une application à mettre sous charge. Nous allons déployer un serveur web simple, mais il y a un détail crucial : **pour que le HPA fonctionne, les conteneurs du Deployment doivent avoir des `requests` de ressources définies**. Le HPA utilise cette valeur de `request` comme référence pour calculer le pourcentage d'utilisation.

1.  **Créez le fichier `hpa-app-deployment.yaml` :**

    ```yaml
    apiVersion: v1
    kind: Namespace
    metadata:
      name: hpa-lab
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: php-apache
      namespace: hpa-lab
    spec:
      selector:
        matchLabels:
          run: php-apache
      replicas: 1 # Nous commençons avec une seule réplique
      template:
        metadata:
          labels:
            run: php-apache
        spec:
          containers:
          - name: php-apache
            image: k8s.gcr.io/hpa-example
            ports:
            - containerPort: 80
            resources:
              requests:
                cpu: 200m # Demande 200 millicores de CPU
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: php-apache
      namespace: hpa-lab
    spec:
      selector:
        run: php-apache
      ports:
      - protocol: TCP
        port: 80
    ```

2.  **Appliquez le manifeste :**

    ```bash
    kubectl apply -f hpa-app-deployment.yaml
    ```

#### **Étape 2 : Créer l'Objet HorizontalPodAutoscaler**

Maintenant, nous allons créer l'objet HPA qui surveillera notre `Deployment`.

1.  **Créez le fichier `hpa.yaml` :**
    Nous allons configurer le HPA pour qu'il maintienne une utilisation moyenne de CPU de 50%. S'il dépasse ce seuil, il ajoutera des pods, jusqu'à un maximum de 10.

    ```yaml
    apiVersion: autoscaling/v2
    kind: HorizontalPodAutoscaler
    metadata:
      name: php-apache-hpa
      namespace: hpa-lab
    spec:
      scaleTargetRef:
        apiVersion: apps/v1
        kind: Deployment
        name: php-apache # Le nom du Deployment à surveiller
      minReplicas: 1   # Nombre minimum de répliques
      maxReplicas: 10  # Nombre maximum de répliques
      metrics:
      - type: Resource
        resource:
          name: cpu
          target:
            type: Utilization
            averageUtilization: 50 # Cible : 50% de la CPU demandée (soit 100m)
    ```

2.  **Appliquez le manifeste :**

    ```bash
    kubectl apply -f hpa.yaml
    ```

3.  **Observez l'état du HPA :**
    Ouvrez un terminal et lancez une commande `watch` pour surveiller le HPA en temps réel.

    ```bash
    watch kubectl get hpa -n hpa-lab
    ```

    **Résultat initial attendu :**

    ```
    NAME             REFERENCE               TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
    php-apache-hpa   Deployment/php-apache   0%/50%    1         10        1          15s
    ```

    La colonne `TARGETS` montre l'utilisation actuelle par rapport à la cible (`0%/50%`). Comme il n'y a pas de charge, l'utilisation est nulle et le nombre de répliques (`REPLICAS`) est de 1, ce qui correspond à notre `minReplicas`.

#### **Étape 3 : Simuler une Charge CPU**

Pour voir le HPA en action, nous devons augmenter la consommation de CPU de notre application. Nous allons lancer un pod "générateur de charge" qui enverra des requêtes en boucle à notre service.

1.  **Lancez le pod générateur de charge :**
    Ouvrez un **second terminal** et exécutez la commande suivante. Elle crée un pod temporaire qui envoie des requêtes en continu au service `php-apache`.

    ```bash
    kubectl run -n hpa-lab -it --rm load-generator --image=busybox -- /bin/sh -c "while true; do wget -q -O- http://php-apache; done"
    ```

    Ce terminal sera maintenant occupé à générer la charge.

#### **Étape 4 : Observer le "Scale-Up" (Montée en charge)**

Retournez à votre **premier terminal** où la commande `watch kubectl get hpa` est en cours d'exécution.

  * **Observation :** Après une minute ou deux, vous verrez la valeur dans la colonne `TARGETS` commencer à grimper bien au-delà de 50%.
  * **Action du HPA :** Dès que le HPA détecte que l'utilisation moyenne dépasse la cible, il va commencer à augmenter le nombre de `REPLICAS`. Vous verrez ce nombre passer de 1 à 2, puis 3, 4, etc., jusqu'à ce que l'utilisation moyenne de CPU sur l'ensemble des pods redescende autour de 50% ou que le `maxReplicas` (10) soit atteint.

Vous pouvez également ouvrir un **troisième terminal** pour observer la création des nouveaux pods :

```bash
watch kubectl get pods -n hpa-lab
```

#### **Étape 5 : Arrêter la Charge et Observer le "Scale-Down" (Baisse de charge)**

Maintenant, nous allons simuler la fin du pic de trafic.

1.  **Arrêtez le générateur de charge :**
    Retournez au **second terminal** (celui qui exécute la commande `kubectl run...`) et arrêtez le processus en appuyant sur `Ctrl+C`. Le pod `load-generator` sera automatiquement supprimé.

2.  **Observation du comportement :**
    Retournez à votre **premier terminal** (celui qui surveille le HPA).

      * **Observation :** Vous verrez l'utilisation du CPU dans la colonne `TARGETS` chuter rapidement vers `0%`.
      * **Action du HPA :** Le HPA ne réduit pas immédiatement le nombre de répliques. Par défaut, il attend une période de stabilisation (environ 5 minutes) pour s'assurer que la baisse de charge n'est pas temporaire.
      * **Après la période de stabilisation :** Vous verrez le nombre de `REPLICAS` diminuer progressivement jusqu'à revenir à la valeur de `minReplicas` (1).

**Conclusion du Lab :** Vous avez configuré et observé un cycle complet d'autoscaling horizontal. Le HPA est un outil indispensable pour construire des applications résilientes et économiques sur Kubernetes, car il permet d'ajuster automatiquement les ressources allouées à la charge réelle de l'application, évitant ainsi le sur-provisionnement coûteux ou le sous-provisionnement risqué.

-----

### **Nettoyage**

Pour supprimer toutes les ressources créées pendant cet atelier, supprimez simplement le namespace.

```bash
kubectl delete namespace hpa-lab
```
