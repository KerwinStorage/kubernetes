# 1.资源规划

官方：https://ceph.com/en

官方文档：https://docs.ceph.com/en/latest/start/intro

ceph 是一种开源的分布式的存储系统 包含以下几种存储类型： 块存储（rbd），对象存储(RADOS Fateway)，文件系统（cephfs）

| 主机                                | 磁盘                           | ceph版本 | ceph-deploy版本 | 系统         |
| ----------------------------------- | ------------------------------ | -------- | --------------- | ------------ |
| 192.168.80.55（ceph-master1-admin） | 系统盘：200GB、数据裸盘：100GB | 17.2.5   | 2.1.0           | Ubuntu 22.04 |
| 192.168.80.66（ceph-node1-monitor） | 系统盘：200GB、数据裸盘：100GB | 17.2.5   | 2.1.0           | Ubuntu 22.04 |
| 192.168.80.77（ceph-node2-osd）     | 系统盘：200GB、数据裸盘：100GB | 17.2.5   | 2.1.0           | Ubuntu 22.04 |

这里已经加入新的数据盘：`/dev/sdb`，记得`reboot`重启服务器。

| k8s节点       | role         | k8s版本 | 系统         |
| ------------- | ------------ | ------- | ------------ |
| 192.168.80.45 | k8s-master01 | v1.26.5 | Ubuntu 22.04 |
| 192.168.80.48 | k8s-node01   | v1.26.5 | Ubuntu 22.04 |
| 192.168.80.49 | k8s-node02   | v1.26.5 | Ubuntu 22.04 |

# 2.在Ubuntu上部署Ceph

## 2.1.初始化

- 所有主机操作

### 2.1.1 host文件加入ip地址

```shell
echo -e "192.168.80.55  master1-admin\n192.168.80.66  node1-monitor\n192.168.80.77  node2-osd" >> /etc/hosts
```

### 2.1.2 免密操作

```shell
apt install -y sshpass
ssh-keygen -f /root/.ssh/id_rsa -P ''
export IP="master1-admin node1-monitor node2-osd"
export SSHPASS=1qazZSE$
for HOST in $IP;do
     sshpass -e ssh-copy-id -o StrictHostKeyChecking=no $HOST
done
```

### 2.1.3 时间同步

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

### 2.1.4 安装基础软件

```shell
apt install net-tools wget vim bash-completion git lrzsz unzip zip -y
```

### 2.1.5 修改主机名

```sh
hostnamectl set-hostname master1-admin
hostnamectl set-hostname node1-monitor
hostnamectl set-hostname node2-osd
```

注：节点的hostname与/etc/hosts要相符，这个很重要否则后面ceph集群会出问题。

## 2.2.安装ceph集群

注：ubuntu22.04添加key的方式跟之前的版本不一样。

### 2.2.1 更新源

https://docs.ceph.com/en/quincy/install/get-packages

```shell
curl -fsSL https://mirrors.tuna.tsinghua.edu.cn/ceph/keys/release.asc | gpg --dearmor | sudo tee /etc/apt/trusted.gpg.d/ceph.gpg
echo deb https://mirrors.tuna.tsinghua.edu.cn/ceph/debian-pacific/ focal main | sudo tee /etc/apt/sources.list.d/ceph.list
apt update

# 全部节点
apt-get install ceph radosgw -y
```

### 2.2.2 安装ecph-deploy

https://download.ceph.com/debian-pacific/pool/main/c/ceph

只要主节点`master1-admin`操作

```shell
# 在 ceph01 节点安装 ceph-deploy
apt-get install python3 python3-pip -y
mkdir -p /home/cephadmin ; cd  /home/cephadmin
git clone https://github.com/ceph/ceph-deploy.git
cd ceph-deploy
pip3 install setuptools
python3 setup.py install
$ ceph-deploy --version
2.0.1
```

注：因为ubuntu 22.04不能apt install方式去安装的命令只能使用pip去安装，因为源里面没找到。

### 2.2.3 创建 monitor 节点

```shell
# 创建目录存放配置文件信息
mkdir -p /etc/ceph ; cd /etc/ceph
ceph-deploy new master1-admin node1-monitor node2-osd
[ceph_deploy.conf][DEBUG ] found configuration file at: /root/.cephdeploy.conf
[ceph_deploy.cli][INFO  ] Invoked (2.1.0): /usr/local/bin/ceph-deploy new master1-admin node1-monitor node2-osd
[ceph_deploy.cli][INFO  ] ceph-deploy options:
[ceph_deploy.cli][INFO  ]  verbose                       : False
[ceph_deploy.cli][INFO  ]  quiet                         : False
[ceph_deploy.cli][INFO  ]  username                      : None
[ceph_deploy.cli][INFO  ]  overwrite_conf                : False
[ceph_deploy.cli][INFO  ]  ceph_conf                     : None
[ceph_deploy.cli][INFO  ]  cluster                       : ceph
[ceph_deploy.cli][INFO  ]  mon                           : ['master1-admin', 'node1-monitor', 'node2-osd']
[ceph_deploy.cli][INFO  ]  ssh_copykey                   : True
[ceph_deploy.cli][INFO  ]  fsid                          : None
[ceph_deploy.cli][INFO  ]  cluster_network               : None
[ceph_deploy.cli][INFO  ]  public_network                : None
[ceph_deploy.cli][INFO  ]  cd_conf                       : <ceph_deploy.conf.cephdeploy.Conf object at 0x7f1ddd3d6800>
[ceph_deploy.cli][INFO  ]  default_release               : False
[ceph_deploy.cli][INFO  ]  func                          : <function new at 0x7f1ddd3f30a0>
[ceph_deploy.new][DEBUG ] Creating new cluster named ceph
[ceph_deploy.new][INFO  ] making sure passwordless SSH succeeds
[master1-admin][DEBUG ] connected to host: master1-admin
[master1-admin][INFO  ] Running command: /bin/ip link show
[master1-admin][INFO  ] Running command: /bin/ip addr show
[master1-admin][DEBUG ] IP addresses found: ['172.19.0.1', '192.168.80.55', '172.18.0.1', '172.17.0.1']
[ceph_deploy.new][DEBUG ] Resolving host master1-admin
[ceph_deploy.new][DEBUG ] Monitor master1-admin at 192.168.80.55
[ceph_deploy.new][INFO  ] making sure passwordless SSH succeeds
[node1-monitor][DEBUG ] connected to host: master1-admin
[node1-monitor][INFO  ] Running command: ssh -CT -o BatchMode=yes node1-monitor true
[node1-monitor][DEBUG ] connected to host: node1-monitor
[node1-monitor][INFO  ] Running command: /bin/ip link show
[node1-monitor][INFO  ] Running command: /bin/ip addr show
[node1-monitor][DEBUG ] IP addresses found: ['192.168.80.66']
[ceph_deploy.new][DEBUG ] Resolving host node1-monitor
[ceph_deploy.new][DEBUG ] Monitor node1-monitor at 192.168.80.66
[ceph_deploy.new][INFO  ] making sure passwordless SSH succeeds
[node2-osd][DEBUG ] connected to host: master1-admin
[node2-osd][INFO  ] Running command: ssh -CT -o BatchMode=yes node2-osd true
[node2-osd][DEBUG ] connected to host: node2-osd
[node2-osd][INFO  ] Running command: /bin/ip link show
[node2-osd][INFO  ] Running command: /bin/ip addr show
[node2-osd][DEBUG ] IP addresses found: ['192.168.80.77']
[ceph_deploy.new][DEBUG ] Resolving host node2-osd
[ceph_deploy.new][DEBUG ] Monitor node2-osd at 192.168.80.77
[ceph_deploy.new][DEBUG ] Monitor initial members are ['master1-admin', 'node1-monitor', 'node2-osd']
[ceph_deploy.new][DEBUG ] Monitor addrs are ['192.168.80.55', '192.168.80.66', '192.168.80.77']
[ceph_deploy.new][DEBUG ] Creating a random mon key...
[ceph_deploy.new][DEBUG ] Writing monitor keyring to ceph.mon.keyring...
[ceph_deploy.new][DEBUG ] Writing initial config to ceph.conf...

ls /etc/ceph/
ceph.conf  ceph-deploy-ceph.log  ceph.mon.keyring  rbdmap
#Ceph 配置文件、一个 monitor 密钥环和一个日志文件
```

