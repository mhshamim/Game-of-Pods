# Tyro

Figure below explains the Jekyll SSG:

![Figure-Bravo](https://github.com/mhshamim/Game-of-Pods/blob/master/scenarios/Game-of-Pods-Tyro-Deploy.JPG)

* [jekyll-pvc](jekyll-pvc.yml) : persistent volumes claim
* [jekyll-pod](jekyll-pod.yml) : Pod manifest file
* [jekyll-service](jekyll-node-service.yml) : Service NodePort
* [developer-RBAC](developer-rbac.yml) : RBAC Config


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

### RBAC Config

#### Role
'developer-role', should have all(*) permissions for services in development namespace
'developer-role', should have all(*) permissions for persistentvolumeclaims in development namespace
'developer-role', should have all(*) permissions for pods in development namespace

#### Rolebinding
create rolebinding = developer-rolebinding, role= 'developer-role', namespace = development
rolebinding = developer-rolebinding associated with user = 'drogo'

```sh
master $ kubectl apply -f developer-rbac.yml
role.rbac.authorization.k8s.io/developer-role created
rolebinding.rbac.authorization.k8s.io/developer-rolebinding created
```

### User Certificate

Certificate and key pair for user drogo is created under /root. Add this user to kubeconfig = /root/.kube/config, User = drogo, client-key = /root/drogo.key client-certificate = /root/drogo.crt
Create a new context in the default config file (/root/.kube/config) called 'developer' with user = drogo and cluster = kubernetes


```sh
master $ openssl genrsa -out drogo.key 2048
Generating RSA private key, 2048 bit long modulus
.................................................................+++
....................................................................................................................................................................................................+++
e is 65537 (0x10001)

master $ openssl req -new -key drogo.key -out drogo.csr -subj "/CN=drogo/O=developer"

master $ openssl x509 -req -in drogo.csr -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -out drogo.crt -days 500
Signature ok
subject=/CN=drogo/O=developer
Getting CA Private Key

master $ kubectl config set-credentials drogo --client-certificate=/root/drogo.crt  --client-key=/root/drogo.key
User "drogo" set.
```

### Kube Config

set context 'developer' with user = 'drogo' and cluster = 'kubernetes' as the current context.

```sh
master $ kubectl config set-context developer --user=drogo --cluster=kubernetes --namespace=development
Context "developer" created.

master $ kubectl config use-context developer --user=drogo --cluster=kubernetes
Switched to context "developer".

master $ kubectl config get-contexts
CURRENT   NAME                          CLUSTER      AUTHINFO           NAMESPACE
*         developer                     kubernetes   drogo
          kubernetes-admin@kubernetes   kubernetes   kubernetes-admin
```
