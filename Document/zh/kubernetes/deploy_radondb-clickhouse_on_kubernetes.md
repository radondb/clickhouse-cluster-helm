Contents
=================

- [在 Kubernetes 上部署 RadonDB ClickHouse](#在-kubernetes-上部署-radondb-clickhouse)
  - [简介](#简介)
  - [部署准备](#部署准备)
  - [部署步骤](#部署步骤)
    - [步骤 1 : 添加仓库](#步骤-1--添加仓库)
    - [步骤 2 : 部署](#步骤-2--部署)
    - [步骤 3 : 部署校验](#步骤-3--部署校验)
      - [查看集群 Pod](#查看集群-pod)
      - [查询 Pod 状态](#查询-pod-状态)
  - [访问 RadonDB ClickHouse](#访问-radondb-clickhouse)
    - [通过 Pod](#通过-pod)
    - [通过 Service](#通过-service)
  - [持久化](#持久化)

# 在 Kubernetes 上部署 RadonDB ClickHouse

## 简介

RadonDB ClickHouse 是基于 [ClickHouse](https://clickhouse.tech/) 的开源、高可用、云原生集群解决方案。

本教程演示如何使用命令行在 Kubernetes 上部署 RadonDB ClickHouse。

## 部署准备

- 已成功部署 Kubernetes 集群。

## 部署步骤

### 步骤 1 : 添加仓库

添加并更新 helm 仓库。

```shell
$ helm repo add radonck https://radondb.github.io/clickhouse-cluster-helm/
$ helm repo update
```

### 步骤 2 : 部署

> 因 ZooKeeper 存储了 ClickHouse 元数据。您可以一次性安装  ZooKeeper 组件和 ClickHouse 集群。

1. 执行如下命令，安装 ClickHouse 集群。默认生成一个主节点，两个从节点。
   ```
   helm install clickhouse radonck/clickhouse
   ```
   **预期结果**

   ```shell
   $ helm install clickhouse radonck/clickhouse
   $  helm list 
   NAME      	NAMESPACE	REVISION	UPDATED                                	STATUS  	CHART           	APP VERSION
   clickhouse	default  	1       	2021-06-07 07:55:42.860240764 +0000 UTC	deployed	clickhouse-0.1.0	21.1    
   
   ```

2. 更多配置项和变量配置，请查看 [values.yaml](../../clickhouse/charts/values.yaml)。

### 步骤 3 : 部署校验

#### 查看集群 Pod

执行如下命令，查看创建的集群。

``
kubectl get all   --selector app.kubernetes.io/instance=clickhouse
```

**预期结果**

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

#### 查询 Pod 状态

执行如行命令，查看 `Events` 返回结果。当返回结果为 `Started` 且稳定后，即可正常访问 ClickHouse 集群。

```
kubectl describe pod <pod name>
```
**预期结果**

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

## 访问 RadonDB ClickHouse

### 通过 Pod

通过 `kubectl` 工具直接访问 ClickHouse Pod ，连接示例如下。

```shell
$ kubectl get pods |grep clickhouse
clickhouse-s0-r0-0   1/1     Running   0          8m50s
clickhouse-s0-r1-0   1/1     Running   0          8m50s

$ kubectl exec -it clickhouse-s0-r0-0 -- clickhouse client -u default --password=C1ickh0use --query='select hostName()'
clickhouse-s0-r0-0

```

### 通过 Service

由于 Service 的`spec.type` 类型为 `ClusterIP`，故需通过客户端连接服务。

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

## 持久化

配置 Pod 使用 PersistentVolumeClaim 作为存储，实现 ClickHouse 持久化。

默认情况下，每个 Pod 将创建一个 PVC ，并将其挂载到 `/var/lib/clickhouse` 目录。

1. 创建一个使用 PVC 作为存储的 Pod。
2. 创建一个 PVC 自动绑定到合适的 PersistentVolume。

> **注意** 
> 在 PersistentVolumeClaim 中，可以配置不同特性的 PersistentVolume。