### 2.2.4  安装 ceph-monitor

1、修改 ceph 配置文件

把 ceph.conf 配置文件里的默认副本数从 3 改成 1 。把 osd_pool_default_size = 2 加入[global]段，这样只有 2 个 osd 也能达到 active+clean 状态：

```shell
vim /etc/ceph/ceph.conf
[global]
fsid = 1ed6d2bd-52f1-4a95-bc36-2753f9892589
mon_initial_members = master1-admin, node1-monitor, node2-osd
mon_host = 192.168.80.55,192.168.80.66,192.168.80.77
auth_cluster_required = cephx
auth_service_required = cephx
auth_client_required = cephx
filestore_xattr_use_omap = true
# 存储集群副本个数
osd_pool_default_size = 2
mon clock drift allowed = 0.500
mon clock drift warn backoff = 10
public_network = 192.168.80.0/24
mon_allow_pool_delete = true
```

- mon clock drift allowed #监视器间允许的时钟漂移量默认值 0.05
- mon clock drift warn backoff #时钟偏移警告的退避指数。默认值 5

ceph 对每个 mon 之间的时间同步延时默认要求在 0.05s 之间，这个时间有的时候太短 了。所以如果 ceph 集群如果出现 clock 问题就检查 ntp 时间同步或者适当放宽这个误差时间。

2、配置初始 monitor、收集所有的密钥

```shell
cd /etc/ceph
# 文件同步到其他节点
scp /etc/ceph/ceph.conf node1-monitor:/etc/ceph/
scp /etc/ceph/ceph.conf node2-osd:/etc/ceph/
ceph-deploy mon create-initial
ls *.keyring
ceph.bootstrap-mds.keyring  ceph.bootstrap-mgr.keyring  ceph.bootstrap-osd.keyring  ceph.bootstrap-rgw.keyring  ceph.client.admin.keyring  ceph.mon.keyring
```

如果报错请参考：https://blog.whsir.com/post-4604.html

### 2.2.5 部署 osd 服务

```shell
cd /etc/ceph
ceph-deploy osd create --data  /dev/sdb  master1-admin
ceph-deploy osd create --data  /dev/sdb  node1-monitor
ceph-deploy osd create --data  /dev/sdb  node2-osd
# 查看状态
ceph-deploy osd list master1-admin node1-monitor node2-osd
```

### 2.2.6 创建 ceph 文件系统

```shell
ceph-deploy mds create master1-admin node1-monitor node2-osd
# 查看 ceph 当前文件系统
ceph fs ls
No filesystems enabled
# 创建存储池
ceph osd pool create cephfs_data 128
pool 'cephfs_data' created
ceph osd pool create cephfs_metadata 128
pool 'cephfs_metadata' created
```

一个 cephfs 至少要求两个 librados 存储池，一个为 data，一个为 metadata。当配置这 两个存储池时，注意： 

1. 为 metadata pool 设置较高级别的副本级别，因为 metadata 的损坏可能导致整个文 件系统不用 

2. 建议，metadata pool 使用低延时存储，比如 SSD，因为 metadata 会直接影响客户端 的响应速度。

关于创建存储池，确定 pg_num 取值是强制性的，因为不能自动计算。

下面是几个常用的值： 

- *少于 5 个 OSD 时可把 pg_num 设置为 128
- *OSD 数量在 5 到 10 个时，可把 pg_num 设置为 512
- *OSD 数量在 10 到 50 个时，可把 pg_num 设置为 4096
- *OSD 数量大于 50 时，你得理解权衡方法、以及如何自己计算 pg_num 取值
- *自己计算 pg_num 取值时可借助 pgcalc 工具

```sh
# 使用 fs new 命令创建文件系统
ceph fs new storageclass cephfs_metadata cephfs_data
new fs with metadata pool 2 and data pool 1
# 查看创建后的 cephfs
ceph fs ls
# 查看 mds 节点状态
ceph mds stat
storageclass:1 {0=ceph-node2-osd=up:active} 2 up:standby
# 禁用不安全模式
ceph config set mon auth_allow_insecure_global_id_reclaim false
# active 是活跃的，另 1 个是处于热备份的状态
ceph -s

# 这里发现mgr没启动，执行下面的命令创建mgr进程
ceph-deploy mgr create master1-admin node1-monitor node2-osd
ceph mgr metadata
```

注：HEALTH_OK 表示 ceph 集群正常
排查方式：

```shell
ceph health detail
# 问题描述
HEALTH_WARN 1 pool(s) have non-power-of-two pg_num
[WRN] POOL_PG_NUM_NOT_POWER_OF_TWO: 1 pool(s) have non-power-of-two pg_num
    pool 'k8srbd1' pg_num 56 is not a power of two
    
# 排查过程
sudo ceph osd pool get k8srbd1 pg_num
pg_num: 56
sudo ceph osd pool get k8srbd1 pgp_num
pgp_num: 56

ceph osd pool set k8srbd1  pg_num 128
```

# 3.测试 k8s 挂载 ceph rbd

## 3.1.测试环境

k8s节点

```sh
kubectl get nodes
NAME           STATUS                        ROLES    AGE   VERSION
k8s-master01   Ready                         master   12d   v1.26.5 # 192.168.80.45
k8s-master02   NotReady,SchedulingDisabled   master   12d   v1.26.5
k8s-master03   NotReady,SchedulingDisabled   master   12d   v1.26.5
k8s-node01     Ready                         node     12d   v1.26.5 # 192.168.80.48
k8s-node02     NotReady                      node     12d   v1.26.5 # 192.168.80.49
```

k8s节点准备：

