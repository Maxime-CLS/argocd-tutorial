+++
title = 'Kustomize'
date = 2024-10-09T11:00:06+02:00
draft = false
weight = 1
+++


Kustomize parcourt un manifeste Kubernetes pour ajouter, supprimer ou mettre à jour des options de configuration sans fork. Il est disponible à la fois en tant que binaire autonome et en tant que fonctionnalité native de kubectl.

Les principes de kustomizesont :

* Approche purement déclarative de la personnalisation de la configuration
* Gérer un nombre arbitraire de configurations Kubernetes distinctement personnalisées
* Chaque artefact utilisé par Kustomize est un simple YAML et peut être validé et traité comme tel
* En tant que système de création de modèles « sans modèle », il encourage l'utilisation de YAML sans forker le référentiel.

## À la découverte de Kustomize

### Découverte de la CLI Kustomize

L'interface kustomize de ligne de commande doit avoir été installée dans le cadre de la configuration du laboratoire. Vérifiez qu'elle a été installée.

``` 
kustomize version --short
```

Cela devrait afficher la version, cela devrait ressembler à ceci.

``` 
{kustomize/v4.0.5  2021-02-13T21:21:14Z  }
```

Kustomize, à la base, est destiné à créer des manifestes Kubernetes natifs basés sur YAML, tout en laissant le YAML d'origine intact. Il y parvient dans un format de modèle « sans modèle ». Cela se fait en fournissant un kustomization.yamlfichier.

Nous nous concentrerons sur deux sous-commandes : la build commande et la editcommande.

La build commande prend la source YAML (via un chemin ou une URL) et crée un nouveau YAML qui peut être redirigé vers kubectl create. Nous allons travailler avec un exemple :


``` 
mkdir kustomize-build
vi welcome.yaml
```

```
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: welcome-php
  name: welcome-php
spec:
  replicas: 1
  selector:
    matchLabels:
      app: welcome-php
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: welcome-php
    spec:
      containers:
      - image: quay.io/redhatworkshops/welcome-php:latest
        name: welcome-php
        resources: {}
```

Ce fichier ne montre rien de spécial. Juste un manifeste Kubernetes standard.

Et si, par exemple, nous voulions ajouter un label à ce manifeste sans le modifier ? C'est là kustomization.yaml qu'intervient le fichier.


``` 
vi kustomization.yaml
```

```
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- ./welcome.yaml
images:
- name: quay.io/redhatworkshops/welcome-php
  newTag: ffcd15
patches:
- patch: |-
    - op: add
      path: /metadata/labels/testkey
      value: testvalue
  target:
    group: apps
    kind: Deployment
    name: welcome-php
    version: v1
```

Comme vous pouvez le voir dans le résultat, il n'y a pas grand chose. Les deux sections de cet exemple sont les resourcessections et patchesJson6902.

resources est un tableau de fichiers individuels, de répertoires et/ou d'URL où d'autres manifestes sont stockés. Dans cet exemple, nous chargeons simplement un fichier. Le ( patchesJson6902 est une RFC de correctif qui kustomizeprend en charge. Comme vous pouvez le voir, dans le patchesJson6902 fichier, j'ajoute une étiquette à ce manifeste.

> [!INFO]
> Vous pouvez en savoir plus sur les options disponibles pour la mise à jour des correctifs sur le site de documentation officiel.


Créez ce manifeste en exécutant :

``` 
kustomize build
```

Vous pouvez voir que la nouvelle étiquette a été ajoutée au manifeste !

``` 
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: welcome-php
    testkey: testvalue
  name: welcome-php
spec:
  replicas: 1
  selector:
    matchLabels:
      app: welcome-php
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: welcome-php
    spec:
      containers:
      - image: quay.io/redhatworkshops/welcome-php:ffcd15
        name: welcome-php
        resources: {}
```

Vous pouvez utiliser la kustomize edit commande au lieu d'écrire du YAML. Par exemple, vous pouvez modifier la balise d'image Deployment utilisée de latest à ffcd15en exécutant la commande suivante :

```
kustomize edit set image quay.io/redhatworkshops/welcome-php:ffcd15
```

Cela mettra à jour le kustomization.yaml fichier avec une images section.

```
cat kustomization.yaml
```

```
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- ./welcome.yaml
images:
- name: quay.io/redhatworkshops/welcome-php
  newTag: ffcd15
patches:
- patch: |-
    - op: add
      path: /metadata/labels/testkey
      value: testvalue
  target:
    group: apps
    kind: Deployment
    name: welcome-php
    version: v1
```

Maintenant, lorsque vous exécutez :

```
kustomize build .
```

Vous devriez voir non seulement la nouvelle étiquette, mais également la nouvelle ffcd15 balise d’image.

> [!INFO]
> Vous devrez peut-être fermer l'onglet kustomization.yaml et le rouvrir pour voir les modifications.

Vous pouvez voir comment vous pouvez prendre un YAML déjà existant et le modifier pour votre environnement spécifique sans avoir besoin de copier ou de modifier l'original.

Kustomize peut être utilisé pour écrire un nouveau fichier YAML ou être inséré dans la commande kubectl

```
kustomize build . | kubectl apply -f -
```

#### Découvrir Kustomize avec Kubectl

Depuis Kubernetes 1.14, la kubectlcommande prend en charge Kustomize de manière intégrée.

Vous pouvez le voir en exécutant :

```
kubectl kustomize --help
```

Cela exécute la kustomize buildcommande. Vous pouvez le voir en exécutant :

```
kubectl kustomize
```

Bien que vous puissiez l'utiliser pour l'intégrer à la commande apply, ce n'est pas obligatoire. La commande kubectl apply dispose de l'option -k qui exécutera la build avant d'appliquer le manifeste.


Pour tester cela, créez d’abord un espace de noms :

```
kubectl create namespace kustomize-test
```

Ensuite, assurez-vous que vous êtes dans l’espace de noms :

```
kubectl config set-context --current --namespace=kustomize-test
```

Enfin, exécutez la commande pour créer et appliquer les manifestes :

```
kubectl apply -k ./
```

> [!INFO]
> Vous pouvez transmettre non seulement des répertoires, mais également des URL. La seule exigence est que vous ayez un fichier kustomization.yaml dans le chemin.

Cela devrait créer le déploiement et vous devriez voir les pods s'exécuter dans l'espace de noms :

```
kubectl get pods -n kustomize-test
```

Vous pouvez voir que le déploiement a été créé avec les étiquettes supplémentaires :

```
kubectl get deployment welcome-php -o jsonpath='{.metadata.labels}' | jq -r
```

De plus, l'image a été mise à jour en fonction de la personnalisation qui a été effectuée :

```
kubectl get deploy welcome-php  -o jsonpath='{.spec.template.spec.containers[].image}{"\n"}'
```

Comme vous pouvez le voir, kustomize peut être un outil puissant.

Vous pouvez désormais supprimer cet espace de noms en toute sécurité :

```
kubectl delete namespace kustomize-test
```



