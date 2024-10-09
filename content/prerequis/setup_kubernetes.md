+++
title = 'Lancement Kubernetes'
date = 2024-10-09T09:50:38+02:00
draft = false
weight = 3
+++

### Installation de minikube

#### Linux

```
mkdir bin && cd bin
curl -Lo minikube https://storage.googleapis.com/minikube/releases/v1.29.0/minikube-linux-amd64
chmod +x minikube
curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.26.1/bin/linux/amd64/kubectl
chmod +x kubectl
cd ..
```

#### MacOs 

```
mkdir bin && cd bin
curl -Lo minikube https://storage.googleapis.com/minikube/releases/v1.29.0/minikube-darwin-amd64
chmod +x minikube
curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.26.1/bin/darwin/amd64/kubectl
chmod +x kubectl
cd ..
```

Et ajouter les variables d'environnement : 

```
export MINIKUBE_HOME=$(pwd);
export PATH=$MINIKUBE_HOME/bin:$PATH
export KUBECONFIG=$MINIKUBE_HOME/.kube/config
export KUBE_EDITOR="code -w"
```

Conserver les paramètres de Vim dans .vimrc

Nous examinons les paramètres importants de Vim si vous souhaitez travailler avec YAML pendant le TP K8s.

**Paramètres**

Créez d'abord ou ouvrez (s'il existe déjà) le fichier .vimrc :

```
vim ~/.vimrc
```

Saisissez ensuite (en mode insertion activé avec i) les lignes suivantes :

```
alias k=kubectl
```

Sauvegardez et fermez le fichier en appuyant sur Esc suivi de :x et Enter.

### Démarrer le cluster Kubernetes

#### Linux

```
minikube start --memory=8192 --cpus=3 --kubernetes-version=v1.26.1 --vm-driver=docker -p gitops
```

Avec un proxy :

```
minikube start --memory=8192 --cpus=3 --docker-env HTTPS_PROXY=$HTTPS_PROXY --docker-env HTTP_PROXY=$HTTP_PROXY --docker-env=NO_PROXY=$NO_PROXY --kubernetes-version=v1.26.1 --vm-driver=docker -p gitops
```


#### MacOs 

```
minikube start --memory=8192 --cpus=3 --kubernetes-version=v1.26.1 --vm-driver=docker -p gitops
```

Avec un proxy :

```
minikube start --memory=8192 --cpus=3 --docker-env HTTPS_PROXY=$HTTPS_PROXY --docker-env HTTP_PROXY=$HTTP_PROXY --docker-env=NO_PROXY=$NO_PROXY --kubernetes-version=v1.26.1 --vm-driver=docker -p gitops
```

Et le résultat doit être quelque chose de similaire :

```
😄  [devnation] minikube v1.20.0 on Darwin 11.3
✅  Created a new profile : devnation
✅  minikube profile was successfully set to devnation
😄  [default] minikube v1.29.0 on Darwin 11.3
✨  Selecting 'virtualbox' driver from user configuration (alternates: [hyperkit])
🔥  Creating virtualbox VM (CPUs=2, Memory=8192MB, Disk=50000MB) ...
🐳  Preparing Kubernetes v1.26.1 on Docker '20.10.6' ...
    ▪ apiserver.enable-admission-plugins=LimitRanger,NamespaceExists,NamespaceLifecycle,ResourceQuota,ServiceAccount,DefaultStorageClass,MutatingAdmissionWebhook
🚜  Pulling images ...
🚀  Launching Kubernetes ...
⌛  Waiting for cluster to come online ...
🏄  Done! kubectl is now configured to use "devnation"
```

Enfin, configurez l'utilisation de minikube internal docker comme docker host :

```
eval $(minikube docker-env)
```