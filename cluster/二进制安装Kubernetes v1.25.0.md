 

部署步骤：

1. 先部署一个master + 两个node节点集群
2. 在加入两个master进来
3. 加入负载均衡

# 1. 部署规划表

| **hostname** | **IP**                    | Software                                                     | **Version**    |
| ------------ | ------------------------- | ------------------------------------------------------------ | -------------- |
| k8s-master01 | 192.168.80.61             | kube-apiserver、kube-controller-manager、kube-scheduler、etcd、 kubelet、kube-proxy、nfs-client、haproxy、keepalived | 20.04.1-Ubuntu |
| k8s-master02 | 192.168.80.62（最后部署） | kube-apiserver、kube-controller-manager、kube-scheduler、etcd、 kubelet、kube-proxy、nfs-client、haproxy、keepalived | 20.04.1-Ubuntu |
| k8s-master03 | 192.168.80.63（最后部署） | kube-apiserver、kube-controller-manager、kube-scheduler、etcd、 kubelet、kube-proxy、nfs-client、haproxy、keepalived | 20.04.1-Ubuntu |
| k8s-node01   | 192.168.80.64             | kubelet、kube-proxy、nfs-client                              | 20.04.1-Ubuntu |
| k8s-node02   | 192.168.80.65             | kubelet、kube-proxy、nfs-client                              | 20.04.1-Ubuntu |
| VIP          | 192.168.80.66             |                                                              |                |

**版本表：**

| **软件**                                                     | **版本**  |
| ------------------------------------------------------------ | --------- |
| kernel                                                       | 5.4.0-125 |
| Ubuntu                                                       | 20.04.1   |
| kube-apiserver、kube-controller-manager、kube-scheduler、kubelet、kube-proxy | v1.25.0   |
| etcd                                                         | v3.5.4    |
| containerd                                                   | v1.6.8    |
| cfssl                                                        | v1.6.1    |
| cni                                                          | v1.1.1    |
| crictl                                                       | v1.24.2   |
| haproxy                                                      | v1.8.27   |
| keepalived                                                   | v2.1.5    |

网段：

物理主机：192.168.80.0/24

service：10.96.0.0/12

pod：172.16.0.0/12

 

# 2. 环境准备

- **k8s-master01、k8s-node01、k8s-node02操作**

```bash
# 1. 修改主机名
hostnamectl set-hostname k8s-master01
hostnamectl set-hostname k8s-node01
hostnamectl set-hostname k8s-node02

# 2. 主机名解析
cat >> /etc/hosts <<EOF
192.168.80.61  k8s-master01
192.168.80.62  k8s-master02
192.168.80.63  k8s-master03
192.168.80.64  k8s-node01
192.168.80.65  k8s-node02
EOF

# 3. 禁用 swap
sed -ri 's/.*swap.*/#&/' /etc/fstab
swapoff -a && sysctl -w vm.swappiness=0

# 4. 将桥接的IPv4流量传递到iptables的链 
cat > /etc/sysctl.d/k8s.conf << EOF 
net.bridge.bridge-nf-call-ip6tables = 1 
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system

# 5. 域名解析
echo "nameserver 8.8.8.8" >> /etc/resolv.conf
 
# 6. 进行时间同步（服务端）
apt install chrony -y
cat > /etc/chrony.conf << EOF 
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

# 查看系统时间与日期
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
hwclock --systohc
date -R
timedatectl set-local-rtc yes
timedatectl

systemctl restart chronyd.service

# 进行实际同步（客户端）
apt install chrony -y
cat > /etc/chrony.conf << EOF 
pool 192.168.80.61 iburst
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

# 7. 日志目录
mkdir -p /var/log/kubernetes

# 8.配置ulimit
ulimit -SHn 65535
cat >> /etc/security/limits.conf <<EOF
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
export IP="192.168.80.61 192.168.80.64 192.168.80.65 "
export SSHPASS=kerwin
for HOST in $IP;do
     sshpass -e ssh-copy-id -o StrictHostKeyChecking=no $HOST
done

# 10.ipvsadm安装（lb除外）
apt-get install ipvsadm
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
net.ipv6.conf.all.forwarding = 1
EOF

sysctl --system

# 命令补全
apt install bash-completion -y
source /usr/share/bash-completion/bash_completion
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc
```

# 3. k8s基本组件安装

## 3.1 k8s节点安装Containerd作为Runtime

- **k8s-master01、k8s-node01、k8s-node02操作**

```bash
# wget -P $HOME/k8s-install  https://github.com/containernetworking/plugins/releases/download/v1.1.1/cni-plugins-linux-amd64-v1.1.1.tgz

#创建cni插件所需目录
mkdir -p /etc/cni/net.d /opt/cni/bin
#解压cni二进制包
mkdir -p $HOME/k8s-install && cd $HOME/k8s-install
tar xf cni-plugins-linux-amd64-v1.1.1.tgz -C /opt/cni/bin/

# wget -P $HOME/k8s-install https://github.com/containerd/containerd/releases/download/v1.6.8/cri-containerd-cni-1.6.8-linux-amd64.tar.gz

#解压
cd $HOME/k8s-install
tar -xzf cri-containerd-cni-1.6.8-linux-amd64.tar.gz -C /

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

### 3.1.1 配置Containerd所需的模块

```bash
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
```

### 3.1.2 加载模块

```bash
systemctl restart systemd-modules-load.service
```

### 3.1.3 配置Containerd所需的内核

```bash
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

# 加载内核

sysctl --system
```

### 3.1.4 创建Containerd的配置文件

```bash
# 创建默认配置文件
mkdir -p /etc/containerd
containerd config default | tee /etc/containerd/config.toml

# 修改Containerd的配置文件
sed -i "s#SystemdCgroup\ \=\ false#SystemdCgroup\ \=\ true#g" /etc/containerd/config.toml
cat /etc/containerd/config.toml | grep SystemdCgroup

sed -i "s#k8s.gcr.io#registry.cn-hangzhou.aliyuncs.com/chenby#g" /etc/containerd/config.toml
cat /etc/containerd/config.toml | grep sandbox_image

sed -i "s#k8s.gcr.io#registry.cn-hangzhou.aliyuncs.com/chenby#g" /etc/containerd/config.toml

sed -i "s#config_path\ \=\ \"\"#config_path\ \=\ \"/etc/containerd/certs.d\"#g" /etc/containerd/config.toml
 
mkdir /etc/containerd/certs.d/docker.io -pv

cat > /etc/containerd/certs.d/docker.io/hosts.toml << EOF
server = "https://docker.io"
[host."https://hub-mirror.c.163.com"]
  capabilities = ["pull", "resolve"]
EOF
```

### 3.1.5 启动并设置为开机启动

```bash
systemctl daemon-reload
systemctl enable --now containerd
systemctl restart containerd
```

### 3.1.6 配置crictl客户端连接的运行时位置

```bash
# wget -P $HOME/k8s-install https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.24.2/crictl-v1.24.2-linux-amd64.tar.gz

#解压
cd $HOME/k8s-install
tar xf crictl-v1.24.2-linux-amd64.tar.gz -C /usr/bin/
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

## 3.2 k8s与etcd下载及安装

- k8s-master01节点操作

### 3.2.1解压k8s安装包

