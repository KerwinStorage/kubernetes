# 							kube-vip的实例应用

# 1. 简介

`kube-vip` 是一个用于 Kubernetes 集群的轻量级负载均衡器和虚拟 IP 解决方案，主要用于为 Kubernetes 控制平面和服务提供高可用性和负载均衡功能。它可以在 Kubernetes 集群中用作多种角色，包括作为 Kubernetes 控制平面节点的静态端点或者为 Kubernetes 服务提供负载均衡。

## 1.1. kube-vip的主要特点和功能

1. **高可用性**：kube-vip 可以通过虚拟 IP 和多个节点之间的故障转移来实现 Kubernetes API服务器的高可用性。

2. **负载均衡**：它还可以作为一个内置的负载均衡器，用于分发到 Kubernetes 服务和 Pod 的流量，从而平衡负载和提高性能。

3. **简单实用**：kube-vip 提供了简化的配置和部署流程，可以很容易地集成到现有的 Kubernetes 集群中。

4. **没有外部依赖**：kube-vip 不依赖于外部负载均衡器或额外的硬件，因此适合在裸机、本地开发环境、云环境或边缘计算场景中使用。

5. **支持多种环境**：可以在多种环境中使用，包括本地部署、边缘计算和云平台。

6. **使用 ARP 或 BGP 协议**：kube-vip 可以使用 ARP（地址解析协议）或 BGP（边界网关协议）广播虚拟 IP 地址。

7. **与Kubernetes集成**：尽管是一个独立的组件，kube-vip 仍然和 Kubernetes 控制平面以及服务高度集成，可以作为静态 Pod 或者 DaemonSet 运行在 Kubernetes 集群中。

## 1.2. 如何使用 kube-vip

kube-vip 的部署和配置通常涉及几个步，包括创建配置文件、部署 kube-vip 作为静态 Pod 或 DaemonSet，以及配置虚拟 IP 和选举机制。kube-vip 也可以集成到 Kubernetes 集群安装工具中，比如`kubeadm`。

官方文档和 GitHub 仓库提供了详细的部署指导，地址为：https://github.com/kube-vip/kube-vip

在部署`kube-vip`之前，建议详细阅读官方文档和安装指南，确保正确理解如何部署和配置它以及如何解读提供的数据。此外，了解你的环境和需求来选择最合适的部署方式和配置选项。

# 2.部署使用

链接地址：

