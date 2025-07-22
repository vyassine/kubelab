#  Déploiement d'une application de démonstration
Avant de créer des services, nous allons déployer une application web simple (Nginx) à l'aide d'un Deployment. Ce Deployment créera 3 Pods que nos services pourront cibler.

## Créez le fichier nginx-deployment.yaml :

```YAML

apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
```
Appliquez le manifeste pour créer le Deployment :

```bash

kubectl apply -f nginx-deployment.yaml
```
Vérifiez que les Pods sont en cours d'exécution :

```bash

kubectl get pods -l app=webapp
```
Vous devriez voir 3 pods avec le statut Running. Ces pods ont des adresses IP internes qui peuvent changer à tout moment, ce qui illustre le problème que les Services viennent résoudre.   

## Lab 1 : Le Service ClusterIP - Communication Interne
Objectif : Comprendre comment les services communiquent à l'intérieur du cluster. Le ClusterIP est le type de service par défaut et le plus courant. Il fournit une adresse IP stable accessible uniquement depuis le cluster, idéale pour la communication entre microservices.   

Créez le fichier webapp-clusterip-service.yaml :

```YAML

apiVersion: v1
kind: Service
metadata:
  name: webapp-internal-service
spec:
  type: ClusterIP # Ce type est le défaut, il peut donc être omis
  selector:
    app: webapp
  ports:
    - protocol: TCP
      port: 80       # Le port sur lequel le service est exposé
      targetPort: 80 # Le port sur lequel les conteneurs des pods écoutent
```
Appliquez le manifeste pour créer le Service :

```bash

kubectl apply -f webapp-clusterip-service.yaml
```
Inspectez le Service :

```bash

kubectl get service webapp-internal-service
```
Résultat attendu :

NAME                      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
webapp-internal-service   ClusterIP   10.108.111.22   <none>        80/TCP    15s
Notez le CLUSTER-IP. C'est l'adresse IP virtuelle et stable de votre service.   

Vérification de la communication interne :
Nous ne pouvons pas accéder à cette IP depuis l'extérieur. Pour tester, nous allons lancer un pod temporaire et utiliser curl depuis l'intérieur du cluster.

```bash

# Lance un pod de test et exécute une commande shell à l'intérieur
kubectl run -it --rm --image=busybox test-pod -- sh


# Une fois dans le shell du pod, utilisez le nom DNS du service pour le contacter
# Le nom DNS est au format <nom-du-service>.<namespace>.svc.cluster.local
wget -q -O - http://webapp-internal-service

```
Résultat et Nettoyage :
Vous devriez voir la page d'accueil HTML de Nginx. Cela prouve que votre pod de test a pu résoudre le nom webapp-internal-service et atteindre l'un des 3 pods Nginx via l'IP stable du service.   


Tapez exit pour quitter et supprimer le pod de test.

Conclusion du Lab 1 : Le service ClusterIP fournit un point d'entrée DNS et une IP stable pour la communication interne, découplant ainsi les services les uns des autres.   

##  Lab 2 : Le Service NodePort - Exposition Externe de Base
Objectif : Exposer une application à l'extérieur du cluster de manière simple, principalement pour le développement et les tests. Un service    

NodePort ouvre un port statique sur l'IP de chaque nœud du cluster.   

Créez le fichier webapp-nodeport-service.yaml :

```YAML

apiVersion: v1
kind: Service
metadata:
  name: webapp-external-service
spec:
  type: NodePort # Nous spécifions explicitement le type NodePort
  selector:
    app: webapp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      # nodePort: 30080 # Optionnel: vous pouvez spécifier un port, sinon Kubernetes en choisit un
```
Appliquez le manifeste :

```bash

kubectl apply -f webapp-nodeport-service.yaml
```
Inspectez le Service :

```bash

kubectl get service webapp-external-service
```
Résultat attendu :

NAME                      TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
webapp-external-service   NodePort   10.101.22.118   <none>        80:31568/TCP   20s
Notez la colonne PORT(S). Le service est accessible sur le port 80 en interne (via son CLUSTER-IP), et sur le port 31568 (ou un autre port dans la plage 30000-32767) sur l'IP de chaque nœud.   

Vérification de l'accès externe :

Obtenez l'adresse IP de votre nœud (si vous utilisez Minikube) :