```bash
# 下载安装包
# wget -P $HOME/k8s-install https://dl.k8s.io/v1.25.0/kubernetes-server-linux-amd64.tar.gz
# wget -P $HOME/k8s-install https://github.com/etcd-io/etcd/releases/download/v3.5.4/etcd-v3.5.4-linux-amd64.tar.gz

# 解压k8s安装文件
mkdir -p $HOME/k8s-install && cd $HOME/k8s-install
tar -xf kubernetes-server-linux-amd64.tar.gz  --strip-components=3 -C /usr/local/bin kubernetes/server/bin/kube{let,ctl,-apiserver,-controller-manager,-scheduler,-proxy}

# 解压etcd安装文件
tar -xf etcd*.tar.gz && mv etcd-*/etcd /usr/local/bin/ && mv etcd-*/etcdctl /usr/local/bin/

# 查看/usr/local/bin下内容

ls /usr/local/bin/
containerd  containerd-shim-runc-v1  containerd-stress  critest  ctr   etcdctl  kube-controller-manager  kubelet  kube-scheduler  containerd-shim  containerd-shim-runc-v2  crictl  ctd-decoder  etcd  kube-apiserver  kubectl  kube-proxy
```

### 3.2.2 查看版本

```bash
[root@k8s-master01 ~]# kubelet --version
Kubernetes v1.25.0
[root@k8s-master01 ~]# etcdctl version
etcdctl version: 3.5.4
API version: 3.5
```

### 3.2.3 将组件发送至其他k8s节点

```bash
Work='k8s-node01 k8s-node02'

for NODE in $Work; do     scp /usr/local/bin/kube{let,-proxy} $NODE:/usr/local/bin/ ; done

mkdir -p /opt/cni/bin
```

## 3.3 创建证书相关文件

- **k8s-master01上操作**

```bash
mkdir -p /root/ssl && cd /root/ssl
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

cd .. && mkdir -p bootstrap && cd bootstrap
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


cd .. && mkdir -p coredns && cd coredns
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


cd .. && mkdir -p metrics-server && cd metrics-server
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

# 4. 相关证书生成

```bash
# master01节点下载证书生成工具
# wget "https://github.com/cloudflare/cfssl/releases/download/v1.6.1/cfssl_1.6.1_linux_amd64" -O /usr/local/bin/cfssl
# wget "https://github.com/cloudflare/cfssl/releases/download/v1.6.1/cfssljson_1.6.1_linux_amd64" -O /usr/local/bin/cfssljson

chmod +x /usr/local/bin/cfssl /usr/local/bin/cfssljson
```

https://github.com/cloudflare/cfssl/releases

## 4.1 生成etcd证书

特别说明除外，以下操作在所有master节点操作

```bash
mkdir /etc/etcd/ssl -p
```

## 4.1.1 master01节点生成etcd证书

```bash
mkdir -p /root/ssl && cd /root/ssl
# 生成etcd证书和etcd证书的key（如果你觉得以后可能会扩容，可以在ip那多写几个预留出来）
# 若没有IPv6 可删除可保留 
cfssl gencert -initca etcd-ca-csr.json | cfssljson -bare /etc/etcd/ssl/etcd-ca
cfssl gencert \
   -ca=/etc/etcd/ssl/etcd-ca.pem \
   -ca-key=/etc/etcd/ssl/etcd-ca-key.pem \
   -config=ca-config.json \
   -hostname=127.0.0.1,k8s-master01,k8s-node01,k8s-node02,192.168.80.61,192.168.80.64,192.168.80.65  \
   -profile=kubernetes \
   etcd-csr.json | cfssljson -bare /etc/etcd/ssl/etcd
   
# 我们将etcd放在了k8s-master01、k8s-node01、k8s-node02中，上面的地址关联
```

## 4.1.2 将证书复制到其他节点

```bash
Master='k8s-node01 k8s-node02'
for NODE in $Master; do ssh $NODE "mkdir -p /etc/etcd/ssl"; for FILE in etcd-ca-key.pem  etcd-ca.pem  etcd-key.pem  etcd.pem; do scp /etc/etcd/ssl/${FILE} $NODE:/etc/etcd/ssl/${FILE}; done; done
```

## 4.2 生成k8s相关证书

特别说明除外，以下操作在所有master节点操作

### 4.2.1 所有k8s节点创建证书存放目录

```bash
mkdir -p /etc/kubernetes/pki
```

### 4.2.2 master01节点生成k8s证书

```bash
cfssl gencert -initca ca-csr.json | cfssljson -bare /etc/kubernetes/pki/ca

# 生成一个根证书 ，多写了一些IP作为预留IP，为将来添加node做准备
# 10.96.0.1是service网段的第一个地址，需要计算，192.168.80.66为高可用vip地址
# 若没有IPv6 可删除可保留 

cfssl gencert   \
-ca=/etc/kubernetes/pki/ca.pem   \
-ca-key=/etc/kubernetes/pki/ca-key.pem   \
-config=ca-config.json   \
-hostname=10.96.0.1,192.168.80.66,127.0.0.1,kubernetes,kubernetes.default,kubernetes.default.svc,kubernetes.default.svc.cluster,kubernetes.default.svc.cluster.local,192.168.80.61,192.168.80.62,192.168.80.63,192.168.80.64,192.168.80.65,192.168.80.66,192.168.80.67,192.168.80.68,192.168.80.69,192.168.80.70,10.0.0.40,10.0.0.41 \
-profile=kubernetes   apiserver-csr.json | cfssljson -bare /etc/kubernetes/pki/apiserver
```

### 4.2.3 生成apiserver聚合证书

```bash
cfssl gencert   -initca front-proxy-ca-csr.json | cfssljson -bare /etc/kubernetes/pki/front-proxy-ca 

# 有一个警告，可以忽略
cfssl gencert  \
-ca=/etc/kubernetes/pki/front-proxy-ca.pem   \
-ca-key=/etc/kubernetes/pki/front-proxy-ca-key.pem   \
-config=ca-config.json   \
-profile=kubernetes   front-proxy-client-csr.json | cfssljson -bare /etc/kubernetes/pki/front-proxy-client
```

### 4.2.4 生成controller-manage的证书

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
     --server=https://192.168.80.61:6443 \
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
     --server=https://192.168.80.61:6443 \
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
  --server=https://192.168.80.61:6443     \
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

### 4.2.5 创建kube-proxy证书

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
  --server=https://192.168.80.61:6443     \
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

### 4.2.6 创建ServiceAccount Key ——secret

```bash
openssl genrsa -out /etc/kubernetes/pki/sa.key 2048
openssl rsa -in /etc/kubernetes/pki/sa.key -pubout -out /etc/kubernetes/pki/sa.pub
```

### 4.2.7 查看证书

```bash
ls /etc/kubernetes/pki/
admin.csr          ca.csr                      front-proxy-ca.csr          kube-proxy.csr      scheduler-key.pem
admin-key.pem      ca-key.pem                  front-proxy-ca-key.pem      kube-proxy-key.pem  scheduler.pem
admin.pem          ca.pem                      front-proxy-ca.pem          kube-proxy.pem
apiserver.csr      controller-manager.csr      front-proxy-client.csr      sa.key
apiserver-key.pem  controller-manager-key.pem  front-proxy-client-key.pem  sa.pub
apiserver.pem      controller-manager.pem      front-proxy-client.pem      scheduler.csr

