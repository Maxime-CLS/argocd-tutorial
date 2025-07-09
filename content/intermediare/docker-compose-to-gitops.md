+++
title = 'De Docker Compose à GitOps avec ArgoCD'
date = 2024-10-09T12:45:13+02:00
draft = false
weight = 3
+++


Maintenant que vous maîtrisez les bases d’ArgoCD, c’est le moment de passer à la pratique complète !
L’objectif est de migrer une application Docker Compose vers Kubernetes et de l’automatiser avec GitOps via ArgoCD, GitLab et Kustomize.

Voici le docker-compose.yaml 


``` 
version: '2.2'

services:
  nginx:
    image: nginx:1.27.5
    container_name: nginx
    labels:
      "co.elastic.logs/module": "nginx"
      "co.elastic.logs/fileset.stdout": "access"
      "co.elastic.logs/fileset.stderr": "error"
      "co.elastic.metrics/module": "nginx"
      "co.elastic.metrics/hosts": "nginx:8081"
      "co.elastic.metrics/metricsets": "stubstatus"
    ports:
      - 8081:8081
    volumes:
      - ./conf/default.conf:/etc/nginx/conf.d/default.conf:ro

  petclinic:
    image: docker.io/michaelhyatt/elastic-k8s-o11y-workshop-petclinic:1.25.0
    container_name: petclinic
    labels:
      "co.elastic.metrics/module": "prometheus"
      "co.elastic.metrics/hosts": "$${data.host}:$${data.port}"
      "co.elastic.metrics/metrics_path": "/metrics/prometheus"
      "co.elastic.metrics/period": "1m"
    environment:
      ELASTIC_APM_SERVER_URLS: "http://fleet-server:8200"
      ELASTIC_APM_SERVER_URLS_FOR_RUM: "http://localhost:8200"
      ELASTIC_APM_SECRET_TOKEN: ""
      ELASTIC_APM_SERVICE_NAME: "spring-petclinic-monolith"
      ELASTIC_APM_APPLICATION_PACKAGES: "org.springframework.samples"
      ELASTIC_APM_ENABLE_LOG_CORRELATION: "true"
      ELASTIC_APM_CAPTURE_JMX_METRICS: >
        object_name[java.lang:type=GarbageCollector,name=*] attribute[CollectionCount:metric_name=collection_count] attribute[CollectionTime:metric_name=collection_time],
        object_name[java.lang:type=Memory] attribute[HeapMemoryUsage:metric_name=heap]
      JAVA_OPTS: >
        -Xms100m
        -Xmx256m
        -Dspring.profiles.active=mysql
        -Ddatabase=mysql
        -Dspring.datasource.username=root
        -Dspring.datasource.password=petclinic
        -Dspring.datasource.initialization-mode=always
        -Dspring.datasource.url=jdbc:mysql://mysql:3306/petclinic?autoReconnect=true&useSSL=false
        -XX:+StartAttachListener
    ports:
      - 8080

  mysql:
    image: mariadb:10.5.8
    container_name: mysql
    labels:
      "co.elastic.logs/module": "mysql"
      "co.elastic.metrics/module": "mysql"
      "co.elastic.metrics/hosts": "root:petclinic@tcp($${data.host}:3306)/"
    environment:
      MYSQL_ROOT_PASSWORD: petclinic
      MYSQL_DATABASE: petclinic
    ports:
      - 3306

networks:
  default:
    name: elastic
    driver: bridge
``` 

Voici le fichier de configuration nginx (default.conf) : 

``` 
server {
        error_log  /var/log/nginx/error.log debug;

        listen       8081;
        listen [::]:8001;
        server_name  localhost;
        location /intake {
          if ($request_method = 'OPTIONS') {
            # This is a bit too wide, would be enough to replace it with only APM server url for CORS support.
            add_header 'Access-Control-Allow-Origin' '*';
            add_header 'Access-Control-Allow-Methods' 'GET,POST,OPTIONS';
            add_header 'Access-Control-Allow-Headers' 'Accept:,Accept-Encoding,Accept-Language:,Cache-Control,Connection,DNT,Pragma,Host,Referer,Upgrade-Insecure-Requests,User-Agent,elastic-apm-traceparent';
            add_header 'Access-Control-Max-Age' 1728000;
            add_header 'Access-Control-Allow-Credentials' 'true';
            add_header 'Content-Type' 'text/plain; charset=utf-8';
            add_header 'Content-Length' 0;
            return 200;
          }
        }
        location / {
          proxy_pass http://petclinic:8080/;
        }
        location /nginx_status {
         	stub_status on;
        	allow all; # A bit too wide
         	# deny all;
        }
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   /usr/share/nginx/html;
        }
    }
``` 

### Objectif final
L’application doit être déployée dans votre cluster Kubernetes à partir d’un dépôt GitLab, via ArgoCD et Kustomize.


### Application cible

Voici les services à déployer :

nginx (exposé sur le port 8081)

petclinic (application Spring Boot)

mysql (avec secret pour les credentials)

Le Docker Compose de départ vous a été fourni.

#### Étapes à réaliser

1. Créez un dépôt GitLab public contenant votre projet Kubernetes.

2. Structurez votre dépôt comme suit :

``` 
└── k8s/
    ├── base/
    │   ├── nginx.yaml
    │   ├── petclinic.yaml
    │   ├── mysql.yaml
    │   ├── secret-mysql.yaml
    └── kustomization.yaml
``` 

3. Convertissez le docker-compose.yml fourni en manifests Kubernetes :

Utilisez Deployment, Service, ConfigMap, Secret comme montré dans les exemples précédents.

Sécurisez les credentials de MySQL via un Secret.

4. Écrivez un fichier kustomization.yaml :

``` 
resources:
  - nginx.yaml
  - petclinic.yaml
  - mysql.yaml
  - secret-mysql.yaml
``` 

5. Ajoutez une Application ArgoCD pointant vers votre dépôt GitLab :

Repo URL : votre dépôt GitLab

Path : k8s/base

Namespace : myapp 

Activez la synchronisation automatique 

6. Lancez le déploiement avec ArgoCD :

Via kubectl apply d'un kind ArgoCD de type Application

Vérifiez que les pods sont Running et que les services sont exposés

7. Accéder au service de la l'application 

`minikube service nginx -n default --url`

Vous devez avoir accès à l'applciaiton comme ceci : 

![image](/argocd-tutorial/images/attachments/intermediare/petclinic.png)



#### Critères de réussite

 Votre dépôt GitLab contient bien la stack Kubernetes

 Le fichier kustomization.yaml est fonctionnel

 Les secrets ne sont pas exposés en clair

 ArgoCD déploie automatiquement votre stack dans le cluster

 nginx est accessible sur le port 8081

 petclinic est opérationnel sur le port 8080 avec MySQL fonctionnel

