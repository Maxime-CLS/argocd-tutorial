+++
title = 'Créer une application'
date = 2024-10-09T11:00:06+02:00
draft = false
+++

##### Déployer l'application

Une collection gérée de manifestes est appelée Applicationdans Argo CD. Par conséquent, vous devez la définir comme telle à l'aide d'un CR d'application (CustomResource) afin qu'Argo CD applique ces manifestes dans votre cluster.

Examinons le manifeste de l'application Argo CD utilisé pour déployer cette application et décomposons-le un peu :

``` 
mkdir bgd-app
vi bgd-app/bgd-app.yaml
```

```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: bgd-app
  namespace: argocd
spec:
  destination:
    namespace: bgd
    server: https://kubernetes.default.svc 
  project: default 
  source: 
    path: apps/bgd/overlays/bgd
    repoURL: https://github.com/redhat-developer-demos/openshift-gitops-examples
    targetRevision: minikube
  syncPolicy: 
    automated:
      prune: true
      selfHeal: false
    syncOptions:
    - CreateNamespace=true
```

1 - Le serveur de destination est le même serveur sur lequel nous avons installé Argo CD.
2- Ici, vous installez l'application dans defaultle projet Argo CD ( .spec.project).
3- Le référentiel manifeste et le chemin d'accès où réside le YAML.
4- Le syncPolicy est défini sur automated. Il supprimera automatiquement les ressources qui ont été supprimées du dépôt Git, mais ne corrigera pas automatiquement les ressources qui s'écartent de la définition stockée dans le dépôt, c'est-à-dire que les modifications manuelles kubectl ne seront pas « healthy ».


Appliquez l'application CR en exécutant la commande suivante :

``` 
kubectl apply -f bgd-app/bgd-app.yaml
```

L'application nouvellement créée apparaît sous forme de tuile avec le titre bgd-appdans l'interface utilisateur du CD Argo.

![image](/argocd-tutorial/images/attachments/debutant/argocd-app1.png)

En cliquant sur cette vignette, vous accédez à la page des détails de l'application. Vous pouvez la voir comme étant toujours en cours ou entièrement synchronisée.

![image](/argocd-tutorial/images/attachments/debutant/argocd-app2.png)

> [!INFO]
> Vous devrez peut-être cliquer sur show hidden resourcescette page pour voir toutes les ressources.

À ce stade, l'application doit être opérationnelle. Vérifiez que les ressources ont été créées :

``` 
kubectl get all -n bgd
```

La sortie doit répertorier un service, un déploiement et un pod :

``` 
NAME                       READY   STATUS    RESTARTS   AGE
pod/bgd-788cb756f7-kz448   1/1     Running   0          10m

NAME          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service/bgd   ClusterIP   172.30.111.118   <none>        8080/TCP   10m

NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/bgd   1/1     1            1           10m
```

Vérifiez que le déploiement est terminé :

``` 
kubectl rollout status deploy/bgd -n bgd 
```

Obtenez l'URL et visitez votre application dans un navigateur Web :

``` 
minikube service bgd --url -n bgd -p gitops
```

``` 
http://127.0.0.1:54134
```

Votre application devrait ressembler à ceci.

![image](/argocd-tutorial/images/attachments/debutant/argocd-app-blue.png)

##### Gestion des dérives de configuration

Introduisons un changement dans l'environnement de l'application ! Appliquez un correctif au manifeste de déploiement en direct pour changer la couleur des bulles de l'application du bleu au vert :

``` 
kubectl -n bgd patch deploy/bgd --type='json' -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/env/0/value", "value":"green"}]'
```

Attendez que le déploiement ait lieu :

``` 
kubectl rollout status deploy/bgd -n bgd
```

Actualisez l'onglet dans lequel votre application est en cours d'exécution. Vous devriez voir des bulles vertes.

![image](/argocd-tutorial/images/attachments/debutant/argocd-app-green.png)


En regardant l'interface Web de votre CD Argo, vous pouvez voir qu'Argo détecte votre application comme « désynchronisée ».

![image](/argocd-tutorial/images/attachments/debutant/out-of-sync.png)

Vous pouvez synchroniser votre application via le CD Argo en :

 * Premier clicSYNC
 * Puis en cliquantSYNCHRONIZE

Alternativement, vous pouvez exécuter la commande suivante :

``` 
argocd app sync bgd-app
```

Une fois le processus de synchronisation terminé, l'interface utilisateur du CD Argo doit marquer l'application comme synchronisée.

![image](/argocd-tutorial/images/attachments/debutant/fullysynced.png)

Rechargez la page sur l'onglet où l'application est en cours d'exécution. Les bulles devraient avoir repris leur couleur bleue d'origine.

![image](/argocd-tutorial/images/attachments/debutant/argocd-app-blue.png)

Vous pouvez configurer Argo CD pour corriger automatiquement la dérive en définissant le Application manifeste pour le faire. Exemple :

``` 
spec:
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Ou, comme dans notre cas, après coup en exécutant la commande suivante :

``` 
kubectl patch application/bgd-app -n argocd --type=merge -p='{"spec":{"syncPolicy":{"automated":{"prune":true,"selfHeal":true}}}}'
```