# 一共26个就对了
ls /etc/kubernetes/pki/ |wc -l
26
```

# 5. k8s系统组件配置

## 5.1 etcd配置

### 5.1.1 master01配置

- k8s-master01操作

```bash
# 如果要用IPv6那么把IPv4地址修改为IPv6即可
cat > /etc/etcd/etcd.config.yml << EOF
name: 'k8s-master01'
data-dir: /var/lib/etcd
wal-dir: /var/lib/etcd/wal
snapshot-count: 5000
heartbeat-interval: 100
election-timeout: 1000
quota-backend-bytes: 0
listen-peer-urls: 'https://192.168.80.61:2380'
listen-client-urls: 'https://192.168.80.61:2379,http://127.0.0.1:2379'
max-snapshots: 3
max-wals: 5
cors:
initial-advertise-peer-urls: 'https://192.168.80.61:2380'
advertise-client-urls: 'https://192.168.80.61:2379'
discovery:
discovery-fallback: 'proxy'
discovery-proxy:
discovery-srv:
initial-cluster: 'k8s-master01=https://192.168.80.61:2380,k8s-node01=https://192.168.80.64:2380,k8s-node02=https://192.168.80.65:2380'
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

### 5.1.2 node01配置

- k8s-node01操作

```bash
# 如果要用IPv6那么把IPv4地址修改为IPv6即可
cat > /etc/etcd/etcd.config.yml << EOF 
name: 'k8s-node01'
data-dir: /var/lib/etcd
wal-dir: /var/lib/etcd/wal
snapshot-count: 5000
heartbeat-interval: 100
election-timeout: 1000
quota-backend-bytes: 0
listen-peer-urls: 'https://192.168.80.64:2380'
listen-client-urls: 'https://192.168.80.64:2379,http://127.0.0.1:2379'
max-snapshots: 3
max-wals: 5
cors:
initial-advertise-peer-urls: 'https://192.168.80.64:2380'
advertise-client-urls: 'https://192.168.80.64:2379'
discovery:
discovery-fallback: 'proxy'
discovery-proxy:
discovery-srv:
initial-cluster: 'k8s-master01=https://192.168.80.61:2380,k8s-node01=https://192.168.80.64:2380,k8s-node02=https://192.168.80.65:2380'
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

### 5.1.3 node02配置

- k8s-node02操作

```bash
# 如果要用IPv6那么把IPv4地址修改为IPv6即可
cat > /etc/etcd/etcd.config.yml << EOF
name: 'k8s-node02'
data-dir: /var/lib/etcd
wal-dir: /var/lib/etcd/wal
snapshot-count: 5000
heartbeat-interval: 100
election-timeout: 1000
quota-backend-bytes: 0
listen-peer-urls: 'https://192.168.80.65:2380'
listen-client-urls: 'https://192.168.80.65:2379,http://127.0.0.1:2379'
max-snapshots: 3
max-wals: 5
cors:
initial-advertise-peer-urls: 'https://192.168.80.65:2380'
advertise-client-urls: 'https://192.168.80.65:2379'
discovery:
discovery-fallback: 'proxy'
discovery-proxy:
discovery-srv:
initial-cluster: 'k8s-master01=https://192.168.80.61:2380,k8s-node01=https://192.168.80.64:2380,k8s-node02=https://192.168.80.65:2380'
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

## 5.2 创建service

- k8s-master01操作

### 5.2.1 创建etcd.service并启动

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

### 5.2.2 其他etcd节点复制二进制文件

```bash
ETCD="k8s-node01 k8s-node02"
for NODE in $ETCD;
do
    scp /usr/local/bin/etcd root@$NODE:/usr/local/bin/etcd
done
```

### 5.2.3 创建etcd证书目录

```bash
mkdir -p /etc/kubernetes/pki/etcd
ln -s /etc/etcd/ssl/* /etc/kubernetes/pki/etcd/
systemctl daemon-reload
systemctl enable --now etcd
systemctl restart etcd3.service
```

### 5.2.4 查看etcd状态

```bash
# 如果要用IPv6那么把IPv4地址修改为IPv6即可
export ETCDCTL_API=3
etcdctl --endpoints="192.168.80.61:2379,192.168.80.64:2379,192.168.80.65:2379" --cacert=/etc/kubernetes/pki/etcd/etcd-ca.pem --cert=/etc/kubernetes/pki/etcd/etcd.pem --key=/etc/kubernetes/pki/etcd/etcd-key.pem  endpoint status --write-out=table
+--------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|      ENDPOINT      |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+--------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| 192.168.80.61:2379 | 602a829d2119d074 |   3.5.4 |   20 kB |     false |      false |         3 |         12 |                 12 |        |
| 192.168.80.64:2379 | 1de61889ab854b65 |   3.5.4 |   20 kB |      true |      false |         3 |         12 |                 12 |        |
| 192.168.80.65:2379 | 3a43ac97176c28c6 |   3.5.4 |   20 kB |     false |      false |         3 |         12 |                 12 |        |
+--------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
```

# 6. k8s组件配置

- 所有k8s节点创建以下目录

```bash
mkdir -p /etc/kubernetes/manifests/ /etc/systemd/system/kubelet.service.d /var/lib/kubelet /var/log/kubernetes
```

## 6.1 创建apiserver（所有master节点）

### 6.1.1 master01节点配置

```bash
cat > /usr/lib/systemd/system/kube-apiserver.service << EOF

[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
ExecStart=/usr/local/bin/kube-apiserver \\
      --v=2  \\
      --logtostderr=true  \\
      --allow-privileged=true  \\
      --bind-address=0.0.0.0  \\
      --secure-port=6443  \\
      --advertise-address=192.168.80.61 \\
      --service-cluster-ip-range=10.96.0.0/12  \\
      --service-node-port-range=30000-32767  \\
      --etcd-servers=https://192.168.80.61:2379,https://192.168.80.64:2379,https://192.168.80.65:2379 \\
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
      --enable-aggregator-routing=true
      # --feature-gates=IPv6DualStack=true
      # --token-auth-file=/etc/kubernetes/token.csv

Restart=on-failure
RestartSec=10s
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target

EOF
```

### 6.1.2 启动apiserver（所有master节点）

```bash
systemctl daemon-reload && systemctl enable --now kube-apiserver

# 注意查看状态是否启动正常
# systemctl status kube-apiserver
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
      # --feature-gates=IPv6DualStack=true

Restart=always
RestartSec=10s

[Install]
WantedBy=multi-user.target

EOF
```

### 6.2.1启动kube-controller-manager，并查看状态

```bash
systemctl daemon-reload
systemctl enable --now kube-controller-manager
# systemctl  status kube-controller-manager
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
      --logtostderr=true \\
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
# systemctl status kube-scheduler
```

# 7.TLS Bootstrapping配置

## 7.1在master01上配置

