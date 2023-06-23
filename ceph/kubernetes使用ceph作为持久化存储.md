# 1.资源规划

官方：https://ceph.com/en，https://docs.ceph.com/en/latest/start/intro

ceph 是一种开源的分布式的存储系统 包含以下几种存储类型： 块存储（rbd），对象存储(RADOS Fateway)，文件系统（cephfs）

介绍：本篇文件主要是在ubuntu 22.04本地去搭建一套ceph集群，后续使用storageclasses插件去控制ceph集群做持久化。

| 主机                                | 磁盘                           | ceph版本 |
| ----------------------------------- | ------------------------------ | -------- |
| 192.168.80.55（ceph-master1-admin） | 系统盘：200GB、数据裸盘：100GB | 17.2.5   |
| 192.168.80.66（ceph-node1-monitor） | 系统盘：200GB、数据裸盘：100GB | 17.2.5   |
| 192.168.80.77（ceph-node2-osd）     | 系统盘：200GB、数据裸盘：100GB | 17.2.5   |

这里已经加入新的数据盘：`/dev/sdb`

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







参考：https://docs.ceph.com/en/quincy/install/get-packages

cephadm 安装 ceph 集群：https://xwls.github.io/2023/02/16/ceph-cluster-install/