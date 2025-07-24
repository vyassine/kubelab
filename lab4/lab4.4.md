#  Lab 4.4 : Mettre en Place une Politique de Conformité
Objectif : Installer OPA Gatekeeper et créer une politique qui oblige tous les nouveaux Pods à avoir une étiquette owner.

## Étape 1 : Installer Gatekeeper

https://github.com/open-policy-agent/gatekeeper/tree/master/demo/basic

https://github.com/bikram20/opa-gatekeeper-examples/tree/master/templates


Cette commande déploie tous les composants de Gatekeeper dans le cluster.

```bash

kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/release-3.11/deploy/gatekeeper.yaml
```
Attendez quelques minutes que tous les pods dans le namespace gatekeeper-system soient Running.

## Étape 2 : Définir le Modèle de Politique (ConstraintTemplate)
Créez le fichier template-labels.yaml.

```YAML

# template-labels.yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels
      validation:
        openAPIV3Schema:
          type: object 
          properties:
            labels:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredlabels

        violation[{"msg": msg, "details": {"missing_labels": missing}}] {
          provided := {label | input.review.object.metadata.labels[label]}
          required := {label | label := input.parameters.labels[_]}
          missing := required - provided
          count(missing) > 0
          msg := sprintf("You must provide labels: %v", [missing])
        }
```
Appliquez-le :

```bash

kubectl apply -f template-labels.yaml
```
## Étape 3 : Créer la Politique (Constraint)
Créez le fichier constraint-owner-label.yaml.

```YAML

# constraint-owner-label.yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: pod-must-have-owner
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
  parameters:
    labels: ["owner"]

```
Appliquez-le :

```bash

kubectl apply -f constraint-owner-label.yaml

```
## Étape 4 : Tester la Politique
Tentative de Violation (doit échouer) :

```bash

kubectl run nginx-sans-label --image=nginx
```
Résultat attendu : ÉCHEC! La requête est rejetée par Gatekeeper avec un message d'erreur clair : [pod-must-have-owner] You must provide labels: {"owner"}

Tentative de Conformité (doit réussir) :

```bash

kubectl run nginx-avec-label --image=nginx --labels=owner=equipe-alpha
```
Résultat attendu : SUCCÈS! Le Pod est créé car il respecte la politique.
