# Pento

Figure below explains the Pento setup:

![Figure-Bravo](https://github.com/mhshamim/Game-of-Pods/blob/master/scenarios/Game-of-Pods-Pento-Deploy.JPG)

* [gop-fs-service](gop-fileserver-service.yml) : FileServer Expose Service
* [gop-fileserver](gop-fileserver-pod.yml) : FileServer Pod Deployment
* [data-pvc](data-pvc.yml) : data persistent volume claim used by the FileServer pod
* [data-pv](data-pv-hostpath.yml) : persistent volume using hostpath


## Configuration

### Master Node Troubleshooting

- kubeconfig = /root/.kube/config, User = 'kubernetes-admin' Cluster: Server Port = '6443'

```sh
master $ kubectl get nodes
The connection to the server 172.17.0.9:2379 was refused - did you specify the right host or port?

master $ vi /root/.kube/config
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJ....0tLQo=
    server: https://172.17.0.9:6443
  name: kubernetes
```

- Fix kube-apiserver. Make sure its running and healthy.


```sh
master $ docker ps -a |grep apiserver
master $ docker container logs 0aec
error: unable to load client CA file: unable to load client CA file: open /etc/kubernetes/pki/ca-authority.crt: no such file or directory

master $ vi /etc/kubernetes/manifests/kube-apiserver.yaml
spec:
  containers:
  - command:
    - kube-apiserver
    - --client-ca-file=/etc/kubernetes/pki/ca.crt
```

- Master node: coredns deployment has image: 'k8s.gcr.io/coredns:1.3.1'

```sh

master $ kubectl edit -n kube-system deployments coredns
deployment.extensions/coredns edited

image: k8s.gcr.io/coredns:1.3.1
```

### Worker Node Troubleshooting

- node01 is ready and can schedule pods?

```sh
master $ kubectl get nodes
NAME      STATUS                     ROLES     AGE       VERSION
master    Ready                      master    2h        v1.11.3
node01    Ready,SchedulingDisabled   <none>    2h        v1.11.3

master $ kubectl uncordon node01
node/node01 uncordoned

master $ kubectl get nodes
NAME      STATUS    ROLES     AGE       VERSION
master    Ready     master    2h        v1.11.3
node01    Ready     <none>    2h        v1.11.3
```

### Persistent Volume and Persistent Volume Claim setup for the file-server

- Create new PersistentVolume = 'data-pv'
- PersistentVolume = data-pv, accessModes = 'ReadWriteMany'
- PersistentVolume = data-pv, hostPath = '/web'
- PersistentVolume = data-pv, storage = '1Gi'

```sh
master $ kubectl apply -f data-pv-hostpath.yml
persistentvolume/data-pv created

master $ kubectl apply -f data-pvc.yml
persistentvolumeclaim/data-pvc created

master $ kubectl get pvc
NAME       STATUS    VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS   AGE
data-pvc   Bound     data-pv   1Gi        RWX                           19s
```

### File-Server pod deployment

- Create a pod for fileserver, name: 'gop-fileserver'
- pod: gop-fileserver image: 'kodekloud/fileserver'
- pod: gop-fileserver mountPath: '/web'
- pod: gop-fileserver volumeMount name: 'data-store'
- pod: gop-fileserver persistent volume name: data-store
- pod: gop-fileserver persistent volume claim used: 'data-pvc'

```sh
master $ kubectl apply -f gop-fileserver-pod.yml
pod/gop-fileserver created
```


### File-Server Expose Ports

- New Service, name: 'gop-fs-service'
- Service name: gop-fs-service, port: '8080'
- Service name: gop-fs-service, targetPort: '8080'
- Service name: gop-fs-service, NodePort: '31200'

```sh
master $ kubectl apply -f gop-fileserver-service.yml
service/gop-fs-service created

master $ kubectl get svc
NAME             TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
gop-fs-service   NodePort    10.106.147.144   <none>        8080:31200/TCP   3s
```
