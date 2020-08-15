# Voting App

Figure below explains the Voting App architecture:

![Figure-Bravo](https://github.com/mhshamim/Game-of-Pods/blob/master/scenarios/Game-of-Pods-VotingApp-Deploy.JPG)


* [vote-app](vote-app.yml) : Vote App with Service and Deployment manifest
* [redis-app](redis-app.yml) : Redis App with Service and Deployment manifest
* [worker-deployment](worker-deployment.yml) : Worker Deployment manifest
* [Postgres-db](db-app.yml) : Postgres DB with Service, ConfigMap and Deployment manifest
* [result-app](result-app.yml) : Result App with Service and Deployment manifest
\
\

Create a new namespace: name = 'vote'

```sh
master $ kubectl create ns vote
namespace/vote created
```

## Vote App

### Vote Service

Create a new service: name = vote-service \
port = '5000' \
targetPort = '80' \
nodePort= '31000' \
service endpoint exposes deployment 'vote-deployment' \

### Vote Deployment

Create a deployment: name = 'vote-deployment' \
image = 'kodekloud/examplevotingapp_vote:before' \
status: 'Running' \

```sh
master $ kubectl apply -f vote-app.yml
service/vote-service created
deployment.apps/vote-deployment created
```

## Redis App

### Redis Service

New Service, name = 'redis' \
port: '6379' \
targetPort: '6379' \
type: 'ClusterIP' \
service endpoint exposes deployment 'redis-deployment' \

### Redis Deployment

Create new deployment, name: 'redis-deployment' \
image: 'redis:alpine' \
Volume Type: 'EmptyDir' \
Volume Name: 'redis-data' \
mountPath: '/data' \
status: 'Running' \

```sh
master $ kubectl apply -f redis-app.yml
service/redis created
deployment.apps/redis-deployment created
```

## Worker App

### Worker Deployment

Create new deployment. name: 'worker' \
image: 'kodekloud/examplevotingapp_worker' \
status: 'Running' \

```sh
master $ kubectl apply -f worker-deployment.yml
deployment.apps/worker created
```


## Postgres DB

### DB Service

Create new service: 'db' \
port: '5432' \
targetPort: '5432' \
type: 'ClusterIP' \

### DB Deployment

Create new deployment. name: 'db-deployment' \
image: 'postgres:9.4' \
Volume Type: 'EmptyDir' \
Volume Name: 'db-data' \
mountPath: '/var/lib/postgresql/data' \
status: 'Running' \

```sh
master $ kubectl apply -f db-app.yml
service/db created
configmap/db-config created
deployment.apps/db-deployment created
```



## Result App

### Result Deployment

Create new deployment, name: 'result-deployment' \
image: 'kodekloud/examplevotingapp_result:before' \
status: 'Running' \

### Result Service

service 'result-service' endpoint exposes deployment 'result-deployment' \
port: '5001' \
targetPort: '80' \
NodePort: '31001' \

```sh
master $ kubectl apply -f result-app.yml
service/result created
deployment.apps/result-deployment created
```
\
\

## Running Deployments
\
```sh
master $ kubectl get deploy,pod,svc -n vote
NAME                                      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.extensions/db-deployment       1         1         1            1           6m
deployment.extensions/redis-deployment    1         1         1            1           4m
deployment.extensions/result-deployment   1         1         1            1           17s
deployment.extensions/vote-deployment     1         1         1            1           4m
deployment.extensions/worker              1         1         1            1           5m

NAME                                     READY     STATUS    RESTARTS   AGE
pod/db-deployment-5869f8d889-4n2sz       1/1       Running   0          6m
pod/redis-deployment-58b47874dc-p4fdd    1/1       Running   0          4m
pod/result-deployment-7dbf8c5f9b-6ks8l   1/1       Running   0          16s
pod/vote-deployment-6b796c8f5c-9nc4w     1/1       Running   0          4m
pod/worker-78bd46fb9b-7hlhn              1/1       Running   0          5m

NAME                     TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
service/db               ClusterIP   10.102.86.131   <none>        5432/TCP         6m
service/redis            ClusterIP   10.105.152.28   <none>        6379/TCP         4m
service/result-service   NodePort    10.109.43.81    <none>        5001:31001/TCP   17s
service/vote-service     NodePort    10.108.72.24    <none>        5000:31000/TCP   4m
```

