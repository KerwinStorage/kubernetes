# 1.环境准备

## 1.1.集群信息

**注：本次部署没有做master节点和node节点扩展操作，直接部署三主两从。**

| 主机名称 | IP地址         | 说明       | 软件                                                         |
| -------- | -------------- | ---------- | ------------------------------------------------------------ |
| Master01 | 192.168.80.45  | master节点 | kube-apiserver、kube-controller-manager、kube-scheduler、etcd、 kubelet、kube-proxy、nfs-client、haproxy、keepalived |
| Master02 | 192.168.80.46  | master节点 | kube-apiserver、kube-controller-manager、kube-scheduler、etcd、 kubelet、kube-proxy、nfs-client、haproxy、keepalived |
| Master03 | 192.168.80.47  | master节点 | kube-apiserver、kube-controller-manager、kube-scheduler、etcd、 kubelet、kube-proxy、nfs-client、haproxy、keepalived |
| Node01   | 192.168.80.48  | node节点   | kubelet、kube-proxy、nfs-client                              |
| Node02   | 192.168.80.49  | node节点   | kubelet、kube-proxy、nfs-client                              |
|          | 192.168.80.100 | VIP        |                                                              |

系统相关版本：

| 内核                                                         | Linux 5.19.0-35-generic x86_64 |
| ------------------------------------------------------------ | ------------------------------ |
| [Ubuntu](https://ubuntu.com/download/desktop/thank-you?version=22.04.2&architecture=amd64) | 22.04                          |
| [kube-apiserver、kube-controller-manager、kube-scheduler、kubelet、kube-proxy](https://github.com/kubernetes/kubernetes/tree/v1.26.5) | v1.26.5                        |
| [etcd](https://github.com/etcd-io/etcd)                      | v3.5.9                         |
| [containerd](https://github.com/containerd/containerd)       | v1.6.21                        |
| [cfssl](https://github.com/cloudflare/cfssl)                 | v1.6.1                         |
| [cni](https://github.com/containernetworking/plugins/releases) | v1.2.0                         |
| [crictl](https://github.com/kubernetes-sigs/cri-tools)       | v1.26.1                        |
| haproxy                                                      | 2.4.18-0ubuntu1.2              |
| keepalived                                                   | v2.2.4                         |

## 1.2.系统设置

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
echo "K8S_VIP='192.168.80.100'" >>  ~/.bashrc
source ~/.bashrc
```

## 1.3.设置主机名

```bash
hostnamectl set-hostname k8s-master01
hostnamectl set-hostname k8s-master02
hostnamectl set-hostname k8s-master03
hostnamectl set-hostname k8s-node01
hostnamectl set-hostname k8s-node02
```

## 1.4.配置apt源

清华源：https://mirrors.tuna.tsinghua.edu.cn/help/ubuntu，阿里源：https://developer.aliyun.com/mirror/ubuntu

22.04版本：

```bash
cp /etc/apt/sources.list /etc/apt/sources.list_bak
cat > /etc/apt/sources.list << EOF
# 默认注释了源码镜像以提高 apt update 速度，如有需要可自行取消注释
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-updates main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-updates main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-backports main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-backports main restricted universe multiverse

# deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-security main restricted universe multiverse
# # deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-security main restricted universe multiverse

deb http://security.ubuntu.com/ubuntu/ jammy-security main restricted universe multiverse
# deb-src http://security.ubuntu.com/ubuntu/ jammy-security main restricted universe multiverse

# 预发布软件源，不建议启用
# deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-proposed main restricted universe multiverse
# # deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-proposed main restricted universe multiverse
EOF

# 更新软件列表
sudo apt-get update
```

## 1.5.安装一些必备工具

```bash
apt install net-tools nfs-kernel-server curl vim git lvm2 telnet htop jq lrzsz tree bash-completion telnet wget make -y

vim ~/.bashrc
# 去除注释
if [ -f /etc/bash_completion ] && ! shopt -oq posix; then
    . /etc/bash_completion
fi
source ~/.bashrc
```

## 1.6.主机名解析

```bash
cat >> /etc/hosts << EOF
$K8S_MASTER01  k8s-master01
$K8S_MASTER02  k8s-master02
$K8S_MASTER03  k8s-master03
$K8S_NODE01  k8s-node01
$K8S_NODE02  k8s-node02
EOF
```

## 1.7.禁用swap

```bash
swapoff -a && sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
swapoff -a && sysctl -w vm.swappiness=0

cat /etc/fstab | grep swap
```

## 1.8.时间同步

Linux大全：https://www.linuxcool.com/chronyc

```bash
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

## 1.9.配置ulimit

```bash
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

## 1.10.配置免密登录

```bash
apt install -y sshpass
ssh-keygen -f /root/.ssh/id_rsa -P ''
export IP="k8s-master01 k8s-master02 k8s-master03 k8s-node01 k8s-node02"
export SSHPASS=1qazZSE$
for HOST in $IP;do
     sshpass -e ssh-copy-id -o StrictHostKeyChecking=no $HOST
done
```

## 1.11.低内核升级内核至4.18版本以上

```bash
sudo apt-get upgrade linux-image-generic
dpkg --list | grep linux-image
uname -r
```

## 1.12.安装ipvsadm

```bash
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

## 1.13.修改内核参数

```bash
cat <<EOF > /etc/sysctl.d/k8s.conf
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
net.ipv6.conf.all.forwarding = 0

EOF

sysctl --system
```

## 1.14.域名解析

```bash
apt install resolvconf
systemctl enable resolvconf.service
echo -e "nameserver 8.8.8.8\nnameserver 8.8.4.4" >>  /etc/resolvconf/resolv.conf.d/head

systemctl start resolvconf.service
sudo systemctl restart resolvconf.service
sudo systemctl restart systemd-resolved.service
sudo systemctl status resolvconf.service
echo -e "nameserver 8.8.8.8\nnameserver 8.8.4.4" >> /etc/resolv.conf
```

# 2.k8s基本组件安装

## 2.1.所有k8s节点安装Containerd作为Runtime

https://containerd.io/downloads

https://github.com/containerd/containerd/blob/main/docs/getting-started.md

```bash
wget https://github.com/containernetworking/plugins/releases/download/v1.2.0/cni-plugins-linux-amd64-v1.2.0.tgz

#创建cni插件所需目录
mkdir -p /etc/cni/net.d /opt/cni/bin
#解压cni二进制包
tar xf cni-plugins-linux-amd64-v1.2.0.tgz -C /opt/cni/bin/

wget https://github.com/containerd/containerd/releases/download/v1.6.21/cri-containerd-cni-1.6.21-linux-amd64.tar.gz

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

# 修改Containerd的配置文件
sed -i "s#SystemdCgroup\ \=\ false#SystemdCgroup\ \=\ true#g" /etc/containerd/config.toml
cat /etc/containerd/config.toml | grep SystemdCgroup

sed -i "s#registry.k8s.io#registry.cn-hangzhou.aliyuncs.com/image-storage#g" /etc/containerd/config.toml
cat /etc/containerd/config.toml | grep sandbox_image
```

镜像加速：

https://help.aliyun.com/document_detail/60750.html#section-hif-tpa-zl8

```bash
# 镜像加速
sed -i "s#config_path = \"\"#config_path = \"/etc/containerd/certs.d\"#g" /etc/containerd/config.toml
cat /etc/containerd/config.toml | grep config_path

mkdir /etc/containerd/certs.d/docker.io -pv
cat > /etc/containerd/certs.d/docker.io/hosts.toml << EOF
server = "https://docker.io"
[host."https://registry-1.docker.io"]
  capabilities = ["pull", "resolve"]
[host."https://tlwy9ltd.mirror.aliyuncs.com"]
  capabilities = ["pull", "resolve"]
[host."https://2lefsjdg.mirror.aliyuncs.com"]
  capabilities = ["pull", "resolve"]
EOF
```

### 2.1.5启动并设置为开机启动

```bash
systemctl daemon-reload
systemctl enable --now containerd
systemctl restart containerd
```

### 2.1.6配置crictl客户端连接的运行时位置

```bash
wget https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.26.1/crictl-v1.26.1-linux-amd64.tar.gz

#解压
tar xf crictl-v1.26.1-linux-amd64.tar.gz -C /usr/bin/
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

### 2.1.7nerdctli和buildkit命令安装

- https://github.com/containerd/nerdctl/releases
- https://github.com/moby/buildkit/releases

`nerdctl` 也可以和 `buildkit` 结合使用来构建容器镜像。

buildkit安装：

```bash
# buildkit安装
wget https://github.com/moby/buildkit/releases/download/v0.11.6/buildkit-v0.11.6.linux-amd64.tar.gz
tar -zxvf buildkit-v0.11.6.linux-amd64.tar.gz --strip-components=1 bin/buildctl bin/buildkit-runc bin/buildkitd
mv buildctl buildkit-runc buildkitd /usr/local/bin

# https://github.com/moby/buildkit/tree/master/examples/systemd/system
cat > /usr/lib/systemd/system/buildkit.service <<EOF
[Unit]
Description=BuildKit
Requires=buildkit.socket
After=buildkit.socket
Documentation=https://github.com/moby/buildkit

[Service]
Type=notify
ExecStart=/usr/local/bin/buildkitd --oci-worker=false --containerd-worker=true

[Install]
WantedBy=multi-user.target
EOF

cat > /usr/lib/systemd/system/buildkit.socket <<EOF
[Unit]
Description=BuildKit
Documentation=https://github.com/moby/buildkit

[Socket]
ListenStream=%t/buildkit/buildkitd.sock
SocketMode=0660

[Install]
WantedBy=sockets.target
EOF

systemctl daemon-reload
systemctl enable --now buildkit
```

nerdctl安装：

```bash
wget https://github.com/containerd/nerdctl/releases/download/v1.3.1/nerdctl-1.3.1-linux-amd64.tar.gz
mkdir -p /usr/local/containerd/bin && \
tar -zxvf nerdctl-1.3.1-linux-amd64.tar.gz nerdctl && \
mv nerdctl /usr/local/containerd/bin
ln -s /usr/local/containerd/bin/nerdctl /usr/bin/nerdctl
echo "source <(nerdctl completion bash)" >> /etc/profile
source /etc/profile
nerdctl version
```

## 2.2.k8s与etcd下载及安装（仅在master01操作）

### 2.2.1解压k8s安装包

GitHub：https://github.com/kubernetes/kubernetes/tree/v1.26.5

华为国内地址：https://mirrors.huaweicloud.com/etcd/v3.5.9/etcd-v3.5.9-linux-amd64.tar.gz

GitHub：https://github.com/etcd-io/etcd/releases/download/v3.5.9/etcd-v3.5.9-linux-amd64.tar.gz

```bash
# k8s二进制包
mkdir -p $HOME/k8s-install && cd $HOME/k8s-install
K8S_VERSION="v1.26.5"
wget https://dl.k8s.io/$K8S_VERSION/kubernetes-server-linux-amd64.tar.gz
tar -xf kubernetes-server-linux-amd64.tar.gz  --strip-components=3 -C /usr/local/bin kubernetes/server/bin/kube{let,ctl,-apiserver,-controller-manager,-scheduler,-proxy}

#etcd二进制包
# Desc:
ETCDCTL_API=3
ETCD_VER=v3.5.9
ETCD_DIR=etcd-download
DOWNLOAD_URL=https://mirrors.huaweicloud.com/etcd
# Download
mkdir -p  ${ETCD_DIR} && cd ${ETCD_DIR} && rm -rf etcd-${ETCD_VER}-linux-amd64.tar.gz  etcd-${ETCD_VER}-linux-amd64
wget ${DOWNLOAD_URL}/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz
tar -xf etcd-$ETCD_VER-linux-amd64.tar.gz --strip-components=1 -C /usr/local/bin etcd-$ETCD_VER-linux-amd64/etcd{,ctl}

# 查看/usr/local/bin下内容
ls /usr/local/bin/
etcd  etcdctl  kube-apiserver  kube-controller-manager  kubectl  kubelet  kube-proxy  kube-scheduler
```

### 2.2.2查看版本

```bash
# kubelet --version
Kubernetes v1.26.5
# etcdctl version
etcdctl version: 3.5.9
API version: 3.5
```

### 2.2.3将组件发送至其他k8s节点

```bash
Master='k8s-master02 k8s-master03'
Work='k8s-node01 k8s-node02'


for NODE in $Master;
do 
    echo "正在传输的主机地址：$NODE ...";
    scp /usr/local/bin/kube{let,ctl,-apiserver,-controller-manager,-scheduler,-proxy} $NODE:/usr/local/bin/;
    scp /usr/local/bin/etcd* $NODE:/usr/local/bin/;
done

for NODE in $Work;
do  
    echo "正在传输的主机地址：$NODE ...";
    scp /usr/local/bin/kube{let,-proxy,ctl} $NODE:/usr/local/bin/ ; 
done

# 添加tab功能
apt install -y bash-completion
source /usr/share/bash-completion/bash_completion
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc
source ~/.bashrc
```

## 2.3.创建证书相关文件

```bash
mkdir -p /root/k8s/ssl && cd /root/k8s/ssl
cat > admin-csr.json << EOF 
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

cat > ca-config.json << EOF 
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

cat > etcd-ca-csr.json  << EOF 
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

cat > front-proxy-ca-csr.json  << EOF 
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

cat > kubelet-csr.json  << EOF 
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

cat > manager-csr.json << EOF 
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

cat > apiserver-csr.json << EOF 
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


cat > ca-csr.json   << EOF 
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

cat > etcd-csr.json << EOF 
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


cat > front-proxy-client-csr.json  << EOF 
{
  "CN": "front-proxy-client",
  "key": {
     "algo": "rsa",
     "size": 2048
  }
}
EOF


cat > kube-proxy-csr.json  << EOF 
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


cat > scheduler-csr.json << EOF 
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

cd .. ; mkdir -p bootstrap ; cd bootstrap
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


cd .. ; mkdir -p coredns && cd coredns
cat > coredns.yaml << EOF 
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
  template:
    metadata:
      labels:
        k8s-app: kube-dns
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
           preferredDuringSchedulingIgnoredDuringExecution:
           - weight: 100
             podAffinityTerm:
               labelSelector:
                 matchExpressions:
                   - key: k8s-app
                     operator: In
                     values: ["kube-dns"]
               topologyKey: kubernetes.io/hostname
      containers:
      - name: coredns
        image: registry.cn-beijing.aliyuncs.com/dotbalo/coredns:1.8.6 
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
spec:
  selector:
    k8s-app: kube-dns
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
EOF
```

# 3.相关证书生成

```bash
# master01节点下载证书生成工具
wget "https://github.com/cloudflare/cfssl/releases/download/v1.6.1/cfssl_1.6.1_linux_amd64" -O /usr/local/bin/cfssl
wget "https://github.com/cloudflare/cfssl/releases/download/v1.6.1/cfssljson_1.6.1_linux_amd64" -O /usr/local/bin/cfssljson
chmod +x /usr/local/bin/cfssl*
```

## 3.1.生成etcd证书

- master节点操作

### 3.1.1所有master节点创建证书存放目录

```bash
mkdir /etc/etcd/ssl -p
```

### 3.1.2master01节点生成etcd证书

```bash
mkdir -p /root/k8s/ssl && cd /root/k8s/ssl
# 生成etcd证书和etcd证书的key（如果你觉得以后可能会扩容，可以在ip那多写几个预留出来）
cfssl gencert -initca etcd-ca-csr.json | cfssljson -bare /etc/etcd/ssl/etcd-ca

cfssl gencert \
   -ca=/etc/etcd/ssl/etcd-ca.pem \
   -ca-key=/etc/etcd/ssl/etcd-ca-key.pem \
   -config=ca-config.json \
   -hostname=127.0.0.1,k8s-master01,k8s-master02,k8s-master03,k8s-node01,k8s-node02,$K8S_MASTER01,$K8S_MASTER02,$K8S_MASTER03,$K8S_NODE01,$K8S_NODE02 \
   -profile=kubernetes \
   etcd-csr.json | cfssljson -bare /etc/etcd/ssl/etcd
```

### 3.1.3将证书复制到其他etcd节点

```bash
Master='k8s-master02 k8s-master03'

for NODE in $Master;
do
    ssh $NODE "mkdir -p /etc/etcd/ssl";
    for FILE in etcd-ca-key.pem  etcd-ca.pem  etcd-key.pem  etcd.pem;
    do
        scp /etc/etcd/ssl/${FILE} $NODE:/etc/etcd/ssl/${FILE}; 
    done;
done
```

## 3.2.生成k8s相关证书

- master节点操作

### 3.2.1所有k8s节点创建证书存放目录

```bash
mkdir -p /etc/kubernetes/pki
```

### 3.2.2master01节点生成k8s证书

```bash
cfssl gencert -initca ca-csr.json | cfssljson -bare /etc/kubernetes/pki/ca

# 生成一个根证书 ，多写了一些IP作为预留IP，为将来添加node做准备
# 10.96.0.1是service网段的第一个地址，需要计算，192.168.80.100为高可用vip地址
# 若没有IPv6 可删除可保留 

cfssl gencert   \
-ca=/etc/kubernetes/pki/ca.pem   \
-ca-key=/etc/kubernetes/pki/ca-key.pem   \
-config=ca-config.json   \
-hostname=10.96.0.1,$K8S_VIP,127.0.0.1,kubernetes,kubernetes.default,kubernetes.default.svc,kubernetes.default.svc.cluster,kubernetes.default.svc.cluster.local,$K8S_MASTER01,$K8S_MASTER02,$K8S_MASTER03,$K8S_NODE01,$K8S_NODE02,192.168.80.50,192.168.80.51,192.168.80.52,192.168.80.53,192.168.80.54,10.0.0.40,10.0.0.41 \
-profile=kubernetes   apiserver-csr.json | cfssljson -bare /etc/kubernetes/pki/apiserver

ls /etc/kubernetes/pki/apiserver*
```

### 3.2.3生成apiserver聚合证书

```bash
cfssl gencert   -initca front-proxy-ca-csr.json | cfssljson -bare /etc/kubernetes/pki/front-proxy-ca

# 有一个警告，可以忽略
cfssl gencert  \
-ca=/etc/kubernetes/pki/front-proxy-ca.pem   \
-ca-key=/etc/kubernetes/pki/front-proxy-ca-key.pem   \
-config=ca-config.json   \
-profile=kubernetes   front-proxy-client-csr.json | cfssljson -bare /etc/kubernetes/pki/front-proxy-client
```

### 3.2.4生成kube-proxy证书

```bash
cd /root/k8s/ssl
cfssl gencert \
   -ca=/etc/kubernetes/pki/ca.pem \
   -ca-key=/etc/kubernetes/pki/ca-key.pem \
   -config=ca-config.json \
   -profile=kubernetes \
   kube-proxy-csr.json | cfssljson -bare /etc/kubernetes/pki/kube-proxy
   
kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/pki/ca.pem  \
  --embed-certs=true \
  --server=https://$K8S_VIP:8443     \
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

### 3.2.4生成controller-manage的证书

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
     --server=https://$K8S_VIP:8443 \
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
     --server=https://$K8S_VIP:8443 \
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
  --server=https://$K8S_VIP:8443     \
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

### 3.2.5创建ServiceAccount Key ——secret

```bash
openssl genrsa -out /etc/kubernetes/pki/sa.key 2048
openssl rsa -in /etc/kubernetes/pki/sa.key -pubout -out /etc/kubernetes/pki/sa.pub
```

### 3.2.6将证书发送到其他master节点

```bash
for NODE in k8s-master02 k8s-master03;
do
    for FILE in $(ls /etc/kubernetes/pki | grep -v etcd); 
    do
        ssh $NODE "mkdir -p /etc/kubernetes/pki"
        scp /etc/kubernetes/pki/${FILE} $NODE:/etc/kubernetes/pki/${FILE}; 
    done;
    for FILE in admin.kubeconfig controller-manager.kubeconfig scheduler.kubeconfig kube-proxy.kubeconfig; 
    do
        ssh $NODE "mkdir -p /etc/kubernetes"
        scp /etc/kubernetes/${FILE} $NODE:/etc/kubernetes/${FILE}; 
    done;
done
```

### 3.2.7查看证书

```bash
ls /etc/kubernetes/pki/
admin.csr      apiserver-key.pem  ca.pem                      etcd                    front-proxy-client.csr      kubelet.key         sa.key             scheduler.pem
admin-key.pem  apiserver.pem      controller-manager.csr      front-proxy-ca.csr      front-proxy-client-key.pem  kube-proxy.csr      sa.pub
admin.pem      ca.csr             controller-manager-key.pem  front-proxy-ca-key.pem  front-proxy-client.pem      kube-proxy-key.pem  scheduler.csr
apiserver.csr  ca-key.pem         controller-manager.pem      front-proxy-ca.pem      kubelet.crt                 kube-proxy.pem      scheduler-key.pem

ls /etc/kubernetes/pki/ | wc -l
26
```

# 4.k8s系统组件配置

## 4.1.配置文件内容

- 所有master节点操作

添加环境变量：

```bash
cd ~
if [ $LOCALHOST == $K8S_MASTER01 ];then
   echo "etcd='k8s-etcd01'" >>  ~/.bashrc
elif [ $LOCALHOST == $K8S_MASTER02 ];then
   echo "etcd='k8s-etcd02'" >>  ~/.bashrc
elif [ $LOCALHOST == $K8S_MASTER03 ];then
  echo "etcd='k8s-etcd03'" >>  ~/.bashrc
fi
source ~/.bashrc
```

配置`etcd`启动文件内容：

```bash
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
initial-cluster: "k8s-etcd01=https://$K8S_MASTER01:2380,k8s-etcd02=https://$K8S_MASTER02:2380,k8s-etcd03=https://$K8S_MASTER03:2380"
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

## 4.2.创建service（所有master节点操作）

### 4.2.1创建etcd.service并启动

```bash
cat > /usr/lib/systemd/system/etcd.service << EOF

[Unit]
Description=Etcd Service
Documentation=https://coreos.com/etcd/docs/latest/
After=network.target

[Service]
Type=notify
ExecStart=/usr/local/bin/etcd --config-file=/etc/etcd/etcd.config.yml
Restart=on-failure
RestartSec=10
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
Alias=etcd3.service

EOF
```

### 4.2.2创建etcd证书目录

```bash
mkdir /etc/kubernetes/pki/etcd
ln -s /etc/etcd/ssl/* /etc/kubernetes/pki/etcd/
systemctl daemon-reload
systemctl enable --now etcd
systemctl status etcd.service
```

注：好的状态全部是`info`。

### 4.2.3查看etcd状态

```bash
export ETCDCTL_API=3
etcdctl --endpoints="$K8S_MASTER01:2379,$K8S_MASTER02:2379,$K8S_MASTER03:2379" --cacert=/etc/kubernetes/pki/etcd/etcd-ca.pem --cert=/etc/kubernetes/pki/etcd/etcd.pem --key=/etc/kubernetes/pki/etcd/etcd-key.pem  endpoint status --write-out=table
+--------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|      ENDPOINT      |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+--------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| 192.168.80.45:2379 | aeceb08f95191d44 |   3.5.4 |   20 kB |      true |      false |         2 |          9 |                  9 |        |
| 192.168.80.46:2379 | e33e882bc326495d |   3.5.4 |   20 kB |     false |      false |         2 |          9 |                  9 |        |
| 192.168.80.47:2379 | c39d0bf63332ca37 |   3.5.4 |   20 kB |     false |      false |         2 |          9 |                  9 |        |
+--------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
```

# 5.高可用配置

## 5.1.在master01和master02和master03服务器上操作

### 5.1.1安装keepalived和haproxy服务

```bash
apt -y install keepalived haproxy
```

### 5.1.2修改haproxy配置文件（配置文件一样）

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

### 5.1.3Master01配置keepalived master节点

- 全部master节点操作

注：这里keepalived使用的是非抢占模式：priority id

```bash
if [ $LOCALHOST == $K8S_MASTER01 ];then
    export PRIORITY="100"
elif [ $LOCALHOST == $K8S_MASTER02 ];then
    export PRIORITY="80"
elif [ $LOCALHOST == $K8S_MASTER03 ];then
    export PRIORITY="50"
fi

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
    priority $PRIORITY
    nopreempt
    advert_int 2
    authentication {
        auth_type PASS
        auth_pass K8SHA_KA_AUTH
    }
    virtual_ipaddress {
        $K8S_VIP
    }
    track_script {
      chk_apiserver 
 } }

EOF
```

### 5.1.4健康检查脚本配置

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
```

### 5.1.5启动服务

```bash
systemctl daemon-reload
systemctl enable --now haproxy
systemctl enable --now keepalived
systemctl restart haproxy.service && netstat -lntup | grep 443
```

### 5.1.6测试高可用

```bash
ping 192.168.80.100
# 关闭主节点，看vip是否漂移到备节点
```

**注：现在apiserver还有创建，所以先不`telnet`8443端口，因为负载不到`6443`端口。**

# 6.k8s组件配置（区别于第4点）

## 6.1.创建apiserver

### 6.1.1准备目录

所有k8s节点创建以下目录

```bash
mkdir -p /etc/kubernetes/manifests/ /etc/systemd/system/kubelet.service.d /var/lib/kubelet /var/log/kubernetes
```

### 6.1.2master所有节点配置

`--service-cluster-ip-range=10.96.0.0/12`: Service IP 段

```bash
cat > /usr/lib/systemd/system/kube-apiserver.service << EOF

[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
ExecStart=/usr/local/bin/kube-apiserver \\
      --v=2  \\
      --allow-privileged=true  \\
      --bind-address=0.0.0.0  \\
      --secure-port=6443  \\
      --advertise-address=$LOCALHOST \\
      --service-cluster-ip-range=10.96.0.0/12,fd00::/108  \\
      --service-node-port-range=30000-32767  \\
      --etcd-servers=https://$K8S_MASTER01:2379,https://$K8S_MASTER02:2379,https://$K8S_MASTER03:2379 \\
      --etcd-cafile=/etc/etcd/ssl/etcd-ca.pem  \\
      --etcd-certfile=/etc/etcd/ssl/etcd.pem  \\
      --etcd-keyfile=/etc/etcd/ssl/etcd-key.pem  \\
      --client-ca-file=/etc/kubernetes/pki/ca.pem  \\
      --tls-cert-file=/etc/kubernetes/pki/apiserver.pem \\
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
      --enable-aggregator-routing=true
      #--feature-gates=IPv6DualStack=true

Restart=on-failure
RestartSec=10s
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target

EOF
```

注：在1.24+ 版本中已经遗弃了 --insecure-port、–enable-swagger-ui 这两个参数。

### 6.1.3启动apiserver（所有master节点）

```bash
systemctl daemon-reload && systemctl enable --now kube-apiserver
journalctl -u kube-apiserver
# 注意查看状态是否启动正常

systemctl restart kube-apiserver
systemctl status kube-apiserver
netstat -lntup | grep 6443
#测试haproxy 8443端口
telnet 192.168.80.100 8443
Trying 192.168.80.100...
Connected to 192.168.80.100.
Escape character is '^]'.
```

## 6.2.配置kube-controller-manager service

```bash
# 所有master节点配置，且配置相同
# 172.16.0.0/12为pod网段，按需求设置你自己的网段

cat > /usr/lib/systemd/system/kube-controller-manager.service << EOF

[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \\
      --v=2 \\
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
      #--feature-gates=IPv6DualStack=true

Restart=always
RestartSec=10s

[Install]
WantedBy=multi-user.target

EOF
```

\# 要是没有配置ipv6就不要打开，打开会出现 Get "http://127.0.0.1:10252/healthz": dial tcp 127.0.0.1:10252: connect: connection refused 报错

### 6.2.1启动kube-controller-manager，并查看状态

```bash
systemctl daemon-reload
systemctl enable --now kube-controller-manager
systemctl status kube-controller-manager
```

## 6.3.配置kube-scheduler service

### 6.3.1所有master节点配置，且配置相同

```bash
cat > /usr/lib/systemd/system/kube-scheduler.service << EOF

[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
ExecStart=/usr/local/bin/kube-scheduler \\
      --v=2 \\
      --bind-address=127.0.0.1 \\
      --leader-elect=true \\
      --kubeconfig=/etc/kubernetes/scheduler.kubeconfig

Restart=always
RestartSec=10s

[Install]
WantedBy=multi-user.target

EOF
```

### 6.3.2启动并查看服务状态

```bash
systemctl daemon-reload
systemctl enable --now kube-scheduler
systemctl status kube-scheduler
```

# 7. TLS Bootstrapping配置

## 7.1.在master01上配置

```bash
kubectl config set-cluster kubernetes     \
--certificate-authority=/etc/kubernetes/pki/ca.pem     \
--embed-certs=true  \
--server=https://$K8S_VIP:8443     \
--kubeconfig=/etc/kubernetes/bootstrap-kubelet.kubeconfig

kubectl config set-credentials tls-bootstrap-token-user     \
--token=c8ad9c.2e4d610cf3e7426e \
--kubeconfig=/etc/kubernetes/bootstrap-kubelet.kubeconfig

kubectl config set-context tls-bootstrap-token-user@kubernetes     \
--cluster=kubernetes     \
--user=tls-bootstrap-token-user     \
--kubeconfig=/etc/kubernetes/bootstrap-kubelet.kubeconfig

kubectl config use-context tls-bootstrap-token-user@kubernetes     \
--kubeconfig=/etc/kubernetes/bootstrap-kubelet.kubeconfig

# token的位置在bootstrap.secret.yaml，如果修改的话到这个文件修改

mkdir -p /root/.kube ; cp /etc/kubernetes/admin.kubeconfig /root/.kube/config
for NODE in k8s-master02 k8s-master03 k8s-node01 k8s-node02;
do
    ssh $NODE  "mkdir -p /root/.kube"
    scp /root/.kube/config $NODE:/root/.kube
done
```

## 7.2.查看集群状态，没问题的话继续后续操作

```bash
kubectl get cs

Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS    MESSAGE                         ERROR
scheduler            Healthy   ok                              
controller-manager   Healthy   ok                              
etcd-0               Healthy   {"health":"true","reason":""}   
etcd-2               Healthy   {"health":"true","reason":""}   
etcd-1               Healthy   {"health":"true","reason":""} 

cd /root/k8s/bootstrap
kubectl create -f bootstrap.secret.yaml
```

# 8.node节点配置

## 8.1.在master01上将证书复制到node节点

```bash
cd /etc/kubernetes/

for NODE in k8s-master02 k8s-master03 k8s-node01 k8s-node02; 
do 
    ssh $NODE mkdir -p /etc/kubernetes/pki; 
    for FILE in pki/ca.pem pki/ca-key.pem pki/front-proxy-ca.pem bootstrap-kubelet.kubeconfig kube-proxy.kubeconfig; 
    do
        scp /etc/kubernetes/$FILE $NODE:/etc/kubernetes/${FILE}; 
    done; 
done
```

## 8.2.kubelet配置

### 8.2.1当使用Containerd作为Runtime

- **k8s-master01、k8s-node01、k8s-node02操作**

```bash
# 所有k8s节点配置kubelet service
cat > /usr/lib/systemd/system/kubelet.service << EOF

[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=containerd.service
Requires=containerd.service

[Service]
ExecStart=/usr/local/bin/kubelet \\
    --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.kubeconfig  \\
    --kubeconfig=/etc/kubernetes/kubelet.kubeconfig \\
    --config=/etc/kubernetes/kubelet-conf.yml \\
    --container-runtime=remote  \\
    --runtime-request-timeout=15m  \\
    --container-runtime-endpoint=unix:///run/containerd/containerd.sock  \\
    --cgroup-driver=systemd \\
    --node-labels=node.kubernetes.io/node=
    # --feature-gates=IPv6DualStack=true

[Install]
WantedBy=multi-user.target
EOF
```

### 8.2.2所有k8s节点创建kubelet的配置文件

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

### 8.2.3启动kubelet

```bash
systemctl daemon-reload
systemctl restart kubelet
systemctl enable --now kubelet
# systemctl status kubelet.service
journalctl -u kubelet -f
```

## 8.3.kube-proxy配置

### 8.3.1所有k8s节点添加kube-proxy的service文件

```bash
cat >  /usr/lib/systemd/system/kube-proxy.service << EOF
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
ExecStart=/usr/local/bin/kube-proxy \\
  --config=/etc/kubernetes/kube-proxy.yaml \\
  --v=2

Restart=always
RestartSec=10s

[Install]
WantedBy=multi-user.target

EOF
```

### 8.3.2所有k8s节点添加kube-proxy的配置

```bash
cat > /etc/kubernetes/kube-proxy.yaml << EOF
apiVersion: kubeproxy.config.k8s.io/v1alpha1
bindAddress: 0.0.0.0
clientConnection:
  acceptContentTypes: ""
  burst: 10
  contentType: application/vnd.kubernetes.protobuf
  kubeconfig: /etc/kubernetes/kube-proxy.kubeconfig
  qps: 5
clusterCIDR: 172.16.0.0/12,fc00::/48 
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

### 8.3.3启动kube-proxy

```bash
systemctl daemon-reload
systemctl enable --now kube-proxy
systemctl restart kube-proxy
netstat -lntup | grep 10249
# 查看当前使用是默认是iptables或ipvs
curl localhost:10249/proxyMode
```

# 9.安装Calico

## 9.1安装Calico

注意对应的版本：https://docs.tigera.io/calico/latest/getting-started/kubernetes/requirements

参考github内容：https://github.com/projectcalico/calico/blob/v3.26.0/manifests/calico.yaml

```bash
mkdir -p $HOME/k8s-install/network && cd $HOME/k8s-install/network
# CIDR的值，与 kube-controller-manager中“--cluster-cidr=172.16.0.0/16” 一致
cat calico.yaml | grep -A 5 CALICO_IPV4POOL_CIDR
            - name: CALICO_IPV4POOL_CIDR
              value: "172.16.0.0/16"
            - name: IP_AUTODETECTION_METHOD
              value: "interface=ens33" # 指定网卡
            # Disable file logging so `kubectl logs` works.
            - name: CALICO_DISABLE_FILE_LOGGING
            
kubectl apply -f calico.yaml

kubectl get pod -n kube-system
NAME                                       READY   STATUS    RESTARTS      AGE
calico-kube-controllers-55fc758c88-7b65k   1/1     Running   0             40m
calico-node-bnkqh                          1/1     Running   0             40m
calico-node-djxn2                          1/1     Running   1 (17m ago)   40m
calico-node-n4x9q                          1/1     Running   0             40m
calico-node-spqbf                          1/1     Running   0             32m
calico-node-sw485                          1/1     Running   0             40m
```

# 10.安装CoreDNS

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
        image: coredns/coredns:1.9.3  #对比下与本地k8s版本是否符合
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
  clusterIP: 10.96.0.10 #这里改成自己的clusterDNS
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
# 启动文件
kubectl apply -f coredns.yaml

#查看pod情况
kubectl get pod  -n kube-system -l k8s-app=kube-dns
coredns-857d6b74c4-tn59p                   1/1     Running   0          95s

# 查看集群
kubectl  get node
NAME           STATUS   ROLES    AGE   VERSION
k8s-master01   Ready    <none>   75m   v1.26.5
k8s-master02   Ready    <none>   72m   v1.26.5
k8s-master03   Ready    <none>   72m   v1.26.5
k8s-node01     Ready    <none>   71m   v1.26.5
k8s-node02     Ready    <none>   71m   v1.26.5
```

# 11.安装Metrics Server

GitHub：https://github.com/kubernetes-sigs/metrics-server

helm：https://artifacthub.io/packages/helm/metrics-server/metrics-server

## 11.1.安装Metrics-server

```bash
wget https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.6.3/components.yaml
sed -i s#registry.k8s.io/metrics-server/metrics-server:v0.6.3#registry.cn-hangzhou.aliyuncs.com/image-storage/metrics-server:v0.6.3#g components.yaml

# 参数部分
      containers:
      - args:
        - --cert-dir=/tmp
        - --secure-port=4443
        - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
        - --kubelet-use-node-status-port
        - --metric-resolution=15s
        - --kubelet-insecure-tls
        - --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.pem
        - --requestheader-username-headers=X-Remote-User
        - --requestheader-group-headers=X-Remote-Group
        - --requestheader-extra-headers-prefix=X-Remote-Extra-
        image: registry.cn-hangzhou.aliyuncs.com/image-storage/metrics-server:v0.6.3
        imagePullPolicy: IfNotPresent
# 挂载部分
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

        
kubectl apply -f components.yaml
```

## 11.2.稍等片刻查看状态

```bash
kubectl  top node
NAME           CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
k8s-master01   248m         12%    3083Mi          81%
k8s-master02   119m         5%     2971Mi          78%
k8s-master03   270m         13%    3040Mi          80%
k8s-node01     184m         9%     2803Mi          73%
k8s-node02     188m         9%     2903Mi          76%
```

# 12.集群验证

## 12.1.部署pod资源

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
```

## 12.2.用pod解析默认命名空间中的kubernete

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

## 12.3.测试跨命名空间是否可以解析

```bash
kubectl exec  busybox -n default -- nslookup kube-dns.kube-system
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      kube-dns.kube-system
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local
```

## 12.4.解析外网地址

```bash
kubectl exec  busybox -n default -- nslookup www.baidu.com
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      www.baidu.com
Address 1: 180.101.50.242
Address 2: 180.101.50.188
```

## 12.5.每个节点都必须要能访问Kubernetes的kubernetes svc 443和kube-dns的service 53

```bash
telnet 10.96.0.1 443
Trying 10.96.0.1...
Connected to 10.96.0.1.
Escape character is '^]'.

telnet 10.96.0.10 53
Trying 10.96.0.10...
Connected to 10.96.0.10.
Escape character is '^]'.
```

## 12.6.Pod和Pod之前要能通

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

## 12.7.创建三个副本，可以看到3个副本分布在不同的节点上

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
```

启动容器：

```bash
kubectl  apply -f deployments.yaml 
deployment.apps/nginx-deployment created

kubectl  get pod -owide -w
kubectl  get pod
NAME                                READY   STATUS    RESTARTS   AGE
busybox                             1/1     Running   0          24m
nginx-deployment-747f4457f5-86tbw   1/1     Running   0          11m
nginx-deployment-747f4457f5-ft5gp   1/1     Running   0          11m
nginx-deployment-747f4457f5-mlt25   1/1     Running   0          11m
```

# 13.打标和污点

```bash
# 标签
kubectl label node k8s-master01 k8s-master02 k8s-master03 node-role.kubernetes.io/master=
kubectl label node k8s-node01 k8s-node02 node-role.kubernetes.io/node=

# 污点
kubectl taint nodes k8s-master01 k8s-master02 k8s-master03 node-role.kubernetes.io/master=:NoSchedule
```

# 14.删除节点

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

# 8. 彻底删除加回来
kubectl get csr
kubectl certificate approve csr-pjsd8
# https://kubernetes.io/zh-cn/docs/tasks/tls/certificate-rotation
```