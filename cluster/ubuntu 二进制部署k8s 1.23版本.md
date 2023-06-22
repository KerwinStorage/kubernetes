# 1. 环境准备



部署步骤：

1. 先部署一个master + 两个node节点集群
2. 在加入两个master进来
3. 加入负载均衡和高可用

![img](https://raw.githubusercontent.com/kubernetes/kubernetes/release-1.2/docs/design/architecture.png)

## 1.1 安装规划

| **角色**     | **IP**         | **组件**                                                     |
| ------------ | -------------- | ------------------------------------------------------------ |
| k8s-master01 | 192.168.80.45  | **etcd**, api-server, controller-manager, scheduler, docker, kubelet, kube-proxy, haproxy、keepalived |
| k8s-master02 | 192.168.80.46  | api-server, controller-manager, scheduler, docker, kubelet, kube-proxy |
| k8s-master03 | 192.168.80.47  | api-server, controller-manager, scheduler, docker, kubelet, kube-proxy |
| k8s-node01   | 192.168.80.48  | **etcd**, kubelet, kube-proxy, docker, haproxy, keepalived   |
| k8s-node02   | 192.168.80.49  | **etcd**, kubelet, kube-proxy, docker, haproxy, keepalived   |
| vip          | 192.168.80.100 |                                                              |

**注释：这里前期先部署三台机器：`k8s-master01`、`k8s-node01`、`k8s-node02`，后期将`k8s-master02`、`k8s-master03`和`vip`高可用地址加入进去。**

| 软件                                                         | 版本                           |
| ------------------------------------------------------------ | ------------------------------ |
| 内核                                                         | Linux 5.19.0-35-generic x86_64 |
| [Ubuntu](https://ubuntu.com/download/desktop/thank-you?version=22.04.2&architecture=amd64) | 22.04                          |
| kube-apiserver、kube-controller-manager、kube-scheduler、kubelet、kube-proxy | v1.23.17                       |
| etcd                                                         | v3.5.4                         |
| docker-ce                                                    | v23.0.1                        |
| containerd                                                   | v1.6.18                        |
| cfssl                                                        | v1.6.1                         |
| cni                                                          | v1.1.1                         |
| crictl                                                       | v1.23.0                        |
| haproxy                                                      | 2.4.18-0ubuntu1.2              |
| keepalived                                                   | v2.2.4                         |

## 1.2 系统设置

- 所有节点执行一遍

定义环境变量：

**这里先将ip地址的环境变量加入到`~/.bashrc`，这样就可以永久保存，因为是为了防止部署时配置的ip地址写错导致集群出现问题，还有环境变量添加只能执行一次因为是重定向到文件内。**

```bash
cd ~
echo "K8S_MASTER01='192.168.80.45'" >>  ~/.bashrc
echo "K8S_MASTER02='192.168.80.46'" >>  ~/.bashrc
echo "K8S_MASTER03='192.168.80.47'" >>  ~/.bashrc
echo "K8S_NODE01='192.168.80.48'" >>  ~/.bashrc
echo "K8S_NODE02='192.168.80.49'" >>  ~/.bashrc
echo "LOCALHOST=`hostname -I |awk '{print $1}'`" >> ~/.bashrc
source ~/.bashrc
```

初始化：

```bash
# 1. 修改主机名
hostnamectl set-hostname k8s-master01
hostnamectl set-hostname k8s-master02
hostnamectl set-hostname k8s-master03
hostnamectl set-hostname k8s-node01
hostnamectl set-hostname k8s-node02

# 2. 主机名解析
cat >> /etc/hosts << EOF
$K8S_MASTER01  k8s-master01
$K8S_MASTER02  k8s-master02
$K8S_MASTER03  k8s-master03
$K8S_NODE01  k8s-node01
$K8S_NODE02  k8s-node02
EOF

# 3. 禁用 swap
swapoff -a && sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# 4. 将桥接的IPv4流量传递到iptables的链 
cat > /etc/sysctl.d/k8s.conf << EOF 
net.bridge.bridge-nf-call-ip6tables = 1 
net.bridge.bridge-nf-call-iptables = 1 
EOF
sysctl --system 

# 5. 域名解析
sudo apt install resolvconf
sudo systemctl enable resolvconf.service

sudo echo "nameserver 8.8.8.8" >>  /etc/resolvconf/resolv.conf.d/head

sudo systemctl start resolvconf.service
sudo systemctl restart resolvconf.service
sudo systemctl restart systemd-resolved.service
sudo systemctl status resolvconf.service
sudo echo "nameserver 8.8.8.8" >> /etc/resolv.conf
 
# 6. 时间同步(服务端)
apt install chrony -y
cat > /etc/chrony.conf <<"EOF" 
pool ntp.aliyun.com iburst
driftfile /var/lib/chrony/drift
makestep 1.0 3
rtcsync
allow 192.168.80.0/24
local stratum 10
keyfile /etc/chrony.keys
leapsectz right/UTC
logdir /var/log/chrony
EOF

systemctl restart chronyd.service

# 时间同步(客户端)
apt install chrony -y
cat > /etc/chrony.conf <<"EOF"
pool k8s-master01 iburst
driftfile /var/lib/chrony/drift
makestep 1.0 3
rtcsync
keyfile /etc/chrony.keys
leapsectz right/UTC
logdir /var/log/chrony
EOF

systemctl restart chronyd.service
#使用客户端进行验证
chronyc sources -v

# 查看系统时间与日期(全部机器)
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
hwclock --systohc
date -R




# 7. 日志目录
mkdir -p /var/log/kubernetes

# 8.配置ulimit
ulimit -SHn 65535
cat >> /etc/security/limits.conf <<"EOF"
* soft nofile 655360
* hard nofile 131072
* soft nproc 655350
* hard nproc 655350
* seft memlock unlimited
* hard memlock unlimitedd
EOF

# 9.配置免密登录
apt install -y sshpass
ssh-keygen
export IP="k8s-master01 k8s-node01 k8s-node02"
export SSHPASS=antCloud@2022
for HOST in $IP;do
     sshpass -e ssh-copy-id -o StrictHostKeyChecking=no $HOST
done

# 10.ipvsadm安装（lb除外）
apt install ipvsadm ipset sysstat conntrack -y
cat >> /etc/modules-load.d/ipvs.conf <<"EOF" 
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

lsmod | grep -e ip_vs -e nf_conntrack
ip_vs_sh               16384  0
ip_vs_wrr              16384  0
ip_vs_rr               16384  0
ip_vs                 155648  6 ip_vs_rr,ip_vs_sh,ip_vs_wrr
nf_conntrack          139264  1 ip_vs
nf_defrag_ipv6         24576  2 nf_conntrack,ip_vs
nf_defrag_ipv4         16384  1 nf_conntrack
libcrc32c              16384  2 nf_conntrack,ip_vs

# 11.修改内核参数 (lb除外)
cat > /etc/sysctl.d/k8s.conf <<"EOF"
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
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
net.ipv6.conf.all.forwarding = 1
EOF

sysctl --system
```

# 2.k8s基本组件安装

## 2.1 所有k8s节点安装Containerd作为Runtime

```bash
wget https://github.com/containernetworking/plugins/releases/download/v1.1.1/cni-plugins-linux-amd64-v1.1.1.tgz

#创建cni插件所需目录
mkdir -p /etc/cni/net.d /opt/cni/bin 
#解压cni二进制包
tar xf cni-plugins-linux-amd64-v1.1.1.tgz -C /opt/cni/bin/


wget https://github.com/containerd/containerd/releases/download/v1.6.6/cri-containerd-cni-1.6.6-linux-amd64.tar.gz

#解压
tar -xzf cri-containerd-cni-*-linux-amd64.tar.gz -C /

rm -rf /etc/cni/net.d/10-containerd-net.conflist

#创建服务启动文件
cat > /etc/systemd/system/containerd.service <<EOF
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target local-fs.target

[Service]
ExecStartPre=-/sbin/modprobe overlay
ExecStart=/usr/local/bin/containerd
Type=notify
Delegate=yes
KillMode=process
Restart=always
RestartSec=5
LimitNPROC=infinity
LimitCORE=infinity
LimitNOFILE=infinity
TasksMax=infinity
OOMScoreAdjust=-999

[Install]
WantedBy=multi-user.target
EOF
```

### 2.1.1配置Containerd所需的模块

```bash
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
```

### 2.1.2加载模块

```bash
systemctl restart systemd-modules-load.service
```

### 2.1.3配置Containerd所需的内核

```bash
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

# 加载内核

sysctl --system
```

### 2.1.4创建Containerd的配置文件

```bash
mkdir -p /etc/containerd
containerd config default | tee /etc/containerd/config.toml


修改Containerd的配置文件
sed -i "s#SystemdCgroup\ \=\ false#SystemdCgroup\ \=\ true#g" /etc/containerd/config.toml

cat /etc/containerd/config.toml | grep SystemdCgroup

sed -i "s#k8s.gcr.io#registry.cn-hangzhou.aliyuncs.com/chenby#g" /etc/containerd/config.toml

cat /etc/containerd/config.toml | grep sandbox_image


# 找到containerd.runtimes.runc.options，在其下加入SystemdCgroup = true

[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
              SystemdCgroup = true
    [plugins."io.containerd.grpc.v1.cri".cni]


# 将sandbox_image默认地址改为符合版本地址

    sandbox_image = "registry.cn-hangzhou.aliyuncs.com/chenby/pause:3.6"
```

### 2.1.5启动并设置为开机启动

```bash
systemctl daemon-reload
systemctl enable --now containerd
```

### 2.1.6 crictl命令安装

```bash
wget https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.23.0/crictl-v1.23.0-linux-amd64.tar.gz

#解压
tar xf crictl-v1.23.0-linux-amd64.tar.gz -C /usr/bin/
#生成配置文件

cat > /etc/crictl.yaml <<EOF
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
debug: false
EOF

#测试
systemctl restart  containerd
crictl info
```

## 2.2 所有k8s节点安装docker作为Runtime

Ubuntu 22.04更新源：

```bash
# 更新 apt 包索引
sudo apt-get update

# 安装 apt 依赖包，用于通过HTTPS来获取仓库
sudo apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release

# 添加 Docker 的官方 GPG 密钥
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# 使用以下指令设置稳定版仓库
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://mirrors.aliyun.com/docker-ce/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
```

Ubuntu 20.04更新源：

```bash
# 更新 apt 包索引
sudo apt-get update

# 安装 apt 依赖包，用于通过HTTPS来获取仓库
sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common -y

# 添加 Docker 的官方 GPG 密钥
curl -fsSL https://mirrors.ustc.edu.cn/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
sudo apt-key fingerprint 0EBFCD88

# 使用以下指令设置稳定版仓库
sudo add-apt-repository \
   "deb [arch=amd64] https://mirrors.ustc.edu.cn/docker-ce/linux/ubuntu/ \
  $(lsb_release -cs) \
  stable"

sudo apt-get update
```

安装最新docker：

```bash
# 安装最新版本的 Docker Engine-Community 和 containerd
sudo apt-get install docker-ce docker-ce-cli containerd.io -y

sudo groupadd docker
sudo gpasswd -a $USER docker


# 使用systemd代替cgroupfs、以及配置仓库镜像地址
cat <<EOF | sudo tee /etc/docker/daemon.json
{
   "registry-mirrors":["https://reg-mirror.qiniu.com/"],
   "exec-opts": ["native.cgroupdriver=systemd"]
}
EOF
systemctl daemon-reload
systemctl restart docker
systemctl enable docker
docker info | grep systemd
```

# 3. 安装准备

kubernetes地址：https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.23.md

```bash
mkdir -p $HOME/k8s-install && cd $HOME/k8s-install
K8S_VERSION="v1.23.17"
wget https://dl.k8s.io/$K8S_VERSION/kubernetes-server-linux-amd64.tar.gz
tar zxvf kubernetes-server-linux-amd64.tar.gz

cd kubernetes/server/bin
cp kube-apiserver kube-scheduler kube-controller-manager kubectl kubelet kube-proxy /usr/bin

# 命令补全
apt install -y bash-completion
source /usr/share/bash-completion/bash_completion
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc
```

# 4. TLS 证书

## 4.1 证书工具

链接：https://github.com/cloudflare/cfssl

```bash
wget -t 0 -c "https://github.com/cloudflare/cfssl/releases/download/v1.6.1/cfssl_1.6.1_linux_amd64" -O /usr/local/bin/cfssl
wget -t 0 -c "https://github.com/cloudflare/cfssl/releases/download/v1.6.1/cfssljson_1.6.1_linux_amd64" -O /usr/local/bin/cfssljson
wget -t 0 -c "https://github.com/cloudflare/cfssl/releases/download/v1.6.1/cfssl-certinfo_1.6.1_linux_amd64" -O /usr/local/bin/cfssl-certinfo

chmod +x /usr/local/bin/cfssl /usr/local/bin/cfssljson /usr/local/bin/cfssl-certinfo
```

## 4.2 证书归类

**生成的 CA 证书和秘钥文件如下：**

| **组件**           | **证书**                            | **密钥**                                    | **备注**         |
| ------------------ | ----------------------------------- | ------------------------------------------- | ---------------- |
| etcd               | ca.pem、etcd.pem                    | etcd-key.pem                                |                  |
| apiserver          | ca.pem、apiserver.pem               | apiserver-key.pem                           |                  |
| controller-manager | ca.pem、kube-controller-manager.pem | ca-key.pem、kube-controller-manager-key.pem | kubeconfig       |
| scheduler          | ca.pem、kube-scheduler.pem          | kube-scheduler-key.pem                      | kubeconfig       |
| kubelet            | ca.pem                              |                                             | kubeconfig+token |
| kube-proxy         | ca.pem、kube-proxy.pem              | kube-proxy-key.pem                          | kubeconfig       |
| kubectl            | ca.pem、admin.pem                   | kube-proxy-key.pem                          |                  |

## 4.3 CA 证书创建

```bash
mkdir -p /root/k8s/ssl && cd /root/k8s/ssl
cat > admin-csr.json <<"EOF" 
{
  "CN": "admin",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Beijing",
      "L": "Beijing",
      "O": "system:masters",
      "OU": "Kubernetes-manual"
    }
  ]
}
EOF

cat > ca-config.json <<"EOF" 
{
  "signing": {
    "default": {
      "expiry": "876000h"
    },
    "profiles": {
      "kubernetes": {
        "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ],
        "expiry": "876000h"
      }
    }
  }
}
EOF

cat > etcd-ca-csr.json  <<"EOF" 
{
  "CN": "etcd",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Beijing",
      "L": "Beijing",
      "O": "etcd",
      "OU": "Etcd Security"
    }
  ],
  "ca": {
    "expiry": "876000h"
  }
}
EOF

cat > front-proxy-ca-csr.json  <<"EOF" 
{
  "CN": "kubernetes",
  "key": {
     "algo": "rsa",
     "size": 2048
  },
  "ca": {
    "expiry": "876000h"
  }
}
EOF

cat > kubelet-csr.json  <<"EOF" 
{
  "CN": "system:node:\$NODE",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "Beijing",
      "ST": "Beijing",
      "O": "system:nodes",
      "OU": "Kubernetes-manual"
    }
  ]
}
EOF

cat > manager-csr.json <<"EOF" 
{
  "CN": "system:kube-controller-manager",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Beijing",
      "L": "Beijing",
      "O": "system:kube-controller-manager",
      "OU": "Kubernetes-manual"
    }
  ]
}
EOF

cat > apiserver-csr.json <<"EOF" 
{
  "CN": "kube-apiserver",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Beijing",
      "L": "Beijing",
      "O": "Kubernetes",
      "OU": "Kubernetes-manual"
    }
  ]
}
EOF


cat > ca-csr.json <<"EOF" 
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Beijing",
      "L": "Beijing",
      "O": "Kubernetes",
      "OU": "Kubernetes-manual"
    }
  ],
  "ca": {
    "expiry": "876000h"
  }
}
EOF

cat > etcd-csr.json <<"EOF" 
{
  "CN": "etcd",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Beijing",
      "L": "Beijing",
      "O": "etcd",
      "OU": "Etcd Security"
    }
  ]
}
EOF

cat > front-proxy-client-csr.json <<"EOF" 
{
  "CN": "front-proxy-client",
  "key": {
     "algo": "rsa",
     "size": 2048
  }
}
EOF

cat > kube-proxy-csr.json <<"EOF"
{
  "CN": "system:kube-proxy",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Beijing",
      "L": "Beijing",
      "O": "system:kube-proxy",
      "OU": "Kubernetes-manual"
    }
  ]
}
EOF

cat > scheduler-csr.json <<"EOF" 
{
  "CN": "system:kube-scheduler",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Beijing",
      "L": "Beijing",
      "O": "system:kube-scheduler",
      "OU": "Kubernetes-manual"
    }
  ]
}
EOF
```

## 4.4 bootstrap

```bash
mkdir -p /root/k8s/bootstrap && cd /root/k8s/bootstrap
cat > bootstrap.secret.yaml << EOF
apiVersion: v1
kind: Secret
metadata:
  name: bootstrap-token-c8ad9c
  namespace: kube-system
type: bootstrap.kubernetes.io/token
stringData:
  description: "The default bootstrap token generated by 'kubelet '."
  token-id: c8ad9c
  token-secret: 2e4d610cf3e7426e
  usage-bootstrap-authentication: "true"
  usage-bootstrap-signing: "true"
  auth-extra-groups:  system:bootstrappers:default-node-token,system:bootstrappers:worker,system:bootstrappers:ingress
 
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kubelet-bootstrap
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:node-bootstrapper
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: system:bootstrappers:default-node-token
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: node-autoapprove-bootstrap
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:certificates.k8s.io:certificatesigningrequests:nodeclient
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: system:bootstrappers:default-node-token
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: node-autoapprove-certificate-rotation
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:certificates.k8s.io:certificatesigningrequests:selfnodeclient
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: system:nodes
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
      - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
    verbs:
      - "*"
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:kube-apiserver
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-apiserver-to-kubelet
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kube-apiserver
EOF
```

## 4.5 metrics-server

```bash
mkdir -p /root/k8s/metrics-server && cd  /root/k8s/metrics-server
cat > metrics-server.yaml << EOF 
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: metrics-server
  name: metrics-server
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    k8s-app: metrics-server
    rbac.authorization.k8s.io/aggregate-to-admin: "true"
    rbac.authorization.k8s.io/aggregate-to-edit: "true"
    rbac.authorization.k8s.io/aggregate-to-view: "true"
  name: system:aggregated-metrics-reader
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
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    k8s-app: metrics-server
  name: system:metrics-server
rules:
- apiGroups:
  - ""
  resources:
  - pods
  - nodes
  - nodes/stats
  - namespaces
  - configmaps
  verbs:
  - get
  - list
  - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  labels:
    k8s-app: metrics-server
  name: metrics-server-auth-reader
  namespace: kube-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: extension-apiserver-authentication-reader
subjects:
- kind: ServiceAccount
  name: metrics-server
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  labels:
    k8s-app: metrics-server
  name: metrics-server:system:auth-delegator
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:auth-delegator
subjects:
- kind: ServiceAccount
  name: metrics-server
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  labels:
    k8s-app: metrics-server
  name: system:metrics-server
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:metrics-server
subjects:
- kind: ServiceAccount
  name: metrics-server
  namespace: kube-system
---
apiVersion: v1
kind: Service
metadata:
  labels:
    k8s-app: metrics-server
  name: metrics-server
  namespace: kube-system
spec:
  ports:
  - name: https
    port: 443
    protocol: TCP
    targetPort: https
  selector:
    k8s-app: metrics-server
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    k8s-app: metrics-server
  name: metrics-server
  namespace: kube-system
spec:
  selector:
    matchLabels:
      k8s-app: metrics-server
  strategy:
    rollingUpdate:
      maxUnavailable: 0
  template:
    metadata:
      labels:
        k8s-app: metrics-server
    spec:
      containers:
      - args:
        - --cert-dir=/tmp
        - --secure-port=4443
        - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
        - --kubelet-use-node-status-port
        - --metric-resolution=15s
        - --kubelet-insecure-tls
        - --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.pem # change to front-proxy-ca.crt for kubeadm
        - --requestheader-username-headers=X-Remote-User
        - --requestheader-group-headers=X-Remote-Group
        - --requestheader-extra-headers-prefix=X-Remote-Extra-
        image: registry.cn-beijing.aliyuncs.com/dotbalo/metrics-server:0.5.0
        imagePullPolicy: IfNotPresent
        livenessProbe:
          failureThreshold: 3
          httpGet:
            path: /livez
            port: https
            scheme: HTTPS
          periodSeconds: 10
        name: metrics-server
        ports:
        - containerPort: 4443
          name: https
          protocol: TCP
        readinessProbe:
          failureThreshold: 3
          httpGet:
            path: /readyz
            port: https
            scheme: HTTPS
          initialDelaySeconds: 20
          periodSeconds: 10
        resources:
          requests:
            cpu: 100m
            memory: 200Mi
        securityContext:
          readOnlyRootFilesystem: true
          runAsNonRoot: true
          runAsUser: 1000
        volumeMounts:
        - mountPath: /tmp
          name: tmp-dir
        - name: ca-ssl
          mountPath: /etc/kubernetes/pki
      nodeSelector:
        kubernetes.io/os: linux
      priorityClassName: system-cluster-critical
      serviceAccountName: metrics-server
      volumes:
      - emptyDir: {}
        name: tmp-dir
      - name: ca-ssl
        hostPath:
          path: /etc/kubernetes/pki
---
apiVersion: apiregistration.k8s.io/v1
kind: APIService
metadata:
  labels:
    k8s-app: metrics-server
  name: v1beta1.metrics.k8s.io
spec:
  group: metrics.k8s.io
  groupPriorityMinimum: 100
  insecureSkipTLSVerify: true
  service:
    name: metrics-server
    namespace: kube-system
  version: v1beta1
  versionPriority: 100
EOF
```

## 4.6 生成etcd证书

特别说明除外，以下操作在所有master节点操作

### 4.6.1所有master节点创建证书存放目录

```bash
mkdir /etc/etcd/ssl -p
```

### 4.6.2 master01节点生成etcd证书

```bash
mkdir -p /root/k8s/ssl && cd /root/k8s/ssl
# 生成etcd证书和etcd证书的key（如果你觉得以后可能会扩容，可以在ip那多写几个预留出来）
# 若没有IPv6 可删除可保留 
cfssl gencert -initca etcd-ca-csr.json | cfssljson -bare /etc/etcd/ssl/etcd-ca
cfssl gencert \
   -ca=/etc/etcd/ssl/etcd-ca.pem \
   -ca-key=/etc/etcd/ssl/etcd-ca-key.pem \
   -config=ca-config.json \
   -hostname=127.0.0.1,k8s-master01,k8s-node01,k8s-node02,$K8S_MASTER01,$K8S_NODE01,$K8S_NODE02  \
   -profile=kubernetes \
   etcd-csr.json | cfssljson -bare /etc/etcd/ssl/etcd
   
# 我们将etcd放在了k8s-master01、k8s-node01、k8s-node02中，上面的地址关联
ls /etc/etcd/ssl/etcd*
```

**注：这里将etcd部署在`master01`、`node01`、`node02`上，因为前期我们部署的是一个单master集群，后期master02和master03才会加入进来。** 

### 4.6.3 将证书复制到其他节点

```bash
Master='k8s-node01 k8s-node02'

for NODE in $Master; 
do
    ssh $NODE "mkdir -p /etc/etcd/ssl"; 
    for FILE in etcd-ca-key.pem  etcd-ca.pem  etcd-key.pem  etcd.pem;
    do
        scp /etc/etcd/ssl/${FILE} $NODE:/etc/etcd/ssl/${FILE}; 
    done;
done
```

## 4.7 生成k8s相关证书

### 4.7.1 所有k8s节点创建证书存放目录

```bash
mkdir -p /etc/kubernetes/pki
```

### 4.7.2 master01节点生成k8s证书

```bash
cfssl gencert -initca ca-csr.json | cfssljson -bare /etc/kubernetes/pki/ca

# 生成一个根证书 ，多写了一些IP作为预留IP，为将来添加node做准备
# 10.96.0.1是service网段的第一个地址，需要计算，192.168.80.100为高可用vip地址
# 若没有IPv6 可删除可保留 

cfssl gencert   \
-ca=/etc/kubernetes/pki/ca.pem   \
-ca-key=/etc/kubernetes/pki/ca-key.pem   \
-config=ca-config.json   \
-hostname=10.96.0.1,192.168.80.100,127.0.0.1,kubernetes,kubernetes.default,kubernetes.default.svc,kubernetes.default.svc.cluster,kubernetes.default.svc.cluster.local,192.168.80.45,192.168.80.46,192.168.80.47,192.168.80.48,192.168.80.49,192.168.80.50,192.168.80.51,192.168.80.52,192.168.80.53,192.168.80.54,10.0.0.40,10.0.0.41 \
-profile=kubernetes   apiserver-csr.json | cfssljson -bare /etc/kubernetes/pki/apiserver

ls /etc/kubernetes/pki/apiserver*
```

### 4.7.3 生成apiserver聚合证书

```bash
cfssl gencert -initca front-proxy-ca-csr.json | cfssljson -bare /etc/kubernetes/pki/front-proxy-ca 

# 有一个警告，可以忽略
cfssl gencert  \
-ca=/etc/kubernetes/pki/front-proxy-ca.pem   \
-ca-key=/etc/kubernetes/pki/front-proxy-ca-key.pem   \
-config=ca-config.json   \
-profile=kubernetes   front-proxy-client-csr.json | cfssljson -bare /etc/kubernetes/pki/front-proxy-client
```

### 4.7.4 生成controller-manage的证书

```bash
cfssl gencert \
   -ca=/etc/kubernetes/pki/ca.pem \
   -ca-key=/etc/kubernetes/pki/ca-key.pem \
   -config=ca-config.json \
   -profile=kubernetes \
   manager-csr.json | cfssljson -bare /etc/kubernetes/pki/controller-manager

# 设置一个集群项
kubectl config set-cluster kubernetes \
     --certificate-authority=/etc/kubernetes/pki/ca.pem \
     --embed-certs=true \
     --server=https://$K8S_MASTER01:6443 \
     --kubeconfig=/etc/kubernetes/controller-manager.kubeconfig

# 设置一个环境项，一个上下文
kubectl config set-context system:kube-controller-manager@kubernetes \
    --cluster=kubernetes \
    --user=system:kube-controller-manager \
    --kubeconfig=/etc/kubernetes/controller-manager.kubeconfig

# 设置一个用户项
kubectl config set-credentials system:kube-controller-manager \
     --client-certificate=/etc/kubernetes/pki/controller-manager.pem \
     --client-key=/etc/kubernetes/pki/controller-manager-key.pem \
     --embed-certs=true \
     --kubeconfig=/etc/kubernetes/controller-manager.kubeconfig

# 设置默认环境
kubectl config use-context system:kube-controller-manager@kubernetes \
     --kubeconfig=/etc/kubernetes/controller-manager.kubeconfig

cfssl gencert \
   -ca=/etc/kubernetes/pki/ca.pem \
   -ca-key=/etc/kubernetes/pki/ca-key.pem \
   -config=ca-config.json \
   -profile=kubernetes \
   scheduler-csr.json | cfssljson -bare /etc/kubernetes/pki/scheduler

kubectl config set-cluster kubernetes \
     --certificate-authority=/etc/kubernetes/pki/ca.pem \
     --embed-certs=true \
     --server=https://$K8S_MASTER01:6443 \
     --kubeconfig=/etc/kubernetes/scheduler.kubeconfig

kubectl config set-credentials system:kube-scheduler \
     --client-certificate=/etc/kubernetes/pki/scheduler.pem \
     --client-key=/etc/kubernetes/pki/scheduler-key.pem \
     --embed-certs=true \
     --kubeconfig=/etc/kubernetes/scheduler.kubeconfig

kubectl config set-context system:kube-scheduler@kubernetes \
     --cluster=kubernetes \
     --user=system:kube-scheduler \
     --kubeconfig=/etc/kubernetes/scheduler.kubeconfig

kubectl config use-context system:kube-scheduler@kubernetes \
     --kubeconfig=/etc/kubernetes/scheduler.kubeconfig

cfssl gencert \
   -ca=/etc/kubernetes/pki/ca.pem \
   -ca-key=/etc/kubernetes/pki/ca-key.pem \
   -config=ca-config.json \
   -profile=kubernetes \
   admin-csr.json | cfssljson -bare /etc/kubernetes/pki/admin

kubectl config set-cluster kubernetes     \
  --certificate-authority=/etc/kubernetes/pki/ca.pem     \
  --embed-certs=true     \
  --server=https://$K8S_MASTER01:6443     \
  --kubeconfig=/etc/kubernetes/admin.kubeconfig

kubectl config set-credentials kubernetes-admin  \
  --client-certificate=/etc/kubernetes/pki/admin.pem     \
  --client-key=/etc/kubernetes/pki/admin-key.pem     \
  --embed-certs=true     \
  --kubeconfig=/etc/kubernetes/admin.kubeconfig

kubectl config set-context kubernetes-admin@kubernetes    \
  --cluster=kubernetes     \
  --user=kubernetes-admin     \
  --kubeconfig=/etc/kubernetes/admin.kubeconfig

kubectl config use-context kubernetes-admin@kubernetes  --kubeconfig=/etc/kubernetes/admin.kubeconfig
```

### 4.7.5 创建kube-proxy证书

```bash
cfssl gencert \
   -ca=/etc/kubernetes/pki/ca.pem \
   -ca-key=/etc/kubernetes/pki/ca-key.pem \
   -config=ca-config.json \
   -profile=kubernetes \
   kube-proxy-csr.json | cfssljson -bare /etc/kubernetes/pki/kube-proxy
   
kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/pki/ca.pem  \
  --embed-certs=true \
  --server=https://$K8S_MASTER01:6443     \
  --kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig

kubectl config set-credentials kube-proxy  \
  --client-certificate=/etc/kubernetes/pki/kube-proxy.pem     \
  --client-key=/etc/kubernetes/pki/kube-proxy-key.pem     \
  --embed-certs=true     \
  --kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig

kubectl config set-context kube-proxy@kubernetes    \
  --cluster=kubernetes     \
  --user=kube-proxy     \
  --kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig

kubectl config use-context kube-proxy@kubernetes  --kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig
```

### 4.7.6 创建ServiceAccount Key ——secret

```bash
openssl genrsa -out /etc/kubernetes/pki/sa.key 2048
openssl rsa -in /etc/kubernetes/pki/sa.key -pubout -out /etc/kubernetes/pki/sa.pub
```

### 4.7.7 将证书发送到其他master节点

```bash
for NODE in k8s-node01 k8s-node02; 
do
  ssh root@$NODE "mkdir -p /etc/kubernetes/pki"
  for FILE in $(ls /etc/kubernetes/pki | grep -v etcd); 
  do
      scp /etc/kubernetes/pki/${FILE} $NODE:/etc/kubernetes/pki/${FILE}; 
  done;
  for FILE in admin.kubeconfig controller-manager.kubeconfig scheduler.kubeconfig; 
  do
      scp /etc/kubernetes/${FILE} $NODE:/etc/kubernetes/${FILE}; 
  done;
done
```

# 5. 部署etcd

## 5.1 etcd下载

先在一个节点上下载，在copy到其他节点。

华为国内地址：https://mirrors.huaweicloud.com/etcd/v3.5.4/etcd-v3.5.4-linux-amd64.tar.gz

GitHub：https://github.com/etcd-io/etcd/releases/download/v3.5.4/etcd-v3.5.4-linux-amd64.tar.gz

```bash
mkdir -p $HOME/k8s-install && cd $HOME/k8s-install
# Desc:
ETCDCTL_API=3
ETCD_VER=v3.5.4
ETCD_DIR=etcd-download
DOWNLOAD_URL=https://mirrors.huaweicloud.com/etcd

# Download
mkdir -p  ${ETCD_DIR}
cd ${ETCD_DIR}
rm -rf etcd-${ETCD_VER}-linux-amd64.tar.gz  etcd-${ETCD_VER}-linux-amd64
wget ${DOWNLOAD_URL}/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz
tar -xzvf etcd-${ETCD_VER}-linux-amd64.tar.gz

# Install
mv etcd-${ETCD_VER}-linux-amd64/{etcd,etcdctl} /usr/bin/
mkdir -p /etc/etcd

# 4. 准备克隆文件
tar zcvf etcd-clone.tar.gz /usr/bin/etcd*
scp etcd-clone.tar.gz root@k8s-node01:/root
scp etcd-clone.tar.gz root@k8s-node02:/root

# 5.其他节点
for ip in k8s-node01 k8s-node02;
do
    ssh root@$ip "tar zxvf /root/etcd-clone.tar.gz -C / && mkdir -p /etc/etcd"
done;
```

## 5.2 etcd配置（所有etcd节点）

### 5.2.1 配置文件内容

```bash
cd
if [ $LOCALHOST == $K8S_MASTER01 ];then
   echo "etcd='k8s-etcd01'" >>  ~/.bashrc
elif [ $LOCALHOST == $K8S_NODE01 ];then
   echo "etcd='k8s-etcd02'" >>  ~/.bashrc
else
  echo "etcd='k8s-etcd03'" >>  ~/.bashrc
fi
source ~/.bashrc

cat > /etc/etcd/etcd.config.yml << EOF
name: $etcd
data-dir: /var/lib/etcd
wal-dir: /var/lib/etcd/wal
snapshot-count: 5000
heartbeat-interval: 100
election-timeout: 1000
quota-backend-bytes: 0
listen-peer-urls: "https://$LOCALHOST:2380"
listen-client-urls: "https://$LOCALHOST:2379,http://127.0.0.1:2379"
max-snapshots: 3
max-wals: 5
cors:
initial-advertise-peer-urls: "https://$LOCALHOST:2380"
advertise-client-urls: "https://$LOCALHOST:2379"
discovery:
discovery-fallback: 'proxy'
discovery-proxy:
discovery-srv:
initial-cluster: "k8s-etcd01=https://$K8S_MASTER01:2380,k8s-etcd02=https://$K8S_NODE01:2380,k8s-etcd03=https://$K8S_NODE02:2380"
initial-cluster-token: 'etcd-k8s-cluster'
initial-cluster-state: 'new'
strict-reconfig-check: false
enable-v2: true
enable-pprof: true
proxy: 'off'
proxy-failure-wait: 5000
proxy-refresh-interval: 30000
proxy-dial-timeout: 1000
proxy-write-timeout: 5000
proxy-read-timeout: 0
client-transport-security:
  cert-file: '/etc/kubernetes/pki/etcd/etcd.pem'
  key-file: '/etc/kubernetes/pki/etcd/etcd-key.pem'
  client-cert-auth: true
  trusted-ca-file: '/etc/kubernetes/pki/etcd/etcd-ca.pem'
  auto-tls: true
peer-transport-security:
  cert-file: '/etc/kubernetes/pki/etcd/etcd.pem'
  key-file: '/etc/kubernetes/pki/etcd/etcd-key.pem'
  peer-client-cert-auth: true
  trusted-ca-file: '/etc/kubernetes/pki/etcd/etcd-ca.pem'
  auto-tls: true
debug: false
log-package-levels:
log-outputs: [default]
force-new-cluster: false
EOF
```

## 5.3 创建service（所有etcd节点）

```bash
cat > /usr/lib/systemd/system/etcd.service <<"EOF"

[Unit]
Description=Etcd Service
Documentation=https://coreos.com/etcd/docs/latest/
After=network.target

[Service]
Type=notify
ExecStart=/usr/bin/etcd --config-file=/etc/etcd/etcd.config.yml
Restart=on-failure
RestartSec=10
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
Alias=etcd3.service

EOF
```

## 5.4 启动etcd

```bash
mkdir -p /etc/kubernetes/pki/etcd
ln -s /etc/etcd/ssl/* /etc/kubernetes/pki/etcd/
systemctl daemon-reload
# 同时启动，在查看下状态
systemctl enable etcd.service
systemctl start etcd.service
systemctl status etcd.service
# 查看状态，要是有报错就重启下
# sudo systemctl restart etcd.service
```

正确的日志：

```bash
etcd.service - Etcd Service
     Loaded: loaded (/lib/systemd/system/etcd.service; enabled; vendor preset: enabled)
     Active: active (running) since Fri 2023-03-17 21:08:52 CST; 45s ago
       Docs: https://coreos.com/etcd/docs/latest/
   Main PID: 142936 (etcd)
      Tasks: 8 (limit: 4573)
     Memory: 16.8M
        CPU: 1.466s
     CGroup: /system.slice/etcd.service
             └─142936 /usr/bin/etcd --config-file=/etc/etcd/etcd.config.yml

3月 17 21:08:52 k8s-master01 etcd[142936]: {"level":"info","ts":"2023-03-17T21:08:52.735+0800","caller":"etcdmain/main.go:50","msg":"successfully notified init daemon"}
3月 17 21:08:52 k8s-master01 etcd[142936]: {"level":"info","ts":"2023-03-17T21:08:52.735+0800","caller":"embed/serve.go:98","msg":"ready to serve client requests"}
3月 17 21:08:52 k8s-master01 systemd[1]: Started Etcd Service.
3月 17 21:08:52 k8s-master01 etcd[142936]: {"level":"info","ts":"2023-03-17T21:08:52.734+0800","caller":"embed/serve.go:140","msg":"serving client traffic insecurely; this is strongly discouraged!","address":"127.0.0.1:2379"}
3月 17 21:08:52 k8s-master01 etcd[142936]: {"level":"info","ts":"2023-03-17T21:08:52.742+0800","caller":"etcdserver/server.go:775","msg":"initialized peer connections; fast-forwarding election ticks","local-member-id":"aeceb08f95191d44","forward-ticks":8,"forward-duration":"800ms","election-ticks":10,"election-timeout":"1s","active-remote-members":2}
3月 17 21:08:52 k8s-master01 etcd[142936]: {"level":"info","ts":"2023-03-17T21:08:52.745+0800","caller":"embed/serve.go:188","msg":"serving client traffic securely","address":"192.168.80.45:2379"}
3月 17 21:08:52 k8s-master01 etcd[142936]: {"level":"info","ts":"2023-03-17T21:08:52.747+0800","caller":"rafthttp/stream.go:249","msg":"set message encoder","from":"aeceb08f95191d44","to":"f86b896d1acd8ebe","stream-type":"stream Message"}
3月 17 21:08:52 k8s-master01 etcd[142936]: {"level":"info","ts":"2023-03-17T21:08:52.747+0800","caller":"rafthttp/stream.go:274","msg":"established TCP streaming connection with remote peer","stream-writer-type":"stream Message","local-member-id":"aeceb08f95191d44","remote-peer-id":"f86b896d1acd8ebe"}
3月 17 21:08:52 k8s-master01 etcd[142936]: {"level":"info","ts":"2023-03-17T21:08:52.756+0800","caller":"rafthttp/stream.go:249","msg":"set message encoder","from":"aeceb08f95191d44","to":"f86b896d1acd8ebe","stream-type":"stream MsgApp v2"}
3月 17 21:08:52 k8s-master01 etcd[142936]: {"level":"info","ts":"2023-03-17T21:08:52.756+0800","caller":"rafthttp/stream.go:274","msg":"established TCP streaming connection with remote peer","stream-writer-type":"stream MsgApp v2","local-member-id":"aeceb08f95191d44","remote-peer-id":"f86b896d1acd8ebe"}
~
```

## 5.5 查看etcd状态

```bash
export ETCDCTL_API=3
# 1.查看状态
etcdctl --endpoints="$K8S_MASTER01:2379,$K8S_NODE01:2379,$K8S_NODE02:2379" --cacert=/etc/kubernetes/pki/etcd/etcd-ca.pem --cert=/etc/kubernetes/pki/etcd/etcd.pem --key=/etc/kubernetes/pki/etcd/etcd-key.pem  endpoint status --write-out=table
+--------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|      ENDPOINT      |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+--------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| 192.168.80.45:2379 | aeceb08f95191d44 |   3.5.4 |   29 kB |     false |      false |         4 |         14 |                 14 |        |
| 192.168.80.48:2379 | f86b896d1acd8ebe |   3.5.4 |   25 kB |      true |      false |         4 |         14 |                 14 |        |
| 192.168.80.49:2379 |  c397b48e51c62c9 |   3.5.4 |   20 kB |     false |      false |         4 |         14 |                 14 |        |
+--------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+

# 2.查看健康
etcdctl endpoint health --endpoints="$K8S_MASTER01:2379,$K8S_NODE01:2379,$K8S_NODE02:2379"  --cacert=/etc/kubernetes/pki/etcd/etcd-ca.pem --cert=/etc/kubernetes/pki/etcd/etcd.pem --key=/etc/kubernetes/pki/etcd/etcd-key.pem --cluster --write-out=table
+----------------------------+--------+-------------+-------+
|          ENDPOINT          | HEALTH |    TOOK     | ERROR |
+----------------------------+--------+-------------+-------+
| https://192.168.80.45:2379 |   true | 28.243526ms |       |
| https://192.168.80.48:2379 |   true |  28.53843ms |       |
| https://192.168.80.49:2379 |   true | 45.906904ms |       |
+----------------------------+--------+-------------+-------+
```

# 6. Master 节点

kubernetes master 节点组件：

- kube-apiserver
- kube-scheduler
- kube-controller-manager
- docker
- kubelet （非必须，但必要）
- kube-proxy（非必须，但必要）

## 6.1 apiserver

### 6.1.1 TLS Bootstrapping Token

**启用 TLS Bootstrapping 机制：**

TLS Bootstraping：Master apiserver启用TLS认证后，Node节点kubelet和kube-proxy要与kube-apiserver进行通信，必须使用CA签发的有效证书才可以，当Node节点很多时，这种客户端证书颁发需要大量工作，同样也会增加集群扩展复杂度。为了简化流程，Kubernetes引入了TLS bootstraping机制来自动颁发客户端证书，kubelet会以一个低权限用户自动向apiserver申请证书，kubelet的证书由apiserver动态签署。所以强烈建议在Node上使用这种方式，目前主要用于kubelet，kube-proxy还是由我们统一颁发一个证书。

`TLS bootstraping` 工作流程：

![img](https://img2023.cnblogs.com/blog/1740081/202303/1740081-20230318113436318-1444892595.png)

```bash
BOOTSTRAP_TOKEN=$(head -c 16 /dev/urandom | od -An -t x | tr -d ' ')
9263997028b08c1674adecf85b4b7e42

# 格式：token，用户名，UID，用户组
cat > /etc/kubernetes/token.csv << EOF
${BOOTSTRAP_TOKEN},kubelet-bootstrap,10001,"system:node-bootstrapper"
EOF
```

### 6.1.2 配置文件

`--service-cluster-ip-range=10.96.0.0/12`: Service IP 段

```bash
cat > /usr/lib/systemd/system/kube-apiserver.service << EOF

[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
ExecStart=/usr/bin/kube-apiserver \\
      --v=2  \\
      --logtostderr=true  \\
      --allow-privileged=true  \\
      --bind-address=0.0.0.0  \\
      --secure-port=6443  \\
      --insecure-port=0  \\
      --advertise-address=$LOCALHOST \\
      --service-cluster-ip-range=10.96.0.0/12 \\
      --feature-gates=IPv6DualStack=true \\
      --service-node-port-range=30000-32767  \\
      --etcd-servers=https://$K8S_MASTER01:2379,https://$K8S_NODE01:2379,https://$K8S_NODE02:2379 \\
      --etcd-cafile=/etc/etcd/ssl/etcd-ca.pem  \\
      --etcd-certfile=/etc/etcd/ssl/etcd.pem  \\
      --etcd-keyfile=/etc/etcd/ssl/etcd-key.pem  \\
      --client-ca-file=/etc/kubernetes/pki/ca.pem  \\
      --tls-cert-file=/etc/kubernetes/pki/apiserver.pem  \\
      --tls-private-key-file=/etc/kubernetes/pki/apiserver-key.pem  \\
      --kubelet-client-certificate=/etc/kubernetes/pki/apiserver.pem  \\
      --kubelet-client-key=/etc/kubernetes/pki/apiserver-key.pem  \\
      --service-account-key-file=/etc/kubernetes/pki/sa.pub  \\
      --service-account-signing-key-file=/etc/kubernetes/pki/sa.key  \\
      --service-account-issuer=https://kubernetes.default.svc.cluster.local \\
      --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname  \\
      --enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,NodeRestriction,ResourceQuota  \
      --authorization-mode=Node,RBAC  \\
      --enable-bootstrap-token-auth=true  \\
      --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.pem  \\
      --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.pem  \\
      --proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client-key.pem  \\
      --requestheader-allowed-names=aggregator  \\
      --requestheader-group-headers=X-Remote-Group  \\
      --requestheader-extra-headers-prefix=X-Remote-Extra-  \\
      --requestheader-username-headers=X-Remote-User \\
      --enable-aggregator-routing=true \\
      --token-auth-file=/etc/kubernetes/token.csv

Restart=on-failure
RestartSec=10s
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target

EOF
```

### 6.1.3 开机启动

```bash
systemctl daemon-reload && \
systemctl start kube-apiserver && \
systemctl enable kube-apiserver

# 注意查看状态是否启动正常

systemctl status kube-apiserver
# sudo systemctl restart kube-apiserver
```

## 6.2 配置kube-controller-manager service

### 6.2.1 kubeconfig 文件

```bash
# 所有master节点配置，且配置相同
# 172.16.0.0/12为pod网段，按需求设置你自己的网段

cat > /usr/lib/systemd/system/kube-controller-manager.service << EOF
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
ExecStart=/usr/bin/kube-controller-manager \
      --v=2 \
      --logtostderr=true \
      --bind-address=127.0.0.1 \
      --root-ca-file=/etc/kubernetes/pki/ca.pem \
      --cluster-signing-cert-file=/etc/kubernetes/pki/ca.pem \
      --cluster-signing-key-file=/etc/kubernetes/pki/ca-key.pem \
      --service-account-private-key-file=/etc/kubernetes/pki/sa.key \
      --kubeconfig=/etc/kubernetes/controller-manager.kubeconfig \
      --leader-elect=true \
      --use-service-account-credentials=true \
      --node-monitor-grace-period=40s \
      --node-monitor-period=5s \
      --pod-eviction-timeout=2m0s \
      --controllers=*,bootstrapsigner,tokencleaner \
      --allocate-node-cidrs=true \
      --service-cluster-ip-range=10.96.0.0/12,fd00::/108 \
      --cluster-cidr=172.16.0.0/12,fc00::/48 \
      --node-cidr-mask-size-ipv4=24 \
      --node-cidr-mask-size-ipv6=64 \
      --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.pem 
      # --feature-gates=IPv6DualStack=true #要是没有配置ipv6就不要打开，打开会出现 Get "http://127.0.0.1:10252/healthz": dial tcp 127.0.0.1:10252: connect: connection refused 报错

Restart=always
RestartSec=10s

[Install]
WantedBy=multi-user.target

EOF
```

### 6.2.2 启动kube-controller-manager，并查看状态

```bash
systemctl daemon-reload && \
systemctl enable kube-controller-manager && \
systemctl start kube-controller-manager 
# systemctl  status kube-controller-manager
```

## 6.3 配置kube-scheduler service

### 6.3.1 所有master节点配置，且配置相同

```bash
cat > /usr/lib/systemd/system/kube-scheduler.service <<EOF

[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
ExecStart=/usr/bin/kube-scheduler \\
      --v=2 \\
      --logtostderr=true \\
      --address=127.0.0.1 \\
      --leader-elect=true \\
      --kubeconfig=/etc/kubernetes/scheduler.kubeconfig

Restart=always
RestartSec=10s

[Install]
WantedBy=multi-user.target

EOF
```

### 6.3.2 启动并查看服务状态

```bash
systemctl daemon-reload && \
systemctl enable kube-scheduler.service && \
systemctl start kube-scheduler.service
# systemctl status kube-scheduler
```

# 7. TLS Bootstrapping配置

## 7.1 在master01上配置

```bash
cd /root/k8s/bootstrap

kubectl config set-cluster kubernetes     \
--certificate-authority=/etc/kubernetes/pki/ca.pem     \
--embed-certs=true     \
--server=https://$K8S_MASTER01:6443     \
--kubeconfig=/etc/kubernetes/bootstrap-kubelet.kubeconfig

kubectl config set-credentials tls-bootstrap-token-user     \
--token=c8ad9c.2e4d610cf3e7426e \
--kubeconfig=/etc/kubernetes/bootstrap-kubelet.kubeconfig
#token 在bootstrap.secret.yaml文件内对应

kubectl config set-context tls-bootstrap-token-user@kubernetes     \
--cluster=kubernetes     \
--user=tls-bootstrap-token-user     \
--kubeconfig=/etc/kubernetes/bootstrap-kubelet.kubeconfig

kubectl config use-context tls-bootstrap-token-user@kubernetes     \
--kubeconfig=/etc/kubernetes/bootstrap-kubelet.kubeconfig

# token的位置在bootstrap.secret.yaml，如果修改的话到这个文件修改

mkdir -p /root/.kube ; cp /etc/kubernetes/admin.kubeconfig /root/.kube/config
```

## 7.2 查看集群状态，没问题的话继续后续操作

```bash
kubectl get cs
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS    MESSAGE                         ERROR
scheduler            Healthy   ok
controller-manager   Healthy   ok
etcd-0               Healthy   {"health":"true","reason":""}
etcd-1               Healthy   {"health":"true","reason":""}
etcd-2               Healthy   {"health":"true","reason":""}

cd /root/k8s/bootstrap
kubectl create -f bootstrap.secret.yaml
```

# 8. node节点配置

Kubernetes node节点组件：

- kubelet
- kube-proxy
- docker

## 8.1 复制证书和二进制文件

### 8.1.1 在master01上将证书复制到node节点

```bash
cd /etc/kubernetes/

for NODE in k8s-node01 k8s-node02; 
do
    ssh $NODE mkdir -p /etc/kubernetes/pki; 
    for FILE in pki/ca.pem pki/ca-key.pem pki/front-proxy-ca.pem bootstrap-kubelet.kubeconfig; 
    do
        scp /etc/kubernetes/$FILE $NODE:/etc/kubernetes/${FILE}; 
    done;
done
```

### 8.1.2 在master01上将二进制包复制到node节点

- master01操作

```bash
mkdir -p $HOME/k8s-install && cd $HOME/k8s-install

tar cvf worker-node-clone.tar.gz /usr/bin/{kubelet,kube-proxy,kubectl} /etc/kubernetes/kubelet* /etc/kubernetes/kube-proxy* /etc/kubernetes/pki /etc/kubernetes/bootstrap-kubelet.kubeconfig

scp worker-node-clone.tar.gz root@$K8S_NODE01:/root
scp worker-node-clone.tar.gz root@$K8S_NODE02:/root
for NODE in k8s-node01 k8s-node02;
do
    ssh root@$NODE "cd && tar xvf worker-node-clone.tar.gz -C / && rm -f worker-node-clone.tar.gz && rm -f /etc/kubernetes/kubelet.kubeconfig && rm -f /etc/kubernetes/pki/kubelet*" 
done
```

## 8.2 kubelet配置

### 8.2.1 cni插件安装

github下载地址：https://github.com/containernetworking/plugins/releases

```bash
mkdir -p $HOME/k8s-install/network && cd $_
CNI_VERSION="v1.1.1"
wget https://github.com/containernetworking/plugins/releases/download/$CNI_VERSION/cni-plugins-linux-amd64-$CNI_VERSION.tgz
#创建cni插件所需目录
mkdir -p /etc/cni/net.d /opt/cni/bin
#解压cni二进制包
tar xf cni-plugins-linux-amd64-$CNI_VERSION.tgz -C /opt/cni/bin/

for NODE in k8s-node01 k8s-node02;
do
    ssh $NODE mkdir -p /etc/cni/net.d /opt/cni/bin $HOME/k8s-install/network; 
    for FILE in cni-plugins-linux-amd64-$CNI_VERSION.tgz; 
    do
        scp cni-plugins-linux-amd64-$CNI_VERSION.tgz $NODE:$HOME/k8s-install/network;
        ssh $NODE "cd $HOME/k8s-install/network && tar xf cni-plugins-linux-amd64-$CNI_VERSION.tgz -C /opt/cni/bin/"
    done;
done
```

### 8.2.2 docker启动服务（本次使用的runtime）

- 所有节点操作

其中：`--kubeconfig=/etc/kubernetes/kubelet.kubeconfig` 在加入集群时自动生成

```bash
mkdir -p /var/lib/kubelet /var/log/kubernetes /etc/systemd/system/kubelet.service.d /etc/kubernetes/manifests/
HOSTNAME=`echo $(hostname)`

cat > /etc/kubernetes/kubelet.conf << EOF
KUBELET_OPTS="--logtostderr=false \\
--v=2 \\
--log-dir=/var/log/kubernetes \\
--hostname-override=$HOSTNAME \\
--network-plugin=cni \\
--container-runtime=docker \\
--kubeconfig=/etc/kubernetes/kubelet.kubeconfig \\
--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.kubeconfig \\
--config=/etc/kubernetes/kubelet-conf.yml \\
--container-runtime-endpoint=unix:///var/run/dockershim.sock \\
--cert-dir=/etc/kubernetes/pki \\
--node-labels=node.kubernetes.io/node='' \\
--pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/imges/pause:3.6"
EOF

cat > /lib/systemd/system/kubelet.service <<EOF
[Unit]
Description=Kubernetes Kubelet
After=docker.service
Requires=docker.service

[Service]
EnvironmentFile=/etc/kubernetes/kubelet.conf
ExecStart=/usr/bin/kubelet \$KUBELET_OPTS
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

### 8.2.3 containerd启动容器

```bash
mkdir -p /var/lib/kubelet /var/log/kubernetes /etc/systemd/system/kubelet.service.d /etc/kubernetes/manifests/

所有k8s节点配置kubelet service
cat > /usr/lib/systemd/system/kubelet.service <<EOF

[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=containerd.service
Requires=containerd.service

[Service]
ExecStart=/usr/bin/kubelet \\
    --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.kubeconfig  \\
    --kubeconfig=/etc/kubernetes/kubelet.kubeconfig \\
    --config=/etc/kubernetes/kubelet-conf.yml \\
    --network-plugin=cni  \\
    --cni-conf-dir=/etc/cni/net.d  \\
    --cni-bin-dir=/opt/cni/bin  \\
    --container-runtime=remote  \\
    --runtime-request-timeout=15m  \\
    --container-runtime-endpoint=unix:///run/containerd/containerd.sock  \\
    --cgroup-driver=systemd \\
    --node-labels=node.kubernetes.io/node=''
    #--feature-gates=IPv6DualStack=true

Restart=always
StartLimitInterval=0
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF
```

注意若是CentOS7，将 `--node-labels=node.kubernetes.io/node=''` 替换为 `--node-labels=node.kubernetes.io/node=` 将 `''` 删除

### 8.2.4 所有k8s节点创建kubelet的配置文件

```bash
cat > /etc/kubernetes/kubelet-conf.yml <<EOF
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
address: 0.0.0.0
port: 10250
readOnlyPort: 10255
authentication:
  anonymous:
    enabled: false
  webhook:
    cacheTTL: 2m0s
    enabled: true
  x509:
    clientCAFile: /etc/kubernetes/pki/ca.pem
authorization:
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: 5m0s
    cacheUnauthorizedTTL: 30s
cgroupDriver: systemd
cgroupsPerQOS: true
clusterDNS:
- 10.96.0.10
clusterDomain: cluster.local
containerLogMaxFiles: 5
containerLogMaxSize: 10Mi
contentType: application/vnd.kubernetes.protobuf
cpuCFSQuota: true
cpuManagerPolicy: none
cpuManagerReconcilePeriod: 10s
enableControllerAttachDetach: true
enableDebuggingHandlers: true
enforceNodeAllocatable:
- pods
eventBurst: 10
eventRecordQPS: 5
evictionHard:
  imagefs.available: 15%
  memory.available: 100Mi
  nodefs.available: 10%
  nodefs.inodesFree: 5%
evictionPressureTransitionPeriod: 5m0s
failSwapOn: true
fileCheckFrequency: 20s
hairpinMode: promiscuous-bridge
healthzBindAddress: 127.0.0.1
healthzPort: 10248
httpCheckFrequency: 20s
imageGCHighThresholdPercent: 85
imageGCLowThresholdPercent: 80
imageMinimumGCAge: 2m0s
iptablesDropBit: 15
iptablesMasqueradeBit: 14
kubeAPIBurst: 10
kubeAPIQPS: 5
makeIPTablesUtilChains: true
maxOpenFiles: 1000000
maxPods: 110
nodeStatusUpdateFrequency: 10s
oomScoreAdj: -999
podPidsLimit: -1
registryBurst: 10
registryPullQPS: 5
resolvConf: /etc/resolv.conf
rotateCertificates: true
runtimeRequestTimeout: 2m0s
serializeImagePulls: true
staticPodPath: /etc/kubernetes/manifests
streamingConnectionIdleTimeout: 4h0m0s
syncFrequency: 1m0s
volumeStatsAggPeriod: 1m0s
EOF
```

### 8.2.5 启动kubelet

```bash
systemctl daemon-reload
systemctl start kubelet
systemctl enable --now kubelet
systemctl restart kubelet
# systemctl status kubelet
```

> 出现以下报错，**tworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized"** **Dec 17 23:19:53 k8s-master01 kubelet[168818]: E1217 23:19:53.926315 168818 kubelet.go:2211] "Container runtime network not ready" networkReady="NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized"。**

**因为cni插件还没有安装**

### 8.2.6 查看集群

```bash
kubectl get node
NAME           STATUS     ROLES    AGE     VERSION
k8s-master01   NotReady   master   4m16s   v1.23.17
k8s-node01     NotReady   node     2m35s   v1.23.17
k8s-node02     NotReady   node     107s    v1.23.17

#设置标签，即更改节点角色
kubectl label node k8s-master01 node-role.kubernetes.io/master=
kubectl label node k8s-node01 node-role.kubernetes.io/node=
kubectl label node k8s-node02 node-role.kubernetes.io/node=
```

## 8.3 kube-proxy配置

### 8.3.1 此配置只在master01操作

```bash
K8S_MASTER01=`cat /etc/hosts | grep k8s-master01 | awk '{print $1}'`
cd /root/k8s/
kubectl -n kube-system create serviceaccount kube-proxy

kubectl create clusterrolebinding system:kube-proxy \
--clusterrole system:node-proxier \
--serviceaccount kube-system:kube-proxy

SECRET=$(kubectl -n kube-system get sa/kube-proxy \
    --output=jsonpath='{.secrets[0].name}')

JWT_TOKEN=$(kubectl -n kube-system get secret/$SECRET \
--output=jsonpath='{.data.token}' | base64 -d)

PKI_DIR=/etc/kubernetes/pki
K8S_DIR=/etc/kubernetes

kubectl config set-cluster kubernetes \
--certificate-authority=/etc/kubernetes/pki/ca.pem \
--embed-certs=true \
--server=https://$K8S_MASTER01:6443 \
--kubeconfig=${K8S_DIR}/kube-proxy.kubeconfig

kubectl config set-credentials kubernetes \
--token=${JWT_TOKEN} \
--kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig

kubectl config set-context kubernetes \
--cluster=kubernetes \
--user=kubernetes \
--kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig

kubectl config use-context kubernetes \
--kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig
```

### 8.3.2 将kubeconfig发送至其他节点

```bash
for NODE in k8s-node01 k8s-node02;
do 
    scp /etc/kubernetes/kube-proxy.kubeconfig $NODE:/etc/kubernetes/kube-proxy.kubeconfig;
done
```

### 8.3.3 节点添加kube-proxy的配置和service文件

- 所有主机操作

```bash
cat >  /usr/lib/systemd/system/kube-proxy.service << EOF
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
ExecStart=/usr/bin/kube-proxy \\
  --config=/etc/kubernetes/kube-proxy.yaml \\
  --v=2

Restart=always
RestartSec=10s

[Install]
WantedBy=multi-user.target

EOF

cat > /etc/kubernetes/kube-proxy.yaml << EOF
apiVersion: kubeproxy.config.k8s.io/v1alpha1
bindAddress: 0.0.0.0
clientConnection:
  acceptContentTypes: ""
  burst: 10
  contentType: application/vnd.kubernetes.protobuf
  kubeconfig: /etc/kubernetes/kube-proxy.kubeconfig
  qps: 5
clusterCIDR: 172.16.0.0/12
configSyncPeriod: 15m0s
conntrack:
  max: null
  maxPerCore: 32768
  min: 131072
  tcpCloseWaitTimeout: 1h0m0s
  tcpEstablishedTimeout: 24h0m0s
enableProfiling: false
healthzBindAddress: 0.0.0.0:10256
hostnameOverride: ""
iptables:
  masqueradeAll: false
  masqueradeBit: 14
  minSyncPeriod: 0s
  syncPeriod: 30s
ipvs:
  masqueradeAll: true
  minSyncPeriod: 5s
  scheduler: "rr"
  syncPeriod: 30s
kind: KubeProxyConfiguration
metricsBindAddress: 127.0.0.1:10249
mode: "ipvs"
nodePortAddresses: null
oomScoreAdj: -999
portRange: ""
udpIdleTimeout: 250ms

EOF
```

### 8.3.4 启动kube-proxy

```bash
systemctl daemon-reload
systemctl enable --now kube-proxy
systemctl restart kube-proxy
# systemctl status kube-proxy

# 查看当前使用是默认是iptables或ipvs
# curl localhost:10249/proxyMode
ipvs
```

 **注：这里能ping通svc的是使用ipvs，不能ping通的是使用iptables。**

## 8.4 安装网络插件

### 8.4.1 calico部署

`Calico`是一个纯三层的数据中心网络方案，是目前Kubernetes主流的网络方案。

注意对应的版本：https://docs.tigera.io/archive/v3.24/getting-started/kubernetes/requirements

```bash
mkdir -p $HOME/k8s-install/network && cd $HOME/k8s-install/network
# 1. 下载插件
curl https://docs.tigera.io/archive/v3.24/manifests/calico.yaml -O


# CIDR的值，与 kube-controller-manager中“--cluster-cidr=172.16.0.0/16” 一致
    - name: CALICO_IPV4POOL_CIDR
      value: "172.16.0.0/16"
      
      containers:
        # Runs calico-node container on each Kubernetes node. This
        # container programs network policy and routes on each
        # host.
        - name: calico-node
          image: docker.io/calico/node:v3.24.5
          envFrom:
          - configMapRef:
              # Allow KUBERNETES_SERVICE_HOST and KUBERNETES_SERVICE_PORT to be overridden for eBPF mode.
              name: kubernetes-services-endpoint
              optional: true
          env:
            - name: IP_AUTODETECTION_METHOD # 指定网卡
              value: "interface=ens33"

              
              
kubectl apply -f calico.yaml

NAME                                       READY   STATUS    RESTARTS   AGE
calico-kube-controllers-7c87c5f9b8-st6tc   1/1     Running   0          2m46s
calico-node-crmqk                          1/1     Running   0          2m47s
calico-node-mdqj4                          1/1     Running   0          2m47s
calico-node-v5qsv                          1/1     Running   0          2m47s
```

**注：需要查看自己使用的网卡名称。**

### 8.4.2 CoreDNS

k8s对比：[CoreDNS version in Kubernetes](https://github.com/coredns/deployment/blob/master/kubernetes/CoreDNS-k8s_version.md) 镜像地址：[coredns/coredns - Docker Image | Docker Hub](https://hub.docker.com/r/coredns/coredns)

官网：https://github.com/coredns/deployment/tree/master/kubernetes 文档参考：[安装配置 CoreDNS](https://jimmysong.io/kubernetes-handbook/practice/coredns.html)

参考：https://github.com/coredns/deployment/blob/master/kubernetes/coredns.yaml.sed

coredns.yaml内容：

```bash
apiVersion: v1
kind: ServiceAccount
metadata:
  name: coredns
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:coredns
rules:
  - apiGroups:
    - ""
    resources:
    - endpoints
    - services
    - pods
    - namespaces
    verbs:
    - list
    - watch
  - apiGroups:
    - discovery.k8s.io
    resources:
    - endpointslices
    verbs:
    - list
    - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:coredns
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:coredns
subjects:
- kind: ServiceAccount
  name: coredns
  namespace: kube-system
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        log
        errors
        health {
          lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
          fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        forward . /etc/resolv.conf {
          max_concurrent 1000
        }
        cache 30
        loop
        reload
        loadbalance
    }
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: coredns
  namespace: kube-system
  labels:
    k8s-app: kube-dns
    kubernetes.io/name: "CoreDNS"
    app.kubernetes.io/name: coredns
spec:
  # replicas: not specified here:
  # 1. Default is 1.
  # 2. Will be tuned in real time if DNS horizontal auto-scaling is turned on.
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  selector:
    matchLabels:
      k8s-app: kube-dns
      app.kubernetes.io/name: coredns
  template:
    metadata:
      labels:
        k8s-app: kube-dns
        app.kubernetes.io/name: coredns
    spec:
      priorityClassName: system-cluster-critical
      serviceAccountName: coredns
      tolerations:
        - key: "CriticalAddonsOnly"
          operator: "Exists"
      nodeSelector:
        kubernetes.io/os: linux
      affinity:
         podAntiAffinity:
           requiredDuringSchedulingIgnoredDuringExecution:
           - labelSelector:
               matchExpressions:
               - key: k8s-app
                 operator: In
                 values: ["kube-dns"]
             topologyKey: kubernetes.io/hostname
      containers:
      - name: coredns
        image: coredns/coredns:1.8.6
        imagePullPolicy: IfNotPresent
        resources:
          limits:
            memory: 170Mi
          requests:
            cpu: 100m
            memory: 70Mi
        args: [ "-conf", "/etc/coredns/Corefile" ]
        volumeMounts:
        - name: config-volume
          mountPath: /etc/coredns
          readOnly: true
        ports:
        - containerPort: 53
          name: dns
          protocol: UDP
        - containerPort: 53
          name: dns-tcp
          protocol: TCP
        - containerPort: 9153
          name: metrics
          protocol: TCP
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            add:
            - NET_BIND_SERVICE
            drop:
            - all
          readOnlyRootFilesystem: true
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 60
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 5
        readinessProbe:
          httpGet:
            path: /ready
            port: 8181
            scheme: HTTP
      dnsPolicy: Default
      volumes:
        - name: config-volume
          configMap:
            name: coredns
            items:
            - key: Corefile
              path: Corefile
---
apiVersion: v1
kind: Service
metadata:
  name: kube-dns
  namespace: kube-system
  annotations:
    prometheus.io/port: "9153"
    prometheus.io/scrape: "true"
  labels:
    k8s-app: kube-dns
    kubernetes.io/cluster-service: "true"
    kubernetes.io/name: "CoreDNS"
    app.kubernetes.io/name: coredns
spec:
  selector:
    k8s-app: kube-dns
    app.kubernetes.io/name: coredns
  clusterIP: 10.96.0.10
  ports:
  - name: dns
    port: 53
    protocol: UDP
  - name: dns-tcp
    port: 53
    protocol: TCP
  - name: metrics
    port: 9153
    protocol: TCP
```

启动容器：

```bash
# 下载文件
# wget https://storage.googleapis.com/kubernetes-the-hard-way/coredns.yaml
image: coredns/coredns:1.8.6
# 修改文件内容cluster ip
clusterIP: 10.96.0.10
# 启动文件
kubectl apply -f coredns.yaml

# 查询状态
kubectl get pods -n kube-system | grep coredns
coredns-c975ddbcc-4tdml                    1/1     Running   0          46s
coredns-c975ddbcc-t5zmr                    1/1     Running   0          46s
```

# 9. 集群验证

## 9.1 部署pod资源

```bash
cat<<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: busybox
  namespace: default
spec:
  containers:
  - name: busybox
    image: docker.io/library/busybox:1.28
    command:
      - sleep
      - "3600"
    imagePullPolicy: IfNotPresent
  restartPolicy: Always
EOF

# 查看
kubectl  get pod
NAME      READY   STATUS    RESTARTS   AGE
busybox   1/1     Running   0          22m
```

## 9.2 用pod解析默认命名空间中的kubernete

```bash
kubectl get svc
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   3h44m

kubectl exec  busybox -n default -- nslookup kubernetes
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      kubernetes
Address 1: 10.96.0.1 kubernetes.default.svc.cluster.local
```

## 9.3 测试跨命名空间是否可以解析

```bash
kubectl exec  busybox -n default -- nslookup kube-dns.kube-system
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      kube-dns.kube-system
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local
```

## 9.4 每个节点都必须要能访问Kubernetes的kubernetes svc 443和kube-dns的service 53

```bash
telnet 10.96.0.1 443
Trying 10.96.0.1...
Connected to 10.96.0.1.
Escape character is '^]'.

telnet 10.96.0.10 53
Trying 10.96.0.10...
Connected to 10.96.0.10.
Escape character is '^]'.

curl 10.96.0.10:53
curl: (52) Empty reply from server
```

## 9.5 Pod和Pod之前要能通

```bash
# 进入busybox ping其他节点上的pod
kubectl exec -ti busybox -- sh
# ping node主机
/ # ping 192.168.80.45
PING 192.168.80.46 (192.168.80.46): 56 data bytes
64 bytes from 192.168.80.46: seq=0 ttl=63 time=3.778 ms

# ping coredns pod地址
/ # ping 172.16.85.196
PING 172.16.85.196 (172.16.85.196): 56 data bytes
64 bytes from 172.16.85.196: seq=0 ttl=62 time=1.648 ms

# ping baidu地址
/ # ping baidu.com
PING baidu.com (110.242.68.66): 56 data bytes
64 bytes from 110.242.68.66: seq=0 ttl=127 time=38.964 ms
64 bytes from 110.242.68.66: seq=1 ttl=127 time=40.936 ms
```

## 9.6 创建三个副本，可以看到3个副本分布在不同的节点上

```bash
cat > deployments.yaml << EOF
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
        image: docker.io/library/nginx:1.14.2
        ports:
        - containerPort: 80

EOF

kubectl  apply -f deployments.yaml 
deployment.apps/nginx-deployment created

kubectl  get pod -owide
NAME                                READY   STATUS    RESTARTS   AGE    IP              NODE           NOMINATED NODE   READINESS GATES
busybox                             1/1     Running   0          9m2s   172.16.58.197   k8s-node02     <none>           <none>
nginx-deployment-5dd7c4f676-clh6f   1/1     Running   0          105s   172.16.32.131   k8s-master01   <none>           <none>
nginx-deployment-5dd7c4f676-mhggd   1/1     Running   0          105s   172.16.85.197   k8s-node01     <none>           <none>
nginx-deployment-5dd7c4f676-tpg77   1/1     Running   0          105s   172.16.58.198   k8s-node02     <none>           <none>
```

# 10. master集群高可用

| **角色**     | **IP**        | **组件**                                                     |
| ------------ | ------------- | ------------------------------------------------------------ |
| k8s-master02 | 192.168.80.46 | kube-apiserver kube-controller-manager kubectl kubelet kube-proxy kube-scheduler docker, haproxy, keepalived |
| k8s-master03 | 192.168.80.47 | kube-apiserver kube-controller-manager kubectl kubelet kube-proxy kube-scheduler docker, haproxy, keepalived |

## 10.1 初始化操作

注：请回到 **[1. 环境准备](#T1)**、**[2.k8s基本组件安装](#T2)** 执行一遍。

## 10.2 安装cni插件

- master01操作

```bash
mkdir -p $HOME/k8s-install/network && cd $_
CNI_VERSION="v1.1.1"

for NODE in k8s-master02 k8s-master03;
do
    ssh $NODE mkdir -p /etc/cni/net.d /opt/cni/bin $HOME/k8s-install/network;
    for FILE in cni-plugins-linux-amd64-$CNI_VERSION.tgz;
    do 
        scp cni-plugins-linux-amd64-$CNI_VERSION.tgz $NODE:$HOME/k8s-install/network;
        ssh $NODE "cd $HOME/k8s-install/network && tar xf cni-plugins-linux-amd64-$CNI_VERSION.tgz -C /opt/cni/bin/"
    done;
done
```

## 10.3 复制证书

- master01操作

```bash
for NODE in k8s-master02 k8s-master03; do
  ssh root@$NODE "mkdir -p /etc/kubernetes/pki"
  ssh root@$NODE "mkdir -p /etc/etcd/ssl"
  for FILE in $(ls /etc/kubernetes/pki | grep -v etcd);
  do
    scp /etc/kubernetes/pki/${FILE} $NODE:/etc/kubernetes/pki/${FILE};
    scp /etc/kubernetes/token.csv $NODE:/etc/kubernetes;
  done
  for FILE in $(ls /etc/kubernetes | grep kubeconfig);
  do
    scp /etc/kubernetes/${FILE} $NODE:/etc/kubernetes/${FILE}; 
  done
  for FILE in $(ls /etc/etcd/ssl | grep etcd);
  do
    scp /etc/etcd/ssl/${FILE} $NODE:/etc/etcd/ssl/${FILE};
  done
done
```

## 10.4 复制k8s master组件

- master01操作

```bash
for NODE in k8s-master02 k8s-master03; do
  ssh $NODE "apt install -y bash-completion && \
             source /usr/share/bash-completion/bash_completion && \
             source <(kubectl completion bash) && \
             echo "source <(kubectl completion bash)" >> ~/.bashrc"
  for FILE in $(ls /usr/bin/ | grep kube); do
     scp /usr/bin/${FILE} $NODE:/usr/bin/${FILE}
  done
done
```

## 10.5 apiserver

- master02、master03操作

`--service-cluster-ip-range=10.96.0.0/12`: Service IP 段

```bash
cat > /usr/lib/systemd/system/kube-apiserver.service << EOF

[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
ExecStart=/usr/bin/kube-apiserver \\
      --v=2  \\
      --logtostderr=true  \\
      --allow-privileged=true  \\
      --bind-address=0.0.0.0  \\
      --secure-port=6443  \\
      --insecure-port=0  \\
      --advertise-address=$LOCALHOST \\
      --service-cluster-ip-range=10.96.0.0/12 \\
      --feature-gates=IPv6DualStack=true \\
      --service-node-port-range=30000-32767  \\
      --etcd-servers=https://$K8S_MASTER01:2379,https://$K8S_NODE01:2379,https://$K8S_NODE02:2379 \\
      --etcd-cafile=/etc/etcd/ssl/etcd-ca.pem  \\
      --etcd-certfile=/etc/etcd/ssl/etcd.pem  \\
      --etcd-keyfile=/etc/etcd/ssl/etcd-key.pem  \\
      --client-ca-file=/etc/kubernetes/pki/ca.pem  \\
      --tls-cert-file=/etc/kubernetes/pki/apiserver.pem  \\
      --tls-private-key-file=/etc/kubernetes/pki/apiserver-key.pem  \\
      --kubelet-client-certificate=/etc/kubernetes/pki/apiserver.pem  \\
      --kubelet-client-key=/etc/kubernetes/pki/apiserver-key.pem  \\
      --service-account-key-file=/etc/kubernetes/pki/sa.pub  \\
      --service-account-signing-key-file=/etc/kubernetes/pki/sa.key  \\
      --service-account-issuer=https://kubernetes.default.svc.cluster.local \\
      --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname  \\
      --enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,NodeRestriction,ResourceQuota  \\
      --authorization-mode=Node,RBAC  \\
      --enable-bootstrap-token-auth=true  \\
      --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.pem  \\
      --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.pem  \\
      --proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client-key.pem  \\
      --requestheader-allowed-names=aggregator  \\
      --requestheader-group-headers=X-Remote-Group  \\
      --requestheader-extra-headers-prefix=X-Remote-Extra-  \\
      --requestheader-username-headers=X-Remote-User \\
      --enable-aggregator-routing=true \\
      --token-auth-file=/etc/kubernetes/token.csv

Restart=on-failure
RestartSec=10s
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target

EOF
```

开机启动：

```bash
systemctl daemon-reload && systemctl enable --now kube-apiserver

# 注意查看状态是否启动正常
systemctl start  kube-apiserver
# systemctl status kube-apiserver
```

## 10.6 kube-controller-manager service

- master02、master03操作

```bash
# 所有master节点配置，且配置相同
# 172.16.0.0/12为pod网段，按需求设置你自己的网段

cat > /usr/lib/systemd/system/kube-controller-manager.service << EOF
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
ExecStart=/usr/bin/kube-controller-manager \\
      --v=2 \\
      --logtostderr=true \\
      --bind-address=127.0.0.1 \\
      --root-ca-file=/etc/kubernetes/pki/ca.pem \\
      --cluster-signing-cert-file=/etc/kubernetes/pki/ca.pem \\
      --cluster-signing-key-file=/etc/kubernetes/pki/ca-key.pem \\
      --service-account-private-key-file=/etc/kubernetes/pki/sa.key \\
      --kubeconfig=/etc/kubernetes/controller-manager.kubeconfig \\
      --leader-elect=true \\
      --use-service-account-credentials=true \\
      --node-monitor-grace-period=40s \\
      --node-monitor-period=5s \\
      --pod-eviction-timeout=2m0s \\
      --controllers=*,bootstrapsigner,tokencleaner \\
      --allocate-node-cidrs=true \\
      --service-cluster-ip-range=10.96.0.0/12,fd00::/108 \\
      --cluster-cidr=172.16.0.0/12,fc00::/48 \\
      --node-cidr-mask-size-ipv4=24 \\
      --node-cidr-mask-size-ipv6=64 \\
      --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.pem 
      # --feature-gates=IPv6DualStack=true #要是没有配置ipv6就不要打开，打开会出现 Get "http://127.0.0.1:10252/healthz": dial tcp 127.0.0.1:10252: connect: connection refused 报错

Restart=always
RestartSec=10s

[Install]
WantedBy=multi-user.target

EOF
```

开机启动：

```bash
systemctl daemon-reload && systemctl enable --now kube-controller-manager
systemctl restart kube-controller-manager
# systemctl status kube-controller-manager
```

## 10.7 kube-scheduler service

```bash
cat > /usr/lib/systemd/system/kube-scheduler.service << EOF

[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
ExecStart=/usr/bin/kube-scheduler \\
      --v=2 \\
      --logtostderr=true \\
      --address=127.0.0.1 \\
      --leader-elect=true \\
      --kubeconfig=/etc/kubernetes/scheduler.kubeconfig

Restart=always
RestartSec=10s

[Install]
WantedBy=multi-user.target

EOF
```

开机启动：

```bash
systemctl daemon-reload && systemctl enable --now kube-scheduler
systemctl restart kube-scheduler
# systemctl status kube-scheduler
```

## 10.8 kubelet

### 10.8.1 docker启动服务（本地使用）

其中：`--kubeconfig=/etc/kubernetes/kubelet.kubeconfig` 在加入集群时自动生成

```bash
mkdir -p /var/lib/kubelet /var/log/kubernetes /etc/systemd/system/kubelet.service.d /etc/kubernetes/manifests/
HOSTNAME=`echo $(hostname)`

cat > /etc/kubernetes/kubelet.conf << EOF
KUBELET_OPTS="--logtostderr=false \\
--v=2 \\
--log-dir=/var/log/kubernetes \\
--hostname-override=$HOSTNAME \\
--network-plugin=cni \\
--container-runtime=docker \\
--cgroup-driver=systemd \\
--kubeconfig=/etc/kubernetes/kubelet.kubeconfig \\
--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.kubeconfig \\
--config=/etc/kubernetes/kubelet-conf.yml \\
--container-runtime-endpoint=unix:///var/run/dockershim.sock \\
--cert-dir=/etc/kubernetes/pki \\
--node-labels=node.kubernetes.io/node='' \\
--pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/imges/pause:3.6"
EOF

cat > /lib/systemd/system/kubelet.service << EOF
[Unit]
Description=Kubernetes Kubelet
After=docker.service
Requires=docker.service

[Service]
EnvironmentFile=/etc/kubernetes/kubelet.conf
ExecStart=/usr/bin/kubelet \$KUBELET_OPTS
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

### 10.8.2 containerd启动容器

```bash
mkdir -p /var/lib/kubelet /var/log/kubernetes /etc/systemd/system/kubelet.service.d /etc/kubernetes/manifests/

所有k8s节点配置kubelet service
cat > /usr/lib/systemd/system/kubelet.service << EOF

[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=containerd.service
Requires=containerd.service

[Service]
ExecStart=/usr/bin/kubelet \\
    --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.kubeconfig  \\
    --kubeconfig=/etc/kubernetes/kubelet.kubeconfig \\
    --config=/etc/kubernetes/kubelet-conf.yml \
    --network-plugin=cni  \\
    --cni-conf-dir=/etc/cni/net.d  \\
    --cni-bin-dir=/opt/cni/bin  \\
    --container-runtime=remote  \\
    --runtime-request-timeout=15m  \\
    --container-runtime-endpoint=unix:///run/containerd/containerd.sock  \\
    --cgroup-driver=systemd \\
    --node-labels=node.kubernetes.io/node=
    #--feature-gates=IPv6DualStack=true

Restart=always
StartLimitInterval=0
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF
```

### 10.8.3 所有k8s节点创建kubelet的配置文件

```bash
cat > /etc/kubernetes/kubelet-conf.yml <<EOF
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
address: 0.0.0.0
port: 10250
readOnlyPort: 10255
authentication:
  anonymous:
    enabled: false
  webhook:
    cacheTTL: 2m0s
    enabled: true
  x509:
    clientCAFile: /etc/kubernetes/pki/ca.pem
authorization:
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: 5m0s
    cacheUnauthorizedTTL: 30s
cgroupDriver: systemd
cgroupsPerQOS: true
clusterDNS:
- 10.96.0.10
clusterDomain: cluster.local
containerLogMaxFiles: 5
containerLogMaxSize: 10Mi
contentType: application/vnd.kubernetes.protobuf
cpuCFSQuota: true
cpuManagerPolicy: none
cpuManagerReconcilePeriod: 10s
enableControllerAttachDetach: true
enableDebuggingHandlers: true
enforceNodeAllocatable:
- pods
eventBurst: 10
eventRecordQPS: 5
evictionHard:
  imagefs.available: 15%
  memory.available: 100Mi
  nodefs.available: 10%
  nodefs.inodesFree: 5%
evictionPressureTransitionPeriod: 5m0s
failSwapOn: true
fileCheckFrequency: 20s
hairpinMode: promiscuous-bridge
healthzBindAddress: 127.0.0.1
healthzPort: 10248
httpCheckFrequency: 20s
imageGCHighThresholdPercent: 85
imageGCLowThresholdPercent: 80
imageMinimumGCAge: 2m0s
iptablesDropBit: 15
iptablesMasqueradeBit: 14
kubeAPIBurst: 10
kubeAPIQPS: 5
makeIPTablesUtilChains: true
maxOpenFiles: 1000000
maxPods: 110
nodeStatusUpdateFrequency: 10s
oomScoreAdj: -999
podPidsLimit: -1
registryBurst: 10
registryPullQPS: 5
resolvConf: /etc/resolv.conf
rotateCertificates: true
runtimeRequestTimeout: 2m0s
serializeImagePulls: true
staticPodPath: /etc/kubernetes/manifests
streamingConnectionIdleTimeout: 4h0m0s
syncFrequency: 1m0s
volumeStatsAggPeriod: 1m0s
EOF
```

### 10.8.4 启动kubelet

```bash
rm -f /etc/kubernetes/kubelet.kubeconfig 
rm -f /etc/kubernetes/pki/kubelet*

systemctl daemon-reload && \
systemctl enable kubelet && \
systemctl restart kubelet
# systemctl status kubelet
```

## 10.9 kube-proxy

```bash
cat >  /usr/lib/systemd/system/kube-proxy.service << EOF
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
ExecStart=/usr/bin/kube-proxy \\
  --config=/etc/kubernetes/kube-proxy.yaml \\
  --v=2

Restart=always
RestartSec=10s

[Install]
WantedBy=multi-user.target

EOF

cat > /etc/kubernetes/kube-proxy.yaml << EOF
apiVersion: kubeproxy.config.k8s.io/v1alpha1
bindAddress: 0.0.0.0
clientConnection:
  acceptContentTypes: ""
  burst: 10
  contentType: application/vnd.kubernetes.protobuf
  kubeconfig: /etc/kubernetes/kube-proxy.kubeconfig
  qps: 5
clusterCIDR: 172.16.0.0/12
configSyncPeriod: 15m0s
conntrack:
  max: null
  maxPerCore: 32768
  min: 131072
  tcpCloseWaitTimeout: 1h0m0s
  tcpEstablishedTimeout: 24h0m0s
enableProfiling: false
healthzBindAddress: 0.0.0.0:10256
hostnameOverride: ""
iptables:
  masqueradeAll: false
  masqueradeBit: 14
  minSyncPeriod: 0s
  syncPeriod: 30s
ipvs:
  masqueradeAll: true
  minSyncPeriod: 5s
  scheduler: "rr"
  syncPeriod: 30s
kind: KubeProxyConfiguration
metricsBindAddress: 127.0.0.1:10249
mode: "ipvs"
nodePortAddresses: null
oomScoreAdj: -999
portRange: ""
udpIdleTimeout: 250ms

EOF
```

开机启动：

```bash
systemctl daemon-reload && systemctl enable --now kube-proxy
systemctl restart kube-proxy
# systemctl status kube-proxy
# 查看是否使用的是ipvs
# curl localhost:10249/proxyMode
ipvs
```

## 10.10 打标签和检查加入集群情况

```bash
kubectl label node k8s-master02 node-role.kubernetes.io/master=
kubectl label node k8s-master03 node-role.kubernetes.io/master=

kubectl get node
```

# 11. 高可用配置

![img](https://img2023.cnblogs.com/blog/1740081/202304/1740081-20230427210806995-1497478377.png)

主要规划：master01、master02、master03 安装 keepalived和haproxy 服务，虚拟vip：`192.168.80.100` 端口为`8443`，此前签发证书中已经加入了`192.168.80.100`地址。

逻辑：这里的逻辑：keepalived和haproxy 放在一台机器上，然后去访问vip地址设置好的haproxy 8443端口，haproxy 去访问master节点的6443端口。

## 11.1 安装keepalived和haproxy服务

```bash
apt install keepalived haproxy -y
```

## 11.2 修改haproxy配置文件（三台配置文件一样）

注：这里如果有条件也可以将keepalived和haproxy放在单独的机器上面。🤔

- k8s-master01操作、k8s-master02操作、k8s-master03操作

```bash
# cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.bak

cat > /etc/haproxy/haproxy.cfg << EOF
global
 maxconn 2000
 ulimit-n 16384
 log 127.0.0.1 local0 err
 stats timeout 30s

defaults
 log global
 mode http
 option httplog
 timeout connect 5000
 timeout client 50000
 timeout server 50000
 timeout http-request 15s
 timeout http-keep-alive 15s


frontend monitor-in
 bind *:33305
 mode http
 option httplog
 monitor-uri /monitor

frontend k8s-master
 bind 0.0.0.0:8443
 bind 127.0.0.1:8443
 mode tcp
 option tcplog
 tcp-request inspect-delay 5s
 default_backend k8s-master


backend k8s-master
 mode tcp
 option tcplog
 option tcp-check
 balance roundrobin
 default-server inter 10s downinter 5s rise 2 fall 2 slowstart 60s maxconn 250 maxqueue 256 weight 100
 server  k8s-master01  $K8S_MASTER01:6443 check
 server  k8s-master02  $K8S_MASTER02:6443 check
 server  k8s-master03  $K8S_MASTER03:6443 check
EOF
```

> 这里keepalived使用的是非抢占模式：priority id

## 11.3 Master01配置keepalived master节点

- k8s-master01操作

```bash
#cp /etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf.bak

cat > /etc/keepalived/keepalived.conf << EOF
! Configuration File for keepalived

global_defs {
    router_id LVS_DEVEL
}
vrrp_script chk_apiserver {
    script "/etc/keepalived/check_apiserver.sh"
    interval 5 
    weight -5
    fall 2
    rise 1
}
vrrp_instance VI_1 {
    state MASTER
    # 注意网卡名
    interface ens33
    mcast_src_ip $LOCALHOST
    virtual_router_id 51
    priority 100
    nopreempt
    advert_int 2
    authentication {
        auth_type PASS
        auth_pass K8SHA_KA_AUTH
    }
    virtual_ipaddress {
        192.168.80.100
    }
    track_script {
      chk_apiserver 
} }

EOF
```

## 11.4 Master02配置keepalived backup节点

- k8s-master02操作

```bash
# cp /etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf.bak

cat > /etc/keepalived/keepalived.conf << EOF
! Configuration File for keepalived

global_defs {
    router_id LVS_DEVEL
}
vrrp_script chk_apiserver {
    script "/etc/keepalived/check_apiserver.sh"
    interval 5 
    weight -5
    fall 2
    rise 1

}
vrrp_instance VI_1 {
    state BACKUP
    # 注意网卡名
    interface ens33
    mcast_src_ip $LOCALHOST
    virtual_router_id 51
    priority 80
    nopreempt
    advert_int 2
    authentication {
        auth_type PASS
        auth_pass K8SHA_KA_AUTH
    }
    virtual_ipaddress {
        192.168.80.100
    }
    track_script {
      chk_apiserver 
} }

EOF
```

## 11.5 Master03配置keepalived backup节点

- k8s-master03操作

```bash
# cp /etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf.bak

cat > /etc/keepalived/keepalived.conf << EOF
! Configuration File for keepalived

global_defs {
    router_id LVS_DEVEL
}
vrrp_script chk_apiserver {
    script "/etc/keepalived/check_apiserver.sh"
    interval 5
    weight -5
    fall 2
    rise 1

}
vrrp_instance VI_1 {
    state BACKUP
    # 注意网卡名
    interface ens33
    mcast_src_ip $LOCALHOST
    virtual_router_id 51
    priority 50
    nopreempt
    advert_int 2
    authentication {
        auth_type PASS
        auth_pass K8SHA_KA_AUTH
    }
    virtual_ipaddress {
        192.168.80.100
    }
    track_script {
      chk_apiserver 
} }

EOF
```

## 11.6 健康检查脚本配置

- 全部master主机

```bash
cat >  /etc/keepalived/check_apiserver.sh << EOF
#!/bin/bash

err=0
for k in \$(seq 1 3)
do
    check_code=\$(pgrep haproxy)
    if [[ \$check_code == "" ]]; then
        err=\$(expr \$err + 1)
        sleep 1
        continue
    else
        err=0
        break
    fi
done

if [[ \$err != "0" ]]; then
    echo "systemctl stop keepalived"
    /usr/bin/systemctl stop keepalived
    exit 1
else
    exit 0
fi
EOF

# 给脚本授权

chmod +x /etc/keepalived/check_apiserver.sh
sudo useradd keepalived_script
sudo passwd keepalived_script
sudo chown -R keepalived_script:keepalived_script /etc/keepalived/check_apiserver.sh
```

## 11.7 启动服务

```bash
systemctl daemon-reload
systemctl enable --now haproxy
systemctl enable --now keepalived
systemctl start haproxy
systemctl start keepalived

systemctl restart haproxy
# systemctl status haproxy
# systemctl status keepalived
```

注：端口要是没起来就重启下。

## 11.8 测试高可用

注：master机器关机然后轮着来每次只剩下一台机器。

```bash
# 能ping同
ping 192.168.80.100
netstat -lntup | grep 8443

# 能telnet访问
telnet 192.168.80.100 8443

# 关闭主节点，看vip是否漂移到备节点
```

## 11.9 替换所有节点配置文件

- 所有节点执行

```bash
# 批量替换
sed -i 's#192.168.80.45:6443#192.168.80.100:8443#g' `grep "192.168.80.45:6443" -rl /etc/kubernetes/* /root/.kube/config`
grep -nr  '192.168.80.' /etc/kubernetes/* /root/.kube/config

systemctl restart kubelet kube-proxy
# systemctl status kubelet kube-proxy

kubectl get node
```

# 12. 打标和污点

```bash
# 标签
kubectl label node k8s-master01 k8s-master02 k8s-master03 node-role.kubernetes.io/master=
kubectl label node k8s-node01 k8s-node02 node-role.kubernetes.io/node=

# 污点
kubectl taint nodes k8s-master01 k8s-master02 k8s-master03 node-role.kubernetes.io/master=:NoSchedule
```

# 13. 删除节点

```bash
# 1. k8s-master02 上，停止kubelet进程
systemctl stop kubelet

# 2. 检查 k8s-master02 是否已下线
kubectl get nodes

# 3. 删除节点
kubectl drain k8s-master02

# 4.强制下线
kubectl drain k8s-master02 --ignore-daemonsets

# 5. 下线状态
kubectl get nodes

# 6. 恢复操作 (如有必要)
kubectl uncordon k8s-master02

# 7. 彻底删除
kubectl delete node k8s-master02 
```

# 14. 安装Metrics Server

GitHub地址：https://github.com/kubernetes-sigs/metrics-server

```bash
cd /root/k8s/metrics-server && kubectl apply -f .
serviceaccount/metrics-server created
clusterrole.rbac.authorization.k8s.io/system:aggregated-metrics-reader created
clusterrole.rbac.authorization.k8s.io/system:metrics-server created
rolebinding.rbac.authorization.k8s.io/metrics-server-auth-reader created
clusterrolebinding.rbac.authorization.k8s.io/metrics-server:system:auth-delegator created
clusterrolebinding.rbac.authorization.k8s.io/system:metrics-server created
service/metrics-server created
deployment.apps/metrics-server created
apiservice.apiregistration.k8s.io/v1beta1.metrics.k8s.io created

kubectl top node
NAME           CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
k8s-master01   474m         23%    2962Mi          78%
k8s-node01     176m         8%     3147Mi          83%
k8s-node02     212m         10%    2317Mi          125%
```

 