+++
title = 'Resources Hooks'
date = 2024-10-09T12:45:13+02:00
draft = false
weight = 2
+++


Maintenant que vous êtes familiarisé avec syncwaves, nous pouvons commencer à explorer l’application de manifestes par phases à l’aide de resource hooks.

Le contrôle de votre opération de synchronisation peut être redéfini davantage en utilisant des hooks. Ces hooks peuvent s'exécuter avant, pendant et après une opération de synchronisation. Ces hooks sont :

* PreSync – S'exécute avant l'opération de synchronisation. Il peut s'agir par exemple d'une sauvegarde de base de données avant un changement de schéma
* Synchronisation - S'exécute après PreSyncune exécution réussie. Cela s'exécutera parallèlement à vos manifestes normaux.
* PostSync - S'exécute après Syncune exécution réussie. Il peut s'agir d'un message Slack ou d'une notification par e-mail.
* SyncFail - S'exécute si l' Syncopération a échoué. Cette fonction est également utilisée pour envoyer des notifications ou effectuer d'autres actions évasives.

Pour activer une synchronisation, annotez le manifeste d'objet spécifique avec argocd.argoproj.io/hookle type de synchronisation que vous souhaitez utiliser pour cette ressource. Par exemple, si je voulais utiliser le PreSynchook :


``` 
metadata:
  annotations:
    argocd.argoproj.io/hook: PreSync
```

Vous pouvez également faire en sorte que les hooks soient supprimés après une exécution réussie/infructueuse.

HookSucceeded - La ressource sera supprimée une fois l'opération réussie.

HookFailed - La ressource sera supprimée en cas d'échec.

BeforeHookCreation - La ressource sera supprimée avant qu'une nouvelle ne soit créée (lorsqu'une nouvelle synchronisation est déclenchée).

Vous pouvez les appliquer avec l'annotation argocd.argoproj.io/hook-delete-policy. Par exemple :


``` 
metadata:
  annotations:
    argocd.argoproj.io/hook: PostSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
```

> [!IMPORTANT]
> Étant donné qu’une synchronisation peut échouer à n’importe quelle phase, vous pouvez arriver à une situation où l’application ne signale jamais qu’elle est healthy !


Bien que les hooks puissent être n'importe quelle ressource, ce sont généralement des Pods et/ou des Jobs.


### Exploration des manifestes

Jetez un œil à ce PostSyncmanifeste qui envoie une requête HTTP pour insérer un nouvel élément TODO :

``` 
apiVersion: batch/v1
kind: Job
metadata:
  name: todo-insert
  annotations:
    argocd.argoproj.io/hook: PostSync 
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  ttlSecondsAfterFinished: 100
  template:
    spec:
      containers:
        - name: httpie
          image: alpine/httpie:2.4.0
          imagePullPolicy: Always
          command: ["http"]
          args:
            [
              "POST",
              "todo-gitops:8080/api",
              "title=Finish ArgoCD tutorial",
              "--ignore-stdin"
            ]
      restartPolicy: Never
  backoffLimit: 1
```

Cela signifie que ce travail s'exécutera dans la PostSync phase, après l'application des manifestes dans la Sync phase.

> [!IMPORTANT]
> Comme je n'ai pas de politique de suppression, ce travail « restera en place » après avoir été terminé.

L'ordre d'exécution peut être vu dans le diagramme suivant :


![image](/argocd-tutorial/images/attachments/intermediare/presyncpost.png)


### Déploiement de l'application

Jetons un œil à ce fichier manifeste todo-application.yaml :

```
mkdir hook
cd hook
vi application-todo.yaml
```

```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: todo-app
  namespace: argocd
spec:
  destination:
    namespace: todo
    server: https://kubernetes.default.svc
  project: default
  source:
    path: apps/todo
    repoURL: https://github.com/redhat-developer-demos/openshift-gitops-examples
    targetRevision: minikube
  syncPolicy:
    automated:
      prune: true
      selfHeal: false
    syncOptions:
    - CreateNamespace=true
```


Cela montrera que cela déploiera l’application dans le namsepace todo.

Créer cette application :

```
kubectl apply -f todo-application.yaml
```

```
application.argoproj.io/todo-app created
```

![image](/argocd-tutorial/images/attachments/intermediare/todo-card.png)

En cliquant sur cette « carte », vous devriez accéder à l’arborescence.

![image](/argocd-tutorial/images/attachments/intermediare/todo-argocd.png)

Observez le processus de synchronisation. Vous verrez l'ordre dans lequel la ressource a été appliquée, d'abord la création de l'espace de noms et enfin la création de la route pour accéder à l'application.

Une fois l'application entièrement synchronisée, jetez un œil aux pods et aux tâches dans le namespace :


```
kubectl get pods -n todo
```

Vous devriez voir que le travail est terminé, mais toujours là.

```
NAME                           READY   STATUS      RESTARTS   AGE
postgresql-599467fd86-cgj9v    1/1     Running     0          32s
todo-gitops-679d88f6f4-v4djp   1/1     Running     0          19s
todo-table-xhddk               0/1     Completed   0          27s
```

```
minikube service todo-gitops --url -n todo -p gitops
```

```
http://127.0.0.1:62953
```

Pour accéder à votre application : 

```
http://127.0.0.1:62953/todo.html
```

Votre application doit ressembler à cela : 

![image](/argocd-tutorial/images/attachments/intermediare/todo-app-screenshot.png)

La tâche todo-insert n'est pas affichée car elle a été configurée pour être supprimée en cas de succès :

```
argocd.argoproj.io/hook-delete-policy: HookSucceeded
```


### A vous de jouez !

L’objectif est de garantir le bon fonctionnement de l’application. Actuellement, il a été constaté que l’insertion de données via le navigateur web de l’application n’est pas possible.

Il est donc nécessaire d’identifier l’origine de ce dysfonctionnement et de le corriger afin de permettre l’insertion des informations telles que le nom et la date.

* Identifier l’origine du dysfonctionnement 
* Corriger le problème pour permettre l’insertion des données suivantes :
   - Nom
   - Date

  ![image](/argocd-tutorial/images/attachments/intermediare/resolv-app.png)



