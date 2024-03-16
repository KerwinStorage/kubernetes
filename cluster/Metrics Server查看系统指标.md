# 			通过 Metrics Server 查看 Kubernetes 资源指标

# 1.简介

Metrics Server 是一个用于 Kubernetes 集群的监控工具，它用于收集、存储和提供关于集群中各种资源的度量数据。Metrics Server 是 Kubernetes 中一个核心的指标收集器，可以提供关于 CPU 和内存使用情况、节点资源利用率以及其他重要指标的信息。它主要用于水平自动扩展（Horizontal Pod Autoscaling，HPA）和 Kubernetes Dashboard 等 Kubernetes 组件的正常运行。

Metrics Server 通过轮询 Kubernetes API 服务器来获取有关容器、节点和集群级别资源使用情况的数据。然后，它将这些数据存储在内存中，并在请求时返回给用户或其他 Kubernetes 组件。Metrics Server 不存储历史数据，因此它主要用于实时监控和自动化任务。

Metrics Server 的工作原理是通过在每个节点上运行的 kubelet 组件定期收集容器和节点级别的度量数据，并将其暴露给 Metrics Server。Metrics Server 将这些数据聚合并提供给 Kubernetes API 服务器，以便用户可以使用 kubectl 或其他工具查询集群的资源使用情况。

Metrics Server 是 Kubernetes 的一个重要组件，特别是在需要进行自动扩展或监控集群资源使用情况时。它可以帮助管理员和开发人员更好地了解其集群的运行状况，并且可以根据实时数据进行自动化操作。

# 2.helm部署方式

添加 metrics-server 仓库

```shell
helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
helm repo update
```

生成 values.yaml

```shell
helm show values metrics-server/metrics-server > values.yaml
```

修改 values.yaml

```shell
# metrics-server/values.yaml
defaultArgs:
  - --cert-dir=/tmp
  - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
  - --kubelet-use-node-status-port
  - --metric-resolution=15s
  - --kubelet-insecure-tls # 添加这行
```

`--kubelet-insecure-tls` 是 Metrics Server 的一个命令行选项，用于在配置 Metrics Server 时指定。该选项允许 Metrics Server 使用不安全的 TLS 连接来与 kubelet 通信。

在 Kubernetes 中，默认情况下，kubelet 暴露的 API 端点要求客户端使用安全的 TLS 连接进行通信。这是为了确保通信的机密性和完整性。但是，在某些情况下，可能由于测试环境或其他特定的配置要求，管理员可能希望放宽这些安全限制，不建议在生产环境中使用，因为它会降低系统的安全性。

如果想查看将要部署的资源清单，可以执行以下命令

```shell
helm template metrics-server metrics-server/metrics-server -n kube-system -f values.yaml > metrics-server.yaml
```

安装 metrics-server

```shell
helm install metrics-server metrics-server/metrics-server -n kube-system -f values.yaml
```

查看 metrics-server 服务状态

```shell
kubectl get pod -n kube-system | grep metrics-server

# metrics-server-59f6894cb9-lj2lf           1/1     Running   0              52s
```

检查 API Server 是否可以连通 Metrics Server

```shell
kubectl describe svc metrics-server -n kube-system
Name:              metrics-server
Namespace:         kube-system
Labels:            app.kubernetes.io/instance=metrics-server
                   app.kubernetes.io/managed-by=Helm
                   app.kubernetes.io/name=metrics-server
                   app.kubernetes.io/version=0.7.0
                   helm.sh/chart=metrics-server-3.12.0
Annotations:       meta.helm.sh/release-name: metrics-server
                   meta.helm.sh/release-namespace: kube-system
Selector:          app.kubernetes.io/instance=metrics-server,app.kubernetes.io/name=metrics-server
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                10.96.129.181
IPs:               10.96.129.181
Port:              https  443/TCP
TargetPort:        https/TCP
Endpoints:         10.244.85.248:10250
Session Affinity:  None
Events:            <none>
```

## 2.1.查看度量指标

查看node节点cpu和内存使用