```bash
cd bootstrap

kubectl config set-cluster kubernetes \
--certificate-authority=/etc/kubernetes/pki/ca.pem  \
--embed-certs=true     --server=https://192.168.80.61:6443  \
--kubeconfig=/etc/kubernetes/bootstrap-kubelet.kubeconfig

kubectl config set-credentials tls-bootstrap-token-user     \
--token=c8ad9c.2e4d610cf3e7426e \
--kubeconfig=/etc/kubernetes/bootstrap-kubelet.kubeconfig

kubectl config set-context tls-bootstrap-token-user@kubernetes  \
--cluster=kubernetes \
--user=tls-bootstrap-token-user  \
--kubeconfig=/etc/kubernetes/bootstrap-kubelet.kubeconfig

kubectl config use-context tls-bootstrap-token-user@kubernetes \
--kubeconfig=/etc/kubernetes/bootstrap-kubelet.kubeconfig

# token的位置在bootstrap.secret.yaml，如果修改的话到这个文件修改 /root/bootstrap/bootstrap.secret.yaml
mkdir -p /root/.kube ; cp /etc/kubernetes/admin.kubeconfig /root/.kube/config
```

## 7.2 查看集群状态，没问题的话继续后续操作

```bash
kubectl get cs
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS    MESSAGE                         ERROR
controller-manager   Healthy   ok
scheduler            Healthy   ok
etcd-1               Healthy   {"health":"true","reason":""}
etcd-2               Healthy   {"health":"true","reason":""}
etcd-0               Healthy   {"health":"true","reason":""}

kubectl create -f /root/bootstrap/bootstrap.secret.yaml
```

# 8.node节点配置

## 8.1 在master01上将证书复制到node节点

- **k8s-master01操作**

```bash
cd /etc/kubernetes/
 
for NODE in k8s-node01 k8s-node02; do ssh $NODE mkdir -p /etc/kubernetes/pki; for FILE in pki/ca.pem pki/ca-key.pem pki/front-proxy-ca.pem bootstrap-kubelet.kubeconfig kube-proxy.kubeconfig; do scp /etc/kubernetes/$FILE $NODE:/etc/kubernetes/${FILE}; done; done
```

## 8.2 kubelet配置

### 8.2.1 当使用Containerd作为Runtime （推荐）

- **k8s-master01、k8s-node01、k8s-node02操作**

```bash
mkdir -p /var/lib/kubelet /var/log/kubernetes /etc/systemd/system/kubelet.service.d /etc/kubernetes/manifests/

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
    --container-runtime-endpoint=unix:///run/containerd/containerd.sock  \\
    --node-labels=node.kubernetes.io/node=
    # --feature-gates=IPv6DualStack=true
    # --container-runtime=remote
    # --runtime-request-timeout=15m
    # --cgroup-driver=systemd

[Install]
WantedBy=multi-user.target
EOF
```

### 8.2.2 所有k8s节点创建kubelet的配置文件

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
```

### 8.2.4查看集群

```bash
kubectl  get node
NAME           STATUS   ROLES    AGE     VERSION
k8s-master01   Ready    <none>   6h17m   v1.25.0
k8s-master02   Ready    <none>   6h16m   v1.25.0
k8s-master03   Ready    <none>   6h16m   v1.25.0
k8s-node01     Ready    <none>   6h20m   v1.25.0
k8s-node02     Ready    <none>   6h18m   v1.25.0
```

## 8.3 kube-proxy配置

### 8.3.1将kubeconfig发送至其他节点

```bash
for NODE in k8s-node01 k8s-node02; do scp /etc/kubernetes/kube-proxy.kubeconfig $NODE:/etc/kubernetes/kube-proxy.kubeconfig;  done
```

### 8.3.2所有k8s节点添加kube-proxy的service文件

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

### 8.3.3所有k8s节点添加kube-proxy的配置

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

### 8.3.4启动kube-proxy

```bash
systemctl daemon-reload
systemctl restart kube-proxy
systemctl enable --now kube-proxy
# systemctl status kube-proxy.service
```

# 9.安装网络插件

**cilium 暂时不支持IPv4 IPv6双栈**

## 9.1安装Calico

https://projectcalico.docs.tigera.io/archive/v3.23/getting-started/kubernetes/

https://github.com/projectcalico/calico

```bash
mkdir -p $HOME/k8s-install/network && cd $HOME/k8s-install/network

# 1. 下载插件
wget -P $HOME/k8s-install/network/ https://docs.projectcalico.org/manifests/calico.yaml

# CIDR的值，与 kube-controller-manager中“--cluster-cidr=172.16.0.0/12” 一致
vi calico.yaml
4548             # The default IPv4 pool to create on startup if none exists. Pod IPs will be
4549             # chosen from this range. Changing this value after installation will have
4550             # no effect. This should fall within `--cluster-cidr`.
4551             - name: CALICO_IPV4POOL_CIDR
4552               value: "172.16.0.0/12"
   
# 2. 安装网络插件
kubectl apply -f calico.yaml

# 3. 查看情况
kubectl get pod -n kube-system -owide
NAME                                      READY   STATUS    RESTARTS   AGE   IP              NODE           NOMINATED NODE   READINESS GATES
calico-kube-controllers-f79f7749d-kkgmw   1/1     Running   0          39m   10.88.0.2       k8s-master01   <none>           <none>
calico-node-9tgg6                         1/1     Running   0          39m   192.168.80.61   k8s-master01   <none>           <none>
calico-node-ppgl6                         1/1     Running   0          39m   192.168.80.64   k8s-node01     <none>           <none>
calico-node-qb7ns                         1/1     Running   0          39m   192.168.80.65   k8s-node02     <none>           <none>

# 4. 查看集群
kubectl get node
NAME           STATUS   ROLES    AGE   VERSION
k8s-master01   Ready    <none>   59m   v1.25.0
k8s-node01     Ready    <none>   60m   v1.25.0
k8s-node02     Ready    <none>   60m   v1.25.0
```

## 9.2安装cilium

### 9.2.1 安装helm

https://mirrors.huaweicloud.com/helm/

https://helm.sh

```bash
# 1. 官网下载网络太慢
# [root@k8s-master01 ~]# curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
# [root@k8s-master01 ~]# chmod 700 get_helm.sh
# [root@k8s-master01 ~]# ./get_helm.sh

# 2. 华为云
wget https://mirrors.huaweicloud.com/helm/v3.9.4/helm-v3.9.4-linux-amd64.tar.gz
tar zxvf helm-v3.9.4-linux-amd64.tar.gz
cp linux-amd64/helm /usr/local/bin/
helm version
# version.BuildInfo{Version:"v3.9.4", GitCommit:"dbc6d8e20fe1d58d50e6ed30f09a04a77e4c68db", GitTreeState:"clean", GoVersion:"go1.17.13"}
```

### 9.2.2 安装cilium

```bash
[root@k8s-master01 ~]# helm repo add cilium https://helm.cilium.io
[root@k8s-master01 ~]# helm install cilium cilium/cilium    --namespace kube-system    --set hubble.relay.enabled=true     --set hubble.ui.enabled=true    --set prometheus.enabled=true    --set operator.prometheus.enabled=true    --set hubble.enabled=true    --set hubble.metrics.enabled="{dns,drop,tcp,flow,port-distribution,icmp,http}"

NAME: cilium
LAST DEPLOYED: Sun Sep 11 00:04:30 2022
NAMESPACE: kube-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
You have successfully installed Cilium with Hubble.

Your release version is 1.12.1.

