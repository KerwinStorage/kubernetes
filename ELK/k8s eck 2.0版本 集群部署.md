## 一、部署ECK

[2.0官网](https://www.elastic.co/guide/en/cloud-on-k8s/2.0/k8s-deploy-eck.html)

```bash
kubectl create -f https://download.elastic.co/downloads/eck/2.0.0/crds.yaml
kubectl apply -f https://download.elastic.co/downloads/eck/2.0.0/operator.yaml
kubectl -n elastic-system logs -f statefulset.apps/elastic-operator
```

## 二、部署Elasticsearch7.6.2集群

下载压缩包，下面的文件内容就是里面的

```bash
curl -LO https://github.com/elastic/cloud-on-k8s/archive/1.1.0.tar.gz
tar zxvf 1.1.0.tar.gz
cd cloud-on-k8s-1.1.0/config/recipes/beats
```

**创建命名空间：**

```bash
kubectl create namespace beats
```

**创建elasticsearch：**

```bash
cat > 1_monitor.yaml <<EOF
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: monitor
  namespace: beats
spec:
  version: 7.6.2
  nodeSets:
  - name: mdi
    count: 3
    config:
      node.master: true
      node.data: true
      node.ingest: true
      node.store.allow_mmap: false
    volumeClaimTemplates:
    - metadata:
        name: elasticsearch-data
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 50Gi
        storageClassName: nfs-storage
  http:
    service:
      spec:
        type: NodePort
---
apiVersion: kibana.k8s.elastic.co/v1
kind: Kibana
metadata:
  name: monitor
  namespace: beats
spec:
  version: 7.6.2
  count: 1
  elasticsearchRef:
    name: "monitor"
  http:
    service:
      spec:
        type: NodePort
EOF
```

**查看启动情况：**

```bash
kubectl get pod -n beats
NAME                                     READY   STATUS    RESTARTS   AGE
monitor-es-mdi-0                         1/1     Running   0          3m5s
monitor-es-mdi-1                         1/1     Running   0          3m5s
monitor-es-mdi-2                         1/1     Running   0          3m5s
monitor-kb-84bfc69db5-dkwm7              1/1     Running   0          3m3s

kubectl get crd  -n beats | grep elastic
agents.agent.k8s.elastic.co                           2022-11-24T11:37:25Z
apmservers.apm.k8s.elastic.co                         2022-11-24T11:37:25Z
beats.beat.k8s.elastic.co                             2022-11-24T11:37:25Z
elasticmapsservers.maps.k8s.elastic.co                2022-11-24T11:37:25Z
elasticsearches.elasticsearch.k8s.elastic.co          2022-11-24T11:37:25Z
enterprisesearches.enterprisesearch.k8s.elastic.co    2022-11-24T11:37:25Z
kibanas.kibana.k8s.elastic.co                         2022-11-24T11:37:25Z
```

**查看创建的Elasticsearch和kibana资源，包括运行状况，版本和节点数：**

```bash
kubectl get elasticsearch -n beats
NAME      HEALTH   NODES   VERSION   PHASE   AGE
monitor   green    3       7.6.2     Ready   5m51s

kubectl get kibana -n beats
NAME      HEALTH   NODES   VERSION   AGE
monitor   green    1       7.6.2     6m8s
```

**查看创建的pv和pvc：**

```bash
kubectl get pv,pvc -n beats
NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                         STORAGECLASS   REASON   AGE
persistentvolume/pvc-9a1710d9-b839-4dcd-805e-cea77d97dd33   50Gi       RWO            Delete           Bound    default/elasticsearch-data-monitor-es-mdi-1   nfs-storage             7m2s
persistentvolume/pvc-b72c2621-ec3b-4a14-9755-e37dc1e1bb09   50Gi       RWO            Delete           Bound    default/elasticsearch-data-monitor-es-mdi-2   nfs-storage             7m2s
persistentvolume/pvc-cd1285c3-6c81-4ec4-9b2e-b938a8eae96e   50Gi       RWO            Delete           Bound    default/elasticsearch-data-monitor-es-mdi-0   nfs-storage             7m2s

NAME                                                        STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/elasticsearch-data-monitor-es-mdi-0   Bound    pvc-cd1285c3-6c81-4ec4-9b2e-b938a8eae96e   50Gi       RWO            nfs-storage    7m2s
persistentvolumeclaim/elasticsearch-data-monitor-es-mdi-1   Bound    pvc-9a1710d9-b839-4dcd-805e-cea77d97dd33   50Gi       RWO            nfs-storage    7m2s
persistentvolumeclaim/elasticsearch-data-monitor-es-mdi-2   Bound    pvc-b72c2621-ec3b-4a14-9755-e37dc1e1bb09   50Gi       RWO            nfs-storage    7m2s
```

**查看创建的service，部署时已经将es和kibana服务类型改为NodePort，方便从集群外访问。**

```bash
kubectl get svc -n beats
NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
kubernetes             ClusterIP   10.254.0.1      <none>        443/TCP          29d
monitor-es-http        NodePort    10.254.52.175   <none>        9200:31553/TCP   8m5s #es端口
monitor-es-mdi         ClusterIP   None            <none>        9200/TCP         8m2s
monitor-es-transport   ClusterIP   None            <none>        9300/TCP         8m5s
monitor-kb-http        NodePort    10.254.212.41   <none>        5601:32705/TCP   8m4s #kibanna端口
```

登录es地址：https://192.168.80.45:31553

默认elasticsearch启用了验证，获取elastic user的密码：

```bash
PASSWORD=$(kubectl get secret -n beats monitor-es-elastic-user -o=jsonpath='{.data.elastic}' | base64 --decode)
echo $PASSWORD
```

用户名：elastic 密码：Y41Z5M06I3d7rGVj58SLsD5z

登录kibanna：[https://192.168.80.45:32705](https://192.168.80.45:32705/)

注：密码跟上面一样，初次登录页面选择`Explore on my own`

## 三、部署filebeat

```bash
vim 2_filebeat-kubernetes.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: filebeat-config
  namespace: beats
  labels:
    k8s-app: filebeat
data:
  filebeat.yml: |-
    filebeat.autodiscover:
      providers:
        - type: kubernetes
          host: ${NODE_NAME}
          hints.enabled: true
          hints.default_config:
            type: container
            paths:
              - /var/log/containers/*${data.kubernetes.container.id}.log

    processors:
      - add_cloud_metadata:
      - add_host_metadata:

    output.elasticsearch:
      hosts: ['https://${ELASTICSEARCH_HOST:elasticsearch}:${ELASTICSEARCH_PORT:9200}']
      username: ${ELASTICSEARCH_USERNAME}
      password: ${ELASTICSEARCH_PASSWORD}
      ssl.certificate_authorities:
      - /mnt/elastic/tls.crt
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: filebeat
  namespace: beats
  labels:
    k8s-app: filebeat
spec:
  selector:
    matchLabels:
      k8s-app: filebeat
  template:
    metadata:
      labels:
        k8s-app: filebeat
    spec:
      serviceAccountName: filebeat
      terminationGracePeriodSeconds: 30
      hostNetwork: true
      dnsPolicy: ClusterFirstWithHostNet
      containers:
      - name: filebeat
        image: docker.elastic.co/beats/filebeat:7.6.2
        args: [
          "-c", "/etc/filebeat.yml",
          "-e",
        ]
        env:
        - name: ELASTICSEARCH_HOST
          value: monitor-es-http
        - name: ELASTICSEARCH_PORT
          value: "9200"
        - name: ELASTICSEARCH_USERNAME
          value: elastic
        - name: ELASTICSEARCH_PASSWORD
          valueFrom:
            secretKeyRef:
              key: elastic
              name: monitor-es-elastic-user
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        securityContext:
          runAsUser: 0
          # If using Red Hat OpenShift uncomment this:
          #privileged: true
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 100Mi
        volumeMounts:
        - name: config
          mountPath: /etc/filebeat.yml
          readOnly: true
          subPath: filebeat.yml
        - name: data
          mountPath: /usr/share/filebeat/data
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
        - name: varlog
          mountPath: /var/log
          readOnly: true
        - name: es-certs
          mountPath: /mnt/elastic/tls.crt
          readOnly: true
          subPath: tls.crt
      volumes:
      - name: config
        configMap:
          defaultMode: 0600
          name: filebeat-config
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
      - name: varlog
        hostPath:
          path: /var/log
      # data folder stores a registry of read status for all files, so we don't send everything again on a Filebeat pod restart
      - name: data
        hostPath:
          path: /var/lib/filebeat-data
          type: DirectoryOrCreate
      - name: es-certs
        secret:
          secretName: monitor-es-http-certs-public
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: filebeat
subjects:
- kind: ServiceAccount
  name: filebeat
  namespace: beats
roleRef:
  kind: ClusterRole
  name: filebeat
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: filebeat
  labels:
    k8s-app: filebeat
rules:
- apiGroups: [""] # "" indicates the core API group
  resources:
  - namespaces
  - pods
  verbs:
  - get
  - watch
  - list
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: filebeat
  namespace: beats
  labels:
    k8s-app: filebeat
```

**启动pod：** 

```bash
kubectl apply -f 2_filebeat-kubernetes.yaml
# 查看情况
kubectl -n beats get pods -l k8s-app=filebeat
NAME             READY   STATUS    RESTARTS   AGE
filebeat-ckghl   1/1     Running   0          62m
filebeat-f6tnr   1/1     Running   0          62m
filebeat-pqqkw   1/1     Running   0          62m
```

 

**![img](https://img2022.cnblogs.com/blog/1740081/202211/1740081-20221116232452945-867102842.png)**

**点击`Create index pattern`。**

**![img](https://img2022.cnblogs.com/blog/1740081/202211/1740081-20221116232525502-266297784.png)**

## 四、部署metricbeat

```bash
vim 3_metricbeat-kubernetes.yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: metricbeat-daemonset-config
  namespace: beats
  labels:
    k8s-app: metricbeat
data:
  metricbeat.yml: |-
    metricbeat.config.modules:
      # Mounted `metricbeat-daemonset-modules` configmap:
      path: ${path.config}/modules.d/*.yml
      # Reload module configs as they change:
      reload.enabled: false

    # To enable hints based autodiscover uncomment this:
    metricbeat.autodiscover:
      providers:
        - type: kubernetes
          host: ${NODE_NAME}
          hints.enabled: true

    processors:
      - add_cloud_metadata:

    output.elasticsearch:
      hosts: ['https://${ELASTICSEARCH_HOST:elasticsearch}:${ELASTICSEARCH_PORT:9200}']
      username: ${ELASTICSEARCH_USERNAME}
      password: ${ELASTICSEARCH_PASSWORD}
      ssl.certificate_authorities:
      - /mnt/elastic/tls.crt
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: metricbeat-daemonset-modules
  namespace: beats
  labels:
    k8s-app: metricbeat
data:
  system.yml: |-
    - module: system
      period: 10s
      metricsets:
        - cpu
        - load
        - memory
        - network
        - process
        - process_summary
        #- core
        #- diskio
        #- socket
      processes: ['.*']
      process.include_top_n:
        by_cpu: 5      # include top 5 processes by CPU
        by_memory: 5   # include top 5 processes by memory

    - module: system
      period: 1m
      metricsets:
        - filesystem
        - fsstat
      processors:
      - drop_event.when.regexp:
          system.filesystem.mount_point: '^/(sys|cgroup|proc|dev|etc|host|lib)($|/)'
  kubernetes.yml: |-
    - module: kubernetes
      metricsets:
        - node
        - system
        - pod
        - container
        - volume
      period: 10s
      host: ${NODE_NAME}
      hosts: ["https://${HOSTNAME}:10250"]
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      ssl.verification_mode: "none"
      # If using Red Hat OpenShift remove ssl.verification_mode entry and
      # uncomment these settings:
      #ssl.certificate_authorities:
        #- /var/run/secrets/kubernetes.io/serviceaccount/service-ca.crt
    - module: kubernetes
      metricsets:
        - proxy
      period: 10s
      host: ${NODE_NAME}
      hosts: ["localhost:10249"]
---
# Deploy a Metricbeat instance per node for node metrics retrieval
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: metricbeat
  namespace: beats
  labels:
    k8s-app: metricbeat
spec:
  selector:
    matchLabels:
      k8s-app: metricbeat
  template:
    metadata:
      labels:
        k8s-app: metricbeat
    spec:
      serviceAccountName: metricbeat
      terminationGracePeriodSeconds: 30
      hostNetwork: true
      dnsPolicy: ClusterFirstWithHostNet
      containers:
      - name: metricbeat
        image: docker.elastic.co/beats/metricbeat:7.6.2
        args: [
          "-c", "/etc/metricbeat.yml",
          "-e",
          "-system.hostfs=/hostfs",
          "-d", "autodiscover",
          "-d", "kubernetes",
        ]
        env:
        - name: ELASTICSEARCH_HOST
          value: monitor-es-http
        - name: ELASTICSEARCH_PORT
          value: "9200"
        - name: ELASTICSEARCH_USERNAME
          value: elastic
        - name: ELASTICSEARCH_PASSWORD
          valueFrom:
            secretKeyRef:
              key: elastic
              name: monitor-es-elastic-user
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        securityContext:
          runAsUser: 0
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 100Mi
        volumeMounts:
        - name: config
          mountPath: /etc/metricbeat.yml
          readOnly: true
          subPath: metricbeat.yml
        - name: modules
          mountPath: /usr/share/metricbeat/modules.d
          readOnly: true
        - name: dockersock
          mountPath: /var/run/docker.sock
        - name: proc
          mountPath: /hostfs/proc
          readOnly: true
        - name: cgroup
          mountPath: /hostfs/sys/fs/cgroup
          readOnly: true
        - name: es-certs
          mountPath: /mnt/elastic/tls.crt
          readOnly: true
          subPath: tls.crt
      volumes:
      - name: proc
        hostPath:
          path: /proc
      - name: cgroup
        hostPath:
          path: /sys/fs/cgroup
      - name: dockersock
        hostPath:
          path: /var/run/docker.sock
      - name: config
        configMap:
          defaultMode: 0600
          name: metricbeat-daemonset-config
      - name: modules
        configMap:
          defaultMode: 0600
          name: metricbeat-daemonset-modules
      - name: data
        hostPath:
          path: /var/lib/metricbeat-data
          type: DirectoryOrCreate
      - name: es-certs
        secret:
          secretName: monitor-es-http-certs-public
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: metricbeat-deployment-config
  namespace: beats
  labels:
    k8s-app: metricbeat
data:
  metricbeat.yml: |-
    metricbeat.config.modules:
      # Mounted `metricbeat-daemonset-modules` configmap:
      path: ${path.config}/modules.d/*.yml
      # Reload module configs as they change:
      reload.enabled: false

    processors:
      - add_cloud_metadata:

    setup.dashboards.enabled: true

    setup.kibana:
      host: "https://${KIBANA_HOST:kibana}:${KIBANA_PORT:5601}"
      ssl.enabled: true
      ssl.certificate_authorities:
      - /mnt/kibana/ca.crt

    output.elasticsearch:
      hosts: ['https://${ELASTICSEARCH_HOST:elasticsearch}:${ELASTICSEARCH_PORT:9200}']
      username: ${ELASTICSEARCH_USERNAME}
      password: ${ELASTICSEARCH_PASSWORD}
      ssl.certificate_authorities:
      - /mnt/elastic/tls.crt

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: metricbeat-deployment-modules
  namespace: beats
  labels:
    k8s-app: metricbeat
data:
  # This module requires `kube-state-metrics` up and running under `kube-system` namespace
  kubernetes.yml: |-
    - module: kubernetes
      metricsets:
        - state_node
        - state_deployment
        - state_replicaset
        - state_pod
        - state_container
        # Uncomment this to get k8s events:
        #- event
      period: 10s
      host: ${NODE_NAME}
      hosts: ["kube-state-metrics.kube-system:8080"]
---
# Deploy singleton instance in the whole cluster for some unique data sources, like kube-state-metrics
apiVersion: apps/v1
kind: Deployment
metadata:
  name: metricbeat
  namespace: beats
  labels:
    k8s-app: metricbeat
spec:
  selector:
    matchLabels:
      k8s-app: metricbeat
  template:
    metadata:
      labels:
        k8s-app: metricbeat
    spec:
      serviceAccountName: metricbeat
      hostNetwork: true
      dnsPolicy: ClusterFirstWithHostNet
      containers:
      - name: metricbeat
        image: docker.elastic.co/beats/metricbeat:7.6.2
        args: [
          "-c", "/etc/metricbeat.yml",
          "-e",
          "-d", "autodiscover",
        ]
        env:
        - name: ELASTICSEARCH_HOST
          value: monitor-es-http
        - name: ELASTICSEARCH_PORT
          value: "9200"
        - name: ELASTICSEARCH_USERNAME
          value: elastic
        - name: ELASTICSEARCH_PASSWORD
          valueFrom:
            secretKeyRef:
              key: elastic
              name: monitor-es-elastic-user
        - name: KIBANA_HOST
          value: monitor-kb-http
        - name: KIBANA_PORT
          value: "5601"
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        securityContext:
          runAsUser: 0
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 100Mi
        volumeMounts:
        - name: config
          mountPath: /etc/metricbeat.yml
          readOnly: true
          subPath: metricbeat.yml
        - name: modules
          mountPath: /usr/share/metricbeat/modules.d
          readOnly: true
        - name: es-certs
          mountPath: /mnt/elastic/tls.crt
          readOnly: true
          subPath: tls.crt
        - name: kb-certs
          mountPath:  /mnt/kibana/ca.crt
          readOnly: true
          subPath: ca.crt
      volumes:
      - name: config
        configMap:
          defaultMode: 0600
          name: metricbeat-deployment-config
      - name: modules
        configMap:
          defaultMode: 0600
          name: metricbeat-deployment-modules
      - name: es-certs
        secret:
          secretName: monitor-es-http-certs-public
      - name: kb-certs
        secret:
          secretName: monitor-kb-http-certs-public
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: metricbeat
subjects:
- kind: ServiceAccount
  name: metricbeat
  namespace: beats
roleRef:
  kind: ClusterRole
  name: metricbeat
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: metricbeat
  labels:
    k8s-app: metricbeat
rules:
- apiGroups: [""]
  resources:
  - nodes
  - namespaces
  - events
  - pods
  verbs: ["get", "list", "watch"]
- apiGroups: ["extensions"]
  resources:
  - replicasets
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources:
  - statefulsets
  - deployments
  - replicasets
  verbs: ["get", "list", "watch"]
- apiGroups:
  - ""
  resources:
  - nodes/stats
  verbs:
  - get
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: metricbeat
  namespace: beats
  labels:
    k8s-app: metricbeat
---
```

**启动pod：**

```bash
kubectl apply -f 3_metricbeat-kubernetes.yaml
kubectl -n beats get pods -l k8s-app=metricbeat
NAME                          READY   STATUS    RESTARTS   AGE
metricbeat-6f859c8b54-kl2zn   1/1     Running   0          70m
metricbeat-wgcpb              1/1     Running   0          70m
metricbeat-x4t4g              1/1     Running   0          70m
metricbeat-x8w6f              1/1     Running   0          70m
```

总体看：

```bash
kubectl -n beats get pod
NAME                          READY   STATUS    RESTARTS   AGE
filebeat-bzxss                1/1     Running   0          3m36s
filebeat-f6tnr                1/1     Running   0          74m
filebeat-pqqkw                1/1     Running   0          74m
metricbeat-6f859c8b54-kl2zn   1/1     Running   0          70m
metricbeat-wgcpb              1/1     Running   0          70m
metricbeat-x4t4g              1/1     Running   0          70m
metricbeat-x8w6f              1/1     Running   0          70m
monitor-es-mdi-0              1/1     Running   0          60m
monitor-es-mdi-1              1/1     Running   0          60m
monitor-es-mdi-2              1/1     Running   0          60m
monitor-kb-7c74b7887d-82hdx   1/1     Running   0          60m
```

 