```shell
kubectl top nodes
NAME           CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
k8s-master01   348m         17%    4878Mi          62%
k8s-node01     262m         13%    3659Mi          46%
k8s-node02     229m         11%    3579Mi          45%
```

查看default空间下pod的cpu和内存使用

```shell
kubectl top pods
NAME                          CPU(cores)   MEMORY(bytes)
kadalu-csi-nodeplugin-54d7b   4m           73Mi
kadalu-csi-nodeplugin-5d4kf   4m           109Mi
kadalu-csi-nodeplugin-prqg8   4m           73Mi
kadalu-csi-provisioner-0      14m          106Mi
```

附录：

1. metrics-server.yaml

```shell
---
# Source: metrics-server/templates/serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: metrics-server
  namespace: kube-system
  labels:
    helm.sh/chart: metrics-server-3.12.0
    app.kubernetes.io/name: metrics-server
    app.kubernetes.io/instance: metrics-server
    app.kubernetes.io/version: "0.7.0"
    app.kubernetes.io/managed-by: Helm
---
# Source: metrics-server/templates/clusterrole-aggregated-reader.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: system:metrics-server-aggregated-reader
  labels:
    helm.sh/chart: metrics-server-3.12.0
    app.kubernetes.io/name: metrics-server
    app.kubernetes.io/instance: metrics-server
    app.kubernetes.io/version: "0.7.0"
    app.kubernetes.io/managed-by: Helm
    rbac.authorization.k8s.io/aggregate-to-admin: "true"
    rbac.authorization.k8s.io/aggregate-to-edit: "true"
    rbac.authorization.k8s.io/aggregate-to-view: "true"
rules:
  - apiGroups:
      - metrics.k8s.io
    resources:
      - pods
      - nodes
    verbs:
      - get
      - list
      - watch
---
# Source: metrics-server/templates/clusterrole.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: system:metrics-server
  labels:
    helm.sh/chart: metrics-server-3.12.0
    app.kubernetes.io/name: metrics-server
    app.kubernetes.io/instance: metrics-server
    app.kubernetes.io/version: "0.7.0"
    app.kubernetes.io/managed-by: Helm
rules:
  - apiGroups:
    - ""
    resources:
    - nodes/metrics
    verbs:
    - get
  - apiGroups:
    - ""
    resources:
      - pods
      - nodes
      - namespaces
      - configmaps
    verbs:
      - get
      - list
      - watch
---
# Source: metrics-server/templates/clusterrolebinding-auth-delegator.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: metrics-server:system:auth-delegator
  labels:
    helm.sh/chart: metrics-server-3.12.0
    app.kubernetes.io/name: metrics-server
    app.kubernetes.io/instance: metrics-server
    app.kubernetes.io/version: "0.7.0"
    app.kubernetes.io/managed-by: Helm
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:auth-delegator
subjects:
  - kind: ServiceAccount
    name: metrics-server
    namespace: kube-system
---
# Source: metrics-server/templates/clusterrolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:metrics-server
  labels:
    helm.sh/chart: metrics-server-3.12.0
    app.kubernetes.io/name: metrics-server
    app.kubernetes.io/instance: metrics-server
    app.kubernetes.io/version: "0.7.0"
    app.kubernetes.io/managed-by: Helm
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:metrics-server
subjects:
  - kind: ServiceAccount
    name: metrics-server
    namespace: kube-system
---
# Source: metrics-server/templates/rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: metrics-server-auth-reader
  namespace: kube-system
  labels:
    helm.sh/chart: metrics-server-3.12.0
    app.kubernetes.io/name: metrics-server
    app.kubernetes.io/instance: metrics-server
    app.kubernetes.io/version: "0.7.0"
    app.kubernetes.io/managed-by: Helm
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: extension-apiserver-authentication-reader
subjects:
  - kind: ServiceAccount
    name: metrics-server
    namespace: kube-system
---
# Source: metrics-server/templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: metrics-server
  namespace: kube-system
  labels:
    helm.sh/chart: metrics-server-3.12.0
    app.kubernetes.io/name: metrics-server
    app.kubernetes.io/instance: metrics-server
    app.kubernetes.io/version: "0.7.0"
    app.kubernetes.io/managed-by: Helm
spec:
  type: ClusterIP
  ports:
    - name: https
      port: 443
      protocol: TCP
      targetPort: https
  selector:
    app.kubernetes.io/name: metrics-server
    app.kubernetes.io/instance: metrics-server
---
# Source: metrics-server/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: metrics-server
  namespace: kube-system
  labels:
    helm.sh/chart: metrics-server-3.12.0
    app.kubernetes.io/name: metrics-server
    app.kubernetes.io/instance: metrics-server
    app.kubernetes.io/version: "0.7.0"
    app.kubernetes.io/managed-by: Helm
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: metrics-server
      app.kubernetes.io/instance: metrics-server
  template:
    metadata:
      labels:
        app.kubernetes.io/name: metrics-server
        app.kubernetes.io/instance: metrics-server
    spec:
      schedulerName:
      serviceAccountName: metrics-server
      priorityClassName: "system-cluster-critical"
      containers:
        - name: metrics-server
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop:
              - ALL
            readOnlyRootFilesystem: true
            runAsNonRoot: true
            runAsUser: 1000
            seccompProfile:
              type: RuntimeDefault
          image: registry.k8s.io/metrics-server/metrics-server:v0.7.0
          imagePullPolicy: IfNotPresent
          args:
            - --secure-port=10250
            - --cert-dir=/tmp
            - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
            - --kubelet-use-node-status-port
            - --metric-resolution=15s
            - --kubelet-insecure-tls
          ports:
          - name: https
            protocol: TCP
            containerPort: 10250
          livenessProbe:
            failureThreshold: 3
            httpGet:
              path: /livez
              port: https
              scheme: HTTPS
            initialDelaySeconds: 0
            periodSeconds: 10
          readinessProbe:
            failureThreshold: 3
            httpGet:
              path: /readyz
              port: https
              scheme: HTTPS
            initialDelaySeconds: 20
            periodSeconds: 10
          volumeMounts:
            - name: tmp
              mountPath: /tmp
          resources:
            requests:
              cpu: 100m
              memory: 200Mi
      volumes:
        - name: tmp
          emptyDir: {}
---
# Source: metrics-server/templates/apiservice.yaml
apiVersion: apiregistration.k8s.io/v1
kind: APIService
metadata:
  name: v1beta1.metrics.k8s.io
  labels:
    helm.sh/chart: metrics-server-3.12.0
    app.kubernetes.io/name: metrics-server
    app.kubernetes.io/instance: metrics-server
    app.kubernetes.io/version: "0.7.0"
    app.kubernetes.io/managed-by: Helm
spec:
  group: metrics.k8s.io
  groupPriorityMinimum: 100
  insecureSkipTLSVerify: true
  service:
    name: metrics-server
    namespace: kube-system
    port: 443
  version: v1beta1
  versionPriority: 100
```

