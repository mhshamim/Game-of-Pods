# Redis Islands

Figure below explains the Redis setup:

![Figure-Bravo](https://github.com/mhshamim/Game-of-Pods/blob/master/scenarios/Game-of-Pods-Redis-Deploy.JPG)

* [redis-pv-hostpath](redis-pv-hostpath.yml) : persistent volumes setup
* [redis-cluster-service](redis-svc.yml) : service expose redis 'client' and 'gossip' ports
* [redis-cluster-configmap](redis-cluster-configmap.yml) : configmap used by redis cluster
* [redis-cluster](redis-statefulset.yml) : redis statefulset


## Configuration

### Persistent Volume Setup

- PersistentVolume - Name: redis01
- Access modes: ReadWriteOnce
- Size: 1Gi
- hostPath: /redis01, directory should be created on worker node

- PersistentVolume - Name: redis02
- Access modes: ReadWriteOnce
- Size: 1Gi
- hostPath: /redis02, directory should be created on worker node

- PersistentVolume - Name: redis03
- Access modes: ReadWriteOnce
- Size: 1Gi
- hostPath: /redis03, directory should be created on worker node

- PersistentVolume - Name: redis04
- Access modes: ReadWriteOnce
- Size: 1Gi
- hostPath: /redis04, directory should be created on worker node

- PersistentVolume - Name: redis05
- Access modes: ReadWriteOnce
- Size: 1Gi
- hostPath: /redis05, directory should be created on worker node

- PersistentVolume - Name: redis06
- Access modes: ReadWriteOnce
- Size: 1Gi
- hostPath: /redis06, directory should be created on worker node

```sh
node01 $ for x in $(seq 1 6); do mkdir "/redis0$x"; done

master $ kubectl apply -f redis-pv-hostpath.yml
persistentvolume/redis01 created
persistentvolume/redis02 created
persistentvolume/redis03 created
persistentvolume/redis04 created
persistentvolume/redis05 created
persistentvolume/redis06 created
```


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

### Redis Statefulset Config

- StatefulSet - Name: redis-cluster
- Replicas: 6
- Pods status: Running (All 6 replicas)
- Image: redis:5.0.1-alpine
- container name: redis, command: ["/conf/update-node.sh", "redis-server", "/conf/redis.conf"]
- Env: name: 'POD_IP', valueFrom: 'fieldRef', fieldPath: 'status.podIP' (apiVersion: v1)
- Ports - name: 'client', containerPort: '6379'
- Ports - name: 'gossip', containerPort: '16379'
- Volume Mount - name: 'conf', mountPath: '/conf', readOnly:'false' (ConfigMap Mount)
- Volume Mount - name: 'conf', mountPath: '/conf', defaultMode = '0755' (ConfigMap Mount)
- Volume Mount - name: 'data', mountPath: '/data', readOnly:'false' (volumeClaim)
- volumes - name: 'conf', Type: 'ConfigMap', ConfigMap Name: 'redis-cluster-configmap',
- volumeClaimTemplates - name: 'data'
- volumeClaimTemplates - accessModes: 'ReadWriteOnce'
- volumeClaimTemplates - Storage Request: '1Gi'

```sh
master $ kubectl apply -f redis-statefulset.yml
statefulset.apps/redis-cluster created

master $ kubectl get statefulsets.apps,pod
NAME                             DESIRED   CURRENT   AGE
statefulset.apps/redis-cluster   6         6         32s

NAME                  READY     STATUS    RESTARTS   AGE
pod/redis-cluster-0   1/1       Running   0          32s
pod/redis-cluster-1   1/1       Running   0          25s
pod/redis-cluster-2   1/1       Running   0          23s
pod/redis-cluster-3   1/1       Running   0          20s
pod/redis-cluster-4   1/1       Running   0          17s
pod/redis-cluster-5   1/1       Running   0          14s
```

#### Deploy Redis Cluster

- Configure the Cluster. Once the StatefulSet has been deployed with 6 'Running' pods, run the below commands and type 'yes' when prompted.

```sh
master $ kubectl exec -it redis-cluster-0 -- redis-cli --cluster create --cluster-replicas 1 $(kubectl get pods -l app=redis-cluster -o jsonpath='{range.items[*]}{.status.podIP}:6379 ')

>>> Performing hash slots allocation on 6 nodes...
Master[0] -> Slots 0 - 5460
Master[1] -> Slots 5461 - 10922
Master[2] -> Slots 10923 - 16383
Adding replica 10.32.0.5:6379 to 10.32.0.2:6379
Adding replica 10.32.0.6:6379 to 10.32.0.3:6379
Adding replica 10.32.0.7:6379 to 10.32.0.4:6379
M: c1c4d020b4140a62aa2966a3c09675078ba109be 10.32.0.2:6379
   slots:[0-5460] (5461 slots) master
M: fe9b1083ab0a37309399ac2b03e91189e8f2daaf 10.32.0.3:6379
   slots:[5461-10922] (5462 slots) master
M: 38fe8430bf3c6919559d9d50369cf0af0a622a29 10.32.0.4:6379
   slots:[10923-16383] (5461 slots) master
S: dad51055e6df60512f7e4382b1209c081950f684 10.32.0.5:6379
   replicates c1c4d020b4140a62aa2966a3c09675078ba109be
S: 60e97e4f9f25a3fc3f25077db571abd54d9de178 10.32.0.6:6379
   replicates fe9b1083ab0a37309399ac2b03e91189e8f2daaf
S: 6fc60ad328471eb09930763c2448910e37f053f5 10.32.0.7:6379
   replicates 38fe8430bf3c6919559d9d50369cf0af0a622a29
Can I set the above configuration? (type 'yes' to accept): yes
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join
...
>>> Performing Cluster Check (using node 10.32.0.2:6379)
M: c1c4d020b4140a62aa2966a3c09675078ba109be 10.32.0.2:6379
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
S: 60e97e4f9f25a3fc3f25077db571abd54d9de178 10.32.0.6:6379
   slots: (0 slots) slave
   replicates fe9b1083ab0a37309399ac2b03e91189e8f2daaf
M: fe9b1083ab0a37309399ac2b03e91189e8f2daaf 10.32.0.3:6379
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
S: 6fc60ad328471eb09930763c2448910e37f053f5 10.32.0.7:6379
   slots: (0 slots) slave
   replicates 38fe8430bf3c6919559d9d50369cf0af0a622a29
M: 38fe8430bf3c6919559d9d50369cf0af0a622a29 10.32.0.4:6379
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
S: dad51055e6df60512f7e4382b1209c081950f684 10.32.0.5:6379
   slots: (0 slots) slave
   replicates c1c4d020b4140a62aa2966a3c09675078ba109be
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```

#### Verify Cluster Deployment

```sh
master $ kubectl exec -it redis-cluster-0 -- redis-cli cluster info

cluster_state:ok
cluster_slots_assigned:16384
cluster_slots_ok:16384
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:6
cluster_size:3
cluster_current_epoch:6
cluster_my_epoch:1
cluster_stats_messages_ping_sent:45
cluster_stats_messages_pong_sent:45
cluster_stats_messages_sent:90
cluster_stats_messages_ping_received:40
cluster_stats_messages_pong_received:45
cluster_stats_messages_meet_received:5
cluster_stats_messages_received:90
```


```sh
master $ for x in $(seq 0 5); do echo "redis-cluster-$x"; kubectl exec redis-cluster-$x -- redis-cli role; echo; done

redis-cluster-0
master
84
10.32.0.5
6379
84

redis-cluster-1
master
84
10.32.0.6
6379
84

redis-cluster-2
master
84
10.32.0.7
6379
84

redis-cluster-3
slave
10.32.0.2
6379
connected
98

redis-cluster-4
slave
10.32.0.3
6379
connected
84

redis-cluster-5
slave
10.32.0.4
6379
connected
84
```