```sh
# kubernetes安装ceph-common命令
apt install ceph-common -y
#将 ceph 配置文件拷贝到 k8s 的控制节点和工作节点
scp /etc/ceph/* 192.168.80.45:/etc/ceph/
scp /etc/ceph/* 192.168.80.48:/etc/ceph/

#创建 ceph rbd
ceph osd pool create k8srbd1 128
pool 'k8srbd1' created
ceph osd pool application enable k8srbd1 rbda

rbd create rbda -s 1024 -p k8srbd1
rbd feature disable k8srbd1/rbda object-map fast-diff deep-flatten
```

删除的操作：`ceph osd pool rm k8srbd1 k8srbd1 --yes-i-really-really-mean-it`。

## 3.2.基于 ceph rbd 生成 pv

1. 创建 ceph-secret 这个 k8s secret 对象，这个 secret 对象用于 k8s volume 插件访问 ceph 集群，获取 client.admin 的 keyring 值，并用 base64 编码，在 master1-admin（ceph 管理节点）操作

```shell
ceph auth get-key client.admin | base64
QVFESlVKUmtFOGlsRUJBQThNb3dpRlFocHVJTzQ4SGFWNWtZbGc9PQ==
```

2. 创建 ceph 的 secret，在 k8s 的控制节点操作：

```shell
cat > ceph-secret.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: ceph-secret
data:
  key: QVFESlVKUmtFOGlsRUJBQThNb3dpRlFocHVJTzQ4SGFWNWtZbGc9PQ==
EOF
kubectl apply -f ceph-secret.yaml
```

3. 创建 pv

```shell
cat > pv.yaml <<EOF
apiVersion: v1
kind: PersistentVolume
metadata:
  name: ceph-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  rbd:
    monitors:
    - 192.168.80.55:6789
    - 192.168.80.66:6789
    - 192.168.80.77:6789
    pool: k8srbd1
    image: rbda
    user: admin
    secretRef:
      name: ceph-secret
      namespace: default
    fsType: ext4
    readOnly: false
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: ceph
EOF
```

创建pv：

```shell
kubectl apply -f pv.yaml
kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                          STORAGECLASS   REASON   AGE
ceph-pv                                    10Gi       RWO            Recycle          Available                                                          108s
```

## 3.3.创建 PVC

```shell
cat > pvc.yaml <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ceph-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: ceph
EOF
```

创建pvc：

```shell
kubectl apply -f pvc.yaml
kubectl get pvc
NAME                   STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
ceph-pvc               Bound    ceph-pv                                    10Gi       RWO            ceph           17s
```

## 3.4.创建 pod

```shell
cat > tomcat.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: tomcat
  name: tomcat
spec:
  replicas: 3
  selector:
    matchLabels:
      app: tomcat
  template:
    metadata:
      labels:
        app: tomcat
    spec:
      containers:
      - image: tomcat:7.0
        imagePullPolicy: IfNotPresent
        name: tomcat8
        ports:
        - containerPort: 8080
        volumeMounts:
          - mountPath: "/etc"
            name: ceph-data
      volumes:
      - name: ceph-data
        persistentVolumeClaim:
          claimName: ceph-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: tomcat
spec:
   ports:
     - port: 8099
       targetPort: 8080
       nodePort: 30899  #浏览器访问的端口
   selector:
     app: tomcat
   type: NodePort
EOF
```

创建容器：

```shell
kubectl apply -f tomcat.yaml
kubectl get pods -l app=tomcat -owide
NAME                      READY   STATUS              RESTARTS   AGE     IP              NODE         NOMINATED NODE   READINESS GATES
tomcat-57448589bf-mn5bm   1/1     Running             0          4m19s   172.16.85.220   k8s-node01   <none>           <none>
tomcat-57448589bf-nmm4g   1/1     Running             0          4m19s   172.16.85.219   k8s-node01   <none>           <none>
tomcat-57448589bf-zh8lg   0/1     ContainerCreating   0          4m19s   <none>          k8s-node02   <none>           <none> # 这里因为跨节点导致这个pod挂载不上pvc，所以一直存在创建的状态。


#报错信息：
Unable to attach or mount volumes: unmounted volumes=[ceph-data], unattached volumes=[kube-api-access-llxrr ceph-data]: timed out waiting for the condition
```

注意：ceph rbd 块存储的特点，ceph rbd 块存储能在同一个 node 上跨 pod 以 ReadWriteOnce 共享挂载，ceph rbd 块存储能在同一个 node 上同一个 pod 多个容器中以 ReadWriteOnce 共享挂载，ceph rbd 块存储不能跨 node 以 ReadWriteOnce 共享挂载。如果一个使用ceph rdb 的pod所在的node挂掉，这个pod虽然会被调度到其它node， 但是由于 rbd 不能跨 node 多次挂载和挂掉的 pod 不能自动解绑 pv 的问题，这个新 pod 不会正常运行

Deployment 更新特性：

deployment 触发更新的时候，它确保至少所需 Pods 75% 处于运行状态（最大不可用 比例为 25%）。故像一个 pod 的情况，肯定是新创建一个新的 pod，新 pod 运行正常之 后，再关闭老的 pod。 默认情况下，它可确保启动的 Pod 个数比期望个数最多多出 25% 问题： 结合 ceph rbd 共享挂载的特性和 deployment 更新的特性，我们发现原因如下： 由于 deployment 触发更新，为了保证服务的可用性，deployment 要先创建一个 pod 并运行正常之后，再去删除老 pod。而如果新创建的 pod 和老 pod 不在一个 node，就会导致此故障。 解决办法： 1，使用能支持跨 node 和 pod 之间挂载的共享存储，例如 cephfs，GlusterFS 等 2，给 node 添加 label，只允许 deployment 所管理的 pod 调度到一个固定的 node 上。 （不建议，这个 node 挂掉的话，服务就故障了）

# 4.基于ceph-csi创建storageClass(rbd)

kubernetes官网：https://kubernetes.io/zh-cn/docs/concepts/storage/storage-classes/#ceph-rbd

插件仓库：https://github.com/kubernetes-retired/external-storage

文档实例：https://docs.ceph.com/en/latest/rbd/rbd-kubernetes

github：https://github.com/ceph/ceph-csi

## 4.1.创建存储池

pg与pgs要根据实际情况修改。

```shell
mkdir -p /root/ecph && cd /root/ecph
ceph osd pool create kubernetes 128 128
ceph osd pool application enable kubernetes rbd
# 初始化
rbd pool init kubernetes
# 为 Kubernetes 和 ceph-csi 创建一个新用户。执行以下命令并 记录生成的密钥：
ceph auth get-or-create client.kubernetes mon 'profile rbd' osd 'profile rbd pool=kubernetes' mgr 'profile rbd pool=kubernetes'
[client.kubernetes]
        key = AQAusZVkZ6+rLxAAXEs7sAgv+jSKPNOfaykmYA==
```

## 4.2.生成 CEPH-CSI 配置映射

```shell
# ceph-csi 需要一个存储在 Kubernetes 中的 ConfigMap 对象来定义 Ceph 集群的 Ceph 监视器地址。收集两个 Ceph 集群 唯一的 FSID 和监视器地址：
ceph mon dump
epoch 1
fsid 1ed6d2bd-52f1-4a95-bc36-2753f9892589
0: [v2:192.168.80.55:3300/0,v1:192.168.80.55:6789/0] mon.master1-admin
1: [v2:192.168.80.66:3300/0,v1:192.168.80.66:6789/0] mon.node1-monitor
2: [v2:192.168.80.77:3300/0,v1:192.168.80.77:6789/0] mon.node2-osd
dumped monmap epoch 1
```

