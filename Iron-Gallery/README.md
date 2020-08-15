# Iron Gallery

Figure below explains the Iron Gallery architecture:

![Figure-Bravo](https://github.com/mhshamim/Game-of-Pods/blob/master/scenarios/Game-of-Pods-IronGallery-Deploy.JPG)

* [ingress-config](gallery-ingress.yml) : persistent volumes claim
* [irongallery-app](irongallery-app.yml) : Iron Gallery App Service and Deployment Manifest
* [irongallery-netpol](irongallery-netpol.yml) : Network Policy definition
* [iron-db](iron-db.yml) : Iron DB Service and Deployment Manifest


### Ingress Config

Ingress resource configured correctly and application accessible at 'http://iron-gallery-braavos.com:30099/'  \
Ingress Resource, name: 'iron-gallery-ingress'  \
host: iron-gallery-braavos.com  \
http parth: '/'  \
http backend serviceName: 'iron-gallery-service'  \
Name: ingress-spacehttp backend servicePort: '80'  

## Iron Gallery App

### Iron Gallery Service

Service: 'iron-gallery-service' has 'one' endpoint for pods in deployment 'iron-gallery'?  \
targetPort: 80  \
port: 80  

### Iron Gallery Deployment

New Deployment, name: 'iron-gallery'  \
Image: 'kodekloud/irongallery:2.0'  \
volume, name = config, type: emptyDir  \
volume, name = images, type: emptyDir  \
volumeMount, name: 'config', mountPath: '/usr/share/nginx/html/data'  \
volumeMount, name: 'images', mountPath: '/usr/share/nginx/html/uploads'  \
Replicas: 1  \
Pod Label: 'run=iron-gallery'  

```sh
master $ kubectl apply -f irongallery-app.yml
service/iron-gallery-service created
deployment.apps/iron-gallery created
```

### Resource Limits

Deployment: 'iron-gallery', container has CPU limit: '50m'  \
Deployment: 'iron-gallery', container has Memory limit: '100Mi'

```sh
        resources:
            limits:
              memory: 100Mi
              cpu: 50m
```

### Network Policy

NetworkPolicy, name: 'iron-gallery-firewall'  \
Ingress Rule - from Pod labeled: 'run=iron-gallery'  \
Applied to Pod with label: 'db=mariadb'  \
Applied to allow access to port : '3306'  

```sh
master $ kubectl apply -f irongallery-netpol.yml
networkpolicy.networking.k8s.io/iron-gallery-firewall created

master $ kubectl get netpol
NAME                    POD-SELECTOR   AGE
iron-gallery-firewall   db=mariadb     6s
```

## Iron DB

### DB Service

Service: 'iron-db-service' has 'one' endpoint for pods in deployment 'iron-db'?  \
targetPort: 3306  \
Service Port: '3306'  

### DB Deployment

New Deployment, name: 'iron-db'  \
Image: 'kodekloud/irondb:2.0'  \
volume, name = db, type: emptyDir  \
volumeMount, name: 'db', mountPath: '/var/lib/mysql'  \
Replicas: 1  \
env, name: 'MYSQL_ROOT_PASSWORD', value: 'Braavo'  \
env, name: 'MYSQL_DATABASE', value: 'lychee'  \
env, name: 'MYSQL_USER', value: 'lychee'  \
env, name: 'MYSQL_PASSWORD', value: 'lychee'  \
Pod Label: 'db=mariadb'  

```sh
master $ kubectl apply -f iron-db.yml
service/iron-db-service created
deployment.apps/iron-db created
```