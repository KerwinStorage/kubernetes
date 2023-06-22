# 准备

官网：

- [https://prometheus.io](https://prometheus.io/)
- [https://grafana.com](https://grafana.com/)

其他文章：

- [#Prometheus (qq.com)](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzIxNjM0OTk3MQ==&action=getalbum&album_id=2004098737979064320&scene=173&from_msgid=2247484431&from_itemidx=1&count=3&nolastread=1#wechat_redirect)

创建命名空间：

```bash
mkdir -p /root/prometheus && cd /root/prometheus
cat > monitor-sa.yaml <<"EOF"
kind: Namespace
apiVersion: v1
metadata:
  name: monitor-sa
EOF
```

查看命名空间：

```bash
kubectl apply -f  monitor-sa.yaml
kubectl get namespace  monitor-sa
NAME         STATUS   AGE
monitor-sa   Active   37s
```

# 1. node-export安装

镜像地址：https://hub.docker.com/r/prom/node-exporter/tags

github：https://github.com/prometheus/node_exporter

```bash
mkdir -p /root/prometheus && cd /root/prometheus
cat > node-export.yaml <<"EOF"
apiVersion: apps/v1
kind: DaemonSet #可以保证k8s集群的每个节点都运行完全一样的pod
metadata:
  name: node-exporter
  namespace: monitor-sa
  labels:
    name: node-exporter
spec:
  selector:
    matchLabels:
     name: node-exporter
  template:
    metadata:
      labels:
        name: node-exporter
    spec:
      hostPID: true # 使用主机的PID
      hostIPC: true # 使用主机的IPC
      hostNetwork: true # 使用主机的网络
      containers:
      - name: node-exporter
        image: prom/node-exporter:v1.3.0
        ports:
        - containerPort: 9100
        resources:
          requests:
            cpu: 0.15
        securityContext: # 开启特权模式，以root运行容器
          privileged: true
        args:
        - --path.procfs=/host/proc
        - --path.sysfs=/host/sys
        - --path.rootfs=/host
        - --collector.filesystem.ignored-mount-points
        - '"^/(sys|proc|dev|host|etc)($|/)"'
        volumeMounts:
        - name: dev
          mountPath: /host/dev
        - name: proc
          mountPath: /host/proc
        - name: sys
          mountPath: /host/sys
        - name: rootfs
          mountPath: /rootfs
      tolerations: # 配置容忍度，可以容忍master节点的污点，master节点也需要收集数据
      - key: "node-role.kubernetes.io/master"
        operator: "Exists"
        effect: "NoSchedule"
      volumes: # 下面定义的是宿主机的目录，挂载到容器内
        - name: proc
          hostPath:
            path: /proc
        - name: dev
          hostPath:
            path: /dev
        - name: sys
          hostPath:
            path: /sys
        - name: rootfs
          hostPath:
            path: /
EOF
```

创建node-export：

```bash
kubectl apply -f node-export.yaml
kubectl get pod -n monitor-sa
NAME                  READY   STATUS    RESTARTS   AGE
node-exporter-4f568   1/1     Running   0          17s
node-exporter-l2nw7   1/1     Running   0          17s
node-exporter-m58ps   1/1     Running   0          17s

# 查看端口
netstat -lntup | grep 9100
tcp6       0      0 :::9100                 :::*                    LISTEN      1407112/node_export

# 查看数据采集数据
curl http://localhost:9100/metrics

# 显示本地主机cpu使用情况
curl http://localhost:9100/metrics | grep node_cpu_seconds

# 查看负载
curl http://localhost:9100/metrics | grep node_load
```

**注：node-export默认的监听端口是9100，可以看到当前主机获取到的所有监控数据。**

```bash
netstat -lntup | grep 9100
tcp6       0      0 :::9100                 :::*                    LISTEN      3045628/node_export
```

报错处理，**使用**`prom/node-exporter:v0.16.0`**出现的问题，本地是已经挂载了这个目录，但是出现报错，干脆直接升级镜像到**`prom/node-exporter:v1.3.0`**，重启`Prometheus`，发现可以了，可以观察`node-exporter`容器的日志和监控模版磁盘读写速率的数据有没有出现。**

```bash
kubectl logs -f -n monitor-sa node-exporter-4f568
time="2023-03-11T04:54:35Z" level=error msg="ERROR: diskstats collector failed after 0.000281s: invalid line for /host/proc/diskstats for sda" source="collector.go:132"
time="2023-03-11T04:54:50Z" level=error msg="ERROR: diskstats collector failed after 0.001121s: invalid line for /host/proc/diskstats for sr0" source="collector.go:132"
time="2023-03-11T04:55:05Z" level=error msg="ERROR: diskstats collector failed after 0.000325s: invalid line for /host/proc/diskstats for sr0" source="collector.go:132"
time="2023-03-11T04:55:20Z" level=error msg="ERROR: diskstats collector failed after 0.000349s: invalid line for /host/proc/diskstats for sr1" source="collector.go:132"
time="2023-03-11T04:55:35Z" level=error msg="ERROR: diskstats collector failed after 0.001031s: invalid line for /host/proc/diskstats for sr1" source="collector.go:132"
```

# 2. 部署prometheus server

## 2.1 创建sa账号

rbac文件内容：

```bash
mkdir -p /root/prometheus && cd /root/prometheus
cat > rbac.yaml <<"EOF"
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
  namespace: monitor-sa
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus
rules:
- apiGroups:
  - ""
  resources:
  - nodes
  - services
  - endpoints
  - pods
  - nodes/proxy
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - configmaps
  - nodes/metrics
  verbs:
  - get
- nonResourceURLs:
  - /metrics
  verbs:
  - get
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
- kind: ServiceAccount
  name: prometheus
  namespace: monitor-sa
EOF
```

创建rbac：

```bash
kubectl apply -f rbac.yaml
```

## 2.2 创建数据目录

prometheus在哪个节点部署就在哪个节点上创建目录

```bash
mkdir -p /data
chmod 777 /data/
```

## 2.3 部署prometheus

### 2.3.1 prometheus.yml文件配置

ConfigMap文件内容：

```bash
mkdir -p /root/prometheus && cd /root/prometheus
cat > prometheus-cfg.yaml <<"EOF"
---
kind: ConfigMap
apiVersion: v1
metadata:
  labels:
    app: prometheus
  name: prometheus-config
  namespace: monitor-sa
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
      scrape_timeout: 10s
      evaluation_interval: 1m
    scrape_configs:
    - job_name: 'kubernetes-node'
      kubernetes_sd_configs:
      - role: node
      relabel_configs:
      - source_labels: [__address__]
        regex: '(.*):10250'
        replacement: '${1}:9100'
        target_label: __address__
        action: replace
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
    - job_name: 'kubernetes-node-cadvisor'
      kubernetes_sd_configs:
      - role:  node
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
      - target_label: __address__
        replacement: kubernetes.default.svc:443
      - source_labels: [__meta_kubernetes_node_name]
        regex: (.+)
        target_label: __metrics_path__
        replacement: /api/v1/nodes/${1}/proxy/metrics/cadvisor
    - job_name: 'kubernetes-apiserver'
      kubernetes_sd_configs:
      - role: endpoints
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      relabel_configs:
      - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
        action: keep
        regex: default;kubernetes;https
    - job_name: 'kubernetes-service-endpoints'
      kubernetes_sd_configs:
      - role: endpoints
      relabel_configs:
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scheme]
        action: replace
        target_label: __scheme__
        regex: (https?)
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_service_annotation_prometheus_io_port]
        action: replace
        target_label: __address__
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
      - action: labelmap
        regex: __meta_kubernetes_service_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_service_name]
        action: replace
        target_label: kubernetes_name
EOF
```

创建configmap：

```bash
# kubectl apply -f prometheus-cfg.yaml
configmap/prometheus-config created

# kubectl get configmaps  -n monitor-sa
NAME                DATA   AGE
prometheus-config   1      29s
```

### 2.3.2 prometheus Deployment创建

镜像地址：

- https://hub.docker.com/r/prom/prometheus/tags
- https://quay.io/repository/prometheus/prometheus?tab=info

查看node节点名称，将服务创建到`k8s-node01`节点上（这里如果想部署到master需要注意节点的污点）。

```bash
kubectl get node | awk '{print $1}'
NAME
k8s-master01
k8s-node01
k8s-node02
```

deployments文件内容：

```bash
mkdir -p /root/prometheus && cd /root/prometheus
cat > prometheus-deploy.yaml <<"EOF"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus-server
  namespace: monitor-sa
  labels:
    app: prometheus
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
      component: server
    #matchExpressions:
    #- {key: app, operator: In, values: [prometheus]}
    #- {key: component, operator: In, values: [server]}
  template:
    metadata:
      labels:
        app: prometheus
        component: server
      annotations:
        prometheus.io/scrape: 'false'
    spec:
      nodeName: k8s-node01  # 注意这里node01的名称，根据本地填写让pod调度到这个节点上面
      serviceAccountName: prometheus
      containers:
      - name: prometheus
        image: prom/prometheus:v2.37.6
        imagePullPolicy: IfNotPresent
        command:
          - prometheus
          - --config.file=/etc/prometheus/prometheus.yml
          - --storage.tsdb.path=/prometheus
          - --storage.tsdb.retention=720h
          - --web.enable-lifecycle
        ports:
        - containerPort: 9090
          protocol: TCP
        volumeMounts:
        - mountPath: /etc/prometheus/prometheus.yml
          name: prometheus-config
          subPath: prometheus.yml
        - mountPath: /prometheus/
          name: prometheus-storage-volume
      volumes:
        - name: prometheus-config
          configMap:
            name: prometheus-config
            items:
              - key: prometheus.yml
                path: prometheus.yml
                mode: 0644
        - name: prometheus-storage-volume
          hostPath:
           path: /data
           type: Directory
EOF
```

创建deployments：

```bash
# kubectl apply -f prometheus-deploy.yaml
deployment.apps/prometheus-server created

# kubectl get pod -n monitor-sa
NAME                                 READY   STATUS    RESTARTS   AGE
node-exporter-4f568                  1/1     Running   0          5h41m
node-exporter-l2nw7                  1/1     Running   0          5h41m
node-exporter-m58ps                  1/1     Running   0          5h41m
prometheus-server-858c97464f-b4czn   1/1     Running   0          3m57s
```

### 2.3.3 prometheus svc创建

svc文件内容：

```bash
mkdir -p /root/prometheus && cd /root/prometheus
cat > prometheus-svc.yaml <<"EOF"
apiVersion: v1
kind: Service
metadata:
  name: prometheus
  namespace: monitor-sa
  labels:
    app: prometheus
spec:
  type: NodePort
  ports:
    - port: 9090
      targetPort: 9090
      protocol: TCP
  selector:
    app: prometheus
    component: server
EOF
```

创建svc（这里我们使用的是用node节点的port去访问）：

```bash
# kubectl apply -f prometheus-svc.yaml
service/prometheus created

# kubectl get svc -n monitor-sa
NAME         TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
prometheus   NodePort   10.101.117.42   <none>        9090:30170/TCP   11h
```

访问地址：http://{任意主机ip地址}:30170

![img](https://img2023.cnblogs.com/blog/1740081/202303/1740081-20230311173243530-160865586.png)

**注：如果查看这样页面没数据最好去看看`prometheus-server`容器的log日志。**

> **level=error ts=2023-03-09T13:09:18.515651655Z caller=main.go:216 component=k8s_client_runtime err="github.com/prometheus/prometheus/discovery/kubernetes/kubernetes.go:269: Failed to list \*v1.Service: services is forbidden: User \"system:serviceaccount:monitor-sa:monitor\" cannot list resource \"services\" in API group \"\" at the cluster scope: RBAC: clusterrole.rbac.authorization.k8s.io \"cluster-admin\\u00a0\" not found"**

这里是因为`ClusterRole`授权有问题。

# 3. Grafana安装和配置

## 3.1 部署Grafana

镜像地址：https://hub.docker.com/r/grafana/grafana

```bash
mkdir -p /root/prometheus && cd /root/prometheus
cat > Grafana.yaml <<"EOF"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: monitoring-grafana
  namespace: monitor-sa
spec:
  replicas: 1
  selector:
    matchLabels:
      task: monitoring
      k8s-app: grafana
  template:
    metadata:
      labels:
        task: monitoring
        k8s-app: grafana
    spec:
      securityContext:
        fsGroup: 472
        supplementalGroups:
          - 0
      containers:
      - name: grafana
        image: grafana/grafana:9.4.3
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 3000
          protocol: TCP
        volumeMounts:
        - mountPath: /var/lib/grafana
          name: grafana-storage
        - mountPath: /etc/ssl/certs
          name: ca-certificates
          readOnly: true
        env:
        - name: INFLUXDB_HOST
          value: monitoring-influxdb
        - name: GF_SERVER_HTTP_PORT
          value: "3000"
          # The following env variables are required to make Grafana accessible via
          # the kubernetes api-server proxy. On production clusters, we recommend
          # removing these env variables, setup auth for grafana, and expose the grafana
          # service using a LoadBalancer or a public IP.
        - name: GF_AUTH_BASIC_ENABLED
          value: "false"
        - name: GF_AUTH_ANONYMOUS_ENABLED
          value: "true"
        - name: GF_AUTH_ANONYMOUS_ORG_ROLE
          value: Admin
        - name: GF_SERVER_ROOT_URL
          # If you're only using the API Server proxy, set this value instead:
          # value: /api/v1/namespaces/kube-system/services/monitoring-grafana/proxy
          value: /
      volumes:
      - name: ca-certificates
        hostPath:
          path: /etc/ssl/certs
      - name: grafana-storage
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  labels:
    # For use as a Cluster add-on (https://github.com/kubernetes/kubernetes/tree/master/cluster/addons)
    # If you are NOT using this as an addon, you should comment out this line.
    kubernetes.io/cluster-service: 'true'
    kubernetes.io/name: monitoring-grafana
  name: monitoring-grafana
  namespace: monitor-sa
spec:
  # In a production setup, we recommend accessing Grafana through an external Loadbalancer
  # or through a public IP.
  # type: LoadBalancer
  # You could also use NodePort to expose the service at a randomly-generated port
  # type: NodePort
  ports:
  - port: 80
    targetPort: 3000
  selector:
    k8s-app: grafana
  type: NodePort
EOF
```

创建Grafana：

```bash
# kubectl apply -f Grafana.yaml
deployment.apps/monitoring-grafana created
service/monitoring-grafana created

# 查看pod启动情况
kubectl get pods -n kube-system| grep monitor
monitoring-grafana-5b4bb4c64b-5qwxj        1/1     Running   0                41s

# 查看svc
kubectl get svc -n kube-system | grep grafana
monitoring-grafana            NodePort    10.101.25.22   <none>        80:31478/TCP                   17m
```

## 3.2 登陆grafana

在浏览器访问：http://{任意主机ip地址}:31478

账号密码都是：`admin`

![img](https://img2023.cnblogs.com/blog/1740081/202303/1740081-20230311173621664-778016106.png)

## 3.3 Grafana接入Prometheus数据源

![img](https://img2023.cnblogs.com/blog/1740081/202303/1740081-20230311175016097-185367957.png)

找到默认的Prometheus

![img](https://img2023.cnblogs.com/blog/1740081/202303/1740081-20230311175050705-1229078089.png)

- Name：Prometheus
- URL：[http://prometheus.monitor-sa.svc:9090](http://prometheus.monitor-sa.svc:9090/)

![img](https://img2023.cnblogs.com/blog/1740081/202303/1740081-20230311175159683-1834162100.png)

点击

![img](https://img2023.cnblogs.com/blog/1740081/202303/1740081-20230311175320045-260018640.png)

## 3.4 导入Node Exporter模版

模版地址：https://grafana.com/grafana/dashboards/?search=node_exporter&collector=nodeexporter

使用的模版：[Node Exporter for Prometheus Dashboard based on 11074 | Grafana Labs](https://grafana.com/grafana/dashboards/15172-node-exporter-for-prometheus-dashboard-based-on-11074/)

ID：11074

![img](https://img2023.cnblogs.com/blog/1740081/202303/1740081-20230311181037212-1084398314.png)

**注：请注意，当我们选择模版时也会有`Grafana`和`Prometheus`版本要求。**

![img](https://img2023.cnblogs.com/blog/1740081/202303/1740081-20230311181246122-1227064937.png)

效果图：

![img](https://img2023.cnblogs.com/blog/1740081/202303/1740081-20230311181636331-162170855.png)

# 4. 安装配置kube-state-metrics组件

kube-state-metrics是什么？

`kube-state-metrics`通过监听API Server生成有关资源对象的状态指标，比如Deployment、Node、Pod，需要注意的是kube-state-metrics只是简单的提供一个metrics数据，并不会存储这些指标数据，所以我们可以使用Prometheus来抓取这些数据然后存储，主要关注的是业务相关的一些元数据，比如Deployment、Pod、副本状态等；调度了多少个replicas？现在可用的有几个？多少个Pod是running/stopped/terminated状态？Pod重启了多少次？我有多少job在运行中。

## 4.1 安装kube-state-metrics组件

注意：部署`kube-state-metrics`有对应的[版本要求](https://github.com/kubernetes/kube-state-metrics#compatibility-matrix)，请根据自己的k8s集群版本来判断使用什么版本的`kube-state-metrics`。

项目地址：https://github.com/kubernetes/kube-state-metrics

镜像地址：https://hub.docker.com/search?q=kube-state-metrics [coreos/kube-state-metrics · Quay](https://quay.io/repository/coreos/kube-state-metrics?tab=tags)

文件在GitHub的位置：[点击这里](https://github.com/kubernetes/kube-state-metrics/tree/v2.6.0/examples/standard)

```bash
mkdir -p /root/prometheus && cd /root/prometheus
cat > kube-state-metrics.yaml <<"EOF"
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  labels:
    app.kubernetes.io/component: exporter
    app.kubernetes.io/name: kube-state-metrics
    app.kubernetes.io/version: 2.6.0
  name: kube-state-metrics
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: kube-state-metrics
subjects:
- kind: ServiceAccount
  name: kube-state-metrics
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    app.kubernetes.io/component: exporter
    app.kubernetes.io/name: kube-state-metrics
    app.kubernetes.io/version: 2.6.0
  name: kube-state-metrics
rules:
- apiGroups:
  - ""
  resources:
  - configmaps
  - secrets
  - nodes
  - pods
  - services
  - serviceaccounts
  - resourcequotas
  - replicationcontrollers
  - limitranges
  - persistentvolumeclaims
  - persistentvolumes
  - namespaces
  - endpoints
  verbs:
  - list
  - watch
- apiGroups:
  - apps
  resources:
  - statefulsets
  - daemonsets
  - deployments
  - replicasets
  verbs:
  - list
  - watch
- apiGroups:
  - batch
  resources:
  - cronjobs
  - jobs
  verbs:
  - list
  - watch
- apiGroups:
  - autoscaling
  resources:
  - horizontalpodautoscalers
  verbs:
  - list
  - watch
- apiGroups:
  - authentication.k8s.io
  resources:
  - tokenreviews
  verbs:
  - create
- apiGroups:
  - authorization.k8s.io
  resources:
  - subjectaccessreviews
  verbs:
  - create
- apiGroups:
  - policy
  resources:
  - poddisruptionbudgets
  verbs:
  - list
  - watch
- apiGroups:
  - certificates.k8s.io
  resources:
  - certificatesigningrequests
  verbs:
  - list
  - watch
- apiGroups:
  - storage.k8s.io
  resources:
  - storageclasses
  - volumeattachments
  verbs:
  - list
  - watch
- apiGroups:
  - admissionregistration.k8s.io
  resources:
  - mutatingwebhookconfigurations
  - validatingwebhookconfigurations
  verbs:
  - list
  - watch
- apiGroups:
  - networking.k8s.io
  resources:
  - networkpolicies
  - ingresses
  verbs:
  - list
  - watch
- apiGroups:
  - coordination.k8s.io
  resources:
  - leases
  verbs:
  - list
  - watch
- apiGroups:
  - rbac.authorization.k8s.io
  resources:
  - clusterrolebindings
  - clusterroles
  - rolebindings
  - roles
  verbs:
  - list
  - watch
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/component: exporter
    app.kubernetes.io/name: kube-state-metrics
    app.kubernetes.io/version: 2.6.0
  name: kube-state-metrics
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: kube-state-metrics
  template:
    metadata:
      labels:
        app.kubernetes.io/component: exporter
        app.kubernetes.io/name: kube-state-metrics
        app.kubernetes.io/version: 2.6.0
    spec:
      automountServiceAccountToken: true
      containers:
      - image: bitnami/kube-state-metrics:2.6.0
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 5
          timeoutSeconds: 5
        name: kube-state-metrics
        ports:
        - containerPort: 8080
          name: http-metrics
        - containerPort: 8081
          name: telemetry
        readinessProbe:
          httpGet:
            path: /
            port: 8081
          initialDelaySeconds: 5
          timeoutSeconds: 5
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
          readOnlyRootFilesystem: true
          runAsUser: 65534
      nodeSelector:
        kubernetes.io/os: linux
      serviceAccountName: kube-state-metrics
---
apiVersion: v1
automountServiceAccountToken: false
kind: ServiceAccount
metadata:
  labels:
    app.kubernetes.io/component: exporter
    app.kubernetes.io/name: kube-state-metrics
    app.kubernetes.io/version: 2.6.0
  name: kube-state-metrics
  namespace: kube-system
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/component: exporter
    app.kubernetes.io/name: kube-state-metrics
    app.kubernetes.io/version: 2.6.0
    prometheus.io/scrape: 'true'
  name: kube-state-metrics
  namespace: kube-system
spec:
  clusterIP: None
  ports:
  - name: http-metrics
    port: 8080
    targetPort: http-metrics
    protocol: TCP
  - name: telemetry
    port: 8081
    targetPort: telemetry
    protocol: TCP
  selector:
    app.kubernetes.io/name: kube-state-metrics
EOF
```

创建资源：

```bash
kubectl apply -f kube-state-metrics.yaml
# 查看kube-state-metrics是否部署成功
kubectl get pods -n kube-system
NAME                                       READY   STATUS    RESTARTS         AGE
kube-state-metrics-5df6ff465f-zkg58        1/1     Running   0                2m33s
```

查看资源监控情况：

![img](https://img2023.cnblogs.com/blog/1740081/202303/1740081-20230313232426191-1153966262.png)

## 4.2 Grafana加入监控模版

https://grafana.com/grafana/dashboards/?search=kubernetes

参考：[Kubernetes云原生监控之cAdvisor容器资源监控](https://mp.weixin.qq.com/s?__biz=MzIxNjM0OTk3MQ==&mid=2247484449&idx=1&sn=b7799c2cd459b866b793c54de9633c8c&chksm=978b285da0fca14bc08db352898f1a5f9ebbeabfcdbd2f11616fdce8232bca7642c64e8e216b&scene=21#wechat_redirect)

导入：`3125`和`13025`dashboard，`cAdvisor`性能监控指标就展示到模板上。

![img](https://img2023.cnblogs.com/blog/1740081/202303/1740081-20230314143851539-1575312007.png)

# 5. 安装和配置Alertmanager-发送报警到qq邮箱

Alertmanager配置文件：https://prometheus.io/docs/alerting/latest/configuration

## 5.1 QQ邮箱smtp授权

首先要开启QQ邮箱的smtp服务，默认是关闭的。

登录[QQ邮箱](https://mail.qq.com/)，点“设置” - “帐户”。

找到“POP3/SMTP服务”和“IMAP/SMTP服务”项，点“开启”。

![img](https://img2023.cnblogs.com/blog/1740081/202303/1740081-20230315161450922-144154753.png)

开启之后，点击“生成授权码”。这个授权码将作为邮箱的身份认证密码。

![img](https://img2023.cnblogs.com/blog/1740081/202303/1740081-20230315161511638-1551434656.png)

SMTP服务器：`smtp.qq.com`

SMTP端口号：`465`。必须填这个端口号，否则会报错。

身份认证用户名：填完整的邮箱名，如：[123456789@qq.com](mailto:123456789@qq.com)，包括@qq.com部分。

身份认证密码：填上述的QQ邮箱授权码。注意，不是QQ邮箱的登录密码。

## 5.2 configmap配置清单

GitHub：[GitHub - prometheus/alertmanager: Prometheus Alertmanager](https://github.com/prometheus/alertmanager)

```bash
mkdir -p /root/prometheus && cd /root/prometheus
cat > alertmanager-configmap.yaml <<"EOF"
apiVersion: v1
kind: ConfigMap
metadata:
  name: alertmanager-config
  namespace: monitor-sa
data:
  config.yml: |-
    global:
      # 在没有报警的状况下声明为已解决的时间
      resolve_timeout: 5m
      # 配置邮件发送信息
      smtp_smarthost: 'smtp.qq.com:465'
      smtp_from: '123456@qq.com'
      smtp_auth_username: '123456@qq.com'
      smtp_auth_password: '密码'
      smtp_require_tls: false
      # 全部报警信息进入后的根路由，用来设置报警的分发策略
    route:
      # 这里的标签列表是接收到报警信息后的从新分组标签，例如，接收到的报警信息里面有许多具备 cluster=A 和 alertname=LatncyHigh 这样的标签的报警信息将会批量被聚合到一个分组里面
      group_by: ['alertname', 'cluster']
      # 当一个新的报警分组被建立后，须要等待至少group_wait时间来初始化通知，这种方式能够确保您能有足够的时间为同一分组来获取多个警报，而后一块儿触发这个报警信息。
      group_wait: 30s
      # 当第一个报警发送后，等待'group_interval'时间来发送新的一组报警信息。
      group_interval: 5m
      # 若是一个报警信息已经发送成功了，等待'repeat_interval'时间来从新发送他们
      repeat_interval: 5m
      # 默认的receiver：若是一个报警没有被一个route匹配，则发送给默认的接收器
      receiver: default
    receivers:
    - name: 'default'
      email_configs:
      - to: '*@163.com'
        send_resolved: true
EOF
```

**注：发送邮箱跟接收邮箱不能是一个，接收邮箱和发送邮箱自己定义。**

```bash
kubectl apply -f alertmanager-configmap.yaml
configmap/alertmanager created
```

## 5.3 deployment配置清单

镜像地址：[prom/alertmanager Tags | Docker Hub](https://hub.docker.com/r/prom/alertmanager/tags)

```bash
mkdir -p /root/prometheus && cd /root/prometheus
cat > alertmanager-deploy.yaml <<"EOF"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: alertmanager
  namespace: monitor-sa
spec:
  replicas: 1
  selector:
    matchLabels:
      app: alertmanager
  template:
    metadata:
      labels:
        app: alertmanager
    spec:
      nodeName: k8s-node01
      containers:
      - name: alertmanager
        image: prom/alertmanager:v0.20.0
        args:
          - "--config.file=/etc/alertmanager/config.yml"
          - "--storage.path=/alertmanager"
          - "--log.level=debug"
        ports:
        - name: alertmanager
          containerPort: 9093
        volumeMounts:
        - name: alertmanager-cm
          mountPath: /etc/alertmanager
        - name: localtime
          mountPath: /etc/localtime
      volumes:
      - name: alertmanager-cm
        configMap:
          name: alertmanager-config
      - name: localtime
        hostPath:
          path: /usr/share/zoneinfo/Asia/Shanghai       
EOF
```

创建deployment：

```bash
kubectl apply -f alertmanager-deploy.yaml
deployment.apps/alertmanager created
```

## 5.4 svc配置清单

```bash
mkdir -p /root/prometheus && cd /root/prometheus
cat > alertmanager-svc.yaml <<"EOF"
apiVersion: v1
kind: Service
metadata:
  name: alertmanager
  namespace: monitor-sa
spec:
  selector:
    app: alertmanager
  ports:
    - port: 80
      targetPort: 9093
EOF
```

创建svc：

```bash
kubectl apply -f alertmanager-svc.yaml
service/alertmanager created
```

## 5.5 配置告警规则

### 5.5.1 规则文件

```bash
mkdir -p /root/prometheus && cd /root/prometheus
cat > alertmanager-rules.yaml <<"EOF"
apiVersion: v1
kind: ConfigMap
metadata:
  name: alertmanager-rules
  namespace: monitor-sa
data:
  rules.yml: |-
    groups:
    - name: hostStatsAlert
      rules:
      - alert: hostCpuUsageAlert
        expr: sum(avg without (cpu)(irate(node_cpu{mode!='idle'}[5m]))) by (instance) > 0.85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "{{ $labels.instance }} CPU usage above 85% (current value: {{ $value }}%)"
      - alert: hostMemUsageAlert
        expr: (node_memory_MemTotal - node_memory_MemAvailable)/node_memory_MemTotal > 0.85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "{{ $labels.instance }} MEM usage above 85% (current value: {{ $value }}%)"
      - alert: OutOfInodes
        expr: node_filesystem_free{fstype="overlay",mountpoint ="/"} / node_filesystem_size{fstype="overlay",mountpoint ="/"} * 100 < 10
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Out of inodes (instance {{ $labels.instance }})"
          description: "Disk is almost running out of available inodes (< 10% left) (current value: {{ $value }})"
      - alert: OutOfDiskSpace
        expr: node_filesystem_free{fstype="overlay",mountpoint ="/rootfs"} / node_filesystem_size{fstype="overlay",mountpoint ="/rootfs"} * 100 < 10
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Out of disk space (instance {{ $labels.instance }})"
          description: "Disk is almost full (< 10% left) (current value: {{ $value }})"
      - alert: UnusualNetworkThroughputIn
        expr: sum by (instance) (irate(node_network_receive_bytes[2m])) / 1024 / 1024 > 100
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Unusual network throughput in (instance {{ $labels.instance }})"
          description: "Host network interfaces are probably receiving too much data (> 100 MB/s) (current value: {{ $value }})"
      - alert: UnusualNetworkThroughputOut
        expr: sum by (instance) (irate(node_network_transmit_bytes[2m])) / 1024 / 1024 > 100
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Unusual network throughput out (instance {{ $labels.instance }})"
          description: "Host network interfaces are probably sending too much data (> 100 MB/s) (current value: {{ $value }})"
      - alert: UnusualDiskReadRate
        expr: sum by (instance) (irate(node_disk_bytes_read[2m])) / 1024 / 1024 > 50
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Unusual disk read rate (instance {{ $labels.instance }})"
          description: "Disk is probably reading too much data (> 50 MB/s) (current value: {{ $value }})"
      - alert: UnusualDiskWriteRate
        expr: sum by (instance) (irate(node_disk_bytes_written[2m])) / 1024 / 1024 > 50
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Unusual disk write rate (instance {{ $labels.instance }})"
          description: "Disk is probably writing too much data (> 50 MB/s) (current value: {{ $value }})"
      - alert: UnusualDiskReadLatency
        expr: rate(node_disk_read_time_ms[1m]) / rate(node_disk_reads_completed[1m]) > 100
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Unusual disk read latency (instance {{ $labels.instance }})"
          description: "Disk latency is growing (read operations > 100ms) (current value: {{ $value }})"
      - alert: UnusualDiskWriteLatency
        expr: rate(node_disk_write_time_ms[1m]) / rate(node_disk_writes_completedl[1m]) > 100
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Unusual disk write latency (instance {{ $labels.instance }})"
          description: "Disk latency is growing (write operations > 100ms) (current value: {{ $value }})"
    - name: http_status
      rules:
      - alert: ProbeFailed
        expr: probe_success == 0
        for: 1m
        labels:
          severity: error
        annotations:
          summary: "Probe failed (instance {{ $labels.instance }})"
          description: "Probe failed (current value: {{ $value }})"
      - alert: StatusCode
        expr: probe_http_status_code <= 199 OR probe_http_status_code >= 400
        for: 1m
        labels:
          severity: error
        annotations:
          summary: "Status Code (instance {{ $labels.instance }})"
          description: "HTTP status code is not 200-399 (current value: {{ $value }})"
      - alert: SslCertificateWillExpireSoon
        expr: probe_ssl_earliest_cert_expiry - time() < 86400 * 30
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "SSL certificate will expire soon (instance {{ $labels.instance }})"
          description: "SSL certificate expires in 30 days (current value: {{ $value }})"
      - alert: SslCertificateHasExpired
        expr: probe_ssl_earliest_cert_expiry - time()  <= 0
        for: 5m
        labels:
          severity: error
        annotations:
          summary: "SSL certificate has expired (instance {{ $labels.instance }})"
          description: "SSL certificate has expired already (current value: {{ $value }})"
      - alert: BlackboxSlowPing
        expr: probe_icmp_duration_seconds > 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Blackbox slow ping (instance {{ $labels.instance }})"
          description: "Blackbox ping took more than 2s (current value: {{ $value }})"
      - alert: BlackboxSlowRequests
        expr: probe_http_duration_seconds > 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Blackbox slow requests (instance {{ $labels.instance }})"
          description: "Blackbox request took more than 2s (current value: {{ $value }})"
      - alert: PodCpuUsagePercent
        expr: sum(sum(label_replace(irate(container_cpu_usage_seconds_total[1m]),"pod","$1","container_label_io_kubernetes_pod_name", "(.*)"))by(pod) / on(pod) group_right kube_pod_container_resource_limits_cpu_cores *100 )by(container,namespace,node,pod,severity) > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Pod cpu usage percent has exceeded 80% (current value: {{ $value }}%)"
EOF
```

创建configmap：

```bash
kubectl apply -f alertmanager-rules.yaml
configmap/alertmanager-rules created
```

### 5.5.2 prometheus配置文件修改

```bash
vim prometheus-deploy.yaml
# 添加rules.yml
    volumeMounts:
    - mountPath: /data/etc/rules.yml
      name: alertmanager-rules
      subPath: rules.yml
  volumes:
    - name: alertmanager-rules
      configMap:
        name: alertmanager-rules
        items:
          - key: rules.yml
            path: rules.yml
            mode: 0644
            
kubectl apply -f prometheus-deploy.yaml
```

在prometheus配置文件中追加配置：

```bash
vim prometheus-cfg.yaml
alerting:
  alertmanagers:
    - static_configs:
        - targets: ["alertmanager"]
rule_files:
     - "/data/etc/rules.yml"

kubectl apply -f  prometheus-cfg.yaml
kubectl apply -f prometheus-deploy.yaml
```

查看收集情况：

![img](https://img2023.cnblogs.com/blog/1740081/202303/1740081-20230315162107265-645242202.png)