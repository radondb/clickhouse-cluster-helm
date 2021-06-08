# Deploy Radondb ClickHouse On kubernetes

## Introduction

RadonDB ClickHouse is an open-source, cloud-native, highly availability cluster solutions based on [ClickHouse](https://clickhouse.tech/).

This tutorial demonstrates how to deploy RadonDB ClickHouse on Kubernetes.

## Prerequisites

- You have created a Kubernetes Cluster.

## Procedure

### Step 1 : Add Helm Repository

Add and update helm repositor.

```shell
$ helm repo add radonck https://radondb.github.io/clickhouse-cluster-helm/
$ helm repo update
```

### Step 2 :  Install to Kubernetes

> Zookeeper store ClickHouse's metadata. You can install Zookeeper and ClickHouse Cluster at the same time.

1. In default, the chart will create a ClickHouse Cluster with one shard and two replicas.

   ```shell
   helm install clickhouse radonck/clickhouse
   ```

   **Expected output:**

   ```shell
   $ helm install clickhouse radonck/clickhouse
   $  helm list 
   NAME      	NAMESPACE	REVISION	UPDATED                                	STATUS  	CHART           	APP VERSION
   clickhouse	default  	1       	2021-06-07 07:55:42.860240764 +0000 UTC	deployed	clickhouse-0.1.0	21.1    

   ```

2. For more configurable options and variables, see [values.yaml](../../clickhouse/charts/values.yaml).

### Step 3 :  Verification

#### Check the Pod
```shell
kubectl get all   --selector app.kubernetes.io/instance=clickhouse
```

**Expected output:**

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

#### Check the Status of Pod

You should wait a while，then check the output in `Reason` line. When the output persistently return `Started`, indicate that RadonDB ClickHouse is up and running.

```
kubectl describe pod <pod name>
```
**Expected output:**

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

## Access RadonDB ClickHouse

### Use pod

You can directly connect to ClickHouse Pod with `kubectl`.

```
$ kubectl get pods |grep clickhouse
clickhouse-s0-r0-0   1/1     Running   0          8m50s
clickhouse-s0-r1-0   1/1     Running   0          8m50s

$ kubectl exec -it clickhouse-s0-r0-0 -- clickhouse client -u default --password=C1ickh0use --query='select hostName()'
clickhouse-s0-r0-0

```

### Use Service

The Service `spec.type` is `ClusterIP`, so you need to create a client to connect the service.

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

## Persistence

You can configure a Pod to use a PersistentVolumeClaim(PVC) for storage. 
In default, PVC mount on the `/var/lib/clickhouse` directory.

1. You should create a Pod that uses the above PVC for storage.

2. You should create a PVC that is automatically bound to a suitable PersistentVolume(PV). 

> **Note** 
> PVC can use different PV, so using the different PV show the different performance.
