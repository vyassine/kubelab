

### **Prérequis : Création d'un Namespace de Laboratoire**

Pour garder notre environnement de test propre et isolé, nous allons d'abord créer un namespace dédié.

1.  **Créez le fichier `probes-lab-namespace.yaml` :**

    ```yaml
    apiVersion: v1
    kind: Namespace
    metadata:
      name: probes-lab
    ```

2.  **Appliquez le manifeste :**

    ```bash
    kubectl apply -f probes-lab-namespace.yaml
    ```

Toutes les ressources suivantes seront créées dans ce namespace `probes-lab`.

-----

### **Lab 1 : La Sonde de Vivacité (`livenessProbe`) - L'Auto-Réparation**

**Objectif :** Démontrer comment une `livenessProbe` détecte qu'une application est bloquée ou en erreur et déclenche un redémarrage du conteneur pour la restaurer. [1, 2, 3]

1.  **Créez le fichier `liveness-app.yaml` :**
    Nous allons déployer un pod qui simule une application devenant défectueuse. La sonde vérifiera périodiquement la santé du pod.

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: liveness-pod
      namespace: probes-lab
    spec:
      containers:
      - name: liveness-container
        image: busybox:1.28
        # Cette commande crée un fichier "healthy" et le supprime après 30 secondes pour simuler une panne.
        command: ["/bin/sh", "-c", "touch /tmp/healthy; sleep 30; rm -f /tmp/healthy; sleep 600"]
        livenessProbe:
          # Le Kubelet exécutera cette commande pour vérifier la santé.
          exec:
            command:
            - cat
            - /tmp/healthy
          initialDelaySeconds: 5  # Attend 5 secondes avant la première sonde.
          periodSeconds: 5      # Sonde toutes les 5 secondes.
          failureThreshold: 1     # Le pod est considéré en échec après 1 tentative ratée.
    ```

2.  **Déployez le pod et observez son état :**

    ```bash
    kubectl apply -f liveness-app.yaml
    ```

    Utilisez la commande `watch` pour observer les changements en temps réel.

    ```bash
    watch kubectl get pod liveness-pod -n probes-lab
    ```

3.  **Observation du comportement :**

      * **Pendant les 35 premières secondes :** Le pod sera `Running` et la colonne `RESTARTS` sera à `0`. Le fichier `/tmp/healthy` existe, donc la sonde réussit.
      * **Après environ 35-40 secondes :** La commande dans le conteneur supprime le fichier `/tmp/healthy`. La prochaine `livenessProbe` échouera (car `cat /tmp/healthy` renverra une erreur).
      * **L'action de Kubernetes :** Dès que la sonde échoue, Kubernetes tue le conteneur. Vous verrez brièvement le statut passer à `Error` ou `CrashLoopBackOff`, puis la colonne `RESTARTS` passera à `1` et le pod redeviendra `Running`.

4.  **Inspectez les événements pour confirmer :**
    Pour voir exactement ce qui s'est passé, décrivez le pod.

    ```bash
    kubectl describe pod liveness-pod -n probes-lab
    ```

    Dans la section `Events`, vous verrez des messages indiquant que la `Liveness probe failed`, suivis d'un message indiquant que le conteneur a été tué et redémarré.

**Conclusion du Lab 1 :** Vous avez vu comment la `livenessProbe` agit comme un mécanisme de surveillance active, redémarrant automatiquement un conteneur qui ne répond plus correctement, même si son processus principal n'a pas planté. [1, 2]

-----

### **Lab 2 : La Sonde de Disponibilité (`readinessProbe`) - Gestion du Trafic**

**Objectif :** Comprendre comment une `readinessProbe` empêche le trafic d'être envoyé à un pod qui est en cours d'exécution mais pas encore prêt à servir des requêtes (par exemple, pendant son initialisation). [1, 4, 3]

1.  **Créez le fichier `readiness-app.yaml` :**
    Nous allons déployer un pod qui simule un long temps de démarrage et un `Service` pour lui envoyer du trafic.

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: readiness-pod
      namespace: probes-lab
      labels:
        app: web
    spec:
      containers:
      - name: readiness-container
        image: busybox:1.28
        # Cette commande simule un démarrage de 20 secondes avant que l'application soit "prête".
        command: ["/bin/sh", "-c", "sleep 20; touch /tmp/ready; sleep 600"]
        readinessProbe:
          exec:
            command:
            - cat
            - /tmp/ready
          initialDelaySeconds: 5
          periodSeconds: 3
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: readiness-service
      namespace: probes-lab
    spec:
      selector:
        app: web
      ports:
      - protocol: TCP
        port: 80
        targetPort: 8080 # Port fictif pour cet exemple
    ```

