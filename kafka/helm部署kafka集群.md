# 						helm部署Kafka集群

# 1.部署zookeeper

## 1.1.创建命令空间

```shell
kubectl create ns kafka
```

## 1.2.helm包拉取本地

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

## 1.3.修改values

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

## 1.4.创建zookeeper集群

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

### 1.4.1.查看资源状态

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

### 1.4.2.查看集群状态

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

# 2.部署kafka

## 2.1.拉取kafka helm包

```shell
helm pull bitnami/kafka --version 22.1.3
tar zvxf kafka-22.1.3.tgz
# cd kafka && ls
Chart.lock  charts  Chart.yaml  README.md  templates  values.yaml
```

## 2.2.修改values

```shell
vim values.yaml
extraEnvVars: 
  - name: TZ
    value: "Asia/Shanghai"
---
# 副本数
replicaCount: 3                    # 副本数
---
# 持久化存储
persistence:
  enabled: true
  storageClass: "nfs-storage"  # sc 有默认sc可以不写
  accessModes:
    - ReadWriteOnce
  size: 8Gi
---
kraft:
  ## @param kraft.enabled Switch to enable or disable the Kraft mode for Kafka
  ##
  enabled: false   #设置为false
---
# 配置zookeeper外部连接
zookeeper:
  enabled: false                   # 不使用内部zookeeper，默认是false
externalZookeeper:                 # 外部zookeeper
  servers: zookeeper            #Zookeeper svc名称
```

可选配置：

```shell

## 允许删除topic（按需开启）
deleteTopicEnable: true
## 日志保留时间（默认一周）
logRetentionHours: 168
## 自动创建topic时的默认副本数
defaultReplicationFactor: 2
## 用于配置offset记录的topic的partition的副本个数
offsetsTopicReplicationFactor: 2
## 事务主题的复制因子
transactionStateLogReplicationFactor: 2
## min.insync.replicas
transactionStateLogMinIsr: 2
## 新建Topic时默认的分区数
numPartitions: 3
```

## 2.3.创建 Kafka 集群

```shell
helm install kafka -n kafka .
W1216 23:31:29.188611  130502 warnings.go:70] spec.template.spec.containers[0].env[39].name: duplicate name "KAFKA_ENABLE_KRAFT"
NAME: kafka
LAST DEPLOYED: Sat Dec 16 23:31:27 2023
NAMESPACE: kafka
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
CHART NAME: kafka
CHART VERSION: 22.1.3
APP VERSION: 3.4.0

** Please be patient while the chart is being deployed **

Kafka can be accessed by consumers via port 9092 on the following DNS name from within your cluster:

    kafka.kafka.svc.cluster.local

Each Kafka broker can be accessed by producers via port 9092 on the following DNS name(s) from within your cluster:

    kafka-0.kafka-headless.kafka.svc.cluster.local:9092

To create a pod that you can use as a Kafka client run the following commands:

    kubectl run kafka-client --restart='Never' --image docker.io/bitnami/kafka:3.4.0-debian-11-r33 --namespace kafka --command -- sleep infinity
    kubectl exec --tty -i kafka-client --namespace kafka -- bash

    PRODUCER:
        kafka-console-producer.sh \
            --broker-list kafka-0.kafka-headless.kafka.svc.cluster.local:9092 \
            --topic test

    CONSUMER:
        kafka-console-consumer.sh \
            --bootstrap-server kafka.kafka.svc.cluster.local:9092 \
            --topic test \
            --from-beginning
```

查看资源资源状态：

```shell
kubectl get all  -n kafka  -l app.kubernetes.io/component=kafka
NAME          READY   STATUS    RESTARTS      AGE
pod/kafka-0   1/1     Running   2 (16m ago)   17m
pod/kafka-1   1/1     Running   0             17m
pod/kafka-2   1/1     Running   2 (16m ago)   17m

NAME                     TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
service/kafka            ClusterIP   10.96.113.139   <none>        9092/TCP            17m
service/kafka-headless   ClusterIP   None            <none>        9092/TCP,9094/TCP   17m

NAME                     READY   AGE
statefulset.apps/kafka   3/3     17m

#查看pvc
kubectl  get pvc -n kafka -l app.kubernetes.io/component=kafka
NAME           STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
data-kafka-0   Bound    pvc-6161cdd3-1a84-45e2-a1f8-80be74f29111   8Gi        RWO            nfs-storage    11h
data-kafka-1   Bound    pvc-40060655-75e5-4701-92fa-af5a42307a71   8Gi        RWO            nfs-storage    11h
data-kafka-2   Bound    pvc-14469e10-747e-4611-9a9e-651fc954f0bc   8Gi        RWO            nfs-storage    11h
```

## 2.4.创建 topic 查看

```shell
##进入Kafka集群
kubectl exec -it -n kafka kafka-0 -- bash
#创建topic
kafka-topics.sh --create --bootstrap-server kafka:9092  --topic abcdocker
#查看topic列表
kafka-topics.sh --list --bootstrap-server kafka:9092 
#查看topic详细信息
kafka-topics.sh --bootstrap-server kafka:9092  --describe --topic abcdocker
#配置文件配置已经生效，默认分区为3，副本为3，过期时间为168小时
I have no name!@kafka-0:/$ kafka-topics.sh --bootstrap-server kafka:9092  --describe --topic abcdocker
Topic: abcdocker        TopicId: Sc1tx-xiQveowuSqJcS5fw PartitionCount: 3       ReplicationFactor: 2    Configs: flush.ms=1000,segment.bytes=1073741824,flush.messages=10000,max.message.bytes=1000012,retention.bytes=1073741824
        Topic: abcdocker        Partition: 0    Leader: 2       Replicas: 2,0   Isr: 2,0
        Topic: abcdocker        Partition: 1    Leader: 1       Replicas: 1,2   Isr: 1,2
        Topic: abcdocker        Partition: 2    Leader: 0       Replicas: 0,1   Isr: 0,1
```

参考：[文章](https://mp.weixin.qq.com/s/3-mvFNpKOsbhilRh41U5gA)