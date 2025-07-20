
## Étape 1 : Écrire le manifeste du Pod  
Créez un fichier nommé `pod-nginx.yaml` avec le contenu suivant :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mon-pod-nginx
  labels:
    app: webserver
spec:
  containers:
  - name: nginx-container
    image: nginx:1.25.3
    ports:
    - containerPort: 80
```
## Étape 2 : Déployer et inspecter le Pod
Déployez le Pod avec la commande :
```bash
kubectl apply -f pod-nginx.yaml
```
Vérifiez que le Pod est Running :
```bash
kubectl get pods -o wide
```
Inspectez les détails du Pod :
```bash
kubectl describe pod mon-pod-nginx
```
## Étape 3 : Accéder à l'application du Pod
Créez un tunnel sécurisé (laissez ce terminal ouvert) :
```bash
kubectl port-forward --address 0.0.0.0 pod/mon-pod-nginx 8080:80
```
(Ouverture optionnelle du port 8080 sur le pare-feu dans un autre terminal) :
```bash
sudo ufw allow 8080/tcp
```
Ouvrez votre navigateur à l’adresse :
```cpp
http://<IP_PUBLIQUE_DE_LA_VM>:8080
```
## Étape 4 : Nettoyer les ressources
Arrêtez la commande port-forward avec Ctrl+C puis supprimez le Pod :
```bash
kubectl delete -f pod-nginx.yaml
```