For any further help, visit https://docs.cilium.io/en/v1.12/gettinghelp
[root@k8s-master01 ~]#
```

### 9.2.3 查看

```bash
[root@k8s-master01 ~]# kubectl  get pod -A | grep cil
kube-system   cilium-gmr6c                       1/1     Running       0             5m3s
kube-system   cilium-kzgdj                       1/1     Running       0             5m3s
kube-system   cilium-operator-69b677f97c-6pw4k   1/1     Running       0             5m3s
kube-system   cilium-operator-69b677f97c-xzzdk   1/1     Running       0             5m3s
kube-system   cilium-q2rnr                       1/1     Running       0             5m3s
kube-system   cilium-smx5v                       1/1     Running       0             5m3s
kube-system   cilium-tdjq4                       1/1     Running       0             5m3s
[root@k8s-master01 ~]#
```

### 9.2.4 下载专属监控面板

```bash
[root@k8s-master01 yaml]# wget https://raw.githubusercontent.com/cilium/cilium/1.12.1/examples/kubernetes/addons/prometheus/monitoring-example.yaml
[root@k8s-master01 yaml]#
[root@k8s-master01 yaml]# kubectl  apply -f monitoring-example.yaml
namespace/cilium-monitoring created
serviceaccount/prometheus-k8s created
configmap/grafana-config created
configmap/grafana-cilium-dashboard created
configmap/grafana-cilium-operator-dashboard created
configmap/grafana-hubble-dashboard created
configmap/prometheus created
clusterrole.rbac.authorization.k8s.io/prometheus created
clusterrolebinding.rbac.authorization.k8s.io/prometheus created
service/grafana created
service/prometheus created
deployment.apps/grafana created
deployment.apps/prometheus created
[root@k8s-master01 yaml]#
```

### 9.2.5 下载部署测试用例

```bash
[root@k8s-master01 yaml]# wget https://raw.githubusercontent.com/cilium/cilium/master/examples/kubernetes/connectivity-check/connectivity-check.yaml

[root@k8s-master01 yaml]# sed -i "s#google.com#oiox.cn#g" connectivity-check.yaml

[root@k8s-master01 yaml]# kubectl  apply -f connectivity-check.yaml
deployment.apps/echo-a created
deployment.apps/echo-b created
deployment.apps/echo-b-host created
deployment.apps/pod-to-a created
deployment.apps/pod-to-external-1111 created
deployment.apps/pod-to-a-denied-cnp created
deployment.apps/pod-to-a-allowed-cnp created
deployment.apps/pod-to-external-fqdn-allow-google-cnp created
deployment.apps/pod-to-b-multi-node-clusterip created
deployment.apps/pod-to-b-multi-node-headless created
deployment.apps/host-to-b-multi-node-clusterip created
deployment.apps/host-to-b-multi-node-headless created
deployment.apps/pod-to-b-multi-node-nodeport created
deployment.apps/pod-to-b-intra-node-nodeport created
service/echo-a created
service/echo-b created
service/echo-b-headless created
service/echo-b-host-headless created
ciliumnetworkpolicy.cilium.io/pod-to-a-denied-cnp created
ciliumnetworkpolicy.cilium.io/pod-to-a-allowed-cnp created
ciliumnetworkpolicy.cilium.io/pod-to-external-fqdn-allow-google-cnp created
[root@k8s-master01 yaml]#
```

### 9.2.6 查看pod

注：**注意机器内存空间需要给大些。**

```bash
[root@k8s-master01 yaml]# kubectl  get pod -A
NAMESPACE           NAME                                                     READY   STATUS    RESTARTS      AGE
cilium-monitoring   grafana-59957b9549-6zzqh                                 1/1     Running   0             10m
cilium-monitoring   prometheus-7c8c9684bb-4v9cl                              1/1     Running   0             10m
default             chenby-75b5d7fbfb-7zjsr                                  1/1     Running   0             27h
default             chenby-75b5d7fbfb-hbvr8                                  1/1     Running   0             27h
default             chenby-75b5d7fbfb-ppbzg                                  1/1     Running   0             27h
default             echo-a-6799dff547-pnx6w                                  1/1     Running   0             10m
default             echo-b-fc47b659c-4bdg9                                   1/1     Running   0             10m
default             echo-b-host-67fcfd59b7-28r9s                             1/1     Running   0             10m
default             host-to-b-multi-node-clusterip-69c57975d6-z4j2z          1/1     Running   0             10m
default             host-to-b-multi-node-headless-865899f7bb-frrmc           1/1     Running   0             10m
default             pod-to-a-allowed-cnp-5f9d7d4b9d-hcd8x                    1/1     Running   0             10m
default             pod-to-a-denied-cnp-65cc5ff97b-2rzb8                     1/1     Running   0             10m
default             pod-to-a-dfc64f564-p7xcn                                 1/1     Running   0             10m
default             pod-to-b-intra-node-nodeport-677868746b-trk2l            1/1     Running   0             10m
default             pod-to-b-multi-node-clusterip-76bbbc677b-knfq2           1/1     Running   0             10m
default             pod-to-b-multi-node-headless-698c6579fd-mmvd7            1/1     Running   0             10m
default             pod-to-b-multi-node-nodeport-5dc4b8cfd6-8dxmz            1/1     Running   0             10m
default             pod-to-external-1111-8459965778-pjt9b                    1/1     Running   0             10m
default             pod-to-external-fqdn-allow-google-cnp-64df9fb89b-l9l4q   1/1     Running   0             10m
kube-system         cilium-7rfj6                                             1/1     Running   0             56s
kube-system         cilium-d4cch                                             1/1     Running   0             56s
kube-system         cilium-h5x8r                                             1/1     Running   0             56s
kube-system         cilium-operator-5dbddb6dbf-flpl5                         1/1     Running   0             56s
kube-system         cilium-operator-5dbddb6dbf-gcznc                         1/1     Running   0             56s
kube-system         cilium-t2xlz                                             1/1     Running   0             56s
kube-system         cilium-z65z7                                             1/1     Running   0             56s
kube-system         coredns-665475b9f8-jkqn8                                 1/1     Running   1 (36h ago)   36h
kube-system         hubble-relay-59d8575-9pl9z                               1/1     Running   0             56s
kube-system         hubble-ui-64d4995d57-nsv9j                               2/2     Running   0             56s
kube-system         metrics-server-776f58c94b-c6zgs                          1/1     Running   1 (36h ago)   37h
[root@k8s-master01 yaml]#
```

###  9.2.7 修改为NodePort

```bash
[root@k8s-master01 yaml]# kubectl  edit svc  -n kube-system hubble-ui
service/hubble-ui edited
[root@k8s-master01 yaml]#
[root@k8s-master01 yaml]# kubectl  edit svc  -n cilium-monitoring grafana
service/grafana edited
[root@k8s-master01 yaml]#
[root@k8s-master01 yaml]# kubectl  edit svc  -n cilium-monitoring prometheus
service/prometheus edited
[root@k8s-master01 yaml]#