## 4.3.配置ceph-csi configmap

创建ConfigMap对象，将fsid替换为"clusterID"，并将监视器地址替换为"monitors"。

```shell
cat > csi-config-map.yaml << EOF
---
apiVersion: v1
kind: ConfigMap
data:
  config.json: |-
    [
      {
        "clusterID": "1ed6d2bd-52f1-4a95-bc36-2753f9892589",
        "monitors": [
          "192.168.80.55:6789",
          "192.168.80.66:6789",
          "192.168.80.77:6789"
        ]
      }
    ]
metadata:
  name: ceph-csi-config
EOF
kubectl apply -f csi-config-map.yaml
```

\# 新版本的csi还需要一个额外的ConfigMap对象来定义密钥管理服务 (KMS) 提供者的详细信息, 空配置即可以。

```shell
cat > csi-kms-config-map.yaml << EOF
---
apiVersion: v1
kind: ConfigMap
data:
  config.json: |-
    {}
metadata:
  name: ceph-csi-encryption-kms-config
EOF
kubectl apply -f csi-kms-config-map.yaml
```

\# 查看ceph.conf文件

```shell
cat /etc/ceph/ceph.conf
[global]
fsid = 1ed6d2bd-52f1-4a95-bc36-2753f9892589
mon_initial_members = master1-admin, node1-monitor, node2-osd
mon_host = 192.168.80.55,192.168.80.66,192.168.80.77
auth_cluster_required = cephx
auth_service_required = cephx
auth_client_required = cephx
filestore_xattr_use_omap = true
# 存储集群副本个数
osd_pool_default_size = 2
mon clock drift allowed = 0.500
mon clock drift warn backoff = 10
public_network = 192.168.80.0/24
mon_allow_pool_delete = true
```

\# 通过ConfigMap对象来定义 Ceph 配置，以添加到 CSI 容器内的 ceph.conf 文件中。ceph.conf文件内容替换以下的内容。

```shell
cat > ceph-config-map.yaml << EOF
---
apiVersion: v1
kind: ConfigMap
data:
  ceph.conf: |
    [global]
    fsid = 1ed6d2bd-52f1-4a95-bc36-2753f9892589
    mon_initial_members = master1-admin, node1-monitor, node2-osd
    mon_host = 192.168.80.55,192.168.80.66,192.168.80.77
    auth_cluster_required = cephx
    auth_service_required = cephx
    auth_client_required = cephx
    filestore_xattr_use_omap = true
    # 存储集群副本个数
    osd_pool_default_size = 2
    mon clock drift allowed = 0.500
    mon clock drift warn backoff = 10
    public_network = 192.168.80.0/24
    mon_allow_pool_delete = true
  # keyring is a required key and its value should be empty
  keyring: |
metadata:
  name: ceph-config
EOF
kubectl apply -f ceph-config-map.yaml
```

## 4.4.配置ceph-csi cephx secret

\# 创建secret对象, csi需要cephx凭据才能与ceph集群通信。

```shell
cat > csi-rbd-secret.yaml << EOF
---
apiVersion: v1
kind: Secret
metadata:
  name: csi-rbd-secret
  namespace: default
stringData:
  userID: kubernetes
  userKey: AQAusZVkZ6+rLxAAXEs7sAgv+jSKPNOfaykmYA==
EOF
kubectl apply -f csi-rbd-secret.yaml
```

## 4.5.配置 CEPH-CSI 插件

\# 创建所需的`ServiceAccount`和 `RBAC ClusterRole`/`ClusterRoleBinding` Kubernetes 对象。

```shell
# https://raw.githubusercontent.com/ceph/ceph-csi/master/deploy/rbd/kubernetes/csi-provisioner-rbac.yaml
cat > csi-provisioner-rbac.yaml << EOF
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: rbd-csi-provisioner
  # replace with non-default namespace name
  namespace: default

---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: rbd-external-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["list", "watch", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "update", "delete", "patch"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims/status"]
    verbs: ["update", "patch"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["snapshot.storage.k8s.io"]
    resources: ["volumesnapshots"]
    verbs: ["get", "list", "patch"]
  - apiGroups: ["snapshot.storage.k8s.io"]
    resources: ["volumesnapshots/status"]
    verbs: ["get", "list", "patch"]
  - apiGroups: ["snapshot.storage.k8s.io"]
    resources: ["volumesnapshotcontents"]
    verbs: ["create", "get", "list", "watch", "update", "delete", "patch"]
  - apiGroups: ["snapshot.storage.k8s.io"]
    resources: ["volumesnapshotclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["volumeattachments"]
    verbs: ["get", "list", "watch", "update", "patch"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["volumeattachments/status"]
    verbs: ["patch"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["csinodes"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["snapshot.storage.k8s.io"]
    resources: ["volumesnapshotcontents/status"]
    verbs: ["update", "patch"]
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get"]
  - apiGroups: [""]
    resources: ["serviceaccounts"]
    verbs: ["get"]
  - apiGroups: [""]
    resources: ["serviceaccounts/token"]
    verbs: ["create"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: rbd-csi-provisioner-role
subjects:
  - kind: ServiceAccount
    name: rbd-csi-provisioner
    # replace with non-default namespace name
    namespace: default
roleRef:
  kind: ClusterRole
  name: rbd-external-provisioner-runner
  apiGroup: rbac.authorization.k8s.io

---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  # replace with non-default namespace name
  namespace: default
  name: rbd-external-provisioner-cfg
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list", "watch", "create", "update", "delete"]
  - apiGroups: ["coordination.k8s.io"]
    resources: ["leases"]
    verbs: ["get", "watch", "list", "delete", "update", "create"]

---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: rbd-csi-provisioner-role-cfg
  # replace with non-default namespace name
  namespace: default
subjects:
  - kind: ServiceAccount
    name: rbd-csi-provisioner
    # replace with non-default namespace name
    namespace: default
roleRef:
  kind: Role
  name: rbd-external-provisioner-cfg
  apiGroup: rbac.authorization.k8s.io
EOF
kubectl apply -f csi-provisioner-rbac.yaml
```

csi-nodeplugin-rbac

```shell
# https://raw.githubusercontent.com/ceph/ceph-csi/master/deploy/rbd/kubernetes/csi-nodeplugin-rbac.yaml
cat > csi-nodeplugin-rbac.yaml << EOF
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: rbd-csi-nodeplugin
  # replace with non-default namespace name
  namespace: default
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: rbd-csi-nodeplugin
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get"]
  # allow to read Vault Token and connection options from the Tenants namespace
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get"]
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get"]
  - apiGroups: [""]
    resources: ["serviceaccounts"]
    verbs: ["get"]
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["volumeattachments"]
    verbs: ["list", "get"]
  - apiGroups: [""]
    resources: ["serviceaccounts/token"]
    verbs: ["create"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: rbd-csi-nodeplugin
subjects:
  - kind: ServiceAccount
    name: rbd-csi-nodeplugin
    # replace with non-default namespace name
    namespace: default
roleRef:
  kind: ClusterRole
  name: rbd-csi-nodeplugin
  apiGroup: rbac.authorization.k8s.io
EOF
kubectl apply -f csi-nodeplugin-rbac.yaml
```

