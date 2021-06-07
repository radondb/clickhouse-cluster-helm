Contents
=================

- [在 Kubernetes 上部署 RadonDB ClickHouse](#在-kubernetes-上部署-radondb-clickhouse)
  - [简介](#简介)
  - [部署准备](#部署准备)
  - [部署步骤](#部署步骤)
    - [步骤 1 : 添加仓库](#步骤-1--添加仓库)
    - [步骤 2 : 部署](#步骤-2--部署)
      - [安装 ZooKeeper](#安装-zookeeper)
      - [安装 ClickHouse](#安装-clickhouse)
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

```
helm repo add radonck https://radondb.github.io/clickhouse-cluster-helm/
helm repo update
```

### 步骤 2 : 部署

#### 安装 ZooKeeper

> 因 ZooKeeper 存储了 ClickHouse 元数据，故在部署 RadonDB ClickHouse 前，需安装 ZooKeeper 组件。

1. 执行如下命令，安装 ZooKeeper。
   当回显 `deployed` 状态时，即安装成功。

   ```
   $ helm install zookeeper radonck/zookeeper
   $ helm list
   NAME     	NAMESPACE	REVISION	UPDATED                                	STATUS  	CHART          	APP VERSION
   zookeeper	default  	1       	2021-06-02 06:01:32.990928355 +0000 UTC	deployed	zookeeper-0.1.0	3.6    
   ```

2. 执行如下命令，获取服务名称。本示例服务名称为 `zk-server-zookeeper`。

   ```
   $ kubectl get service |grep zk-server-zookeeper
   zk-server-zookeeper   ClusterIP   None           <none>        2888/TCP,3888/TCP   9m36s
   ```

#### 安装 ClickHouse

1. 修改 `zookeeper.external.instances[0].host` 参数值为 `zk-server-zookeeper` 。

   ```
   $ helm install clickhouse --set zookeeper.external.instances[0].host=zk-server-zookeeper radonck/clickhouse
 
   $ helm list
   NAME      	NAMESPACE	REVISION	UPDATED                                	STATUS  	CHART           	APP VERSION
   clickhouse	default  	1       	2021-06-02 06:13:10.623409272 +0000 UTC	deployed	clickhouse-0.1.0	21.1       
   zookeeper 	default  	1       	2021-06-02 06:01:32.990928355 +0000 UTC	deployed	zookeeper-0.1.0 	3.6        

   ```

2. 执行如下命令，创建 ClickHouse 集群。默认生成一个主节点，两个从节点。

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

3. 更多配置项和变量配置，请查看 [values.yaml](../../clickhouse/charts/values.yaml)。

## 访问 RadonDB ClickHouse

### 通过 Pod

通过 `kubectl` 工具直接访问 ClickHouse Pod ，连接示例如下。

```
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
