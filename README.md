# Game of Pods

![KODEKLOUD](https://process.fs.teachablecdn.com/ADNupMnWyR7kCWRvm76Laz/resize=height:20/https://www.filepicker.io/api/file/OapjGZrUQRiPge9Kc2xu)


Solution of the game to testing Kubernestes skills.

  - Bravo
  - Pento
  - Redis Islands
  - Tyro
  - Voting App
  - Iron Gallery

## Bravo

Figure below explains the Drupal deployment:

![Figure-Bravo](https://github.com/mhshamim/Game-of-Pods/blob/master/scenarios/Game-of-Pods-Bravo-Deploy.JPG?raw=true)

* [drupal-service](drupal-service.yml) - drupal service
* [drupal](drupal-service.yml) - drupal frontend deployment
* [drupal-pvc] - drupal persistent volume claim used by the deployment
* [drupal-pv] - persistent volume using hostpath
* [drupal-mysql-service] - drupal backend service exposing mysql deployment port
* [drupal-mysql] - drupal backend deployment using mysql db
* [drupal-mysql-secret] - mysql environment variables
* [drupal-mysql-pvc] - mysql persistent volume claim used by the deployment
* [drupal-mysql-pv] - persistent volume using hostpath


### Installation

Run all the required mentioned kubernetes app resources listed above:

- Create the folders on kubernetes nodes for persistent volumes
```sh
node01 $ mkdir /drupal-data
node01 $ mkdir /drupal-mysql-data
```

- Create the persistent volume resources

```sh
master $ kubectl apply -f drupal-pv-hostpath.yml
persistentvolume/drupal-pv created
master $ kubectl apply -f drupal-mysql-pv-hostpath.yml
persistentvolume/drupal-mysql-pv created
```

- Create persistent volume claim resources

```sh
master $ kubectl apply -f drupal-pvc.yml
persistentvolumeclaim/drupal-pvc created
master $ kubectl apply -f drupal-mysql-pvc.yml
persistentvolumeclaim/drupal-mysql-pvc created
```
- Check and verify the volumes are bounded to their respective persistent volumes
```sh
master $ kubectl get pv,pvc
NAME                               CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS    CLAIM                      STORAGECLASS   REASON    AGE
persistentvolume/drupal-mysql-pv   5Gi        RWO            Retain           Bound     default/drupal-mysql-pvc                             38s
persistentvolume/drupal-pv         5Gi        RWO            Retain           Bound     default/drupal-pvc                                   1m

NAME                                     STATUS    VOLUME            CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/drupal-mysql-pvc   Bound     drupal-mysql-pv   5Gi        RWO                           15s
persistentvolumeclaim/drupal-pvc         Bound     drupal-pv         5Gi        RWO                           22s
```
- Create mysql environment variables as secret
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

- Create mysql & drupal service exposing the mysql ports

```sh
master $ kubectl apply -f drupal-mysql-service.yml
service/drupal-mysql-service created
master $ kubectl apply -f drupal-service.yml
service/drupal-service created

master $ kubectl get svc
NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
drupal-mysql-service   ClusterIP   10.109.55.129   <none>        3306/TCP       1m
drupal-service         NodePort    10.99.78.233    <none>        80:30095/TCP   1m
kubernetes             ClusterIP   10.96.0.1       <none>        443/TCP        1h
```

- Create mysql deployment

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

- Create drupal deployment

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