type: NodePort
```

### 9.2.8 查看端口

```bash
[root@k8s-master01 yaml]# kubectl get svc -A | grep monit
cilium-monitoring   grafana                NodePort    10.100.250.17    <none>        3000:30707/TCP           15m
cilium-monitoring   prometheus             NodePort    10.100.131.243   <none>        9090:31155/TCP           15m
[root@k8s-master01 yaml]#
[root@k8s-master01 yaml]# kubectl get svc -A | grep hubble
kube-system         hubble-metrics         ClusterIP   None             <none>        9965/TCP                 5m12s
kube-system         hubble-peer            ClusterIP   10.100.150.29    <none>        443/TCP                  5m12s
kube-system         hubble-relay           ClusterIP   10.109.251.34    <none>        80/TCP                   5m12s
kube-system         hubble-ui              NodePort    10.102.253.59    <none>        80:31219/TCP             5m12s
[root@k8s-master01 yaml]#
```

### 9.2.9 访问

```bash
http://192.168.80.61:30707
http://192.168.80.61:31155
http://192.168.80.61:31219
```

# 10.安装CoreDNS

https://github.com/coredns/deployment/tree/master/kubernetes

## 10.1以下步骤只在master01操作

### 10.1.1修改文件

```bash
# 下载文件
# wget https://storage.googleapis.com/kubernetes-the-hard-way/coredns.yaml
# 修改文件内容cluster ip
clusterIP: 10.96.0.10
# 启动文件
kubectl apply -f /root/coredns/coredns.yaml

# 查询状态
kubectl get pods -n kube-system | grep coredns
coredns-78cdc77856-9855b                  1/1     Running   0          21m
```

DNS问题排查：

```bash
# dns service
kubectl get svc -n kube-system
NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   13m

# endpoints 是否正常
kubectl get endpoints kube-dns -n kube-system
NAME       ENDPOINTS                                                       AGE
kube-dns   172.17.125.2:53,172.25.244.196:53,172.17.125.2:53 + 3 more...   10m

# coredns 增加解析日志
CoreDNS 配置参数说明：
errors: 输出错误信息到控制台。
health：CoreDNS 进行监控检测，检测地址为 http://localhost:8080/health 如果状态为不健康则让 Pod 进行重启。
ready: 全部插件已经加载完成时，将通过 endpoints 在 8081 端口返回 HTTP 状态 200。
kubernetes：CoreDNS 将根据 Kubernetes 服务和 pod 的 IP 回复 DNS 查询。
prometheus：是否开启 CoreDNS Metrics 信息接口，如果配置则开启，接口地址为 http://localhost:9153/metrics
forward：任何不在Kubernetes 集群内的域名查询将被转发到预定义的解析器 (/etc/resolv.conf)。
cache：启用缓存，30 秒 TTL。
loop：检测简单的转发循环，如果找到循环则停止 CoreDNS 进程。
reload：监听 CoreDNS 配置，如果配置发生变化则重新加载配置。
loadbalance：DNS 负载均衡器，默认 round_robin。