## 4.6.创建所需的`ceph-csi`容器

kubernetes版本和ceph-csi版本对照：https://github.com/ceph/ceph-csi#known-to-work-co-platforms

```shell
# https://raw.githubusercontent.com/ceph/ceph-csi/v3.8.0/deploy/rbd/kubernetes/csi-rbdplugin-provisioner.yaml
cat > csi-rbdplugin-provisioner.yaml <<"EOF"
---
kind: Service
apiVersion: v1
metadata:
  name: csi-rbdplugin-provisioner
  # replace with non-default namespace name
  namespace: default
  labels:
    app: csi-metrics
spec:
  selector:
    app: csi-rbdplugin-provisioner
  ports:
    - name: http-metrics
      port: 8080
      protocol: TCP
      targetPort: 8680

---
kind: Deployment
apiVersion: apps/v1
metadata:
  name: csi-rbdplugin-provisioner
  # replace with non-default namespace name
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: csi-rbdplugin-provisioner
  template:
    metadata:
      labels:
        app: csi-rbdplugin-provisioner
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app
                    operator: In
                    values:
                      - csi-rbdplugin-provisioner
              topologyKey: "kubernetes.io/hostname"
      serviceAccountName: rbd-csi-provisioner
      priorityClassName: system-cluster-critical
      containers:
        - name: csi-provisioner
          image: registry.k8s.io/sig-storage/csi-provisioner:v3.3.0
          args:
            - "--csi-address=$(ADDRESS)"
            - "--v=1"
            - "--timeout=150s"
            - "--retry-interval-start=500ms"
            - "--leader-election=true"
            #  set it to true to use topology based provisioning
            - "--feature-gates=Topology=false"
            - "--feature-gates=HonorPVReclaimPolicy=true"
            - "--prevent-volume-mode-conversion=true"
            # if fstype is not specified in storageclass, ext4 is default
            - "--default-fstype=ext4"
            - "--extra-create-metadata=true"
          env:
            - name: ADDRESS
              value: unix:///csi/csi-provisioner.sock
          imagePullPolicy: "IfNotPresent"
          volumeMounts:
            - name: socket-dir
              mountPath: /csi
        - name: csi-snapshotter
          image: registry.k8s.io/sig-storage/csi-snapshotter:v6.1.0
          args:
            - "--csi-address=$(ADDRESS)"
            - "--v=1"
            - "--timeout=150s"
            - "--leader-election=true"
            - "--extra-create-metadata=true"
          env:
            - name: ADDRESS
              value: unix:///csi/csi-provisioner.sock
          imagePullPolicy: "IfNotPresent"
          volumeMounts:
            - name: socket-dir
              mountPath: /csi
        - name: csi-attacher
          image: registry.k8s.io/sig-storage/csi-attacher:v4.0.0
          args:
            - "--v=1"
            - "--csi-address=$(ADDRESS)"
            - "--leader-election=true"
            - "--retry-interval-start=500ms"
            - "--default-fstype=ext4"
          env:
            - name: ADDRESS
              value: /csi/csi-provisioner.sock
          imagePullPolicy: "IfNotPresent"
          volumeMounts:
            - name: socket-dir
              mountPath: /csi
        - name: csi-resizer
          image: registry.k8s.io/sig-storage/csi-resizer:v1.6.0
          args:
            - "--csi-address=$(ADDRESS)"
            - "--v=1"
            - "--timeout=150s"
            - "--leader-election"
            - "--retry-interval-start=500ms"
            - "--handle-volume-inuse-error=false"
            - "--feature-gates=RecoverVolumeExpansionFailure=true"
          env:
            - name: ADDRESS
              value: unix:///csi/csi-provisioner.sock
          imagePullPolicy: "IfNotPresent"
          volumeMounts:
            - name: socket-dir
              mountPath: /csi
        - name: csi-rbdplugin
          # for stable functionality replace canary with latest release version
          image: quay.io/cephcsi/cephcsi:v3.8.0
          args:
            - "--nodeid=$(NODE_ID)"
            - "--type=rbd"
            - "--controllerserver=true"
            - "--endpoint=$(CSI_ENDPOINT)"
            - "--csi-addons-endpoint=$(CSI_ADDONS_ENDPOINT)"
            - "--v=5"
            - "--drivername=rbd.csi.ceph.com"
            - "--pidlimit=-1"
            - "--rbdhardmaxclonedepth=8"
            - "--rbdsoftmaxclonedepth=4"
            - "--enableprofiling=false"
            - "--setmetadata=true"
          env:
            - name: POD_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
            - name: NODE_ID
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            # - name: KMS_CONFIGMAP_NAME
            #   value: encryptionConfig
            - name: CSI_ENDPOINT
              value: unix:///csi/csi-provisioner.sock
            - name: CSI_ADDONS_ENDPOINT
              value: unix:///csi/csi-addons.sock
          imagePullPolicy: "IfNotPresent"
          volumeMounts:
            - name: socket-dir
              mountPath: /csi
            - mountPath: /dev
              name: host-dev
            - mountPath: /sys
              name: host-sys
            - mountPath: /lib/modules
              name: lib-modules
              readOnly: true
            - name: ceph-csi-config
              mountPath: /etc/ceph-csi-config/
            - name: ceph-csi-encryption-kms-config
              mountPath: /etc/ceph-csi-encryption-kms-config/
            - name: keys-tmp-dir
              mountPath: /tmp/csi/keys
            - name: ceph-config
              mountPath: /etc/ceph/
            - name: oidc-token
              mountPath: /run/secrets/tokens
              readOnly: true
        - name: csi-rbdplugin-controller
          # for stable functionality replace canary with latest release version
          image: quay.io/cephcsi/cephcsi:v3.8.0
          args:
            - "--type=controller"
            - "--v=5"
            - "--drivername=rbd.csi.ceph.com"
            - "--drivernamespace=$(DRIVER_NAMESPACE)"
            - "--setmetadata=true"
          env:
            - name: DRIVER_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          imagePullPolicy: "IfNotPresent"
          volumeMounts:
            - name: ceph-csi-config
              mountPath: /etc/ceph-csi-config/
            - name: keys-tmp-dir
              mountPath: /tmp/csi/keys
            - name: ceph-config
              mountPath: /etc/ceph/
        - name: liveness-prometheus
          image: quay.io/cephcsi/cephcsi:v3.8.0
          args:
            - "--type=liveness"
            - "--endpoint=$(CSI_ENDPOINT)"
            - "--metricsport=8680"
            - "--metricspath=/metrics"
            - "--polltime=60s"
            - "--timeout=3s"
          env:
            - name: CSI_ENDPOINT
              value: unix:///csi/csi-provisioner.sock
            - name: POD_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
          volumeMounts:
            - name: socket-dir
              mountPath: /csi
          imagePullPolicy: "IfNotPresent"
      volumes:
        - name: host-dev
          hostPath:
            path: /dev
        - name: host-sys
          hostPath:
            path: /sys
        - name: lib-modules
          hostPath:
            path: /lib/modules
        - name: socket-dir
          emptyDir: {
            medium: "Memory"
          }
        - name: ceph-config
          configMap:
            name: ceph-config
        - name: ceph-csi-config
          configMap:
            name: ceph-csi-config
        - name: ceph-csi-encryption-kms-config
          configMap:
            name: ceph-csi-encryption-kms-config
        - name: keys-tmp-dir
          emptyDir: {
            medium: "Memory"
          }
        - name: oidc-token
          projected:
            sources:
              - serviceAccountToken:
                  path: oidc-token
                  expirationSeconds: 3600
                  audience: ceph-csi-kms
EOF
sed -i s#registry.k8s.io/sig-storage#registry.cn-hangzhou.aliyuncs.com/image-storage#g csi-rbdplugin-provisioner.yaml
sed -i s#quay.io/cephcsi#registry.cn-hangzhou.aliyuncs.com/image-storage#g csi-rbdplugin-provisioner.yaml
cat csi-rbdplugin-provisioner.yaml  | grep image:
kubectl apply -f csi-rbdplugin-provisioner.yaml
```

