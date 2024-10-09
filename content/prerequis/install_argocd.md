+++
title = 'Installation ArgoCD'
date = 2024-10-09T09:50:48+02:00
draft = false
weight = 4
+++

### Installation de ArgoCD

```
minikube addons enable ingress -p gitops
```

Vérifier que l'ingress controller est bien installé :

``` 
kubectl get pods -n ingress-nginx
```

Résultat :

```
NAME                                        READY   STATUS      RESTARTS    AGE
ingress-nginx-admission-create-g9g49        0/1     Completed   0          11m
ingress-nginx-admission-patch-rqp78         0/1     Completed   1          11m
ingress-nginx-controller-59b45fb494-26npt   1/1     Running     0          11m
```


installer ArgoCD et vérifier que chaque pod fonctionne correctement dans l'espace de noms argocd :

``` 
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

> [!INFO]
> L'installation des composants d'ArgoCD prendra quelques minutes. Vous pouvez suivre l'état de l'installation à l'aide de la commande :

``` 
watch kubectl get pods -n argocd
```

Un déploiement réussi d'ArgoCD affichera les pods suivants :

``` 
NAME                                  READY   STATUS    RESTARTS   AGE
argocd-application-controller-0       1/1     Running   0          2m18s
argocd-dex-server-5dd657bd9-2r24r     1/1     Running   0          2m19s
argocd-redis-759b6bc7f4-bnljg         1/1     Running   0          2m19s
argocd-repo-server-6c495f858f-p5267   1/1     Running   0          2m18s
argocd-server-859b4b5578-cv2qx        1/1     Running   0          2m18s
```

Corrigez le service ArgoCD de ClusterIP vers un LoadBalancer :

``` 
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```

Maintenant, avec la liste des services minikube, vous pouvez vérifier le service argocd exposé :

``` 
minikube -p gitops service list | grep argocd
```