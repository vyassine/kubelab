
# Installation de Minikube avec 2 nœuds sur Ubuntu

Ce guide explique comment installer **Minikube** avec **Docker** comme driver, et démarrer un cluster Kubernetes avec **2 nœuds** (1 master + 1 worker) sur Ubuntu.

> ✅ **Pré-requis :** une VM ou machine avec **2 vCPU** et **4 Go de RAM minimum**

---

## 🔧 Étape 1 : Mise à jour du système et installation des dépendances

```
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl apt-transport-https ca-certificates software-properties-common docker.io
```

---

## Étape 2 : Activer Docker et ajouter l'utilisateur courant au groupe Docker

```
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
newgrp docker
```

> 💡 Conseil : si tu viens d'ajouter ton utilisateur au groupe `docker`, pense à te déconnecter puis reconnecter, ou exécute `newgrp docker` pour prendre en compte le changement.

---

## Étape 3 : Installer `kubectl` (client Kubernetes)

```
curl -LO "https://dl.k8s.io/release/$(curl -Ls https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
kubectl version --client
```

---

## Étape 4 : Installer Minikube

```
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
minikube version
```

---

## Étape 5 : Démarrer un cluster Minikube avec 2 nœuds

```
minikube start --nodes=2 --cpus=2 --memory=2200 --driver=docker
```

> ✅ Explication des options :
> - `--nodes=2` : 1 master (`minikube`) + 1 worker (`minikube-m02`)
> - `--cpus=2` : 2 CPU alloués au cluster
> - `--memory=2200` : 2.2 Go de RAM alloués au cluster
> - `--driver=docker` : utilise Docker pour exécuter les nœuds

---

## Étape 6 : Vérifier que le cluster fonctionne

```
kubectl get nodes
```

Sortie attendue :

```
NAME            STATUS   ROLES                  AGE   VERSION
minikube        Ready    control-plane,master   Xs    vX.X.X
minikube-m02    Ready    <none>                 Xs    vX.X.X
```

---

##  Étape 7 (optionnelle) : Déployer une application de test

```
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --type=NodePort --port=80
kubectl get services
```

Tu peux aussi ouvrir le **dashboard Kubernetes** avec :

```
minikube dashboard
```

---

##  Étape 8 (optionnelle) : Supprimer le cluster

```
minikube delete --all
```

---

## Remarques importantes


- Si Minikube refuse de démarrer pour cause de mémoire, réduis `--memory=2200` à `2000` ou `1800`

---
