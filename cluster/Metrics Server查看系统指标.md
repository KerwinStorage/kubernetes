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

参考：

- [资源指标管道](https://kubernetes.io/zh-cn/docs/tasks/debug/debug-cluster/resource-metrics-pipeline)
- [artifacthub.io](https://artifacthub.io/packages/helm/metrics-server/metrics-server)
- [Metrics Server github](https://github.com/kubernetes-sigs/metrics-server)
- [kubernetes-sigs](https://kubernetes-sigs.github.io/metrics-server)
- [部署参考](https://todoit.tech/k8s/metrics-server)