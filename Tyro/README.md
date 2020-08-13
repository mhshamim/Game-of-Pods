# Tyro

Figure below explains the Jekyll SSG:

![Figure-Bravo](https://github.com/mhshamim/Game-of-Pods/blob/master/scenarios/Game-of-Pods-Tyro-Deploy.JPG)

* [jekyll-pvc](jekyll-pvc.yml) : persistent volumes claim
* [jekyll-pod](jekyll-pod.yml) : Pod manifest file
* [jekyll-service](jekyll-node-service.yml) : Service NodePort


## Configuration

### Namespace

namspace called 'development' has already been created. Inspect it.

```sh
master $ kubectl get ns
NAME          STATUS    AGE
default       Active    56m
development   Active    16m
kube-public   Active    56m
kube-system   Active    56m
```

### Persistent Volume

jekyll-pv is already created. Inspect it before you create the pvc
PV is using hostpath '/site'

```sh
master $ kubectl get pv
NAME          CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM     STORAGECLASS   REASON    AGE
jekyll-site   1Gi        RWX            Retain           Available                                      17m
```

### Persistent Volume Claim

Storage Request: 1Gi
Access modes: ReadWriteMany
pvc name = jekyll-site, namespace development

```sh
master $ kubectl apply -f jekyll-pvc.yml
persistentvolumeclaim/jekyll-site created

master $ kubectl get pvc -n development
NAME          STATUS    VOLUME        CAPACITY   ACCESS MODES   STORAGECLASS   AGE
jekyll-site   Bound     jekyll-site   1Gi        RWX                           2m
```

### Pod Config

pod: 'jekyll' has an initContainer, name: 'copy-jekyll-site', image: 'kodekloud/jekyll'
initContainer: 'copy-jekyll-site' command: [ "jekyll", "new", "/site" ] (command to run: jekyll new /site)
pod: 'jekyll', initContainer: 'copy-jekyll-site', volume name = site, mountPath = /site
pod: 'jekyll', container: 'jekyll', image = kodekloud/jekyll-serve, volume name = site, mountPath = /site
pod: 'jekyll', uses volume called 'site' with pvc = 'jekyll-site'
pod: 'jekyll', uses label 'run=jekyll'

```sh
master $ kubectl apply -f jekyllpod.yml
pod/jekyll created

master $ kubectl get pods -n development
NAME      READY     STATUS    RESTARTS   AGE
jekyll    1/1       Running   0          51s
```

### Service NodePort

Service 'jekyll' uses targetPort: '4000' , namespace: 'development'
Service 'jekyll' uses Port: '8080' , namespace: 'development'
Service 'jekyll' uses NodePort: '30097' , namespace: 'development'
Service 'jekyll' uses selector: 'run=jekyll' (for pod: jekyll) , namespace: 'development'

```sh
master $ kubectl apply -f jekyll-node-service.yml
service/jekyll created

master $ kubectl get svc -n development
NAME      TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)          AGE
jekyll    NodePort   10.104.91.1   <none>        8080:30097/TCP   10s
```
