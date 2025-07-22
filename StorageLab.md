 # LAB 1 — Volume emptyDir

## Objectif :

Découvrir les volumes éphémères internes au pod, qui permettent à plusieurs conteneurs d’un même pod de partager des fichiers.

## Étapes :

1. Créer le pod shared-volume-pod :

```
kubectl apply -f Lab1-emptyDir.yaml
```

Contenu du fichier YAML :
```YAML
apiVersion: v1
kind: Pod
metadata:
  name: shared-volume-pod
spec:
  containers:
  - name: writer
    image: busybox
    command: ["sh", "-c", "echo 'Bonjour depuis writer' > /shared/data.txt && sleep 3600"]
    volumeMounts:
    - name: shared-vol
      mountPath: /shared
  - name: reader
    image: busybox
    command: ["sh", "-c", "sleep 3600"]
    volumeMounts:
    - name: shared-vol
      mountPath: /shared
  volumes:
  - name: shared-vol
    emptyDir: {}
```
2. Vérification :
```
kubectl exec shared-volume-pod -c reader -- cat /shared/data.txt
```

✅ Attendu : Bonjour depuis writer → preuve que le volume est partagé.

⸻

# LAB 2 — Volume hostPath

##  Objectif :

Monter un dossier du nœud hôte dans un pod, par exemple pour partager des logs ou données statiques.

## Étapes :

1. Préparer un dossier sur le nœud (VM) :
```
sudo mkdir -p /mnt/data/hostpath-vol
echo "Fichier sur le nœud" | sudo tee /mnt/data/hostpath-vol/demo.txt
sudo chmod -R 777 /mnt/data/hostpath-vol
```

2. Créer le pod hostpath-pod :
   
```
kubectl apply -f Lab2-hostPath.yaml
```
Contenu du fichier YAML :
```YAML
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-pod
spec:
  containers:
  - name: explorer
    image: busybox
    command: ["sh", "-c", "sleep 3600"]
    volumeMounts:
    - name: host-volume
      mountPath: /data
  volumes:
  - name: host-volume
    hostPath:
      path: /mnt/data/hostpath-vol
      type: Directory
```
3. Vérification :
```
kubectl exec hostpath-pod -- cat /data/demo.txt
```
✅ Attendu : contenu du fichier présent sur le nœud.

⸻

# LAB 3 — PersistentVolume statique + PVC

## Objectif :

Créer manuellement un volume (PV) et le réclamer via un PVC. C’est l’approche “admin → dev”.

## Étapes :

1. Créer un dossier sur le nœud :
```
sudo mkdir -p /mnt/data/static-vol
echo "Fichier du PV" | sudo tee /mnt/data/static-vol/init.txt
sudo chmod -R 777 /mnt/data/static-vol
```
2. Créer le PersistentVolume :
   
```
kubectl apply -f Lab3-pv-static.yaml
```
Contenu :
```YAML

apiVersion: v1
kind: PersistentVolume
metadata:
  name: static-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /mnt/data/static-vol

```

3. Créer le PersistentVolumeClaim :
```
kubectl apply -f Lab3-pvc-static.yaml
```
Contenu :

```YAML

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: static-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  volumeName: static-pv

```

4. Déployer un pod qui utilise ce PVC :
```
kubectl apply -f Lab3-pod-static.yaml
```
Contenu :
```YAML
apiVersion: v1
kind: Pod
metadata:
  name: pod-static-pv
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "sleep 3600"]
    volumeMounts:
    - mountPath: /data
      name: vol
  volumes:
  - name: vol
    persistentVolumeClaim:
      claimName: static-pvc
```
5. Vérification :
```
kubectl exec pod-static-pv -- ls /data
kubectl exec pod-static-pv -- cat /data/init.txt
```

✅ Attendu : le pod peut lire les données créées sur le nœud.

⸻

# LAB 4 — PersistentVolumeClaim dynamique avec StorageClass

##  Objectif :

Utiliser un PVC qui déclenche automatiquement la création d’un PV via une StorageClass.

## Étapes :

1. Vérifier la StorageClass active :
```
kubectl get storageclass
```

Note : Minikube propose en général une classe standard avec le provisionneur k8s.io/minikube-hostpath.

2. Créer un PVC dynamique :
```
kubectl apply -f Lab4-pvc-dynamic.yaml
```
Contenu :
```YAML
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
  storageClassName: standard
```
3. Créer un pod qui l’utilise :
```
kubectl apply -f Lab4-pod-dynamic.yaml
```
Contenu :
```YAML
apiVersion: v1
kind: Pod
metadata:
  name: pod-dynamic-pv
spec:
  containers:
  - name: test-container
    image: busybox
    command: ["sh", "-c", "echo 'Stockage dynamique OK' > /data/info.txt && sleep 3600"]
    volumeMounts:
    - mountPath: /data
      name: storage
  volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: dynamic-pvc
```
4. Vérification :
```
kubectl get pvc
kubectl get pv
kubectl exec pod-dynamic-pv -- cat /data/info.txt
```
✅ Attendu : PV créé automatiquement et données présentes.

