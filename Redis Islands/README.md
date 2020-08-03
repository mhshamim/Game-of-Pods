# Redis Islands

Figure below explains the Redis setup:

![Figure-Bravo](https://github.com/mhshamim/Game-of-Pods/blob/master/scenarios/Game-of-Pods-Redis-Deploy.JPG)

* [redis-pv-hostpath](redis-pv-hostpath.yml) : persistent volumes setup
* [redis-cluster-service](redis-svc.yml) : redis services
* [redis-cluster-configmap](redis-cluster-configmap.yml) : configmap used by redis cluster


## Configuration

### Persistent Volume Setup




### Service

- Ports - service name 'redis-cluster-service', port name: 'client', port: '6379' & targetPort: '6379'
- Ports - service name 'redis-cluster-service', port name: 'gossip', port: '16379' & targetPort: '16379'

```sh
master $ kubectl apply -f redis-svc.yml
service/redis-cluster-service created

master $ kubectl get svc
NAME                    TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)              AGE
redis-cluster-service   ClusterIP   10.99.234.78   <none>        6379/TCP,16379/TCP   55s
```

### Redis ConfigMap

- ConfigMap: redis-cluster-configmap is already created. Inspect it...

```sh
master $ kubectl get cm
NAME                      DATA      AGE
redis-cluster-configmap   2         45m

master $ kubectl describe cm redis-cluster-configmap
Name:         redis-cluster-configmap
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
redis.conf:
----
cluster-enabled yes
cluster-require-full-coverage no
cluster-node-timeout 15000
cluster-config-file /data/nodes.conf
cluster-migration-barrier 1
appendonly yes
protected-mode no

update-node.sh:
----
#!/bin/sh
REDIS_NODES="/data/nodes.conf"
sed -i -e "/myself/ s/[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}/${POD_IP}/" ${REDIS_NODES}
exec "$@"

Events:  <none>
```


### Redis Cluster Setup

#### Deploy Redis Cluster

```sh
master $ kubectl exec -it redis-cluster-0 -- redis-cli --cluster create --cluster-replicas 1 $(kubectl get pods -l app=redis-cluster -o jsonpath='{range.items[*]}{.status.podIP}:6379 ')
```

#### Verify Cluster Deployment

```sh
master $ kubectl exec -it redis-cluster-0 -- redis-cli cluster info
```


```sh
master $ for x in $(seq 0 5); do echo "redis-cluster-$x"; kubectl exec redis-cluster-$x -- redis-cli role; echo; done
```