2.  **Déployez les ressources et observez l'état du pod :**

    ```bash
    kubectl apply -f readiness-app.yaml
    ```

    Observez l'état du pod.

    ```bash
    kubectl get pod readiness-pod -n probes-lab -w
    ```

    Vous remarquerez que la colonne `READY` reste à `0/1` pendant les 20 premières secondes, même si le pod est `Running`.

3.  **Inspectez les points de terminaison (`Endpoints`) du Service :**
    Pendant que le pod n'est pas "prêt", ouvrez un autre terminal et regardez les `Endpoints` du service.

    ```bash
    kubectl describe service readiness-service -n probes-lab
    ```

      * **Pendant les 20 premières secondes :** La section `Endpoints` sera vide (`<none>`). Kubernetes sait que le pod n'est pas prêt à recevoir du trafic et ne l'ajoute donc pas à la liste des destinations du service.
      * **Après 20 secondes :** Le fichier `/tmp/ready` est créé, la `readinessProbe` réussit. La colonne `READY` du pod passe à `1/1`. Si vous redécrivez le service, vous verrez maintenant l'adresse IP du pod dans la section `Endpoints`.

**Conclusion du Lab 2 :** La `readinessProbe` est un mécanisme de contrôle de trafic crucial. Elle garantit qu'un pod ne reçoit des requêtes que lorsqu'il a explicitement signalé qu'il est prêt à les traiter, ce qui est essentiel pour éviter les erreurs lors des démarrages d'applications ou des déploiements. [1, 4]

-----

### **Lab 3 : La Sonde de Démarrage (`startupProbe`) - Protéger les Applications Lentes**

**Objectif :** Démontrer comment une `startupProbe` peut protéger une application avec un long temps de démarrage, en retardant l'activation des autres sondes pour éviter qu'elle ne soit tuée prématurément. [1, 3]

1.  **Créez le fichier `startup-app.yaml` :**
    Nous allons simuler une application qui prend 30 secondes pour démarrer. Nous configurerons une `livenessProbe` agressive qui, seule, tuerait l'application.

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: startup-pod
      namespace: probes-lab
    spec:
      containers:
      - name: startup-container
        image: busybox:1.28
        # Simule un démarrage de 30 secondes.
        command: ["/bin/sh", "-c", "sleep 30; touch /tmp/started; sleep 600"]
        startupProbe:
          exec:
            command:
            - cat
            - /tmp/started
          # Donne à l'application 45 secondes (15 * 3s) pour démarrer.
          failureThreshold: 15
          periodSeconds: 3
        livenessProbe:
          exec:
            command:
            - cat
            - /tmp/started
          # Cette sonde commencerait normalement après 5s et tuerait le pod.
          initialDelaySeconds: 5
          periodSeconds: 5
    ```

2.  **Déployez le pod et observez les événements :**

    ```bash
    kubectl apply -f startup-app.yaml
    ```

    La meilleure façon de voir ce qui se passe est de regarder les événements du pod.

    ```bash
    kubectl describe pod startup-pod -n probes-lab
    ```

3.  **Observation du comportement :**
    Dans la section `Events`, vous observerez la séquence suivante :

      * Le pod est créé et passe à l'état `Running`.
      * Vous verrez plusieurs messages `Warning Unhealthy` provenant de la **`Startup probe`** qui échoue (environ 10 échecs, toutes les 3 secondes).
      * Pendant ce temps, la `livenessProbe` est désactivée. Il n'y aura aucun événement la concernant. [3]
      * Après 30 secondes, le fichier `/tmp/started` est créé. La `startupProbe` suivante réussit.
      * Une fois la `startupProbe` réussie, elle ne s'exécutera plus jamais. La `livenessProbe` prend alors le relais.
      * Le pod reste `Running` avec `0` redémarrage, car la `livenessProbe` commence son travail sur une application déjà démarrée et saine.

**Conclusion du Lab 3 :** La `startupProbe` est un outil essentiel pour les applications "legacy" ou complexes qui ont des temps de démarrage longs et imprévisibles. Elle garantit que la `livenessProbe` n'interfère pas avec le processus de démarrage, améliorant ainsi considérablement la fiabilité des déploiements. [5, 3]

-----

### **Nettoyage**

Pour supprimer toutes les ressources créées pendant ces ateliers, supprimez simplement le namespace.

```bash
kubectl delete namespace probes-lab
```