csi-rbdplugin

```shell
# https://raw.githubusercontent.com/ceph/ceph-csi/v3.8.0/deploy/rbd/kubernetes/csi-rbdplugin.yaml
cat > csi-rbdplugin.yaml <<"EOF"
---
kind: DaemonSet
apiVersion: apps/v1
metadata:
  name: csi-rbdplugin
  # replace with non-default namespace name
  namespace: default
spec:
  selector:
    matchLabels:
      app: csi-rbdplugin
  template:
    metadata:
      labels:
        app: csi-rbdplugin
    spec:
      serviceAccountName: rbd-csi-nodeplugin
      hostNetwork: true
      hostPID: true
      priorityClassName: system-node-critical
      # to use e.g. Rook orchestrated cluster, and mons' FQDN is
      # resolved through k8s service, set dns policy to cluster first
      dnsPolicy: ClusterFirstWithHostNet
      containers:
        - name: driver-registrar
          # This is necessary only for systems with SELinux, where
          # non-privileged sidecar containers cannot access unix domain socket
          # created by privileged CSI driver container.
          securityContext:
            privileged: true
            allowPrivilegeEscalation: true
          image: registry.k8s.io/sig-storage/csi-node-driver-registrar:v2.6.2
          args:
            - "--v=1"
            - "--csi-address=/csi/csi.sock"
            - "--kubelet-registration-path=/var/lib/kubelet/plugins/rbd.csi.ceph.com/csi.sock"
          env:
            - name: KUBE_NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
          volumeMounts:
            - name: socket-dir
              mountPath: /csi
            - name: registration-dir
              mountPath: /registration
        - name: csi-rbdplugin
          securityContext:
            privileged: true
            capabilities:
              add: ["SYS_ADMIN"]
            allowPrivilegeEscalation: true
          # for stable functionality replace canary with latest release version
          image: quay.io/cephcsi/cephcsi:v3.8.0
          args:
            - "--nodeid=$(NODE_ID)"
            - "--pluginpath=/var/lib/kubelet/plugins"
            - "--stagingpath=/var/lib/kubelet/plugins/kubernetes.io/csi/"
            - "--type=rbd"
            - "--nodeserver=true"
            - "--endpoint=$(CSI_ENDPOINT)"
            - "--csi-addons-endpoint=$(CSI_ADDONS_ENDPOINT)"
            - "--v=5"
            - "--drivername=rbd.csi.ceph.com"
            - "--enableprofiling=false"
            # If topology based provisioning is desired, configure required
            # node labels representing the nodes topology domain
            # and pass the label names below, for CSI to consume and advertise
            # its equivalent topology domain
            # - "--domainlabels=failure-domain/region,failure-domain/zone"
            #
            # Options to enable read affinity.
            # If enabled Ceph CSI will fetch labels from kubernetes node and
            # pass `read_from_replica=localize,crush_location=type:value` during
            # rbd map command. refer:
            # https://docs.ceph.com/en/latest/man/8/rbd/#kernel-rbd-krbd-options
            # for more details.
            # - "--enable-read-affinity=true"
            # - "--crush-location-labels=topology.io/zone,topology.io/rack"
          env:
            - name: POD_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
            - name: NODE_ID
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            # - name: KMS_CONFIGMAP_NAME
            #   value: encryptionConfig
            - name: CSI_ENDPOINT
              value: unix:///csi/csi.sock
            - name: CSI_ADDONS_ENDPOINT
              value: unix:///csi/csi-addons.sock
          imagePullPolicy: "IfNotPresent"
          volumeMounts:
            - name: socket-dir
              mountPath: /csi
            - mountPath: /dev
              name: host-dev
            - mountPath: /sys
              name: host-sys
            - mountPath: /run/mount
              name: host-mount
            - mountPath: /etc/selinux
              name: etc-selinux
              readOnly: true
            - mountPath: /lib/modules
              name: lib-modules
              readOnly: true
            - name: ceph-csi-config
              mountPath: /etc/ceph-csi-config/
            - name: ceph-csi-encryption-kms-config
              mountPath: /etc/ceph-csi-encryption-kms-config/
            - name: plugin-dir
              mountPath: /var/lib/kubelet/plugins
              mountPropagation: "Bidirectional"
            - name: mountpoint-dir
              mountPath: /var/lib/kubelet/pods
              mountPropagation: "Bidirectional"
            - name: keys-tmp-dir
              mountPath: /tmp/csi/keys
            - name: ceph-logdir
              mountPath: /var/log/ceph
            - name: ceph-config
              mountPath: /etc/ceph/
            - name: oidc-token
              mountPath: /run/secrets/tokens
              readOnly: true
        - name: liveness-prometheus
          securityContext:
            privileged: true
            allowPrivilegeEscalation: true
          image: quay.io/cephcsi/cephcsi:v3.8.0
          args:
            - "--type=liveness"
            - "--endpoint=$(CSI_ENDPOINT)"
            - "--metricsport=8680"
            - "--metricspath=/metrics"
            - "--polltime=60s"
            - "--timeout=3s"
          env:
            - name: CSI_ENDPOINT
              value: unix:///csi/csi.sock
            - name: POD_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
          volumeMounts:
            - name: socket-dir
              mountPath: /csi
          imagePullPolicy: "IfNotPresent"
      volumes:
        - name: socket-dir
          hostPath:
            path: /var/lib/kubelet/plugins/rbd.csi.ceph.com
            type: DirectoryOrCreate
        - name: plugin-dir
          hostPath:
            path: /var/lib/kubelet/plugins
            type: Directory
        - name: mountpoint-dir
          hostPath:
            path: /var/lib/kubelet/pods
            type: DirectoryOrCreate
        - name: ceph-logdir
          hostPath:
            path: /var/log/ceph
            type: DirectoryOrCreate
        - name: registration-dir
          hostPath:
            path: /var/lib/kubelet/plugins_registry/
            type: Directory
        - name: host-dev
          hostPath:
            path: /dev
        - name: host-sys
          hostPath:
            path: /sys
        - name: etc-selinux
          hostPath:
            path: /etc/selinux
        - name: host-mount
          hostPath:
            path: /run/mount
        - name: lib-modules
          hostPath:
            path: /lib/modules
        - name: ceph-config
          configMap:
            name: ceph-config
        - name: ceph-csi-config
          configMap:
            name: ceph-csi-config
        - name: ceph-csi-encryption-kms-config
          configMap:
            name: ceph-csi-encryption-kms-config
        - name: keys-tmp-dir
          emptyDir: {
            medium: "Memory"
          }
        - name: oidc-token
          projected:
            sources:
              - serviceAccountToken:
                  path: oidc-token
                  expirationSeconds: 3600
                  audience: ceph-csi-kms
---
# This is a service to expose the liveness metrics
apiVersion: v1
kind: Service
metadata:
  name: csi-metrics-rbdplugin
  # replace with non-default namespace name
  namespace: default
  labels:
    app: csi-metrics
spec:
  ports:
    - name: http-metrics
      port: 8080
      protocol: TCP
      targetPort: 8680
  selector:
    app: csi-rbdplugin
EOF
sed -i s#quay.io/cephcsi#registry.cn-hangzhou.aliyuncs.com/image-storage#g csi-rbdplugin.yaml
sed -i s#registry.k8s.io/sig-storage#registry.cn-hangzhou.aliyuncs.com/image-storage#g csi-rbdplugin.yaml
cat csi-rbdplugin.yaml | grep image:
kubectl apply -f csi-rbdplugin.yaml
```