```bash

minikube ip
```
Utilisez curl ou votre navigateur pour accéder à l'application via l'IP du nœud et le NodePort :

```bash

# Remplacez <minikube-ip> et <node-port> par les valeurs obtenues
curl http://<minikube-ip>:<node-port>
```

Conclusion du Lab 2 : Le service NodePort est un moyen rapide d'exposer une application, mais il est généralement déconseillé en production car il expose un port sur chaque nœud et ne fournit pas de load balancing avancé.   

## Lab 3 : Le Service LoadBalancer - Exposition de Production
Objectif : Comprendre la méthode standard pour exposer des services sur Internet dans un environnement cloud. Ce type de service provisionne automatiquement un load balancer externe.   

Note : Ce lab fonctionne mieux sur un vrai fournisseur cloud (AWS, GCP, Azure). Avec Minikube, nous pouvons le simuler.

Créez le fichier webapp-loadbalancer-service.yaml :

```YAML

apiVersion: v1
kind: Service
metadata:
  name: webapp-prod-service
spec:
  type: LoadBalancer # Le type qui déclenche la création d'un load balancer externe
  selector:
    app: webapp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```
Appliquez le manifeste :

```bash

kubectl apply -f webapp-loadbalancer-service.yaml
```
Simulation avec Minikube :
Pour que le LoadBalancer fonctionne sur Minikube, ouvrez un nouveau terminal et lancez la commande suivante. Laissez-la tourner en arrière-plan.

```bash

minikube tunnel
Inspectez le Service :
Retournez à votre premier terminal et inspectez le service. Cela peut prendre une minute pour que l'IP externe soit assignée.

```bash

kubectl get service webapp-prod-service

```
Résultat attendu (après un moment) :

NAME                  TYPE           CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
webapp-prod-service   LoadBalancer   10.98.45.67   10.98.45.67   80:30343/TCP   1m
Grâce à minikube tunnel, une EXTERNAL-IP a été assignée. Sur un vrai cloud, ce serait une adresse IP publique fournie par le fournisseur.   

Vérification de l'accès :
Vous pouvez maintenant accéder à votre application directement via cette EXTERNAL-IP sur le port 80.

```bash

curl http://<EXTERNAL-IP>
```
Conclusion du Lab 3 : Le service LoadBalancer est la solution robuste pour la production, car il s'intègre aux fournisseurs cloud pour créer un point d'entrée unique et scalable pour votre application.   

## Lab 4 : Le Service Headless - Découverte de Pods Individuels
Objectif : Découvrir un type de service spécial qui ne fait pas de load balancing mais est utilisé pour la découverte de services, en donnant des enregistrements DNS pour chaque pod individuel. C'est un prérequis essentiel pour les StatefulSets.   

Créez le fichier webapp-headless-service.yaml :

```YAML

apiVersion: v1
kind: Service
metadata:
  name: webapp-headless
spec:
  clusterIP: None # La clé pour créer un service Headless
  selector:
    app: webapp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80

```
Appliquez le manifeste :

```bash

kubectl apply -f webapp-headless-service.yaml
```
Inspectez le Service :

```bash

kubectl get service webapp-headless
```
Résultat attendu :

NAME              TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
webapp-headless   ClusterIP   None         <none>        80/TCP    10s
Remarquez que le CLUSTER-IP est explicitement None. Ce service n'a pas d'IP virtuelle.   

Vérification de la découverte DNS :
Nous allons à nouveau utiliser un pod de test pour voir comment le DNS fonctionne pour ce service.

```bash

kubectl run -it --rm --image=dnsutils test-dns -- /bin/sh
```

# Une fois dans le shell, faites une requête DNS pour le nom du service
nslookup webapp-headless
Résultat :
Au lieu d'une seule adresse IP (celle du service), nslookup retournera les adresses IP des 3 pods qui correspondent au sélecteur app: webapp.

Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      webapp-headless
Address 1: 172.17.0.5
Address 2: 172.17.0.6
Address 3: 172.17.0.4
Conclusion du Lab 4 : Le service Headless est un outil puissant pour les scénarios où vous avez besoin de vous connecter directement à des pods spécifiques, comme dans les bases de données en cluster gérées par des StatefulSets, plutôt que de passer par un proxy de service. 
