# 						helm部署zookeeper集群

# 1.准备文件

## 1.1.创建命令空间

```shell
kubectl create ns kafka
```

## 1.1.helm包拉取本地

```shell
# 添加bitnami仓库
helm repo add bitnami https://charts.bitnami.com/bitnami
# 查询chart
helm search repo bitnami
# 拉取zookeeper
helm pull bitnami/zookeeper
# 解压
tar zxvf  zookeeper-12.0.0.tgz
#进入Zookeeper
cd zookeeper
# 查看包内容
ls
Chart.lock  charts  Chart.yaml  README.md  templates  values.yaml
```

## 1.1.修改values

```shell
vim values.yaml
extraEnvVars: 
  - name: TZ
    value: "Asia/Shanghai"
# 允许任意用户连接（默认开启）,本地文件没有需要自己添加
allowAnonymousLogin: true
---
# 关闭认证（默认关闭）
auth:
  enable: false
---
# 修改副本数
replicaCount: 3
---
# 4. 配置持久化，按需使用
global:
  imageRegistry: ""
  imagePullSecrets: []
  storageClass: "nfs-storage"
```

## 1.3.创建zookeeper集群

```shell
#部署
helm install zookeeper -n kafka .
NAME: zookeeper
LAST DEPLOYED: Sat Dec 16 23:03:46 2023
NAMESPACE: zookeeper
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
CHART NAME: zookeeper
CHART VERSION: 12.0.0
APP VERSION: 3.9.0

** Please be patient while the chart is being deployed **

ZooKeeper can be accessed via port 2181 on the following DNS name from within your cluster:

    zookeeper.zookeeper.svc.cluster.local

To connect to your ZooKeeper server run the following commands:

    export POD_NAME=$(kubectl get pods --namespace zookeeper -l "app.kubernetes.io/name=zookeeper,app.kubernetes.io/instance=zookeeper,app.kubernetes.io/component=zookeeper" -o jsonpath="{.items[0].metadata.name}")
    kubectl exec -it $POD_NAME -- zkCli.sh

To connect to your ZooKeeper server from outside the cluster execute the following commands:

    kubectl port-forward --namespace zookeeper svc/zookeeper 2181:2181 &
    zkCli.sh 127.0.0.1:2181
```

### 1.3.1.查看资源状态

```shell
kubectl get  all -n  kafka
NAME              READY   STATUS    RESTARTS   AGE
pod/zookeeper-0   1/1     Running   0          64s
pod/zookeeper-1   1/1     Running   0          64s
pod/zookeeper-2   1/1     Running   0          64s

NAME                         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
service/zookeeper            ClusterIP   10.105.92.34   <none>        2181/TCP,2888/TCP,3888/TCP   65s
service/zookeeper-headless   ClusterIP   None           <none>        2181/TCP,2888/TCP,3888/TCP   65s

NAME                         READY   AGE
statefulset.apps/zookeeper   3/3     65s

NAME                                                                            AGE
componentresourceconstraint.apps.kubeblocks.io/kb-resource-constraint-general   13d

#查看挂载状态，本地使用nfs作为持久化
 kubectl get  pvc -n  kafka
NAME               STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
data-zookeeper-0   Bound    pvc-8f76686a-5312-49bf-97e7-5fdf45dc99ce   8Gi        RWO            nfs-storage    17m
data-zookeeper-1   Bound    pvc-d72b2b5b-73dd-409a-8bea-487f1b72ec24   8Gi        RWO            nfs-storage    17m
data-zookeeper-2   Bound    pvc-e6ed9a7f-50b2-490d-b3b0-6b6476e60e53   8Gi        RWO            nfs-storage    17m
```

### 1.3.2.查看集群状态

```shell
#slave节点
root@k8s-master01:~/zookeeper# kubectl exec -it -n  kafka zookeeper-0 -- bash
I have no name!@zookeeper-0:/$ zkServer.sh status
/opt/bitnami/java/bin/java
ZooKeeper JMX enabled by default
Using config: /opt/bitnami/zookeeper/bin/../conf/zoo.cfg
Client port found: 2181. Client address: localhost. Client SSL: false.
Mode: follower
I have no name!@zookeeper-0:/$
#master节点
root@k8s-master01:~/zookeeper# kubectl exec -it -n  kafka zookeeper-1 -- bash
I have no name!@zookeeper-1:/$ zkServer.sh status
/opt/bitnami/java/bin/java
ZooKeeper JMX enabled by default
Using config: /opt/bitnami/zookeeper/bin/../conf/zoo.cfg
Client port found: 2181. Cli
ent address: localhost. Client SSL: false.
Mode: leader
I have no name!@zookeeper-1:/$
#slave节点
root@k8s-master01:~/zookeeper# kubectl exec -it -n  kafka zookeeper-2 -- bash
I have no name!@zookeeper-2:/$ zkServer.sh status
/opt/bitnami/java/bin/java
ZooKeeper JMX enabled by default
Using config: /opt/bitnami/zookeeper/bin/../conf/zoo.cfg
Client port found: 2181. Client address: localhost. Client SSL: false.
Mode: follower
I have no name!@zookeeper-2:/$
```