## 4.7.使用ceph块设备

\# 创建一个存储类
kubernetes storageclass定义了一个存储类。 可以创建多个storageclass对象以映射到不同的服务质量级别（即 NVMe 与基于 HDD 的池）和功能。例如，要创建一个映射到上面创建的kubernetes池的storageclass ，确保"clusterID"与您的ceph集群的fsid一致。

```shell
cat > csi-rbd-sc.yaml << EOF
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
   name: csi-rbd-sc
provisioner: rbd.csi.ceph.com
parameters:
   clusterID: 1ed6d2bd-52f1-4a95-bc36-2753f9892589
   pool: kubernetes
   imageFeatures: layering
   csi.storage.k8s.io/provisioner-secret-name: csi-rbd-secret
   csi.storage.k8s.io/provisioner-secret-namespace: default
   csi.storage.k8s.io/controller-expand-secret-name: csi-rbd-secret
   csi.storage.k8s.io/controller-expand-secret-namespace: default
   csi.storage.k8s.io/node-stage-secret-name: csi-rbd-secret
   csi.storage.k8s.io/node-stage-secret-namespace: default
reclaimPolicy: Delete
allowVolumeExpansion: true
mountOptions:
   - discard
EOF
kubectl apply -f csi-rbd-sc.yaml
```

创建pvc，使用上面创建的storageclass创建PersistentVolumeClaim

```shell
cat > raw-block-pvc.yaml << EOF
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: raw-block-pvc
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 2Gi
  storageClassName: csi-rbd-sc
EOF
kubectl apply -f raw-block-pvc.yaml
```

查看pvc的状态为Bound即正常。

```shell
kubectl get pvc
NAME            STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
raw-block-pvc          Bound    pvc-e3578bd3-b2e1-4cfa-809a-5e9b8a12cf82   2Gi        RWO            csi-rbd-sc     7s
```

将上述 pvc绑定到pod资源作为挂载文件系统的示例。

```shell
cat > raw-block-pod.yaml << EOF
---
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-raw-block-volume
spec:
  containers:
    - name: web-container
      image: nginx
      volumeMounts:
        - name: data
          mountPath: /var/lib/www/html
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: raw-block-pvc
EOF
kubectl apply -f raw-block-pod.yaml
kubectl exec -it pod-with-raw-block-volume -- lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sdb      8:16   0   100G  0 disk
sr0     11:0    1  53.7M  0 rom
sr1     11:1    1   4.6G  0 rom
rbd0   252:0    0     2G  0 disk /var/lib/www/html
```

# 5.基于ceph-csi创建storageClass(cephfs)

```shell
ceph fs ls
name: storageclass, metadata pool: cephfs_metadata, data pools: [cephfs_data ]
```

## 5.1.创建 ceph 子目录

这里主要演示本地怎么去挂载cephfs。

```shell
cat /etc/ceph/ceph.client.admin.keyring |grep key|awk -F" " '{print $3}' > /etc/ceph/admin.secret
# 挂载 cephfs 的根目录到集群的 mon 节点下的一个目录，比如 cephfs_data，因为挂载后，我们就可以直接在 cephfs_data 下面用 Linux 命令创建子目录了。
cd /root ; mkdir -p cephfs_data
mount -t ceph 192.168.80.66:6789:/ /root/cephfs_data -o name=admin,secretfile=/etc/ceph/admin.secret
df -h | grep /root/cephfs_data
192.168.80.66:6789:/  143G     0  143G   0% /root/cephfs_data

cd /root/cephfs_data ; mkdir ceph ; chmod 0777 ceph
```

## 5.2.部署Ceph-CSI

ceph-csib版本对照：https://github.com/ceph/ceph-csi#known-to-work-co-platforms

```shell
cd /root/ceph
git clone https://github.com/ceph/ceph-csi.git
cd ceph-csi/deploy/cephfs/kubernetes

# 替换镜像
cat ./*.yaml | grep image:
          image: registry.k8s.io/sig-storage/csi-provisioner:v3.5.0
          image: registry.k8s.io/sig-storage/csi-resizer:v1.8.0
          image: registry.k8s.io/sig-storage/csi-snapshotter:v6.2.2
          image: quay.io/cephcsi/cephcsi:canary
          image: quay.io/cephcsi/cephcsi:canary
          image: registry.k8s.io/sig-storage/csi-node-driver-registrar:v2.8.0
          image: quay.io/cephcsi/cephcsi:canary
          image: quay.io/cephcsi/cephcsi:canary

sed -i s#registry.k8s.io/sig-storage#registry.cn-hangzhou.aliyuncs.com/image-storage#g *.yaml
sed -i s#quay.io/cephcsi#registry.cn-hangzhou.aliyuncs.com/image-storage#g *.yaml
```

配置csi-config-map.yaml文件链接ceph集群的信息

```shell
cat > csi-config-map.yaml <<EOF
---
apiVersion: v1
kind: ConfigMap
data:
  config.json: |-
    [
      {
        "clusterID": "1ed6d2bd-52f1-4a95-bc36-2753f9892589",
        "monitors": [
          "192.168.80.55:6789",
          "192.168.80.66:6789",
          "192.168.80.77:6789"
        ]
      }
    ]
metadata:
  name: ceph-csi-config
EOF
kubectl apply -f csi-config-map.yaml
```

## 5.3.部署cephfs相关的CSI