Parfait ! Voici le Lab 6 — Utilisation d’un volume partagé via NFS.

⸻

# Lab 6 — NFS : Volume partagé entre plusieurs pods

 ## Objectif pédagogique

Mettre en place un serveur NFS dans le cluster et monter le même volume NFS dans plusieurs pods pour partager des fichiers entre eux.

⸻

## Étape 1 : Créer un namespace pour le lab

kubectl create namespace nfs-lab


⸻

## Étape 2 : Déployer un serveur NFS dans le cluster

Fichier nfs-server.yaml :
```YAML
apiVersion: v1
kind: Service
metadata:
  name: nfs-service
  namespace: nfs-lab
spec:
  selector:
    app: nfs-server
  ports:
    - protocol: TCP
      port: 2049
      targetPort: 2049
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-server
  namespace: nfs-lab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nfs-server
  template:
    metadata:
      labels:
        app: nfs-server
    spec:
      containers:
        - name: nfs-server
          image: itsthenetwork/nfs-server-alpine:12
          ports:
            - name: nfs
              containerPort: 2049
          securityContext:
            privileged: true
          args:
            - /exports
          volumeMounts:
            - name: nfs-volume
              mountPath: /exports
      volumes:
        - name: nfs-volume
          emptyDir: {}

```
⸻

## Étape 3 : Créer le PersistentVolume + PersistentVolumeClaim

Fichier nfs-pv-pvc.yaml :
```YAML
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  nfs:
    server: nfs-service.nfs-lab.svc.cluster.local
    path: /
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc
  namespace: nfs-lab
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 500Mi
  volumeName: nfs-pv

```
⸻

## Étape 4 : Déployer des pods partageant le volume

Fichier shared-pods.yaml :
```YAML
apiVersion: v1
kind: Pod
metadata:
  name: writer
  namespace: nfs-lab
spec:
  containers:
    - name: writer
      image: busybox
      command: ["/bin/sh", "-c"]
      args:
        - while true; do date >> /mnt/shared/date.log; sleep 2; done
      volumeMounts:
        - name: shared-vol
          mountPath: /mnt/shared
  volumes:
    - name: shared-vol
      persistentVolumeClaim:
        claimName: nfs-pvc
---
apiVersion: v1
kind: Pod
metadata:
  name: reader
  namespace: nfs-lab
spec:
  containers:
    - name: reader
      image: busybox
      command: ["/bin/sh", "-c"]
      args:
        - tail -f /mnt/shared/date.log
      volumeMounts:
        - name: shared-vol
          mountPath: /mnt/shared
  volumes:
    - name: shared-vol
      persistentVolumeClaim:
        claimName: nfs-pvc

```
⸻

Déploiement des ressources
```
kubectl apply -f nfs-server.yaml
kubectl apply -f nfs-pv-pvc.yaml
kubectl apply -f shared-pods.yaml
```

⸻

 Vérification
Connecte-toi au pod reader :
```
kubectl exec -it reader -n nfs-lab -- sh
```

Tu dois voir les dates s’écrire en direct par le pod writer dans /mnt/shared/date.log.

⸻

Nettoyage
```
kubectl delete namespace nfs-lab

```
⸻

# Lab 7 — Utiliser une StorageClass CSI dynamique

## Objectif pédagogique

Comprendre comment Kubernetes crée automatiquement un volume via un CSI driver à partir d’une StorageClass, sans avoir besoin de déclarer le PV manuellement.

⸻

💡 Prérequis
	•	Un cluster Kubernetes avec un CSI installé et activé (ex. : hostPath, local-path, ou CSI GCE, AWS EBS…).
	•	Pour Minikube, activez un CSI local de test :
```
minikube addons enable storage-provisioner

```
⸻

## Étape 1 : Créer un namespace

```
kubectl create namespace csi-lab
```

⸻

## Étape 2 : Vérifier la StorageClass active

Liste les StorageClass disponibles :
```
kubectl get sc
```
Note le nom de la StorageClass marquée (default) (souvent standard ou local-path).

⸻

## Étape 3 : Créer une PVC (volumeClaim) avec provisionnement dynamique

Fichier pvc-dynamic.yaml :
```YAML
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
  namespace: csi-lab
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: standard # ou local-path selon ton cluster
```
Kubernetes créera automatiquement un PV via la StorageClass CSI désignée.

⸻

## Étape 4 : Créer un pod qui monte ce volume

