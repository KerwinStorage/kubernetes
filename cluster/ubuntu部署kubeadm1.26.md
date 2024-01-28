# 1.初始化操作

介绍：kubeadm 是一个工具，它帮助初始化和配置 Kubernetes 集群。下面是在 **Ubuntu 22.04** 上安装和部署 Kubernetes 使用 kubeadm 的基本步骤。请注意，在执行以下步骤之前，您需要有一组服务器或虚拟机，至少一台作为控制平面节点（master），以及可选的一台或多台作为工作节点（worker）。

| 主机名称     | IP地址         | 裸盘             | 说明               |
| ------------ | -------------- | ---------------- | ------------------ |
| k8s-master01 | 192.168.80.45  | /dev/sdb（200G） | master节点         |
| k8s-node01   | 192.168.80.46  | /dev/sdb（200G） | node节点           |
| k8s-node02   | 192.168.80.47  | /dev/sdb（200G） | node节点           |
| kube-vip     | 192.168.80.100 |                  | apiserver的vip地址 |

- 所有节点

## 1.1.更新系统并安装所需依赖

```shell
sudo apt update
sudo apt upgrade -y
apt install net-tools nfs-kernel-server curl vim git lvm2 telnet htop jq lrzsz tree bash-completion telnet wget pssh make -y
```

## 1.2.修改主机名

```shell
hostnamectl set-hostname k8s-master01
hostnamectl set-hostname k8s-node01
hostnamectl set-hostname k8s-node02
echo "192.168.80.45  k8s-master01" >> /etc/hosts
echo "192.168.80.46  k8s-node01" >> /etc/hosts
echo "192.168.80.47  k8s-node02" >> /etc/hosts
echo "192.168.80.45  k8s-vip" >> /etc/hosts
```

## 1.3.安装ipvsadm

```shell
apt install ipvsadm ipset sysstat conntrack -y

cat >> /etc/modules-load.d/ipvs.conf <<EOF 
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
nf_conntrack
ip_tables
ip_set
xt_set
ipt_set
ipt_rpfilter
ipt_REJECT
ipip
EOF

systemctl restart systemd-modules-load.service
systemctl enable ipvsadm

lsmod | grep -e ip_vs -e nf_conntrack
ip_vs_sh               16384  0
ip_vs_wrr              16384  0
ip_vs_rr               16384  0
ip_vs                 180224  6 ip_vs_rr,ip_vs_sh,ip_vs_wrr
nf_conntrack          176128  1 ip_vs
nf_defrag_ipv6         24576  2 nf_conntrack,ip_vs
nf_defrag_ipv4         16384  1 nf_conntrack
libcrc32c              16384  2 nf_conntrack,ip_vs
```

## 1.4.修改内核参数