2. values.yaml文件内容

```sh
# Default values for metrics-server.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.
image:
  repository: registry.k8s.io/metrics-server/metrics-server
  # Overrides the image tag whose default is v{{ .Chart.AppVersion }}
  tag: ""
  pullPolicy: IfNotPresent
imagePullSecrets: []
# - name: registrySecretName
nameOverride: ""
fullnameOverride: ""
serviceAccount:
  # Specifies whether a service account should be created
  create: true
  # Annotations to add to the service account
  annotations: {}
  # The name of the service account to use.
  # If not set and create is true, a name is generated using the fullname template
  name: ""
  # The list of secrets mountable by this service account.
  # See https://kubernetes.io/docs/reference/labels-annotations-taints/#enforce-mountable-secrets
  secrets: []
rbac:
  # Specifies whether RBAC resources should be created
  create: true
  pspEnabled: false
apiService:
  # Specifies if the v1beta1.metrics.k8s.io API service should be created.
  #
  # You typically want this enabled! If you disable API service creation you have to
  # manage it outside of this chart for e.g horizontal pod autoscaling to
  # work with this release.
  create: true
  # Annotations to add to the API service
  annotations: {}
  # Specifies whether to skip TLS verification
  insecureSkipTLSVerify: true
  # The PEM encoded CA bundle for TLS verification
  caBundle: ""
commonLabels: {}
podLabels: {}
podAnnotations: {}
podSecurityContext: {}
securityContext:
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  runAsNonRoot: true
  runAsUser: 1000
  seccompProfile:
    type: RuntimeDefault
  capabilities:
    drop:
      - ALL
priorityClassName: system-cluster-critical
containerPort: 10250
hostNetwork:
  # Specifies if metrics-server should be started in hostNetwork mode.
  #
  # You would require this enabled if you use alternate overlay networking for pods and
  # API server unable to communicate with metrics-server. As an example, this is required
  # if you use Weave network on EKS
  enabled: false
replicas: 1
revisionHistoryLimit:
updateStrategy: {}
#   type: RollingUpdate
#   rollingUpdate:
#     maxSurge: 0
#     maxUnavailable: 1
podDisruptionBudget:
  # https://kubernetes.io/docs/tasks/run-application/configure-pdb/
  enabled: false
  minAvailable:
  maxUnavailable:
defaultArgs:
  - --cert-dir=/tmp
  - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
  - --kubelet-use-node-status-port
  - --metric-resolution=15s
  - --kubelet-insecure-tls
args: []
livenessProbe:
  httpGet:
    path: /livez
    port: https
    scheme: HTTPS
  initialDelaySeconds: 0
  periodSeconds: 10
  failureThreshold: 3
readinessProbe:
  httpGet:
    path: /readyz
    port: https
    scheme: HTTPS
  initialDelaySeconds: 20
  periodSeconds: 10
  failureThreshold: 3
service:
  type: ClusterIP
  port: 443
  annotations: {}
  labels: {}
  #  Add these labels to have metrics-server show up in `kubectl cluster-info`
  #  kubernetes.io/cluster-service: "true"
  #  kubernetes.io/name: "Metrics-server"
addonResizer:
  enabled: false
  image:
    repository: registry.k8s.io/autoscaling/addon-resizer
    tag: 1.8.20
  securityContext:
    allowPrivilegeEscalation: false
    readOnlyRootFilesystem: true
    runAsNonRoot: true
    runAsUser: 1000
    seccompProfile:
      type: RuntimeDefault
    capabilities:
      drop:
        - ALL
  resources:
    requests:
      cpu: 40m
      memory: 25Mi
    limits:
      cpu: 40m
      memory: 25Mi
  nanny:
    cpu: 0m
    extraCpu: 1m
    memory: 0Mi
    extraMemory: 2Mi
    minClusterSize: 100
    pollPeriod: 300000
    threshold: 5
metrics:
  enabled: false
serviceMonitor:
  enabled: false
  additionalLabels: {}
  interval: 1m
  scrapeTimeout: 10s
  metricRelabelings: []
  relabelings: []
# See https://github.com/kubernetes-sigs/metrics-server#scaling
resources:
  requests:
    cpu: 100m
    memory: 200Mi
  # limits:
  #   cpu:
  #   memory:
extraVolumeMounts: []
extraVolumes: []
nodeSelector: {}
tolerations: []
affinity: {}
topologySpreadConstraints: []
dnsConfig: {}
# Annotations to add to the deployment
deploymentAnnotations: {}
schedulerName: ""
tmpVolume:
  emptyDir: {}
```

参考：

- [资源指标管道](https://kubernetes.io/zh-cn/docs/tasks/debug/debug-cluster/resource-metrics-pipeline)
- [artifacthub.io](https://artifacthub.io/packages/helm/metrics-server/metrics-server)
- [Metrics Server github](https://github.com/kubernetes-sigs/metrics-server)
- [kubernetes-sigs](https://kubernetes-sigs.github.io/metrics-server)
- [部署参考](https://todoit.tech/k8s/metrics-server)