Fichier pod-csi.yaml :
```YAML
apiVersion: v1
kind: Pod
metadata:
  name: app-using-csi
  namespace: csi-lab
spec:
  containers:
    - name: busybox
      image: busybox
      command: ["/bin/sh", "-c"]
      args: ["echo Bonjour > /mnt/data/bonjour.txt && sleep 3600"]
      volumeMounts:
        - name: data-vol
          mountPath: /mnt/data
  volumes:
    - name: data-vol
      persistentVolumeClaim:
        claimName: dynamic-pvc

```
⸻

Déploiement
```
kubectl apply -f pvc-dynamic.yaml
kubectl apply -f pod-csi.yaml
```

⸻

Vérification

Vérifier que le PVC est Bound :
```
kubectl get pvc -n csi-lab
```
Résultat attendu :

NAME          STATUS   VOLUME                                     STORAGECLASS   AGE
dynamic-pvc   Bound    pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx   standard       1m

Inspecter le contenu dans le pod :
```
kubectl exec -it app-using-csi -n csi-lab -- cat /mnt/data/bonjour.txt
```
Tu devrais voir :

Bonjour


⸻

Nettoyage
```
kubectl delete namespace csi-lab
```

⸻


⸻

# Lab 8 — Utiliser VolumeSnapshot avec CSI

## Objectif pédagogique

Comprendre comment utiliser le mécanisme VolumeSnapshot pour créer une sauvegarde à chaud d’un PVC CSI, puis restaurer cette sauvegarde dans un nouveau volume.

⸻
## Prérequis
	1.	CSI avec support des snapshots doit être activé.
	•	Minikube le supporte via le provisionneur hostpath-csi :

minikube addons enable csi-hostpath-driver
minikube addons enable volumesnapshots


	2.	Installer les CRD nécessaires (si non déjà présents) :

kubectl get volumesnapshotclasses.snapshot.storage.k8s.io

Si la commande échoue, tu peux les créer via :
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/client/config/crd/snapshot.storage.k8s.io_volumesnapshotclasses.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/client/config/crd/snapshot.storage.k8s.io_volumesnapshotcontents.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/client/config/crd/snapshot.storage.k8s.io_volumesnapshots.yaml

```

⸻

## Étape 1 : Créer le PVC avec des données

Fichier pvc-source.yaml :

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: source-pvc
  namespace: snapshot-lab
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: csi-hostpath-sc
  resources:
    requests:
      storage: 1Gi

Fichier writer-pod.yaml :

apiVersion: v1
kind: Pod
metadata:
  name: writer
  namespace: snapshot-lab
spec:
  containers:
    - name: writer
      image: busybox
      command: ["/bin/sh", "-c"]
      args: ["echo 'Kubernetes est génial !' > /data/test.txt && sleep 3600"]
      volumeMounts:
        - mountPath: /data
          name: data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: source-pvc

Déploie le tout :

kubectl create namespace snapshot-lab
kubectl apply -f pvc-source.yaml -n snapshot-lab
kubectl apply -f writer-pod.yaml -n snapshot-lab


⸻

## Étape 2 : Créer un VolumeSnapshot

Fichier snapshot.yaml :
```YAML
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: source-snapshot
  namespace: snapshot-lab
spec:
  volumeSnapshotClassName: csi-hostpath-snapclass
  source:
    persistentVolumeClaimName: source-pvc

```
⸻

## Étape 3 : Créer un PVC à partir du snapshot

Fichier pvc-from-snapshot.yaml :
```YAML
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: restored-pvc
  namespace: snapshot-lab
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: csi-hostpath-sc
  resources:
    requests:
      storage: 1Gi
  dataSource:
    name: source-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io

```
⸻

## Étape 4 : Lire les données restaurées

Fichier reader-pod.yaml :
```YAML
apiVersion: v1
kind: Pod
metadata:
  name: reader
  namespace: snapshot-lab
spec:
  containers:
    - name: reader
      image: busybox
      command: ["/bin/sh", "-c"]
      args: ["cat /data/test.txt && sleep 3600"]
      volumeMounts:
        - mountPath: /data
          name: data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: restored-pvc
```

⸻

 Déploiement final
```
kubectl apply -f snapshot.yaml -n snapshot-lab
kubectl wait --for=condition=ready volumesnapshot/source-snapshot -n snapshot-lab --timeout=60s
kubectl apply -f pvc-from-snapshot.yaml -n snapshot-lab
kubectl apply -f reader-pod.yaml -n snapshot-lab

```
⸻

Vérification
```
kubectl exec -it reader -n snapshot-lab -- cat /data/test.txt
```
Tu devrais voir :

Kubernetes est génial !


⸻

🧹 Nettoyage
```
kubectl delete ns snapshot-lab

```


