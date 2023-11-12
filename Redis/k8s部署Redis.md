# 						kubernetes部署Redis

# 1.使用 Helm 图表部署 Redis

为了简化 Redis 部署过程，我们将使用 [Helm](https://helm.sh/)，一个用于 Kubernetes 应用程序的包管理器。按照[官方文档](https://helm.sh/docs/intro/install/)安装 Helm。

## 1.1.部署

1. **将 Bitnami 存储库添加到 Helm：**

```shell
helm repo add bitnami https://charts.bitnami.com/bitnami
```

2. **更新 Helm 存储库：**

```shell
helm repo update
```

3. **下载chart：**

```shell
helm search repo redis
helm fetch  bitnami/redis

tar zxvf redis-17.15.5.tgz

```

4. **自定义`values.yaml`：**

```shell
vim values.yaml
master:
  password: "your-password-here"
```

5. **使用自定义值部署 Redis ：**

```shell
helm install my-redis bitnami/redis -f redis/values.yaml
```

## 1.2.验证

1. 查看pod状态：

```shell
kubectl get pod
```

2. 下载redis客户端：

```shell
apt install redis-tools -y
```

3. 连接redis容器：

```shell
kubectl get svc | grep redis
my-redis-headless   ClusterIP   None            <none>        6379/TCP   8h
my-redis-master     ClusterIP   10.108.59.255   <none>        6379/TCP   8h
my-redis-replicas   ClusterIP   10.108.27.58    <none>        6379/TCP   8h

redis-cli -h 10.108.59.255 -p 6379 -a "your password"
10.108.59.255:6379>
10.108.59.255:6379>
```

# 2.在 Kubernetes 集群中扩展 Redis

## 2.1.StatefulSet配置

```shell
cat > redis-cluster.yaml <<EOF
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis-cluster
spec:
  serviceName: redis-cluster
  replicas: 6
  selector:
    matchLabels:
      app: redis-cluster
  template:
    metadata:
      labels:
        app: redis-cluster
    spec:
      containers:
      - name: redis
        image: redis:6.2.5
        command: ["redis-server"]
        args: ["/conf/redis.conf"]
        env:
        - name: REDIS_CLUSTER_ANNOUNCE_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        ports:
        - containerPort: 6379
          name: client
        - containerPort: 16379
          name: gossip
        volumeMounts:
        - name: conf
          mountPath: /conf
        - name: data
          mountPath: /data
      volumes:
        - name: conf
          configMap:
            name: redis-cluster-configmap
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
EOF
```

## 2.2.configmap配置

```shell
cat > redis-config.yaml <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-cluster-configmap
data:
  redis.conf: |-
    cluster-enabled yes
    cluster-require-full-coverage no
    cluster-node-timeout 15000
    cluster-config-file /data/nodes.conf
    cluster-migration-barrier 1
    appendonly yes
    protected-mode no
    bind 0.0.0.0
    port 6379
---
EOF
```

## 2.3.创建资源

```shell
kubectl apply  -f  redis-config.yaml
kubectl apply -f  redis-cluster.yaml
```

https://www.dragonflydb.io/guides/redis-kubernetes

https://www.codeleading.com/article/34566409164
