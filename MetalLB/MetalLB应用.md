# 							MetalLB的应用

# 1.概述

MetalLB 是一个用于 Kubernetes 的负载均衡器实现，使得你可以在没有外部负载均衡器硬件（例如，在裸机集群上）的情况下使用标准的负载均衡服务。它为你的 Kubernetes 集群提供了一种创建在本地网络上可访问的 LoadBalancer 类型的服务的方法。

MetalLB 主要有两种工作模式：

1. Layer 2 模式：在这个模式下，MetalLB 响应 ARP 请求，为服务分配的 IP 地址映射到集群中的一个节点。当服务的 IP 地址收到流量时，这个节点就会接收流量并对其进行路由到正确的服务。
2. BGP 模式：在 BGP（Border Gateway Protocol）模式下，MetalLB 与集群外部的路由器通过 BGP 协议对话，动态地宣告服务 IP 地址的位置。这使得流量可以直接路由到提供服务的节点，而不是通过一个中间节点。

- Repo: https://github.com/metallb/metallb
- 官网: https://metallb.universe.tf/installation

## 1.1.限制条件

安装 MetalLB 很简单，不过还是有一些限制：

- 1）需要 Kubernetes v1.13.0 或者更新的版本
- 2）集群中的 CNI 要能兼容 MetalLB，具体兼容性参考这里[network-addons](https://metallb.universe.tf/installation/network-addons/)（本地使用Calico）
  - 像常见的 Flannel、Cilium 等都是兼容的，Calico 的话大部分情况都兼容，BGP 模式下需要额外处理
- 3）提供一下 IPv4 地址给 MetalLB 用于分配
  - 一般在内网使用，提供同一网段的地址即可。
- 4）BGP 模式下需要路由器支持 BGP
- 5）L2 模式下需要各个节点间 7946 端口联通

看起来限制比较多，实际上这些都比较容器满足，除了第四条。

**注：<font color=red>因为 BGP 对路由器有要求，因此建议测试时使用 Layer2 模式</font>** 。

# 2.安装 MetalLB

部署 MetalLB 到 Kubernetes 集群的大致步骤如下：

## 2.1.修改kube-proxy

如果 kube-proxy 使用的是 ipvs 模式，需要修改 kube-proxy 配置文件，启用严格的 ARP

```shell
# https://metallb.universe.tf/installation/#preparation
kubectl edit configmap -n kube-system kube-proxy
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: "ipvs"
ipvs:
  strictARP: true
```

## 2.2.使用 yaml 安装

首先，你需要应用 MetalLB 的清单来安装它。你可以用 kubectl 应用其官方清单文件：

```shell
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.3/config/manifests/metallb-native.yaml
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.3/config/manifests/metallb-frr.yaml
```

这些命令会创建一个名为 metallb-system 的命名空间，并在这个命名空间中部署 MetalLB 所需的组件。

查看资源创建：

```shell
kubectl get  all -n  metallb-system
NAME                              READY   STATUS    RESTARTS   AGE
pod/controller-6dbc8c8797-4xdcd   1/1     Running   0          4m15s
pod/speaker-2g58x                 4/4     Running   0          3m46s
pod/speaker-697n6                 4/4     Running   0          3m54s
pod/speaker-x5n2g                 4/4     Running   0          3m51s

NAME                      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/webhook-service   ClusterIP   10.102.86.213   <none>        443/TCP   5m56s

NAME                     DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
daemonset.apps/speaker   3         3         3       3            3           kubernetes.io/os=linux   5m56s

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/controller   1/1     1            1           5m56s

NAME                                    DESIRED   CURRENT   READY   AGE
replicaset.apps/controller-6dbc8c8797   1         1         1       4m15s
replicaset.apps/controller-7d678cf54    0         0         0       5m56s
```

## 2.3.Layer 2 模式配置

### 2.3.1创建 IPAdressPool

这个 IPAddressPool 指定了 MetalLB 控制的 IP 地址池。在这个例子中，我们指定了 192.168.1.240 到 192.168.1.255 的地址范围。你需要根据自己的网络环境调整这些值。

```shell
mkdir -p ~/metalLb  && cd ~/metalLb
cat <<EOF > IPAddressPool.yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  # 可分配的 IP 地址,可以指定多个，包括 ipv4、ipv6
  - 192.168.1.240/28
EOF
kubectl apply -f IPAddressPool.yaml
```

### 2.3.2创建 L2Advertisement，并关联 IPAdressPool

如果不设置关联到 IPAdressPool，那默认 L2Advertisement 会关联上所有可用的 IPAdressPool

```shell
cat <<EOF > L2Advertisement.yaml
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: example
  namespace: metallb-system
spec:
  ipAddressPools:
  - first-pool #上一步创建的 ip 地址池，通过名字进行关联
EOF

kubectl apply -f L2Advertisement.yaml
```

# 3.测试

创建一个 nginx deploy 以及一个 loadbalance 类型的 svc 来测试。

使用以下命令创建 nginx deploy：

```shell
cat <<EOF > nginx-dp.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: docker.io/nginx:latest
        ports:
        - containerPort: 80
EOF

kubectl apply -f nginx-dp.yaml
```

使用以下命令创建 nginx-svc：

```shell
cat <<EOF > nginx-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx2
  labels:
    app: nginx
spec:
  selector:
    app: nginx
  ports:
  - name: nginx-port
    protocol: TCP
    port: 80
    targetPort: 80
  type: LoadBalancer
EOF

kubectl apply -f nginx-svc.yaml
```

然后查看 svc，看看是不是真的分配了 ExternalIP

```shell
kubectl get svc nginx2
NAME     TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)        AGE
nginx2   LoadBalancer   10.111.75.160   192.168.1.240   80:32083/TCP   51s

telnet 192.168.1.240 80
Trying 192.168.1.240...
Connected to 192.168.1.240.
Escape character is '^]'.
```

当你创建这样的服务时，MetalLB 会为该服务分配一个外部 IP 地址，这个地址来自你在 IPAddressPool 中定义的地址池。

**注意事项：**

- MetalLB 的 Layer 2 模式不能跨多个物理网络传递流量。
- MetalLB 不会在主节点上分配 IP 地址，除非你特别配置它这样做。
- 确保你的网络环境和安全策略允许使用 MetalLB 分配的 IP 地址。
- MetalLB 需要对应的网络权限来广播信息，所以需要确保在你的集群网络策略中合理配置。

在部署和配置 MetalLB 时，请参考官方文档以获得最新的安装步骤和最佳实践。

参考文档：

- https://www.lixueduan.com/posts/cloudnative/01-metallb
- https://metallb.universe.tf/installation