Pour configurer le chiffrement au repos (Encryption at Rest) pour les secrets Kubernetes, vous devez créer une configuration de chiffrement et redémarrer le `kube-apiserver` pour qu'il l'utilise.

Voici comment faire, étape par étape.

-----

### 🔒 Le Principe

Par défaut, les `Secrets` dans Kubernetes sont simplement encodés en **Base64** dans la base de données `etcd`. Cela signifie que quiconque a accès à une sauvegarde de `etcd` peut facilement lire toutes les informations confidentielles.

Le chiffrement au repos ajoute une couche de cryptage réelle. Le `kube-apiserver` chiffrera les données du `Secret` avant de les écrire dans `etcd`. Ainsi, même si la base `etcd` est compromise, les données des secrets restent illisibles sans la clé de chiffrement.

-----

### ⚙️ Comment le Mettre en Place

La mise en place se fait en 3 grandes étapes.

#### Étape 1 : Créer le Fichier de Configuration de Chiffrement

Vous devez d'abord créer un fichier YAML qui définit quel type de ressource chiffrer et avec quelle méthode.

1.  **Générez une clé de chiffrement.** Cette clé doit être une chaîne de 32 octets encodée en Base64.

    ```bash
    head -c 32 /dev/urandom | base64
    ```

    Gardez précieusement le résultat (par exemple : `yL4n...J7g=`). **Si vous perdez cette clé, vous ne pourrez plus jamais déchiffrer vos secrets.**

2.  **Créez le fichier de configuration** (nommons-le `encryption-config.yaml`). Ce fichier doit être placé sur vos nœuds master.

    ```yaml
    # encryption-config.yaml
    apiVersion: apiserver.config.k8s.io/v1
    kind: EncryptionConfiguration
    resources:
      - resources:
          - secrets # On cible les ressources de type "Secret"
        providers:
          - aescbc: # On choisit l'algorithme AES-CBC
              keys:
                - name: key1
                  secret: <VOTRE_CLÉ_EN_BASE64_GÉNÉRÉE_PRÉCÉDEMMENT>
          - identity: {} # Indispensable : c'est le fournisseur "ne rien faire".
                         # Il permet de lire les secrets qui n'étaient pas encore chiffrés.
    ```

#### Étape 2 : Configurer le `kube-apiserver`

Vous devez maintenant indiquer au `kube-apiserver` d'utiliser ce fichier de configuration. La méthode dépend de la façon dont votre cluster est installé.

  * **Pour les clusters gérés (GKE, AKS, EKS) :** Cette option est généralement gérée par le fournisseur de cloud. Vous l'activez souvent via leur interface web ou leur outil en ligne de commande lors de la création ou de la mise à jour du cluster.

  * **Pour les clusters auto-hébergés (avec `kubeadm`) :** Vous devez modifier le fichier de manifeste statique du `kube-apiserver`.

    1.  Placez votre fichier `encryption-config.yaml` dans un répertoire sécurisé sur vos nœuds master (par exemple `/etc/kubernetes/`).
    2.  Modifiez le fichier `/etc/kubernetes/manifests/kube-apiserver.yaml`.
    3.  Ajoutez l'argument `--encryption-provider-config=/etc/kubernetes/encryption-config.yaml` à la section `command`.
    4.  Montez le fichier de configuration et son répertoire en tant que volumes dans le pod.

    Voici un exemple de modification du `kube-apiserver.yaml` :

    ```yaml
    # /etc/kubernetes/manifests/kube-apiserver.yaml
    ...
    spec:
      containers:
      - command:
        - kube-apiserver
        # ... autres arguments
        - --encryption-provider-config=/etc/kubernetes/encryption-config.yaml # <--- AJOUTER CETTE LIGNE
        ...
        volumeMounts:
        # ... autres volumeMounts
        - name: encryption-config
          mountPath: /etc/kubernetes/
          readOnly: true # <--- AJOUTER CE BLOC
      volumes:
      # ... autres volumes
      - name: encryption-config
        hostPath:
          path: /etc/kubernetes/
          type: DirectoryOrCreate # <--- AJOUTER CE BLOC
    ...
    ```

    Le `kubelet` détectera le changement et redémarrera automatiquement le `kube-apiserver` avec la nouvelle configuration.

#### Étape 3 : Chiffrer les Secrets Existants

Le redémarrage du serveur d'API ne chiffre que les **nouveaux** secrets. Pour forcer le chiffrement des secrets qui existaient déjà, exécutez la commande suivante :

```bash
kubectl get secrets --all-namespaces -o json | kubectl replace -f -
```

Cette commande lit chaque secret et le réécrit immédiatement. En étant réécrit, il passera par le nouveau processus de chiffrement.

-----

### ✅ Vérification

Pour vérifier que vos secrets sont bien chiffrés, vous pouvez interroger `etcd` directement.

```bash
# Commande à exécuter sur un nœud master
ETCDCTL_API=3 etcdctl --cacert /etc/kubernetes/pki/etcd/ca.crt --cert /etc/kubernetes/pki/etcd/server.crt --key /etc/kubernetes/pki/etcd/server.key get /registry/secrets/default/mon-secret
```

Le résultat ne doit **PAS** être la valeur en Base64, mais une chaîne qui commence par `k8s:enc:aescbc:v1:...`, prouvant que le chiffrement est actif.

-----

### ⚠️ Points Importants

  * **🔑 Sauvegarde de la clé :** La clé de chiffrement que vous avez générée est vitale. Sauvegardez-la dans un endroit extrêmement sécurisé (comme un gestionnaire de secrets), séparément de vos sauvegardes `etcd`.
  * **Rotation des clés :** Pour améliorer la sécurité, vous pouvez effectuer une rotation des clés. Pour cela, ajoutez une nouvelle clé en haut de la liste `keys` dans votre fichier de configuration, redémarrez le `kube-apiserver`, puis relancez la commande `kubectl replace` pour que tout soit chiffré avec la nouvelle clé.