- [Github地址](https://github.com/kubernetes/kubeadm/blob/main/docs/ha-considerations.md)
- [Daemonset部署地址](https://kube-vip.io/docs/installation/daemonset)

## 2.1. 方式一-命令启动

```shell
# 设置要用于控制平面的地址：VIP
export VIP=192.168.80.100
# 将名称设置为控制平面上将宣布 VIP 的接口的名称。
export INTERFACE=ens33
# 通过解析 GitHub API 获取最新版本的 kube-vip 版本。此步骤需要并安装。
KVVERSION=$(curl -sL https://api.github.com/repos/kube-vip/kube-vip/releases | jq -r ".[0].name")
# 创建一个kube-vip 命令
alias kube-vip="docker run --network host --rm ghcr.io/kube-vip/kube-vip:$KVVERSION"
kube-vip manifest daemonset \
    --interface $INTERFACE \
    --address $VIP \
    --inCluster \
    --taint \
    --controlplane \
    --services \
    --arp \
    --leaderElection | tee kube-vip-ds.yaml
```

注：方式一只是做演示，后续如果对于命令部署感兴趣可以看看官网，目前部署使用第二种方式。

## 2.2. 方式二-DaemonSet启动

### 2.2.1. 创建 RBAC 设置

由于 kube-vip 作为 DaemonSet 作为常规资源而不是静态 Pod 运行，因此它仍然需要正确的访问权限才能监视 Kubernetes 服务和其他对象。为此，必须创建 RBAC 资源，其中包括 ServiceAccount、ClusterRole 和 ClusterRoleBinding，并可通过以下命令应用：

```shell
mkdir -p  ~/kube-vip && cd ~/kube-vip
# https://kube-vip.io/manifests/rbac.yaml
cat > kube-vip-rabc.yaml <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kube-vip
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  name: system:kube-vip-role
rules:
  - apiGroups: [""]
    resources: ["services/status"]
    verbs: ["update"]
  - apiGroups: [""]
    resources: ["services", "endpoints"]
    verbs: ["list","get","watch", "update"]
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["list","get","watch", "update", "patch"]
  - apiGroups: ["coordination.k8s.io"]
    resources: ["leases"]
    verbs: ["list", "get", "watch", "update", "create"]
  - apiGroups: ["discovery.k8s.io"]
    resources: ["endpointslices"]
    verbs: ["list","get","watch", "update"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: system:kube-vip-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-vip-role
subjects:
- kind: ServiceAccount
  name: kube-vip
  namespace: kube-system
EOF
```

备注：vip_cidr环境变量改成24依然还是32。

```shell
- name: vip_cidr
  value: "24"
```

github问题链接：https://github.com/kube-vip/kube-vip/issues/324

### 2.2.2. 创建资源

下载镜像地址实例：

- [docker.io/plndr/kube-vip:v0.6.4](https://hub.docker.com/r/plndr/kube-vip/tags)
- ghcr.io/kube-vip/kube-vip:v0.6.4

```shell
cat > kube-vip-ds.yaml <<EOF
apiVersion: apps/v1
kind: DaemonSet
metadata:
  creationTimestamp: null
  name: kube-vip-ds
  namespace: kube-system
spec:
  selector:
    matchLabels:
      name: kube-vip-ds
  template:
    metadata:
      creationTimestamp: null
      labels:
        name: kube-vip-ds
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: node-role.kubernetes.io/master
                operator: Exists
            - matchExpressions:
              - key: node-role.kubernetes.io/control-plane
                operator: Exists
      containers:
      - args:
        - manager
        env:
        - name: vip_arp
          value: "true"
        - name: port
          value: "6443"
        - name: vip_interface
          value: ens33
        - name: vip_cidr
          value: "24"
        - name: vipSubnet
          value: ""
        - name: cp_enable
          value: "true"
        - name: cp_namespace
          value: kube-system
        - name: vip_ddns
          value: "false"
        - name: svc_enable
          value: "true"
        - name: vip_leaderelection
          value: "true"
        - name: vip_leaseduration
          value: "5"
        - name: vip_renewdeadline
          value: "3"
        - name: vip_retryperiod
          value: "1"
        - name: address
          value: 192.168.80.100  #定义的vip地址
        image: ghcr.io/kube-vip/kube-vip:v0.6.4
        imagePullPolicy: Always
        name: kube-vip
        resources: {}
        securityContext:
          capabilities:
            add:
            - NET_ADMIN
            - NET_RAW
            - SYS_TIME
      hostNetwork: true
      serviceAccountName: kube-vip
      tolerations:
      - effect: NoSchedule
        operator: Exists
      - effect: NoExecute
        operator: Exists
  updateStrategy: {}
status:
  currentNumberScheduled: 0
  desiredNumberScheduled: 0
  numberMisscheduled: 0
  numberReady: 0
EOF
```

创建资源：

```shell
kubectl apply -f kube-vip-rabc.yaml
kubectl apply -f kube-vip-ds.yaml

# 查看资源创建
kubectl get pods  -n kube-system  -l  name=kube-vip-ds
NAME                READY   STATUS    RESTARTS   AGE
kube-vip-ds-bxg4s   1/1     Running   0          5m17s

kubectl get ServiceAccount -n kube-system | grep kube-vip
kube-vip                             0         45m
```

### 2.2.3. 查看master节点网关

```shell
ip addr
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 00:0c:29:7e:01:43 brd ff:ff:ff:ff:ff:ff
    altname enp2s1
    inet 192.168.80.45/24 brd 192.168.80.255 scope global noprefixroute ens33
       valid_lft forever preferred_lft forever
    inet 192.168.80.100/32 scope global ens33 # 这里可以看到我们的vip地址
       valid_lft forever preferred_lft forever
    inet6 fe80::1b63:e692:3239:41db/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
5: kube-ipvs0: <BROADCAST,NOARP> mtu 1500 qdisc noop state DOWN group default
    link/ether 02:67:68:41:61:e5 brd ff:ff:ff:ff:ff:ff
    inet 10.96.0.1/32 scope global kube-ipvs0
       valid_lft forever preferred_lft forever
    inet 10.96.0.10/32 scope global kube-ipvs0
       valid_lft forever preferred_lft forever
       
# 测试ip地址连通性
telnet 192.168.80.100 6443
Trying 192.168.80.100...
Connected to 192.168.80.100.
Escape character is '^]'.
```

结尾：本地`apiserver-vip`地址已经可以访问到，后续逻辑就是需要重创建k8s集群时签发的证书，将vip地址加入进去然后重新将证书在分配到每一个节点上。