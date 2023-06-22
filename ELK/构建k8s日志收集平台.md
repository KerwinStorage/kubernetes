# 前言

架构：fluentd --> elasticsearch --> kibana

相关链接：

- [采集k8s容器和物理节点日志.pdf](https://www.yuque.com/office/yuque/0/2023/pdf/1255012/1678267662101-a5654658-79b4-411b-a8b7-29f27cdecb13.pdf?from=https%3A%2F%2Fwww.yuque.com%2Fjixiashan%2Ffize6l%2Fdo74buqo389kzr7l)
- [Elasticsearch 的数据采集器 | Elastic](https://www.elastic.co/cn/beats)
- [Elasticsearch：官方分布式搜索和分析引擎 | Elastic](https://www.elastic.co/cn/elasticsearch/)
- [Kibana：数据的探索、可视化和分析 | Elastic](https://www.elastic.co/cn/kibana/)
- [流利 |开源数据收集器 |统一日志记录层 (fluentd.org)](https://www.fluentd.org/)

# 1. 安装 elasticsearch 组件

## 1.1 创建命名空间

先创建一个命名空间，在这个名称空间下安装日志收工具`elasticsearch`、`fluentd`、`kibana`。我们创建一个 `kube-logging` 名称空间，将 EFK 组件安装到该名称空间中。

```bash
mkdir -p /root/elf && cd /root/elf
cat > kube-logging.yaml <<"EOF"
kind: Namespace
apiVersion: v1
metadata:
  name: kube-logging
EOF
```

查看结果：

```bash
# kubectl apply -f kube-logging.yaml && kubectl get namespaces kube-logging
namespace/kube-logging created
NAME           STATUS   AGE
kube-logging   Active   1s
```

> 通过这个命令空间去安装 efk，首先部署一个有3个节点的Elasticsearch 集群。我们使用 3 个 Elasticsearch Pods 可以避免高可用中的多节点群集中发生的“裂脑”的问题。
>
> Elasticsearch 脑裂可参考如下：https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-node.html#splitbrain

## 1.2 创建 headless service 服务

创建一个 headless service 的 Kubernetes 服务，服务名称是 elasticsearch，这个服务将为 3 个 Pod 定义一个 DNS 域。headless service 不具备负载均衡也没有 IP。要了解有关 headless service 的更多信息，可参考 https://kubernetes.io/zh-cn/docs/concepts/services-networking/service

```bash
cat > elasticsearch_svc.yaml <<"EOF"
kind: Service
apiVersion: v1
metadata:
  name: elasticsearch
  namespace: kube-logging
  labels:
    app: elasticsearch
spec:
  selector:
    app: elasticsearch
  clusterIP: None
  ports:
    - port: 9200
      name: rest
    - port: 9300
      name: inter-node
EOF
```

查看结果：

```bash
# kubectl apply -f elasticsearch_svc.yaml && kubectl get services --namespace=kube-logging
service/elasticsearch created
NAME            TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)             AGE
elasticsearch   ClusterIP   None         <none>        9200/TCP,9300/TCP   0s
```

> 在 kube-logging 名称空间定义了一个名为 elasticsearch 的 Service 服务，带有 app=elasticsearch 标签，当我们将 Elasticsearch StatefulSet 与此服务关联时，服务将返回带有标签 app=elasticsearch 的 Elasticsearch Pods 的 DNS A 记录，然后设置 clusterIP=None，将该服务设置成无头服务。最后，我们分别定义端口 9200、9300，分别用于与 REST API 交互，以及用于节点间通信。 
>
> 现在我们已经为 Pod 设置了无头服务和一个稳定的域名`elasticsearch.kube-logging.svc.cluster.local`，接下来我们通过 StatefulSet 来创建具体的 Elasticsearch 的 Pod 应用。 

## 1.3 通过 StorageClass 创建 pv

storageclass创建方式：[https://www.cnblogs.com/-k8s/p/16875442.html](https://www.cnblogs.com/Mercury-linux/p/16875442.html)

**这里注意，本地nfs安装是在ubuntu系统进行的。**

## 1.4 安装 elasticsearch 集群

```bash
cat > elasticsearch-statefulset.yaml <<"EOF"
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: es-cluster
  namespace: kube-logging
spec:
  serviceName: elasticsearch
  replicas: 3
  selector:
    matchLabels:
      app: elasticsearch
  template:
    metadata:
      labels:
        app: elasticsearch
    spec:
      containers:
      - name: elasticsearch
        image: docker.elastic.co/elasticsearch/elasticsearch:7.2.0
        imagePullPolicy: IfNotPresent
        resources:
            limits:
              cpu: 1000m
            requests:
              cpu: 100m
        ports:
        - containerPort: 9200  #容器暴露了 9200 和 9300 两个端口，名称要和上面定义的 Service 保持一致.
          name: rest
          protocol: TCP
        - containerPort: 9300
          name: inter-node
          protocol: TCP
        volumeMounts:
        - name: data
          mountPath: /usr/share/elasticsearch/data  # 通过 volumeMount 声明了数据持久化目录，定义了一个 data 数据卷，通过 volumeMount 把它挂载到容器里的/usr/share/elasticsearch/data 目录。
        env:
          - name: cluster.name  # 环境变量：集群名称为：k8s-logs
            value: k8s-logs
          - name: node.name # node.name：节点的名称，通过metadata.name来获取。这将解析为es-cluster-[0,1,2]，取决于节点的指定顺序。
            valueFrom:
              fieldRef:
                fieldPath: metadata.name
          - name: discovery.seed_hosts # 此字段用于设置在 Elasticsearch 集群中节点相互连接的发现方法，它为我们的集群指定了一个静态主机列表。由于我们之前配置的是无头服务，我们的 Pod 具有唯一的 DNS 地址 es-cluster-[0,1,2].elasticsearch.kube-logging.svc.cluster.local，因此我们相应地设置此地址变量即可。由于都在同一个 namespace 下面，所以我们可以将其缩短为 es-cluster-[0,1,2].elasticsearch。https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-discovery.html。
            value: "es-cluster-0.elasticsearch.kube-logging.svc.cluster.local,es-cluster-1.elasticsearch.kube-logging.svc.cluster.local,es-cluster-2.elasticsearch.kube-logging.svc.cluster.local"
          - name: cluster.initial_master_nodes
            value: "es-cluster-0,es-cluster-1,es-cluster-2"
          - name: ES_JAVA_OPTS
            value: "-Xms512m -Xmx512m"  # 使用 resources 字段来指定容器至少需要0.1 个 vCPU，并且容器最多可以使用 1 个 vCPU 了解有关资源请求和限制 https://www.elastic.co/guide/en/elasticsearch/reference/current/heap-size.html
      initContainers:
      - name: fix-permissions
        image: busybox
        imagePullPolicy: IfNotPresent
        command: ["sh", "-c", "chown -R 1000:1000 /usr/share/elasticsearch/data"] # 将 Elasticsearch 数据目录的用户和组更改为 1000:1000（Elasticsearch 用户的 UID）。因为默认情况下，Kubernetes 用 root 用户挂载数据目录，这会使得 Elasticsearch 无法访问该数据目录。可以参考 Elasticsearch 生产中的一些默认注意事项相关文档说明：https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html#_notes_for_production_use_and_defaults。
        securityContext:
          privileged: true
        volumeMounts:
        - name: data
          mountPath: /usr/share/elasticsearch/data
      - name: increase-vm-max-map # increase-vm-max-map 的容器用来增加操作系统对 mmap 计数的限制，默认情况下该值可能太低，导致内存不足的错误，要了解更多关于该设置的信息，可以查看 Elasticsearch 官方文档说明：https://www.elastic.co/guide/en/elasticsearch/reference/current/vm-max-map-count.html
        image: busybox
        imagePullPolicy: IfNotPresent
        command: ["sysctl", "-w", "vm.max_map_count=262144"]
        securityContext:
          privileged: true
      - name: increase-fd-ulimit # 用来执行 ulimit 命令增加打开文件描述符的最大数量的。
        image: busybox
        imagePullPolicy: IfNotPresent
        command: ["sh", "-c", "ulimit -n 65536"]
        securityContext:
          privileged: true
  volumeClaimTemplates:
  - metadata:
      name: data
      labels:
        app: elasticsearch
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: nfs-storage
      resources:
        requests:
          storage: 10Gi
EOF
```

创建StatefulSet：

```bash
# kubectl apply -f elasticsearch-statefulset.yaml
statefulset.apps/es-cluster created

# kubectl get pods -n kube-logging
NAME                      READY   STATUS    RESTARTS        AGE
es-cluster-0              1/1     Running   1 (2d15h ago)   3d12h
es-cluster-1              1/1     Running   0               2d15h
es-cluster-2              1/1     Running   1 (2d15h ago)   3d2h

# kubectl get svc -n kube-logging
NAME            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE
elasticsearch   ClusterIP   None             <none>        9200/TCP,9300/TCP   3d13h
```

> 上面内容的解释：在 kube-logging 的名称空间中定义了一个 es-cluster 的 StatefulSet。然后，我们使用 serviceName 字段与我们之前创建的 headless ElasticSearch 服务相关联。这样可以确保可以使用以下 DNS 地址访问 StatefulSet 中的每个Pod：，es-cluster-[0,1,2].elasticsearch.kube-logging.svc.cluster.local，其中[0,1,2]与 Pod 分配的序号数相对应。我们指定 3 个 replicas（3 个 Pod 副本），将 selector matchLabels 设置为 app: elasticseach。该.spec.selector.matchLabels 和.spec.template.metadata.labels 字段必须匹配。
>
> resources参数显示cpu可参考：[https://kubernetes.io/docs/concepts/configuration/manage-resources-containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)

 

测试集群连接：

```bash
# kubectl port-forward es-cluster-0 9200:9200 --namespace=kube-logging

# curl http://localhost:9200/_cluster/state?pretty |grep transport_address
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
      "transport_address" : "172.16.85.233:9300",
      "transport_address" : "172.16.58.235:9300",
      "transport_address" : "172.16.85.238:9300",
                  "transport_address" : {
                      "transport_address" : {
                  "transport_address" : {
                  "transport_address" : {
                  "transport_address" : {
100  235k  100  235k    0     0  6717k      0 --:--:-- --:--:-- --:--:-- 6717k
```

看到上面的信息就表明我们名为 k8s-logs 的 Elasticsearch 集群成功创建了 3 个节点：es-cluster-0，es-cluster-1，和 es-cluster-2，当前主节点是 es-cluster-0。

# 2. 安装 kibana 可视化 UI 界面

```bash
cat > kibana.yaml <<"EOF"
apiVersion: v1
kind: Service
metadata:
  name: kibana
  namespace: kube-logging
  labels:
    app: kibana
spec:
  ports:
  - port: 5601
  selector:
    app: kibana
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kibana
  namespace: kube-logging
  labels:
    app: kibana
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kibana
  template:
    metadata:
      labels:
        app: kibana
    spec:
      containers:
      - name: kibana
        image: docker.elastic.co/kibana/kibana:7.2.0
        imagePullPolicy: IfNotPresent
        resources:
          limits:
            cpu: 1000m
          requests:
            cpu: 100m
        env:
          - name: ELASTICSEARCH_URL
            value: http://elasticsearch.kube-logging.svc.cluster.local:9200
        ports:
        - containerPort: 5601
EOF
```

- ELASTICSEARCH_URL 这个环境变量来设置 Elasticsearch 集群的端点和端口，直接使用 Kubernetes DNS 即可，此端点对应服务名称为 elasticsearch，由于是一个 headless service，所以该域将解析为 3 个 Elasticsearch Pod 的 IP 地址列表。 

创建kibanna：

```bash
# kubectl apply -f kibana.yaml
service/kibana created
deployment.apps/kibana created

# kubectl get pods -n kube-logging
NAME                      READY   STATUS    RESTARTS        AGE
es-cluster-0              1/1     Running   1 (2d16h ago)   3d13h
es-cluster-1              1/1     Running   0               2d16h
es-cluster-2              1/1     Running   1 (2d16h ago)   3d3h
kibana-66f59798b7-l8gjr   1/1     Running   0               2d16h
```

修改 service 的 type 类型为 NodePort：

```bash
# kubectl edit svc kibana -n kube-logging
把 type: ClusterIP 变成 type: NodePort
保存退出之后


# kubectl get svc -n kube-logging
NAME            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE
elasticsearch   ClusterIP   None             <none>        9200/TCP,9300/TCP   66s
kibana          NodePort    10.110.183.214   <none>        5601:30178/TCP      66s
```

登录界面：http://{任意节点ip}:30178 (端口是随机的，根据自己的端口访问)

![img](https://img2023.cnblogs.com/blog/1740081/202303/1740081-20230309124808254-1448315233.png)

# 3. 安装 fluentd 组件

```bash
cat > fluentd.yaml <<"EOF"
apiVersion: v1
kind: ServiceAccount
metadata:
  name: fluentd
  namespace: kube-logging
  labels:
    app: fluentd
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: fluentd
  labels:
    app: fluentd
rules:
- apiGroups:
  - ""
  resources:
  - pods
  - namespaces
  verbs:
  - get
  - list
  - watch
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: fluentd
roleRef:
  kind: ClusterRole
  name: fluentd
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: ServiceAccount
  name: fluentd
  namespace: kube-logging
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: kube-logging
  labels:
    app: fluentd
spec:
  selector:
    matchLabels:
      app: fluentd
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      serviceAccount: fluentd
      serviceAccountName: fluentd
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - name: fluentd
        image: fluent/fluentd-kubernetes-daemonset:v1.4.2-debian-elasticsearch-1.1
        imagePullPolicy: IfNotPresent
        env:
          - name:  FLUENT_ELASTICSEARCH_HOST
            value: "elasticsearch.kube-logging.svc.cluster.local"
          - name:  FLUENT_ELASTICSEARCH_PORT
            value: "9200"
          - name: FLUENT_ELASTICSEARCH_SCHEME
            value: "http"
          - name: FLUENTD_SYSTEMD_CONF
            value: disable
        resources:
          limits:
            memory: 512Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
EOF
```

创建：

```bash
#  kubectl apply -f fluentd.yaml
serviceaccount/fluentd created
clusterrole.rbac.authorization.k8s.io/fluentd created
clusterrolebinding.rbac.authorization.k8s.io/fluentd created
daemonset.apps/fluentd created

# kubectl get pods -n kube-logging
NAME                      READY   STATUS    RESTARTS        AGE
es-cluster-0              1/1     Running   1 (2d16h ago)   3d13h
es-cluster-1              1/1     Running   0               2d16h
es-cluster-2              1/1     Running   1 (2d16h ago)   3d3h
fluentd-7fgv2             1/1     Running   1 (40h ago)     2d16h
fluentd-lcxll             1/1     Running   0               2d16h
fluentd-mkdjj             1/1     Running   0               2d16h
kibana-66f59798b7-l8gjr   1/1     Running   0               2d16h
```

Fluentd 启动成功后，我们可以前往 Kibana 的 Dashboard 页面中，点击左侧的 Discover

![img](https://img2023.cnblogs.com/blog/1740081/202303/1740081-20230309124900568-1724843293.png)

可以看到如下配置页面： 

![img](https://img2023.cnblogs.com/blog/1740081/202303/1740081-20230309124924264-1380887761.png)

在这里可以配置我们需要的 Elasticsearch 索引，前面 Fluentd 配置文件中我们采集的日志使用的是 logstash 格式，这里只需要在文本框中输入 logstash-*即可匹配到 Elasticsearch 集群中的所有日志数据。

![img](https://img2023.cnblogs.com/blog/1740081/202303/1740081-20230309125000635-1482685181.png)

点击 Next step

![img](https://img2023.cnblogs.com/blog/1740081/202303/1740081-20230309125115530-1980099677.png)

选择@timestamp，创建索引

点击左侧的 discover，可看到如下：

![img](https://img2023.cnblogs.com/blog/1740081/202303/1740081-20230309125219855-905397244.png)

# 4. 测试收集 pod 容器日志

```bash
cat > pod.yaml <<"EOF"
apiVersion: v1
kind: Pod
metadata:
  name: counter
spec:
  containers:
  - name: count
    image: busybox
    imagePullPolicy: IfNotPresent
    args: [/bin/sh, -c,'i=0; while true; do echo "$i: $(date)"; i=$((i+1)); sleep 1; done']
EOF

# kubectl apply -f pod.yaml
pod/counter created
```

Kibana 查询语言 KQL 官方地址： [https://www.elastic.co/guide/en/kibana/7.2/kuery-query.html ](https://www.elastic.co/guide/en/kibana/7.2/kuery-query.html)

登录到 kibana 的控制面板，在 discover 处的搜索栏中输入`kubernetes.pod_name:counter`，这将过滤名为的 Pod 的日志数据 counter，如下所示： 

![img](https://img2023.cnblogs.com/blog/1740081/202303/1740081-20230309125358180-1650241073.png)

 