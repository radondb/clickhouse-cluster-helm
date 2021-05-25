Contents
=================

* [Deploy Radondb ClickHouse On kubernetes](#deploy-radondb-clickhouse-on-kubernetes)
    * [1. Installation](#1-installation)
        * [1.1. Add Helm Repository](#11-add-helm-repository)
        * [1.2. Install to Kubernetes](#12-install-to-kubernetes)
            * [1.2.1. Install Zookeeper](#121-install-zookeeper)
            * [1.2.2. Install ClickHouse](#122-install-clickhouse)
    * [2. Connect to ClickHouse](#2-connect-to-clickhouse)
        * [2.1. Use pod](#21-use-pod)
        * [2.2. Use Service](#22-use-service)
    * [3. Persistence](#3-persistence)

# Deploy Radondb ClickHouse On kubernetes

## 1. Installation

### 1.1. Add Helm Repository

```
helm repo add radonck https://radondb.github.io/clickhouse-cluster-helm/
helm repo update
```

### 1.2. Install to Kubernetes

* Zookeeper: Store ClickHouse's metadata.

So we should install Zookeeper at first.

#### 1.2.1. Install Zookeeper

If you see the STATUS is `deployed` that means the Zookeeper has already been installed.

```
$ helm install zookeeper radonck/zookeeper
$ helm list
NAME     	NAMESPACE	REVISION	UPDATED                                	STATUS  	CHART          	APP VERSION
zookeeper	default  	1       	2021-06-02 06:01:32.990928355 +0000 UTC	deployed	zookeeper-0.1.0	3.6    

```

And we can get the name of the service: `zk-server-zookeeper`. It will be used on next section.

```
$ kubectl get service |grep zk-server-zookeeper
zk-server-zookeeper   ClusterIP   None           <none>        2888/TCP,3888/TCP   9m36s

```

#### 1.2.2. Install ClickHouse

Modify the default value of `zookeeper.external.instances[0].host` to the Zookeeper service name `zk-server-zookeeper` we got on previous section.

```
$ helm install clickhouse --set zookeeper.external.instances[0].host=zk-server-zookeeper radonck/clickhouse

$ helm list
NAME      	NAMESPACE	REVISION	UPDATED                                	STATUS  	CHART           	APP VERSION
clickhouse	default  	1       	2021-06-02 06:13:10.623409272 +0000 UTC	deployed	clickhouse-0.1.0	21.1       
zookeeper 	default  	1       	2021-06-02 06:01:32.990928355 +0000 UTC	deployed	zookeeper-0.1.0 	3.6        

```

In default, the chart will create a ClickHouse Cluster with one shard and two replicas.

```
$ kubectl get all   --selector app.kubernetes.io/instance=clickhouse
NAME                     READY   STATUS    RESTARTS   AGE
pod/clickhouse-s0-r0-0   1/1     Running   0          2m41s
pod/clickhouse-s0-r1-0   1/1     Running   0          2m41s

NAME                       TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)             AGE
service/clickhouse         ClusterIP   10.96.71.193   <none>        9000/TCP,8123/TCP   2m41s
service/clickhouse-s0-r0   ClusterIP   10.96.40.207   <none>        9000/TCP,8123/TCP   2m41s
service/clickhouse-s0-r1   ClusterIP   10.96.63.179   <none>        9000/TCP,8123/TCP   2m41s

NAME                                READY   AGE
statefulset.apps/clickhouse-s0-r0   1/1     2m41s
statefulset.apps/clickhouse-s0-r1   1/1     2m41s

```

For a list of all configurable options and variables please see [values.yaml](../../clickhouse/charts/values.yaml).

## 2. Connect to ClickHouse

### 2.1. Use pod

We can directly connect to ClickHouse Pod with `kubectl`.

```
$ kubectl get pods |grep clickhouse
clickhouse-s0-r0-0   1/1     Running   0          8m50s
clickhouse-s0-r1-0   1/1     Running   0          8m50s

$ kubectl exec -it clickhouse-s0-r0-0 -- clickhouse client -u default --password=C1ickh0use --query='select hostName()'
clickhouse-s0-r0-0

```

### 2.2. Use Service

Because the Service `spec.type` is `ClusterIP`, we need a client to connect to the service.

```
$ kubectl get service |grep clickhouse
clickhouse            ClusterIP   10.96.71.193   <none>        9000/TCP,8123/TCP   12m
clickhouse-s0-r0      ClusterIP   10.96.40.207   <none>        9000/TCP,8123/TCP   12m
clickhouse-s0-r1      ClusterIP   10.96.63.179   <none>        9000/TCP,8123/TCP   12m

$ cat client.yaml
apiVersion: v1
kind: Pod
metadata:
name: clickhouse-client
labels:
app: clickhouse-client
spec:
containers:
- name: clickhouse-client
image: tceason/clickhouse-server:v21.1.3.32-stable
imagePullPolicy: Always

$ kubectl apply -f client.yaml
pod/clickhouse-client unchanged

$ kubectl exec -it clickhouse-client -- clickhouse client -u default --password=C1ickh0use -h 10.96.71.193 --query='select hostName()'
clickhouse-s0-r1-0
$ kubectl exec -it clickhouse-client -- clickhouse client -u default --password=C1ickh0use -h 10.96.71.193 --query='select hostName()'
clickhouse-s0-r0-0

```

## 3. Persistence

It will create a PersistentVolumeClaim for each pod and mounted to `/var/lib/clickhouse`.

**Note:** 
> PersistentVolumeClaim can use different PersistentVolume, so using different PV will produce a different performance.