参考：[https://kubernetes.io/docs/setup/production-environment/container-runtimes/](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)

```shell
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
# 通过运行以下命令验证是否加载了 ，模块：br_netfilteroverla
lsmod | grep br_netfilter
lsmod | grep overlay


cat <<EOF > /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
fs.may_detach_mounts = 1
vm.overcommit_memory=1
vm.panic_on_oom=0
fs.inotify.max_user_watches=89100
fs.file-max=52706963
fs.nr_open=52706963
net.netfilter.nf_conntrack_max=2310720


net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_probes = 3
net.ipv4.tcp_keepalive_intvl =15
net.ipv4.tcp_max_tw_buckets = 36000
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_max_orphans = 327680
net.ipv4.tcp_orphan_retries = 3
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 16384
net.ipv4.ip_conntrack_max = 65536
net.ipv4.tcp_max_syn_backlog = 16384
net.ipv4.tcp_timestamps = 0
net.core.somaxconn = 16384

net.ipv6.conf.all.disable_ipv6 = 0
net.ipv6.conf.default.disable_ipv6 = 0
net.ipv6.conf.lo.disable_ipv6 = 0
net.ipv6.conf.all.forwarding = 0

EOF

sysctl --system
```

## 1.5.节点免密

- master节点操作

```shell
apt install -y sshpass
ssh-keygen -f /root/.ssh/id_rsa -P ''
export IP="k8s-master01 k8s-node01 k8s-node02"
export SSHPASS=1qazZSE$
for HOST in $IP;do
     sshpass -e ssh-copy-id -o StrictHostKeyChecking=no $HOST
done
```

## 1.6.安装 Docker

Kubernetes 推荐 Docker 作为容器运行时，但它同样支持其他容器运行时，如 containerd 或 CRI-O。

```shell
sudo apt install -y apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
sudo apt update
sudo apt install -y docker-ce
systemctl enable docker

#修改docker的Cgroup Driver为systemd
cat > /etc/docker/daemon.json <<-"EOF"
{
  "exec-opts": [
    "native.cgroupdriver=systemd"
  ],
  "registry-mirrors": [
        "https://dockerproxy.com",
        "https://hub-mirror.c.163.com",
        "https://mirror.baidubce.com",
        "https://ccr.ccs.tencentyun.com"
    ]
}
EOF
systemctl restart docker.service
docker info | grep systemd
```

## 1.7.安装cri-dockerd

Github：[https://github.com/Mirantis/cri-dockerd](https://github.com/Mirantis/cri-dockerd)

```shell
wget https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.8/cri-dockerd-0.3.8.amd64.tgz
# Run these commands as root

tar zxvf cri-dockerd-0.3.8.amd64.tgz
cd cri-dockerd
mkdir -p /usr/bin
install -o root -g root -m 0755 cri-dockerd /usr/bin/cri-dockerd
```

获取Service文件：

```shell
wget -O /etc/systemd/system/cri-docker.service https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.service
wget -O /etc/systemd/system/cri-docker.socket https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.socket
```

或者直接填写：

- cri-docker.socket

```shell
cat > /etc/systemd/system/cri-docker.socket <<EOF
[Unit]
Description=CRI Docker Socket for the API
PartOf=cri-docker.service

[Socket]
ListenStream=%t/cri-dockerd.sock
SocketMode=0660
SocketUser=root
SocketGroup=docker

[Install]
WantedBy=sockets.target
EOF
```

- cri-docker.service

```shell
cat > /etc/systemd/system/cri-docker.service <<EOF
[Unit]
Description=CRI Interface for Docker Application Container Engine
Documentation=https://docs.mirantis.com
After=network-online.target firewalld.service docker.service
Wants=network-online.target
Requires=cri-docker.socket

[Service]
Type=notify
ExecStart=/usr/bin/cri-dockerd --container-runtime-endpoint fd:// --pod-infra-container-image=registry.aliyuncs.com/google_containers/pause:3.6
ExecReload=/bin/kill -s HUP $MAINPID
TimeoutSec=0
RestartSec=2
Restart=always

# Note that StartLimit* options were moved from "Service" to "Unit" in systemd 229.
# Both the old, and new location are accepted by systemd 229 and up, so using the old location
# to make them work for either version of systemd.
StartLimitBurst=3

# Note that StartLimitInterval was renamed to StartLimitIntervalSec in systemd 230.
# Both the old, and new name are accepted by systemd 230 and up, so using the old name to make
# this option work for either version of systemd.
StartLimitInterval=60s

# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity

# Comment TasksMax if your systemd version does not support it.
# Only systemd 226 and above support this option.
TasksMax=infinity
Delegate=yes
KillMode=process

[Install]
WantedBy=multi-user.target
EOF
```

在 `ExecStart=/usr/bin/cri-dockerd --container-runtime-endpoint fd://` 后面加上 `--pod-infra-container-image=registry.aliyuncs.com/google_containers/pause:3.6`
启动cri-dockerd：

```shell
systemctl daemon-reload
systemctl enable cri-docker
systemctl start cri-docker
```

## 1.8.下载cni插件

Github： [https://github.com/containernetworking/plugins](https://github.com/containernetworking/plugins)

```shell
wget https://github.com/containernetworking/plugins/releases/download/v1.3.0/cni-plugins-linux-amd64-v1.3.0.tgz
mkdir -pv /opt/cni/bin
tar zxvf cni-plugins-linux-amd64-v1.3.0.tgz -C /opt/cni/bin/
```

## 1.9.关闭 Swap

```shell
sudo swapoff -a
sed -ri 's/.*swap.*/#&/' /etc/fstab
```

编辑 /etc/fstab 文件，注释掉 swap 相关的行，以便在重启时禁用 swap。

## 1.10.配置ulimit

```shell
ulimit -SHn 65535
cat >> /etc/security/limits.conf <<EOF
* soft nofile 655360
* hard nofile 131072
* soft nproc 655350
* hard nproc 655350
* seft memlock unlimited
* hard memlock unlimitedd
EOF
```

## 1.11.时间同步

```shell
# 时间同步(服务端)
apt install chrony -y
cat > /etc/chrony.conf << EOF
pool ntp.aliyun.com iburst
driftfile /var/lib/chrony/drift
makestep 1.0 3
rtcsync
allow 192.168.80.0/24 #允许网段地址同步时间
local stratum 10
keyfile /etc/chrony.keys
leapsectz right/UTC
logdir /var/log/chrony
EOF

systemctl restart chronyd.service
systemctl enable chrony


# 时间同步(客户端)
apt install chrony -y
cat > /etc/chrony.conf << EOF
pool k8s-master01 iburst
driftfile /var/lib/chrony/drift
makestep 1.0 3
rtcsync
keyfile /etc/chrony.keys
leapsectz right/UTC
logdir /var/log/chrony
EOF

systemctl restart chronyd.service
systemctl enable chrony
#使用客户端进行验证
chronyc sources -v

#查看时间同步源的状态
chronyc sourcestats -v

# 查看系统时间与日期(全部机器)
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
date -R
timedatectl
```

# 2.准备kubeadm条件

## 2.1.安装特定版本的 kubeadm, kubelet 和 kubectl

- 全部节点操作

添加 Kubernetes APT 仓库：

```shell
curl -s https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo gpg --dearmor -o /usr/share/keyrings/kubernetes-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

更新 APT 包索引并安装 kubeadm, kubelet, 和 kubectl 的 1.26 版本。确保先检查 1.26 版本是否已经发布并可用。

```shell
sudo apt update
sudo apt install -y kubelet=1.26.0-00 kubeadm=1.26.0-00 kubectl=1.26.0-00
sudo apt-mark hold kubelet kubeadm kubectl
```

## 2.2.初始化 Kubernetes 集群

配置kubelet驱动参考：[https://kubernetes.io/zh-cn/docs/tasks/administer-cluster/kubeadm/configure-cgroup-driver](https://kubernetes.io/zh-cn/docs/tasks/administer-cluster/kubeadm/configure-cgroup-driver/)
在控制平面节点上执行：

```shell
kubeadm config print init-defaults > kubeadm-config.yaml
```

初始化：
`kubeadm init`命令参数参考：[链接](https://kubernetes.io/zh-cn/docs/reference/setup-tools/kubeadm/kubeadm-init/)
kubeadm高可用：[链接](https://github.com/kubernetes/kubeadm/blob/main/docs/ha-considerations.md#options-for-software-load-balancing)

```shell
#kubeadm init --config kubeadm-config.yaml
kubeadm init --image-repository registry.aliyuncs.com/google_containers --control-plane-endpoint=k8s-vip --apiserver-advertise-address=192.168.80.45 --kubernetes-version=v1.26.0 --pod-network-cidr=10.244.0.0/16 --cri-socket=unix:///var/run/cri-dockerd.sock

# 执行结果
[init] Using Kubernetes version: v1.26.0
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [k8s-master01 k8s-vip kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 192.168.80.45]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [k8s-master01 localhost] and IPs [192.168.80.45 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [k8s-master01 localhost] and IPs [192.168.80.45 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 12.510984 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node k8s-master01 as control-plane by adding the labels: [node-role.kubernetes.io/control-plane node.kubernetes.io/exclude-from-external-load-balancers]
[mark-control-plane] Marking the node k8s-master01 as control-plane by adding the taints [node-role.kubernetes.io/control-plane:NoSchedule]
[bootstrap-token] Using token: mo2u0f.yvg991926b6uksfk
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] Configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] Configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of control-plane nodes by copying certificate authorities
and service account keys on each node and then running the following as root:

  kubeadm join k8s-vip:6443 --token mo2u0f.yvg991926b6uksfk \
        --discovery-token-ca-cert-hash sha256:663e9f5fc3306526a9bb7fc39ac080b121d736b05202829cba082830873fa1ee \
        --control-plane

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join k8s-vip:6443 --token mo2u0f.yvg991926b6uksfk \
        --discovery-token-ca-cert-hash sha256:663e9f5fc3306526a9bb7fc39ac080b121d736b05202829cba082830873fa1ee
```

- `--control-plane-endpoint=k8s-vip` 参数为后面部署kube-vip做预留准备，初始化时指向的是k8s-master01的地址，后续部署完指向kube-vip内的vip地址，本次vip为：`192.168.80.100`。

# 3.启动kubeadm

## 3.1.配置config文件

```shell
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# 配置自动补全
apt install -y bash-completion
source /usr/share/bash-completion/bash_completion
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc
source ~/.bashrc
```

## 3.2.安装Calico

注意对应的版本：[https://docs.tigera.io/calico/latest/getting-started/kubernetes/requirements](https://docs.tigera.io/calico/latest/getting-started/kubernetes/requirements)
参考github内容：[https://github.com/projectcalico/calico/blob/v3.26.0/manifests/calico.yaml](https://github.com/projectcalico/calico/blob/v3.26.0/manifests/calico.yaml)

```shell
mkdir -p $HOME/k8s-install/network && cd $HOME/k8s-install/network
# CIDR的值，与 kube-controller-manager中“--cluster-cidr=172.16.0.0/16” 一致
cat calico.yaml | grep -A 5 CALICO_IPV4POOL_CIDR
            # kubeadm init 时指定的--pod-network-cidr参数
            - name: CALICO_IPV4POOL_CIDR
              value: "10.244.0.0/16"
            - name: IP_AUTODETECTION_METHOD
              value: "interface=ens33" # 指定网卡
            # Disable file logging so `kubectl logs` works.
            - name: CALICO_DISABLE_FILE_LOGGING
            
kubectl apply -f calico.yaml

kubectl get pod -n kube-system
NAME                                      READY   STATUS    RESTARTS      AGE
calico-kube-controllers-b48d575fb-vpv9x   1/1     Running   1 (18m ago)   34m
calico-node-lrqf6                         1/1     Running   1 (18m ago)   34m
calico-node-vpt4p                         1/1     Running   0             5m5s
calico-node-zpgtj                         1/1     Running   0             6m1s
```

# 4.启用ipvs模式

修改kube-proxy模式：

```shell
kubectl edit configmap -n kube-system kube-proxy
 
    mode: "ipvs"
    nodePortAddresses: null
 
 
如果要修改调度算法使用
    scheduler: ""
```

默认为空：

```shell
ipvsadm -L -n
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
```

依次重启/删除kube-proxy的pod：

```shell
kubectl get pod -n kube-system  | grep kube-proxy
kube-proxy-49mgv                          1/1     Running   2 (28m ago)   73m
kube-proxy-lclt7                          1/1     Running   2 (67m ago)   79m
kube-proxy-lm8jv                          1/1     Running   1 (28m ago)   57m

# 删除pod
kubectl delete pod -n kube-system kube-proxy-49mgv
kubectl delete pod -n kube-system kube-proxy-lclt7
kubectl delete pod -n kube-system kube-proxy-lm8jv
```

等容器重启完毕再去看看：

```shell
ipvsadm -L -n
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.96.0.1:443 rr
  -> 192.168.80.45:6443           Masq    1      1          0
TCP  10.96.0.10:53 rr
  -> 10.244.58.197:53             Masq    1      0          0
  -> 10.244.58.198:53             Masq    1      0          0
TCP  10.96.0.10:9153 rr
  -> 10.244.58.197:9153           Masq    1      0          0
  -> 10.244.58.198:9153           Masq    1      0          0
UDP  10.96.0.10:53 rr
  -> 10.244.58.197:53             Masq    1      0          0
  -> 10.244.58.198:53             Masq    1      0          0
```

# 5.启动kube-vip

链接地址：

- [Github地址](https://github.com/kubernetes/kubeadm/blob/main/docs/ha-considerations.md)
- [Daemonset部署地址](https://kube-vip.io/docs/installation/daemonset)

## 5.1.方式一-命令启动

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

## 5.2.方式二-DaemonSet启动

### 5.2.1创建 RBAC 设置

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

### 5.2.2创建资源

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

备注：vip_cidr环境变量改成24依然还是32。

```shell
- name: vip_cidr
  value: "24"
```

github问题链接：[https://github.com/kube-vip/kube-vip/issues/324](https://github.com/kube-vip/kube-vip/issues/324)
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

### 5.2.3查看master节点网关

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

# 6.其他节点加入集群

- work节点操作

修改hosts：

```shell
echo "192.168.80.100  k8s-vip" >> /etc/hosts
```

node节点初始化：

```shell
kubeadm join k8s-vip:6443 --cri-socket=unix:///var/run/cri-dockerd.sock --token mo2u0f.yvg991926b6uksfk \
        --discovery-token-ca-cert-hash sha256:663e9f5fc3306526a9bb7fc39ac080b121d736b05202829cba082830873fa1ee
[preflight] Running pre-flight checks
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Starting the kubelet
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...
This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```

清理控制平面实例（如果有需要）

```shell
# 设置为不可调度及驱逐节点
kubectl cordon  k8s-node01
kubectl drain k8s-node01 --delete-local-data --force --ignore-daemonsets
# 删除节点
kubectl delete node  k8s-node01
# 清空节点
kubeadm reset --cri-socket /var/run/cri-dockerd.sock
```

- 复制config配置文件

```shell
# master操作
scp /etc/kubernetes/admin.conf root@192.168.80.46:$HOME/.kube/config
scp /etc/kubernetes/admin.conf root@192.168.80.47:$HOME/.kube/config

# node节点操作
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

# 7.openlocal

Github：[https://github.com/alibaba/open-local/blob/main/README_zh_CN.md](https://github.com/alibaba/open-local/blob/main/README_zh_CN.md)
镜像下载地址参考：[https://hub.docker.com/search?q=openlocal](https://hub.docker.com/search?q=openlocal)

## 7.1.创建lvm

参考：[https://en.wikipedia.org/wiki/Logical_Volume_Manager_(Linux)](https://en.wikipedia.org/wiki/Logical_Volume_Manager_(Linux))

```shell
apt install lvm2 -y
```

## 7.2.helm安装指南

前提：[helm3](https://helm.sh/zh/docs/intro/install/)已经部署完成。

## 7.3.部署open-local

kube-scheduler 配置：[https://github.com/alibaba/open-local/blob/main/docs/user-guide/kube-scheduler-configuration.md](https://github.com/alibaba/open-local/blob/main/docs/user-guide/kube-scheduler-configuration.md)

| 主机         | 裸盘           |
| ------------ | -------------- |
| k8s-master01 | /dev/sdb：200G |
| k8s-node01   | /dev/sdb：200G |
| k8s-node02   | /dev/sdb：200G |

- 每个节点都需要创建

初始化磁盘：

```shell
pvcreate /dev/sdb
pvscan
vgcreate open-local-pool-0  /dev/sdb
vgdisplay
```

关于`apiVersion`在1.26版本内废弃`v1beta1`：[链接](https://kubernetes.io/zh-cn/docs/reference/scheduling/config/)

```shell
cat > /etc/kubernetes/kube-scheduler-configuration.yaml <<EOF
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
clientConnection:
  kubeconfig: /etc/kubernetes/scheduler.conf # your kubeconfig filepath 也就是我们的.kube/config文件
extenders:
- urlPrefix: http://open-local-scheduler-extender.kube-system:23000/scheduler
  filterVerb: predicates
  prioritizeVerb: priorities
  weight: 10
  ignorable: true
  nodeCacheCapable: true
EOF
```

如果你的 kube-scheduler 是静态 pod，请像这样配置 kube-scheduler 文件：

```shell
vim /etc/kubernetes/manifests/kube-scheduler.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    component: kube-scheduler
    tier: control-plane
  name: kube-scheduler
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-scheduler
    - --address=0.0.0.0
    - --authentication-kubeconfig=/etc/kubernetes/scheduler.conf
    - --authorization-kubeconfig=/etc/kubernetes/scheduler.conf
    - --bind-address=0.0.0.0
    - --feature-gates=TTLAfterFinished=true,EphemeralContainers=true
    - --kubeconfig=/etc/kubernetes/scheduler.conf
    - --leader-elect=true
    - --port=10251
    - --profiling=false
    - --config=/etc/kubernetes/kube-scheduler-configuration.yaml # add --config option
    image: sea.hub:5000/oecp/kube-scheduler:v1.20.4-aliyun.1
    imagePullPolicy: IfNotPresent
    livenessProbe:
      failureThreshold: 8
      httpGet:
        path: /healthz
        port: 10259
        scheme: HTTPS
      initialDelaySeconds: 10
      periodSeconds: 10
      timeoutSeconds: 15
    name: kube-scheduler
    resources:
      requests:
        cpu: 100m
    startupProbe:
      failureThreshold: 24
      httpGet:
        path: /healthz
        port: 10259
        scheme: HTTPS
      initialDelaySeconds: 10
      periodSeconds: 10
      timeoutSeconds: 15
    volumeMounts:
    - mountPath: /etc/kubernetes/kube-scheduler-configuration.yaml # mount this file into the container
      name: scheduler-policy-config
      readOnly: true
    - mountPath: /etc/kubernetes/scheduler.conf
      name: kubeconfig
      readOnly: true
    - mountPath: /etc/localtime
      name: localtime
      readOnly: true
  hostNetwork: true
  dnsPolicy: ClusterFirstWithHostNet
  priorityClassName: system-node-critical
  volumes:
  - hostPath:
      path: /etc/kubernetes/kube-scheduler-configuration.yaml
      type: File
    name: scheduler-policy-config # this is kube-scheduler-configuration.yaml that we created before
  - hostPath:
      path: /etc/kubernetes/scheduler.conf
      type: FileOrCreate
    name: kubeconfig
  - hostPath:
      path: /etc/localtime
      type: File
    name: localtime
```

部署openlocal：

```shell
wget -O "open-local-0.7.1.tar.gz" https://github.com/alibaba/open-local/archive/refs/tags/v0.7.1.tar.gz
tar xf open-local-0.7.1.tar.gz
cd open-local-0.7.1/helm
# 批量替换values.yaml镜像地址，ack-agility-registry.cn-shanghai.cr.aliyuncs.com 这个地址给去掉
sed -i "s/ecp_builder\///g" values.yaml
sed -i "s/ack-agility-registry.cn-shanghai.cr.aliyuncs.com/openlocal/g" values.yaml
sed -i "s/init_job: true/init_job: false/g" values.yaml

# 部署
helm install open-local .
```

values.yaml文件参考：

```shell
cat > values.yaml <<EOF
name: open-local
namespace: kube-system
driver: local.csi.aliyun.com
customLabels: {}
images:
  local:
    image: open-local
    tag: v0.7.1
  registrar:
    image: csi-node-driver-registrar
    tag: v2.3.0
  provisioner:
    image: csi-provisioner
    tag: v2.2.2
  resizer:
    image: csi-resizer
    tag: v1.3.0
  snapshotter:
    image: csi-snapshotter
    tag: v4.2.1
  snapshot_controller:
    image: snapshot-controller
    tag: v4.2.1
agent:
  name: open-local-agent
  # This block device will be used to create as a Volume Group in every node
  # Open-Local does nothing if the device has been formatted or mountted
  device: /dev/sdb  #配置成你自己的磁盘
  kubelet_dir: /var/lib/kubelet
  volume_name_prefix: local
  spdk: false
  # driver mode can be 'all' or 'node'
  # all: agent will start as csi controller and csi node
  # node: agent will start as csi node
  driverMode: node
extender:
  name: open-local-scheduler-extender
  # scheduling strategy: binpack/spread
  strategy: spread
  # scheduler extender http port
  port: 23000
  # you can also configure your kube-scheduler manually, see docs/user-guide/kube-scheduler-configuration.md to get more details
  init_job: false
controller:
  update_nls: "true"
storageclass:
  lvm:
    name: open-local-lvm
  lvm_xfs:
    name: open-local-lvm-xfs
  device_ssd:
    name: open-local-device-ssd
  device_hdd:
    name: open-local-device-hdd
monitor:
  # install grafana dashboard
  enabled: false
  # grafana namespace
  namespace: monitoring
global:
  RegistryURL: openlocal
EOF
```

遇到的问题：

1. **0/3 nodes are available: 1 node(s) had untolerated taint {node-role.kubernetes.io/control-plane: }, 3 node(s) didn't match Pod's node affinity/selector. preemption: 0/3 nodes are available: 3 Preemption is not helpful for scheduling..**

```shell
kubectl describe node k8s-master01  | grep Taints
Taints:             node-role.kubernetes.io/control-plane:NoSchedule

kubectl describe nodes | grep -E '(Roles|Taints)'
kubectl taint node k8s-master01  node-role.kubernetes.io/control-plane-


# 查看yaml内的内容，哪些不符合规则
kubectl get replicasets.apps -n kube-system open-local-scheduler-extender-6fffd89747 -o yaml
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: node-role.kubernetes.io/master
                operator: Exists
```

- NoSchedule：表示k8s将不会将Pod调度到具有该污点的Node上
- PreferNoSchedule：表示k8s将尽量避免将Pod调度到具有该污点的Node上
- NoExecute：表示k8s将不会将Pod调度到具有该污点的Node上，同时会将Node上已经存在的Pod驱逐出去

具体来说，这是一个节点选择器（Node Selector）的配置。这段配置的含义是：

- `requiredDuringSchedulingIgnoredDuringExecution`: 这是节点亲和性（Node Affinity）的一种类型，表示 Kubernetes 在调度时会尽量遵守这些规则，但在 Pod 运行期间，即使这些规则不再满足，也不会重新调度 Pod。
- `nodeSelectorTerms`: 这是一个列表，其中的每一项都表示一组必须满足的规则。
- `matchExpressions`: 这是一个列表，其中的每一项都是一个规则，表示节点必须满足的条件。
- `key: node-role.kubernetes.io/master` 和 `operator: Exists`：这个规则的含义是，只有标签 `node-role.kubernetes.io/master` 存在的节点才能调度这个 Pod。

总的来说，这段配置的含义是，Pod 在被调度时，会被尽量调度到标签为 `node-role.kubernetes.io/master` 的节点上。

```shell
# 加上需要的标签
kubectl label node k8s-master01 node-role.kubernetes.io/master=Exists
```

## 7.4.查看创建资源情况

```shell
# 查看容器启动
kubectl get pod -n kube-system  | grep open-local
open-local-agent-kjt5p                           3/3     Running   0            27m
open-local-agent-nk8xq                           3/3     Running   0            27m
open-local-agent-r7rl7                           3/3     Running   0            27m
open-local-controller-85987bffd9-z5kjn           6/6     Running   0            27m
open-local-scheduler-extender-6fffd89747-nmt58   1/1     Running   0            13m

# 查看
kubectl get  sc
NAME                    PROVISIONER            RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
open-local-device-hdd   local.csi.aliyun.com   Delete          WaitForFirstConsumer   false                  16m
open-local-device-ssd   local.csi.aliyun.com   Delete          WaitForFirstConsumer   false                  16m
open-local-lvm          local.csi.aliyun.com   Delete          WaitForFirstConsumer   true                   16m
open-local-lvm-xfs      local.csi.aliyun.com   Delete          WaitForFirstConsumer   true                   16m
```

## 7.5.配置nlsc

```shell
kubectl edit nlsc open-local
# yaml内容
apiVersion: csi.aliyun.com/v1alpha1
kind: NodeLocalStorageInitConfig
metadata:
  annotations:
    meta.helm.sh/release-name: open-local
    meta.helm.sh/release-namespace: default
  creationTimestamp: "2024-01-28T12:25:42Z"
  generation: 2
  labels:
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: open-local
    app.kubernetes.io/version: v0.7.1
    helm.sh/chart: open-local-v0.7.1
  name: open-local
  resourceVersion: "20420"
  uid: 20c75b0c-4a32-431d-bf25-244480046eb7
spec:
  globalConfig:
    listConfig:
      vgs:
        include:
        - open-local-pool-[0-9]+
        - yoda-pool[0-9]+
        - ackdistro-pool
    resourceToBeInited:
      vgs:
      - devices:
        - /dev/sdb # 这里就是我们本地的磁盘
        name: open-local-pool-0
```

## 7.6.查看磁盘分配情况

```shell
# vg
vgdisplay
  --- Volume group ---
  VG Name               open-local-pool-0
  System ID
  Format                lvm2
  Metadata Areas        1
  Metadata Sequence No  1
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                0
  Open LV               0
  Max PV                0
  Cur PV                1
  Act PV                1
  VG Size               <200.00 GiB
  PE Size               4.00 MiB
  Total PE              51199
  Alloc PE / Size       0 / 0
  Free  PE / Size       51199 / <200.00 GiB
  VG UUID               QD7qVJ-myrj-ezts-jBL1-3D3C-3zlp-ReoquF

# pv
pvdisplay
  --- Physical volume ---
  PV Name               /dev/sdb
  VG Name               open-local-pool-0
  PV Size               200.00 GiB / not usable 4.00 MiB
  Allocatable           yes
  PE Size               4.00 MiB
  Total PE              51199
  Free PE               51199
  Allocated PE          0
  PV UUID               nDlewm-IKTg-qR5V-BBig-6jO4-8V0x-jIhD4X
```

# 8.卸载

```shell
kubeadm reset -f
modprobe -r ipip
lsmod
rm -rf ~/.kube/
rm -rf /etc/kubernetes/
rm -rf /etc/systemd/system/kubelet.service.d
rm -rf /etc/systemd/system/kubelet.service
rm -rf /usr/bin/kube*
rm -rf /etc/cni
rm -rf /opt/cni
rm -rf /var/lib/etcd
rm -rf /var/etcd
apt-get autoremove kubeadm kubectl kubelet  --allow-change-held-packages -y
```

注：慎用上面的命令，加入的新机器最好是没有使用过的。
问题链接：[地址](https://stackoverflow.com/questions/60166842/pod-is-in-pending-stage-error-failedscheduling-nodes-didnt-match-node-sel)
参考：

- [https://ost.51cto.com/posts/12603](https://ost.51cto.com/posts/12603)
- [https://blog.csdn.net/zhanglu0302/article/details/128111040](https://blog.csdn.net/zhanglu0302/article/details/128111040)
- [https://blog.csdn.net/guanfengliang1988/article/details/128456753c](https://blog.csdn.net/guanfengliang1988/article/details/128456753c)