# Bravo

Figure below explains the Drupal deployment:

![Figure-Bravo](https://github.com/mhshamim/Game-of-Pods/blob/master/scenarios/Game-of-Pods-Bravo-Deploy.JPG?raw=true)

* [drupal-service](drupal-service.yml) : drupal service
* [drupal](drupal-deploy.yml) : drupal frontend deployment
* [drupal-pvc](drupal-pvc.yml) : drupal persistent volume claim used by the deployment
* [drupal-pv](drupal-pv-hostpath.yml) : persistent volume using hostpath
* [drupal-mysql-service](drupal-mysql-service.yml) : drupal backend service exposing mysql deployment port
* [drupal-mysql](drupal-mysql-deploy.yml) : drupal backend deployment using mysql db
* [drupal-mysql-secret](drupal-mysql-secret.yml) : mysql environment variables
* [drupal-mysql-pvc](drupal-mysql-pvc.yml) : mysql persistent volume claim used by the deployment
* [drupal-mysql-pv](drupal-mysql-pv-hostpath.yml) : persistent volume using hostpath


## Configuration

Run all the required mentioned kubernetes app resources listed above:


- Access modes: ReadWriteOnce
- Volume Name: drupal-pv
- Storage: 5Gi
- Configure drupal-pv with hostPath = /drupal-data (create the directory on Worker Nodes)

```sh
node01 $ mkdir /drupal-data

master $ kubectl apply -f drupal-pv-hostpath.yml
persistentvolume/drupal-pv created
```

- Volume Name: drupal-mysql-pv
- Storage: 5Gi
- Access modes: ReadWriteOnce
- Configure drupal-mysql-pv with hostPath = /drupal-mysql-data (create the directory on Worker Nodes)

```sh
node01 $ mkdir /drupal-mysql-data

master $ kubectl apply -f drupal-mysql-pv-hostpath.yml
persistentvolume/drupal-mysql-pv created
```

- Claim Name: drupal-pvc
- Storage Request: 5Gi
- Access modes: ReadWriteOnce

```sh
master $ kubectl apply -f drupal-pvc.yml
persistentvolumeclaim/drupal-pvc created
```

- Claim Name: drupal-mysql-pvc
- Storage Request: 5Gi
- Access modes: ReadWriteOnce

```sh
master $ kubectl apply -f drupal-mysql-pvc.yml
persistentvolumeclaim/drupal-mysql-pvc created
```

> Check and verify the volumes are bounded to their respective persistent volumes

```sh
master $ kubectl get pv,pvc
NAME                               CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS    CLAIM                      STORAGECLASS   REASON    AGE
persistentvolume/drupal-mysql-pv   5Gi        RWO            Retain           Bound     default/drupal-mysql-pvc                             38s
persistentvolume/drupal-pv         5Gi        RWO            Retain           Bound     default/drupal-pvc                                   1m

NAME                                     STATUS    VOLUME            CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/drupal-mysql-pvc   Bound     drupal-mysql-pv   5Gi        RWO                           15s
persistentvolumeclaim/drupal-pvc         Bound     drupal-pv         5Gi        RWO                           22s
```

- Secret Name: drupal-mysql-secret
- Secret: MYSQL_ROOT_PASSWORD=root_password
- Secret: MYSQL_DATABASE=drupal-database
- Secret: MYSQL_USER=root

```sh
master $ echo -n 'root_password' | base64
cm9vdF9wYXNzd29yZA==
master $ echo -n 'drupal-database' | base64
ZHJ1cGFsLWRhdGFiYXNl
master $ echo -n 'root' | base64
cm9vdA==

master $ kubectl apply -f drupal-mysql-secret.yml
secret/drupal-mysql-secret created
```

> Create env variables of secret using imperative command

```sh
kubectl create secret generic drupal-mysql-secret --from-literal=MYSQL_ROOT_PASSWORD='root_password' --from-literal=MYSQL_DATABASE=drupal-database --from-literal=MYSQL_USER=root 
```

- Name: drupal-mysql-service
- Type: ClusterIP
- Port: 3306
- Service Endpoint: drupal-mysql pod

```sh
master $ kubectl apply -f drupal-mysql-service.yml
service/drupal-mysql-service created
```

- frontend service name: drupal-service
- drupal-service configured as NodePort
- drupal-service uses NodePort 30095
- Service "drupal-service" connects to "drupal" pod with label "app=drupal"

```sh
master $ kubectl apply -f drupal-service.yml
service/drupal-service created

master $ kubectl get svc
NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
drupal-mysql-service   ClusterIP   10.109.55.129   <none>        3306/TCP       1m
drupal-service         NodePort    10.99.78.233    <none>        80:30095/TCP   1m
kubernetes             ClusterIP   10.96.0.1       <none>        443/TCP        1h
```

- Name: drupal-mysql
- Replicas: 1
- Image: mysql:5.7
- Deployment Volume uses PVC : drupal-mysql-pvc
- Volume Mount Path: /var/lib/mysql, subPath: dbdata
- Deployment: 'drupal-mysql' running
- ENV: MYSQL_DATABASE valueFrom secret: drupal-mysql-secret
- ENV: MYSQL_ROOT_PASSWORD valueFrom secret
- ENV: MYSQL_USER valueFrom secret: drupal-mysql-secret
- Volume: drupal-mysql-pv
- Volume Claim: drupal-mysql-pvc

```sh
master $ kubectl apply -f drupal-mysql-deploy.yml
deployment.apps/drupal-mysql created

master $ kubectl get deploy,rs,pods
NAME                                 DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.extensions/drupal-mysql   1         1         1            1           1m
NAME                                            DESIRED   CURRENT   READY     AGE
replicaset.extensions/drupal-mysql-55865dd69c   1         1         1         1m
NAME                                READY     STATUS    RESTARTS   AGE
pod/drupal-mysql-55865dd69c-hxrdm   1/1       Running   0          1m
```

- Deployment Name: drupal
- Replicas: 1
- Image: drupal:8.6
- Deployment has an initContainer, name: 'init-sites-volume'
- initContainer 'init-sites-volume', image: drupal:8.6
- initContainer 'init-sites-volume', persistentVolumeClaim: drupal-pvc
- initContainer 'init-sites-volume', mountPath: /data
- initContainer 'init-sites-volume', Command: [ "/bin/bash", "-c" ], initContainer: Args: [ 'cp -r /var/www/html/sites/ /data/; chown www-data:www-data /data/ -R' ]
- Deployment 'drupal' uses correct pvc: drupal-pvc
- Deployment has a regular container, name: 'drupal', image: 'drupal:8.6'
- container: 'drupal', Volume mountPath: /var/www/html/modules, subPath: modules
- container: 'drupal', Volume mountPath: /var/www/html/profiles, subPath: profiles
- container: 'drupal', Volume mountPath: /var/www/html/sites, subPath: sites
- container: 'drupal', Volume mountPath: /var/www/html/themes, subPath: themes
- Deployment: "drupal" running
- Deployment: 'drupal' has label 'app=drupal'
- Volume Claim: drupal-pvc

```sh
master $ kubectl apply -f drupal-deploy.yml
deployment.apps/drupal created

master $ kubectl get deploy,rs,pods
NAME                                 DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.extensions/drupal         1         1         1            1           20s
NAME                                           DESIRED   CURRENT   READY     AGE
replicaset.extensions/drupal-75d8567f9         1         1         1         20s
NAME                               READY     STATUS     RESTARTS   AGE
pod/drupal-75d8567f9-xdtbc         1/1       Running    1          20s
```