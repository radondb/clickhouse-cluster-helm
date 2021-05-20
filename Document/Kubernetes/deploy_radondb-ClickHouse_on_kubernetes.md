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

```shell
$ helm repo add radonck https://radondb.github.io/clickhouse-cluster-helm/
$ helm repo update
```

### 1.2. Install to Kubernetes

* Zookeeper: Store ClickHouse's metadata.

The ClickHouse Cluster chart has already contains Zookeeper. So we just need install this chart.

#### 1.2.2. Install ClickHouse

```shell
$ helm install clickhouse radonck/clickhouse

$  helm list 
NAME      	NAMESPACE	REVISION	UPDATED                                	STATUS  	CHART           	APP VERSION
clickhouse	default  	1       	2021-06-07 07:55:42.860240764 +0000 UTC	deployed	clickhouse-0.1.0	21.1    

```

In default, the chart will create a ClickHouse Cluster with one shard and two replicas.

```shell
$ kubectl get all   --selector app.kubernetes.io/instance=clickhouse
NAME                     READY   STATUS    RESTARTS   AGE
pod/clickhouse-s0-r0-0   1/1     Running   0          72s
pod/clickhouse-s0-r1-0   1/1     Running   0          72s

NAME                       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
service/clickhouse         ClusterIP   10.96.230.92    <none>        9000/TCP,8123/TCP   72s
service/clickhouse-s0-r0   ClusterIP   10.96.83.41     <none>        9000/TCP,8123/TCP   72s
service/clickhouse-s0-r1   ClusterIP   10.96.240.111   <none>        9000/TCP,8123/TCP   72s

NAME                                READY   AGE
statefulset.apps/clickhouse-s0-r0   1/1     72s
statefulset.apps/clickhouse-s0-r1   1/1     72s

```

For a list of all configurable options and variables please see [values.yaml](../../clickhouse/charts/values.yaml).

## 2. Connect to ClickHouse

Because the chart starts Zookeeper and ClickHouse, the ClickHouse maybe started before Zookeeper.

We should wait `90+`s until see this `event` then we can enjoy ClickHouse Cluster.

```shell
$ kubectl describe pod  clickhouse-s0-r0-0
...
Events:
  Type     Reason                  Age                    From                     Message
  ----     ------                  ----                   ----                     -------
  Warning  FailedScheduling        7m30s (x3 over 7m42s)  default-scheduler        error while running "VolumeBinding" filter plugin for pod "clickhouse-s0-r0-0": pod has unbound immediate PersistentVolumeClaims
  Normal   Scheduled               7m28s                  default-scheduler        Successfully assigned default/clickhouse-s0-r0-0 to worker-p004
  Normal   SuccessfulAttachVolume  7m6s                   attachdetach-controller  AttachVolume.Attach succeeded for volume "pvc-21c5de1f-c396-4743-a31b-2b094ecaf79b"
  Warning  Unhealthy               5m4s (x3 over 6m4s)    kubelet, worker-p004     Liveness probe failed: Code: 210. DB::NetException: Connection refused (localhost:9000)
  Normal   Killing                 5m4s                   kubelet, worker-p004     Container clickhouse failed liveness probe, will be restarted
  Normal   Pulled                  4m34s (x2 over 6m50s)  kubelet, worker-p004     Container image "tceason/clickhouse-server:v21.1.3.32-stable" already present on machine
  Normal   Created                 4m34s (x2 over 6m50s)  kubelet, worker-p004     Created container clickhouse
  Normal   Started                 4m33s (x2 over 6m48s)  kubelet, worker-p004     Started container clickhouse

```

### 2.1. Use pod

We can directly connect to ClickHouse Pod with `kubectl`.

```shell
$ kubectl get pods |grep clickhouse
clickhouse-s0-r0-0   1/1     Running   0          8m50s
clickhouse-s0-r1-0   1/1     Running   0          8m50s

$ kubectl exec -it clickhouse-s0-r0-0 -- clickhouse client -u default --password=C1ickh0use --query='select hostName()'
clickhouse-s0-r0-0

```

### 2.2. Use Service

Because the Service `spec.type` is `ClusterIP`, we need a client to connect to the service.

```shell
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