# 编辑 coredns 配置
kubectl edit configmap coredns -n kube-system
apiVersion: v1
data:
  Corefile: |
    .:53 {
        log     # new add
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
kind: ConfigMap
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","data":{"Corefile":".:53 {\n    errors\n    health {\n      lameduck 5s\n    }\n    ready\n    kubernetes cluster.local in-addr.arpa ip6.arpa {\n      fallthrough in-addr.arpa ip6.arpa\n    }\n    prometheus :9153\n    forward . /etc/resolv.conf {\n      max_concurrent 1000\n    }\n    cache 30\n    loop\n    reload\n    loadbalance\n}\n"},"kind":"ConfigMap","metadata":{"annotations":{},"name":"coredns","namespace":"kube-system"}}
  creationTimestamp: "2021-05-13T11:57:45Z"
  name: coredns
  namespace: kube-system
  resourceVersion: "38460"
  selfLink: /api/v1/namespaces/kube-system/configmaps/coredns
  uid: c62a856d-1fc3-4fe9-b5f1-3ca0dbeb39c1
```

# 11.安装Metrics Server

## 11.1以下步骤只在master01操作

### 11.1.1安装Metrics-server

在新版的Kubernetes中系统资源的采集均使用Metrics-server，可以通过Metrics采集节点和Pod的内存、磁盘、CPU和网络的使用率

```bash
# 安装metrics server
kubectl apply -f /root/metrics-server/metrics-server/metrics-server.yaml

# 查看资源使用
NAME           CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
k8s-master01   726m         36%    1186Mi          64%
k8s-node01     229m         11%    973Mi           52%
k8s-node02     215m         10%    973Mi           52%
```

# 12.集群验证

## 12.1部署pod资源

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

## 12.2用pod解析默认命名空间中的kubernetes

```bash
kubectl get svc
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   140m

kubectl exec  busybox -n default -- nslookup kubernetes
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      kubernetes
Address 1: 10.96.0.1 kubernetes.default.svc.cluster.local
```

## 12.3测试跨命名空间是否可以解析

```bash
kubectl exec  busybox -n default -- nslookup kube-dns.kube-system
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      kube-dns.kube-system
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local
```

## 12.4每个节点都必须要能访问Kubernetes的kubernetes svc 443和kube-dns的service 53

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

## 12.5Pod和Pod之前要能通

```bash
NAME      READY   STATUS    RESTARTS   AGE   IP             NODE         NOMINATED NODE   READINESS GATES
busybox   1/1     Running   0          27m   172.17.125.2   k8s-node01   <none>           <none>

kubectl get po -n kube-system -owide
NAME                                      READY   STATUS    RESTARTS   AGE   IP              NODE           NOMINATED NODE   READINESS GATES
calico-kube-controllers-f79f7749d-kkgmw   1/1     Running   0          50m   10.88.0.2       k8s-master01   <none>           <none>
calico-node-9tgg6                         1/1     Running   0          50m   192.168.80.61   k8s-master01   <none>           <none>
calico-node-ppgl6                         1/1     Running   0          50m   192.168.80.64   k8s-node01     <none>           <none>
calico-node-qb7ns                         1/1     Running   0          50m   192.168.80.65   k8s-node02     <none>           <none>
coredns-78cdc77856-9855b                  1/1     Running   0          30m   172.27.14.193   k8s-node02     <none>           <none>
metrics-server-6bbcb9f574-ncj9r           1/1     Running   0          29m   172.17.125.1    k8s-node01     <none>           <none>

# 进入busybox ping其他节点上的pod
kubectl exec -ti busybox -- sh
/ # ping 192.168.80.64
PING 192.168.80.64 (192.168.80.64): 56 data bytes
64 bytes from 192.168.80.64: seq=0 ttl=64 time=0.443 ms
64 bytes from 192.168.80.64: seq=1 ttl=64 time=0.126 ms
64 bytes from 192.168.80.64: seq=2 ttl=64 time=0.170 ms
```

## 12.6创建三个副本，可以看到3个副本分布在不同的节点上

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

kubectl  get pod
NAME                                READY   STATUS    RESTARTS   AGE
busybox                             1/1     Running   0          29m
nginx-deployment-6ff84ff895-4ktgm   1/1     Running   0          27s
nginx-deployment-6ff84ff895-g6dms   1/1     Running   0          27s
nginx-deployment-6ff84ff895-v4v8p   1/1     Running   0          27s
```

# 13. master集群高可用

## 13.1系统初始化

- **k8s-master02、k8s-master03操作**

 

```bash
# 1. 修改主机名
hostnamectl set-hostname k8s-master02
hostnamectl set-hostname k8s-master03

# 2. 主机名解析
cat >> /etc/hosts <<EOF
192.168.80.61  k8s-master01
192.168.80.62  k8s-master02
192.168.80.63  k8s-master03
192.168.80.64  k8s-node01
192.168.80.65  k8s-node02
EOF

# 3. 禁用 swap
sed -ri 's/.*swap.*/#&/' /etc/fstab
swapoff -a && sysctl -w vm.swappiness=0

# 4. 将桥接的IPv4流量传递到iptables的链 
cat > /etc/sysctl.d/k8s.conf << EOF 
net.bridge.bridge-nf-call-ip6tables = 1 
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system

# 5. 域名解析
echo "nameserver 8.8.8.8" >> /etc/resolv.conf

# 6.进行实际同步（客户端）
apt install chrony -y
cat > /etc/chrony.conf << EOF 
pool 192.168.80.61 iburst
driftfile /var/lib/chrony/drift
makestep 1.0 3
rtcsync
keyfile /etc/chrony.keys
leapsectz right/UTC
logdir /var/log/chrony
EOF

ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
hwclock --systohc
date -R
timedatectl set-local-rtc yes
timedatectl

systemctl restart chrony.service

#使用客户端进行验证
chronyc sources -v

# 7. 日志目录
mkdir -p /var/log/kubernetes

# 8.配置ulimit
ulimit -SHn 65535
cat >> /etc/security/limits.conf <<EOF
* soft nofile 655360
* hard nofile 131072
* soft nproc 655350
* hard nproc 655350
* seft memlock unlimited
* hard memlock unlimitedd
EOF

# 9.ipvsadm安装（lb除外）
apt-get install ipvsadm
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

lsmod | grep -e ip_vs -e nf_conntrack
ip_vs_sh               16384  0
ip_vs_wrr              16384  0
ip_vs_rr               16384  0
ip_vs                 155648  6 ip_vs_rr,ip_vs_sh,ip_vs_wrr
nf_conntrack          139264  1 ip_vs
nf_defrag_ipv6         24576  2 nf_conntrack,ip_vs
nf_defrag_ipv4         16384  1 nf_conntrack
libcrc32c              16384  2 nf_conntrack,ip_vs

# 10.修改内核参数 (lb除外)
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
net.ipv6.conf.all.forwarding = 1
EOF

sysctl --system

# 11.命令补全
apt bash-completion -y
source /usr/share/bash-completion/bash_completion
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc
```

## 13.2 安装Containerd作为Runtime

### 13.2.1 master01操作

```bash
export master_other="k8s-master02 k8s-master03"
for HOST in $master_other;
do
    sshpass -e ssh-copy-id -o StrictHostKeyChecking=no $HOST
    ssh root@$HOST "mkdir -p /usr/local/bin/ && mkdir -p $HOME/k8s-install"
    scp /opt/cni/bin $HOST:/opt/cni/bin
    scp /usr/bin/crictl $HOST:/usr/bin/
    # kubernetes-server-linux-amd64.tar.gz
    scp /usr/local/bin/kube{let,ctl,-apiserver,-controller-manager,-scheduler,-proxy}  $HOST:/usr/local/bin/
    scp $HOME/k8s-install/cri-containerd-cni-1.6.8-linux-amd64.tar.gz $HOST:$HOME/k8s-install/
    scp /usr/local/bin/kube{let,-proxy} $HOST:/usr/local/bin/
done

# etcd密钥
for NODE in $master_other; do ssh $NODE "mkdir -p /etc/etcd/ssl"; for FILE in etcd-ca-key.pem  etcd-ca.pem  etcd-key.pem  etcd.pem; do scp /etc/etcd/ssl/${FILE} $NODE:/etc/etcd/ssl/${FILE}; done; done

# 其他master节点
mkdir  /etc/kubernetes/pki/ -p

for NODE in $master_other; do  for FILE in $(ls /etc/kubernetes/pki | grep -v etcd); do  scp /etc/kubernetes/pki/${FILE} $NODE:/etc/kubernetes/pki/${FILE}; done;  for FILE in admin.kubeconfig controller-manager.kubeconfig scheduler.kubeconfig; do  scp /etc/kubernetes/${FILE} $NODE:/etc/kubernetes/${FILE}; done; done
```

###  13.2.2 创建配置文件

```bash
#创建cni插件所需目录
mkdir -p /etc/cni/net.d

#解压
tar -xzf $HOME/k8s-install/cri-containerd-cni-1.6.8-linux-amd64.tar.gz -C /

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

###  13.2.3 配置Containerd所需的模块

```bash
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
```

### 13.2.4 加载模块

```bash
systemctl restart systemd-modules-load.service
```

### 13.2.5 配置Containerd所需的内核

```bash
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

# 加载内核
sysctl --system
```

### 13.2.6 创建Containerd的配置文件

```bash
# 创建默认配置文件
mkdir -p /etc/containerd
containerd config default | tee /etc/containerd/config.toml

# 修改Containerd的配置文件
sed -i "s#SystemdCgroup\ \=\ false#SystemdCgroup\ \=\ true#g" /etc/containerd/config.toml
cat /etc/containerd/config.toml | grep SystemdCgroup

sed -i "s#k8s.gcr.io#registry.cn-hangzhou.aliyuncs.com/chenby#g" /etc/containerd/config.toml
cat /etc/containerd/config.toml | grep sandbox_image

sed -i "s#k8s.gcr.io#registry.cn-hangzhou.aliyuncs.com/chenby#g" /etc/containerd/config.toml

sed -i "s#config_path\ \=\ \"\"#config_path\ \=\ \"/etc/containerd/certs.d\"#g" /etc/containerd/config.toml
 
mkdir /etc/containerd/certs.d/docker.io -pv

cat > /etc/containerd/certs.d/docker.io/hosts.toml << EOF
server = "https://docker.io"
[host."https://hub-mirror.c.163.com"]
  capabilities = ["pull", "resolve"]
EOF
```

### 13.2.7 启动并设置为开机启动

```bash
systemctl daemon-reload
systemctl enable --now containerd
systemctl restart containerd
```

### 13.2.8 配置crictl客户端连接的运行时位置

```bash
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

## 13.3 组件配置

### 13.3.1 k8s-master02创建apiserver

```bash
cat > /usr/lib/systemd/system/kube-apiserver.service << EOF

[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
ExecStart=/usr/local/bin/kube-apiserver \\
      --v=2  \\
      --logtostderr=true  \\
      --allow-privileged=true  \\
      --bind-address=0.0.0.0  \\
      --secure-port=6443  \\
      --advertise-address=192.168.80.62 \\
      --service-cluster-ip-range=10.96.0.0/12  \\
      --service-node-port-range=30000-32767  \\
      --etcd-servers=https://192.168.80.61:2379,https://192.168.80.64:2379,https://192.168.80.65:2379 \\
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
      --enable-aggregator-routing=true
      # --feature-gates=IPv6DualStack=true
      # --token-auth-file=/etc/kubernetes/token.csv

Restart=on-failure
RestartSec=10s
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target

EOF
```

### 13.3.2 k8s-master03创建apiserver

```bash
cat > /usr/lib/systemd/system/kube-apiserver.service << EOF

[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
ExecStart=/usr/local/bin/kube-apiserver \\
      --v=2  \\
      --logtostderr=true  \\
      --allow-privileged=true  \\
      --bind-address=0.0.0.0  \\
      --secure-port=6443  \\
      --advertise-address=192.168.80.63 \\
      --service-cluster-ip-range=10.96.0.0/12  \\
      --service-node-port-range=30000-32767  \\
      --etcd-servers=https://192.168.80.61:2379,https://192.168.80.64:2379,https://192.168.80.65:2379 \\
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
      --enable-aggregator-routing=true
      # --feature-gates=IPv6DualStack=true
      # --token-auth-file=/etc/kubernetes/token.csv

Restart=on-failure
RestartSec=10s
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target

EOF
```

### 13.3.3 启动apiserver

```bash
systemctl daemon-reload && systemctl enable --now kube-apiserver

# 注意查看状态是否启动正常
# systemctl status kube-apiserver
```

### 13.3.4 配置kube-controller-manager service

- k8s-master02、k8s-master03操作

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
      # --feature-gates=IPv6DualStack=true

Restart=always
RestartSec=10s

[Install]
WantedBy=multi-user.target

EOF

systemctl daemon-reload
systemctl enable --now kube-controller-manager
# systemctl  status kube-controller-manager
```

### 13.3.5 配置kube-scheduler service

- k8s-master02、k8s-master03操作

```bash
cat > /usr/lib/systemd/system/kube-scheduler.service << EOF

[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
ExecStart=/usr/local/bin/kube-scheduler \\
      --v=2 \\
      --logtostderr=true \\
      --bind-address=127.0.0.1 \\
      --leader-elect=true \\
      --kubeconfig=/etc/kubernetes/scheduler.kubeconfig

Restart=always
RestartSec=10s

[Install]
WantedBy=multi-user.target

EOF
```

### 13.3.6 启动并查看服务状态

```bash
systemctl daemon-reload
systemctl enable --now kube-scheduler
# systemctl status kube-scheduler
```

复制证书：

```bash
cd /etc/kubernetes/
 
for NODE in k8s-master02 k8s-master03; do ssh $NODE mkdir -p /etc/kubernetes/pki; for FILE in pki/ca.pem pki/ca-key.pem pki/front-proxy-ca.pem bootstrap-kubelet.kubeconfig kube-proxy.kubeconfig; do scp /etc/kubernetes/$FILE $NODE:/etc/kubernetes/${FILE}; done; done
```

### 13.3.7 kubelet配置，Containerd作为Runtime

```bash
mkdir -p /var/lib/kubelet /var/log/kubernetes /etc/systemd/system/kubelet.service.d /etc/kubernetes/manifests/

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
    --container-runtime-endpoint=unix:///run/containerd/containerd.sock  \\
    --node-labels=node.kubernetes.io/node=
    # --feature-gates=IPv6DualStack=true
    # --container-runtime=remote
    # --runtime-request-timeout=15m
    # --cgroup-driver=systemd

[Install]
WantedBy=multi-user.target
EOF
```

### 13.3.8 所有k8s节点创建kubelet的配置文件

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

### 13.3.9 启动kubelet

```bash
systemctl daemon-reload
systemctl restart kubelet
systemctl enable --now kubelet
# systemctl status kubelet.service
```

### 13.3.10 查看集群

```bash
kubectl get node
NAME           STATUS   ROLES    AGE    VERSION
k8s-master01   Ready    <none>   21h    v1.25.0
k8s-master02   Ready    <none>   113s   v1.25.0
k8s-master03   Ready    <none>   38s    v1.25.0
k8s-node01     Ready    <none>   21h    v1.25.0
k8s-node02     Ready    <none>   21h    v1.25.0
```

### 13.3.11 kube-proxy配置

```bash
for NODE in k8s-master02 k8s-master03; do scp /etc/kubernetes/kube-proxy.kubeconfig $NODE:/etc/kubernetes/kube-proxy.kubeconfig; done

# 添加kube-proxy的service文件
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

# 添加kube-proxy的配置
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

# 启动kube-proxy
systemctl daemon-reload
systemctl restart kube-proxy
systemctl enable --now kube-proxy
# systemctl status kube-proxy.service
```

# 14. 高可用配置

主要规划：master01、master02、master03 安装 keepalived 服务，虚拟vip：192.168.80.66 端口为8443，此前公钥中已经加入了192.168.80.66地址。haproxy 是 k8s-master01和 k8s-master02 安装。

## 14.1 安装keepalived和haproxy服务

```bash
apt install keepalived haproxy -y
```

## 14.2 修改haproxy配置文件（两台配置文件一样）

- k8s-master01、k8s-master02操作

```bash
# cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.bak

cat >/etc/haproxy/haproxy.cfg<<"EOF"
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
 server  k8s-master01  192.168.80.61:6443 check
 server  k8s-master02  192.168.80.62:6443 check
 server  k8s-master03  192.168.80.63:6443 check
EOF
```

## 14.3 Master01配置keepalived master节点

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
    mcast_src_ip 192.168.80.61
    virtual_router_id 51
    priority 100
    nopreempt
    advert_int 2
    authentication {
        auth_type PASS
        auth_pass K8SHA_KA_AUTH
    }
    virtual_ipaddress {
        192.168.80.66
    }
    track_script {
      chk_apiserver 
} }

EOF
```

## 14.4 Master02配置keepalived backup节点

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
    mcast_src_ip 192.168.80.62
    virtual_router_id 51
    priority 80
    nopreempt
    advert_int 2
    authentication {
        auth_type PASS
        auth_pass K8SHA_KA_AUTH
    }
    virtual_ipaddress {
        192.168.80.66
    }
    track_script {
      chk_apiserver 
} }

EOF
```

## 14.5 Master03配置keepalived backup节点

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
    mcast_src_ip 192.168.80.63
    virtual_router_id 51
    priority 50
    nopreempt
    advert_int 2
    authentication {
        auth_type PASS
        auth_pass K8SHA_KA_AUTH
    }
    virtual_ipaddress {
        192.168.80.66
    }
    track_script {
      chk_apiserver 
} }

EOF
```

## 14.6 健康检查脚本配置（两台lb主机）

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

## 14.7 启动服务

```bash
systemctl daemon-reload
systemctl enable --now haproxy
systemctl enable --now keepalived
systemctl start haproxy
systemctl start keepalived

# systemctl status haproxy
# systemctl status keepalived
```

## 14.8 测试高可用

```bash
# 能ping同
ping 192.168.80.66
netstat -lntup | grep 8443

# 能telnet访问
telnet 192.168.80.66 8443

# 关闭主节点，看vip是否漂移到备节点
```

## 14.9 替换所有节点配置文件

```bash
# 批量替换
sed -i 's#192.168.80.61:6443#192.168.80.66:8443#' /etc/kubernetes/*
sed -i 's#192.168.80.61:6443#192.168.80.66:8443#' /etc/kubernetes/pki/*
grep -nr  '192.168.80.' /etc/kubernetes/*

systemctl restart kubelet kube-proxy
# systemctl status kubelet kube-proxy

kubectl get node
```

# 15. 打标和污点

```bash
# 标签
kubectl label node k8s-master01 k8s-master02 k8s-master03 node-role.kubernetes.io/master=
kubectl label node k8s-node01 k8s-node02 node-role.kubernetes.io/node=

# 污点
kubectl taint nodes k8s-master01 k8s-master02 k8s-master03 node-role.kubernetes.io/master=:NoSchedule
```

# 16. 删除节点

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