```shell
cd /root/ceph/ceph-csi/deploy/cephfs/kubernetes/
kubectl apply -f .

# 查看pod情况
kubectl get pods | grep cephfs
csi-cephfsplugin-6dnlt                          3/3     Running   0                39m
csi-cephfsplugin-6nqdj                          3/3     Running   0                39m
csi-cephfsplugin-n8lff                          3/3     Running   0                39m
csi-cephfsplugin-provisioner-7d5b787d54-2pwx6   5/5     Running   0                39m
csi-cephfsplugin-provisioner-7d5b787d54-k5fhh   5/5     Running   0                39m
csi-cephfsplugin-provisioner-7d5b787d54-mcdf4   5/5     Running   0                39m
```

## 5.4.创建存储池

查看集群状态

```shell
ceph -s
  cluster:
    id:     1ed6d2bd-52f1-4a95-bc36-2753f9892589
    health: HEALTH_OK

  services:
    mon: 3 daemons, quorum master1-admin,node1-monitor,node2-osd (age 2h)
    mgr: master1-admin(active, since 3h), standbys: node1-monitor, node2-osd
    mds: 1/1 daemons up, 2 standby
    osd: 3 osds: 3 up (since 53m), 3 in (since 8d)

  data:
    volumes: 1/1 healthy
    pools:   5 pools, 113 pgs
    objects: 51 objects, 8.0 MiB
    usage:   83 MiB used, 300 GiB / 300 GiB avail
    pgs:     113 active+clean
```

创建cephfs存储池fs_metadata，fs_data：

```shell
ceph osd pool create fs_metadata 128 128
pool 'fs_metadata' created

ceph osd pool create fs_data 128 128
pool 'fs_data' created

ceph fs new cephfs fs_metadata fs_data
new fs with metadata pool 7 and data pool 8

ceph osd lspools
```

获取集群信息和查看用户key

```shell
ceph mon dump
epoch 1
fsid 1ed6d2bd-52f1-4a95-bc36-2753f9892589
last_changed 2023-06-22T21:46:24.547734+0800
created 2023-06-22T21:46:24.547734+0800
min_mon_release 17 (quincy)
election_strategy: 1
0: [v2:192.168.80.55:3300/0,v1:192.168.80.55:6789/0] mon.master1-admin
1: [v2:192.168.80.66:3300/0,v1:192.168.80.66:6789/0] mon.node1-monitor
2: [v2:192.168.80.77:3300/0,v1:192.168.80.77:6789/0] mon.node2-osd
dumped monmap epoch 1

ceph auth get client.admin
[client.admin]
        key = AQDJUJRkE8ilEBAA8MowiFQhpuIO48HaV5kYlg==
        caps mds = "allow *"
        caps mgr = "allow *"
        caps mon = "allow *"
        caps osd = "allow *"
exported keyring for client.admin
```

## 5.5.验证

### 5.5.1创建连接ceph集群的秘钥

```shell
cat > cephfs-secret.yaml <<EOF
---
apiVersion: v1
kind: Secret
metadata:
  name: csi-cephfs-secret
  namespace: default
stringData:
  # Required for statically provisioned volumes
  userID: admin
  userKey: AQDJUJRkE8ilEBAA8MowiFQhpuIO48HaV5kYlg==

  # Required for dynamically provisioned volumes
  adminID: admin
  adminKey: AQDJUJRkE8ilEBAA8MowiFQhpuIO48HaV5kYlg==	
EOF
kubectl apply -f cephfs-secret.yaml
```

### 5.5.2 创建storeclass

```shell
cat > cephfs-storageclass.yaml <<EOF
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: csi-cephfs-sc
provisioner: cephfs.csi.ceph.com
parameters:
  clusterID: 1ed6d2bd-52f1-4a95-bc36-2753f9892589
  fsName: cephfs
  pool: fs_data
  rootPath: /ceph
  csi.storage.k8s.io/provisioner-secret-name: csi-cephfs-secret
  csi.storage.k8s.io/provisioner-secret-namespace: default
  csi.storage.k8s.io/controller-expand-secret-name: csi-cephfs-secret
  csi.storage.k8s.io/controller-expand-secret-namespace: default
  csi.storage.k8s.io/node-stage-secret-name: csi-cephfs-secret
  csi.storage.k8s.io/node-stage-secret-namespace: default
reclaimPolicy: Delete
allowVolumeExpansion: true
#mountOptions:
  #- discard
EOF
kubectl apply -f cephfs-storageclass.yaml
```

### 5.5.3 基于storeclass创建pvc

```shell
cat > csi-cephfs-pvc.yaml <<EOF
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: csi-cephfs-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  storageClassName: csi-cephfs-sc
EOF
```

创建pvc：

```shell
kubectl apply -f csi-cephfs-pvc.yaml
kubectl get pvc csi-cephfs-pvc
NAME             STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS    AGE
csi-cephfs-pvc   Bound    pvc-187aad99-c5a0-40de-80fe-7df93258983d   1Gi        RWX            csi-cephfs-sc   4s
```

### 5.5.4 创建pod应用pvc

```shell
cat > cephfs-pod.yaml <<EOF
---
apiVersion: v1
kind: Pod
metadata:
  name: csi-cephfs-demo-pod
spec:
  containers:
    - name: web-server
      image: nginx
      volumeMounts:
        - name: mypvc
          mountPath: /var/lib/www
  volumes:
    - name: mypvc
      persistentVolumeClaim:
        claimName: csi-cephfs-pvc
        readOnly: false
EOF
```

创建测试pod：

```shell
kubectl apply -f cephfs-pod.yaml
kubectl get pods csi-cephfs-demo-pod

kubectl exec -ti csi-cephfs-demo-pod -- bash
df -h
Filesystem                                                                                                                                               Size  Used Avail Use% Mounted on
overlay                                                                                                                                                  196G   34G  152G  19% /
tmpfs                                                                                                                                                     64M     0   64M   0% /dev
/dev/sdb3                                                                                                                                                196G   34G  152G  19% /etc/hosts
shm                                                                                                                                                       64M     0   64M   0% /dev/shm
192.168.80.55:6789,192.168.80.66:6789,192.168.80.77:6789:/volumes/csi/csi-vol-e944f84d-8de4-4e67-bb2f-6350feea6bcb/824971d2-d2f6-41cf-b20f-5d05539c1e98  1.0G     0  1.0G   0% /var/lib/www
tmpfs                                                                                                                                                    5.7G   12K  5.7G   1% /run/secrets/kubernetes.io/serviceaccount
tmpfs                                                                                                                                                    2.9G     0  2.9G   0% /proc/acpi
tmpfs                                                                                                                                                    2.9G     0  2.9G   0% /proc/scsi
tmpfs                                                                                                                                                    2.9G     0  2.9G   0% /sys/firmware
```

**注：<font color=red>处理pod挂载不上问题：</font>https://github.com/ceph/ceph-csi/issues/3927，<font color=red>sc内容注释掉mountOptions部分。</font>**



参考：http://docs.ceph.com/en/quincy/install/get-packages

cephadm 安装 ceph 集群：https://xwls.github.io/2023/02/16/ceph-cluster-install