+++
title = 'Créer une application avec Kustomize'
date = 2024-10-09T12:05:09+02:00
draft = false
weight = 2
+++

## Déploiement d'une application personnalisée

Dans le chapitre précédent, vous avez appris que dans un workflow GitOps, l'intégralité de la pile d'applications (y compris l'infrastructure) est reflétée dans un dépôt Git. Le défi consiste à savoir comment y parvenir sans dupliquer YAML.

Maintenant que vous avez exploré kustomize, voyons comment il s'intègre dans Argo CD et comment il peut être utilisé dans un workflow GitOps.

Avant de continuer, revenez au répertoire personnel :

``` 
cd -
```

### La console Web du CD Argo

Accédez à la console Web Argo CD.

Une fois que vous avez accepté le certificat auto-signé, l'écran de connexion d'Argo CD devrait s'afficher.

![image](/argocd-tutorial/images/attachments/elementaire/argocd-login.png)

Vous pouvez vous connecter avec les éléments suivants

Nom d'utilisateur :

``` 
admin
```


Mot de passe :

``` 
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

### Application personnalisée

Argo CD prend en charge Kustomize en natif. Vous pouvez l'utiliser pour éviter de dupliquer YAML pour chaque déploiement. Cela est particulièrement utile si vous déployez dans différents environnements ou clusters.

Jetez un oeil à la définition Application :


``` 
mkdir bgdk-app
vi bgdk-app/bgdk-app.yaml
```

``` 
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: bgdk-app
  namespace: argocd
spec:
  destination:
    namespace: bgdk
    server: https://kubernetes.default.svc
  project: default
  source:
    path: apps/bgd/overlays/bgdk
    repoURL: https://github.com/redhat-developer-demos/openshift-gitops-examples
    targetRevision: minikube
  syncPolicy:
    automated:
      prune: true
      selfHeal: false
    syncOptions:
    - CreateNamespace=true
```

Cette application pointe vers le même référentiel mais vers un répertoire différent .

Il s'agit d'utiliser un concept de « superposition », où vous disposez d'un ensemble de manifestes « de base » et vous superposez vos personnalisations.

Jetez un oeil au fichier kustomization.yaml :

``` 
vi bgdk-app/kustomize.yaml
```

``` 
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: bgdk
resources:
- ../../base
- bgdk-ns.yaml
patchesJson6902:
  - target:
      version: v1
      group: apps
      kind: Deployment
      name: bgd
      namespace: bgdk
    patch: |-
      - op: replace
        path: /spec/template/spec/containers/0/env/0/value
        value: yellow
  - target:
      version: v1
      group: networking.k8s.io
      kind: Ingress
      name: bgd
      namespace: bgdk
    patch: |-
      - op: replace
        path: /spec/rules/0/host
        value: bgdk.devnation
```

Cela kustomization.yamlprend l'application de base et corrige le manifeste de sorte que nous obtenions un carré jaune au lieu d'un bleu. 

Il déploie également l'application dans le namespace bgd (indiqué par la namespace:section du fichier) et met à jour le nom d'hôte Ingress pour ce nouvel espace de noms à bgdk.devnation.

Déployer cette application :

``` 
kubectl apply -f bgdk-app/bgdk-app.yaml
```

Cela devrait vous montrer deux applications sur l'interface utilisateur du CD Argo.


![image](/argocd-tutorial/images/attachments/elementaire/two-apps.png)


Obtenez l'URL et visitez votre application dans un navigateur Web :

``` 
minikube service bgd --url -n bgdk -p gitops
```

``` 
http://127.0.0.1:60458
```

![image](/argocd-tutorial/images/attachments/elementaire/yellow-square.png)


Comme vous pouvez le voir, l'application s'est déployée avec vos personnalisations ! Pour revoir ce que nous venons de faire.

J'ai déployé une application appelée bgd avec un carré bleu.

Déploiement d'une autre application basée sur bgd qui appel bgdk

L'application bgdk a été déployée dans son propre espace de noms, avec des personnalisations de déploiement.

TOUT sans avoir à dupliquer YAML !
