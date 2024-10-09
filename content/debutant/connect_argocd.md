+++
title = 'Connexion à ArgoCD'
date = 2024-10-09T10:00:57+02:00
draft = false
weight = 1
+++


Maintenant que vous avez vérifié qu'Argo CD est opérationnel, explorons comment accéder et gérer Argo CD.

### Connexion à Argo CD

ArgoCD génère un admin utilisateur par défaut et un mot de passe aléatoire lors du premier déploiement.

Vous pouvez vous connecter à Argo CD en utilisant ce compte utilisateur via la CLI ou la console Web.

Connexion avec la CLI
Pour vous connecter à l'aide de la CLI, vous devez obtenir le mot de passe administrateur et l'URL de l'instance Argo CD :

``` 
minikube -p gitops service argocd-server -n argocd --url -p gitops
```

``` 
http://127.0.0.1:59466
```

terminal 2 
 
``` 
argoPass=$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d)
echo $argoPass
```

``` 
argoURL=127.0.0.1:59466
```

Connectez-vous à Argo CD avec la argocdCLI en utilisant l'URL et le mot de passe :

``` 
argocd login --insecure --grpc-web $argoURL  --username admin --password $argoPass
```

Le message suivant sera imprimé une fois la connexion réussie :

``` 
'admin:login' logged in successfully
```

#### Connexion à la console Web

Exposez la console ArgoCD à l'aide du service minikube.

``` 
http://127.0.0.1:59466
```

Accédez à la console Argo CD en vous connectant avec le nom d'utilisateur adminet le mot de passe extraits à l'étape précédente :

![image](/argocd-tutorial/images/attachments/debutant/argocd-login.png)

Une fois connecté, vous devriez voir la page suivante. Il s'agit de l'interface Web d'Argo CD.

![image](/argocd-tutorial/images/attachments/debutant/argocd-login2.png)


#### Déployer un exemple d'application

##### Examiner les manifestes de l'application

Les manifestes d'application incluent un namespace, un déploiement et des manifestes de mise en réseau pour Minikube. 
Le déploiement de ces manifestes sur un cluster génère une application prenant un Ingress.

> [!IMPORTANT]
> Vérifiez, mais n'appliquez pas ces manifestes à votre cluster. Nous le ferons bientôt à l'aide d'Argo CD.

Un namespace :

```
apiVersion: v1
kind: Namespace
metadata:
  name: bgd
spec: {}
status: {}
```

Un déploiement : 

```
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: bgd
  name: bgd
  namespace: bgd
spec:
  replicas: 1
  selector:
    matchLabels:
      app: bgd
  strategy: {}
  template:
    metadata:
      labels:
        app: bgd
    spec:
      containers:
      - image: quay.io/redhatworkshops/bgd:latest
        name: bgd
        env:
        - name: COLOR
          value: "blue"
        resources: {}
---
```

Un service :

```
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: bgd
  name: bgd
  namespace: bgd
spec:
  type: NodePort
  ports:
  - port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    app: bgd
---
```

Un Ingress

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: bgd
spec:
  rules:
    - host: bgd.devnation
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: bgd
                port:
                  number: 8080
```


