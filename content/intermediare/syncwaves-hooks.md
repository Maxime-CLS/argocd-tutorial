+++
title = 'SyncWaves et Hooks'
date = 2024-10-09T12:24:30+02:00
draft = false
weight = 1
+++

Les Syncwaves sont utilisés dans Argo CD pour ordonner la manière dont les manifestes sont appliqués au cluster.

D'autre part, les hooks de ressources divisent la livraison de ces manifestes en différentes phases.

En utilisant une combinaison de syncwaves et de hooks de ressources, vous pouvez contrôler le déploiement de votre application.

Cet exemple vous guidera à travers les étapes suivantes :

* Utilisation de Syncwaves pour commander un déploiement
* Exploration des crochets de ressources
* Utiliser Syncwaves et Hooks ensemble

L'exemple d'application que nous allons déployer est une application TODO avec une base de données et, outre les fichiers de déploiement, des syncwaves et des hooks de ressources sont utilisés :

![image](/argocd-tutorial/images/attachments/intermediare/todo-app.png)

### Utilisation des Waves de synchronisation

Un Syncwave est un moyen d'ordonner la manière dont Argo CD applique les manifestes qui sont stockés dans git. Tous les manifestes ont une wave de zéro par défaut, mais vous pouvez les définir en utilisant l' argocd.argoproj.io/sync-wave annotation.

Exemple:

``` 
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "2"
```

La wave peut également être négative.

``` 
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "-5"
```

Lorsque Argo CD démarre une action de synchronisation, le manifeste est placé dans l'ordre suivant :

* La phase dans laquelle ils se trouvent (nous aborderons les phases dans la section suivante)
* L'onde dans laquelle la ressource est annotée (en partant de la valeur la plus basse jusqu'à la plus haute)
* Par type (espaces de noms d’abord, puis services, puis déploiements, etc…)
* Par nom (ordre croissant)


#### À la découverte des manifestes

L'exemple d'application que nous allons déployer possède les manifestes suivants :

Le namespace avec syncwave comme -1 :

``` 
apiVersion: v1
kind: Namespace
metadata:
  name: todo
  annotations:
    argocd.argoproj.io/sync-wave: "-1"
```

Le PostgreSQL avec syncwave à 0 :


``` 
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgresql
  namespace: todo
  annotations:
    argocd.argoproj.io/sync-wave: "0"
spec:
  selector:
    matchLabels:
      app: postgresql
  template:
    metadata:
      labels:
        app: postgresql
    spec:
      containers:
        - name: postgresql
          image: postgres:12
          imagePullPolicy: Always
          ports:
            - name: tcp
              containerPort: 5432
          env:
            - name: POSTGRES_PASSWORD
              value: admin
            - name: POSTGRES_USER
              value: admin
            - name: POSTGRES_DB
              value: todo
```

Le service PostgreSQL avec syncwave à 0 :

``` 
---
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: todo
  annotations:
    argocd.argoproj.io/sync-wave: "0"
spec:
  selector:
    app: postgresql
  ports:
    - name: pgsql
      port: 5432
      targetPort: 5432
```

Création de la table de base de données avec syncwave comme 1 :

``` 
apiVersion: batch/v1
kind: Job
metadata:
  name: todo-table
  namespace: todo
  annotations:
    argocd.argoproj.io/sync-wave: "1"
spec:
  ttlSecondsAfterFinished: 100
  template:
    spec:
      containers:
        - name: postgresql-client
          image: postgres:12
          imagePullPolicy: Always
          env:
            - name: PGPASSWORD
              value: admin
          command: ["psql"]
          args:
            [
              "--host=postgresql",
              "--username=admin",
              "--no-password",
              "--dbname=todo",
              "--command=create table Todo (id bigint not null,completed boolean not null,ordering integer,title varchar(255),url varchar(255),primary key (id));create sequence hibernate_sequence start with 1 increment by 1;",
            ]
      restartPolicy: Never
  backoffLimit: 1
```

Le déploiement de l'application TODO avec syncwave en 2 :

``` 
---
apiVersion: "v1"
kind: "ServiceAccount"
metadata:
  labels:
    app.kubernetes.io/name: "todo-gitops"
    app.kubernetes.io/version: "1.0.0"
  name: "todo-gitops"
  namespace: todo
  annotations:
    argocd.argoproj.io/sync-wave: "2"
---
apiVersion: "apps/v1"
kind: "Deployment"
metadata:
  labels:
    app.kubernetes.io/name: "todo-gitops"
    app.kubernetes.io/version: "1.0.0"
  name: "todo-gitops"
  namespace: todo
  annotations:
    argocd.argoproj.io/sync-wave: "2"
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: "todo-gitops"
      app.kubernetes.io/version: "1.0.0"
  template:
    metadata:
      labels:
        app.kubernetes.io/name: "todo-gitops"
        app.kubernetes.io/version: "1.0.0"
    spec:
      containers:
      - env:
        - name: "KUBERNETES_NAMESPACE"
          valueFrom:
            fieldRef:
              fieldPath: "metadata.namespace"
        image: "quay.io/rhdevelopers/todo-gitops:1.0.0"
        imagePullPolicy: "Always"
        name: "todo-gitops"
        ports:
        - containerPort: 8080
          name: "http"
          protocol: "TCP"
      serviceAccount: "todo-gitops"
```

La configuration TODO Ingress avec syncwave en 3 :


``` 
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: todo
  namespace: todo
  annotations:
    argocd.argoproj.io/sync-wave: "3"
spec:
  rules:
    - host: todo.devnation
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: todo-gitops
                port:
                  number: 8080
```

Argo CD appliquera d'abord le namespace (puisqu'il s'agit de la valeur la plus basse) et s'assurera qu'il renvoie un statut « healthy » avant de continuer.

Ensuite, le déploiement PostgreSQL sera appliqué. Après cela, les rapports sains continueront avec le reste des ressources.

> [!INFO]
> Argo CD n'appliquera pas le manifeste suivant tant que les rapports précédents ne seront pas « healthy ».

