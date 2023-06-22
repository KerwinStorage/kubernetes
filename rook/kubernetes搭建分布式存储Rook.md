# 1.介绍🙈

> 部署rook时会非常消耗内存资源，资源方面建议内存至少4G，核数不能低于2个如果有条件尽量往上调整，这里添加了磁盘sdb为50G大小用于实验，rook版本为v1.6.3。

Rook官网：[https://rook.io](https://rook.io/)

Github地址：https://github.com/rook/rook

# 2.Rook部署🙉

资源分配情况表：

| 主机         | 内存 | 磁盘     | 核数 |
| ------------ | ---- | -------- | ---- |
| k8s-master01 | 4    | sdb：50G | 2    |
| k8s-master03 | 4    | sdb：50G | 2    |
| k8s-master03 | 4    | sdb：50G | 2    |
| k8s-node01   | 6    | sdb：50G | 2    |
| k8s-node02   | 6    | sdb：50G | 2    |

## 2.1.部署部分

```bash
# 1.下载
wget https://github.com/rook/rook/archive/refs/tags/v1.6.3.tar.gz
tar zxf v1.6.3.tar.gz
cd rook-1.6.3/cluster/examples/kubernetes/ceph

# 2.处理镜像下载
# 方式一：直接修改文件镜像指到aliyun上面
vim operator.yaml

  # ROOK_CSI_REGISTRAR_IMAGE: "k8s.gcr.io/sig-storage/csi-node-driver-registrar:v2.0.1"
  # ROOK_CSI_RESIZER_IMAGE: "k8s.gcr.io/sig-storage/csi-resizer:v1.0.1"
  # ROOK_CSI_PROVISIONER_IMAGE: "k8s.gcr.io/sig-storage/csi-provisioner:v2.0.4"
  # ROOK_CSI_SNAPSHOTTER_IMAGE: "k8s.gcr.io/sig-storage/csi-snapshotter:v4.0.0"
  # ROOK_CSI_ATTACHER_IMAGE: "k8s.gcr.io/sig-storage/csi-attacher:v3.0.2"
  # 替换成
  ROOK_CSI_REGISTRAR_IMAGE: "registry.cn-beijing.aliyuncs.com/dotbalo/csi-node-driver-registrar:v2.0.1"
  ROOK_CSI_RESIZER_IMAGE: "registry.cn-beijing.aliyuncs.com/dotbalo/csi-resizer:v1.0.1"
  ROOK_CSI_PROVISIONER_IMAGE: "registry.cn-beijing.aliyuncs.com/dotbalo/csi-provisioner:v2.0.4"
  ROOK_CSI_SNAPSHOTTER_IMAGE: "registry.cn-beijing.aliyuncs.com/dotbalo/csi-snapshotter:v4.0.0"
  ROOK_CSI_ATTACHER_IMAGE: "registry.cn-beijing.aliyuncs.com/dotbalo/csi-attacher:v3.0.2"

# 方式二：直接下载镜像并打tag，文件不修改
docker pull registry.aliyuncs.com/google_containers/csi-node-driver-registrar:v2.0.1
docker pull registry.aliyuncs.com/google_containers/csi-resizer:v1.0.1
docker pull registry.aliyuncs.com/google_containers/csi-provisioner:v2.0.4
docker pull registry.aliyuncs.com/google_containers/csi-snapshotter:v4.0.0
docker pull registry.aliyuncs.com/google_containers/csi-attacher:v3.0.2


docker tag registry.aliyuncs.com/google_containers/csi-node-driver-registrar:v2.0.1 k8s.gcr.io/sig-storage/csi-node-driver-registrar:v2.0.1
docker tag registry.aliyuncs.com/google_containers/csi-resizer:v1.0.1 k8s.gcr.io/sig-storage/csi-resizer:v1.0.1
docker tag registry.aliyuncs.com/google_containers/csi-provisioner:v2.0.4 k8s.gcr.io/sig-storage/csi-provisioner:v2.0.4
docker tag registry.aliyuncs.com/google_containers/csi-snapshotter:v4.0.0 k8s.gcr.io/sig-storage/csi-snapshotter:v4.0.0
docker tag registry.aliyuncs.com/google_containers/csi-attacher:v3.0.2 k8s.gcr.io/sig-storage/csi-attacher:v3.0.2


# 3.部署

#还需要修改operator文件，新版本rook默认关闭了自动发现容器的部署，可以找到ROOK_ENABLE_DISCOVERY_DAEMON改成true即可   
ROOK_ENABLE_DISCOVERY_DAEMON: "true"

kubectl create -f crds.yaml -f common.yaml -f operator.yaml
kubectl -n rook-ceph get pods
NAME                                                     READY   STATUS      RESTARTS   AGE
rook-ceph-operator-675f59664d-b9nch                      1/1     Running     0          32m
rook-discover-4m68r                                      1/1     Running     0          40m
rook-discover-chscc                                      1/1     Running     0          40m
rook-discover-mmk69                                      1/1     Running     0          40m


# 等上面容器启动完在下执行下面这条命令
# 配置cluster.yaml
nodes:
  - name: "k8s-node01"
    devices:
      - name: "sdb"
  - name: "k8s-node2"
    devices:
     - name: "sdb"
  - name: "k8s-node03"
    devices:
      - name: "sdb"
# 因为资源有限mon的三个pod拉不起来，这里我们改成1个
  mon:
    count: 3

# 6.部署Ceph集群
kubectl create -f cluster.yaml

# 查看pod启动情况
kubectl -n rook-ceph get pods
NAME                                                     READY   STATUS      RESTARTS      AGE
csi-cephfsplugin-4mrdn                                   3/3     Running     0             38m
csi-cephfsplugin-cbsx5                                   3/3     Running     0             38m
csi-cephfsplugin-g52xd                                   3/3     Running     0             38m
csi-cephfsplugin-jmhx8                                   3/3     Running     0             38m
csi-cephfsplugin-provisioner-77c7f8f674-blftx            6/6     Running     0             38m
csi-cephfsplugin-provisioner-77c7f8f674-hh62m            6/6     Running     0             38m
csi-cephfsplugin-vt6kg                                   3/3     Running     0             38m
csi-rbdplugin-5jrd5                                      3/3     Running     0             39m
csi-rbdplugin-7qrh6                                      3/3     Running     0             39m
csi-rbdplugin-kjzg8                                      3/3     Running     0             39m
csi-rbdplugin-l9475                                      3/3     Running     0             39m
csi-rbdplugin-provisioner-5b78cf5f59-m7rwc               6/6     Running     0             39m
csi-rbdplugin-provisioner-5b78cf5f59-z4drt               6/6     Running     0             39m
csi-rbdplugin-pzmj2                                      3/3     Running     0             39m
rook-ceph-crashcollector-k8s-master01-5bcf84d7cc-8drhh   1/1     Running     0             32m
rook-ceph-crashcollector-k8s-master02-5c988b7f8-kthxz    1/1     Running     0             35m
rook-ceph-crashcollector-k8s-master03-796c459867-bn4df   1/1     Running     0             33m
rook-ceph-crashcollector-k8s-node01-6d5b45bf75-pnvtd     1/1     Running     0             33m
rook-ceph-crashcollector-k8s-node02-56c795b5b-7bstp      1/1     Running     0             34m
rook-ceph-mgr-a-6989b96648-rpwpx                         1/1     Running     2 (29m ago)   35m
rook-ceph-mon-a-7669dfd79f-b5dn6                         1/1     Running     6 (15m ago)   37m
rook-ceph-mon-b-cc86f6fb-r5zzt                           1/1     Running     3 (16m ago)   36m
rook-ceph-mon-c-784b67954d-kjtq9                         1/1     Running     1 (33m ago)   36m
rook-ceph-operator-76948f86f7-f8vr2                      1/1     Running     0             48m
rook-ceph-osd-0-5c6b459677-4jcqd                         1/1     Running     0             33m
rook-ceph-osd-1-6b5458d9b7-c7595                         1/1     Running     2 (16m ago)   33m
rook-ceph-osd-2-57d794889f-ls69b                         1/1     Running     6 (13m ago)   33m
rook-ceph-osd-3-866bf56d58-gzhg2                         1/1     Running     2 (13m ago)   33m
rook-ceph-osd-4-7d96fcf7d6-ln5dm                         1/1     Running     1 (15m ago)   32m
rook-ceph-osd-prepare-k8s-master01-2brd6                 0/1     Completed   0             31m
rook-ceph-osd-prepare-k8s-master02-mtczp                 0/1     Completed   0             31m
rook-ceph-osd-prepare-k8s-master03-vd8qg                 0/1     Completed   0             31m
rook-ceph-osd-prepare-k8s-node01-bnglb                   0/1     Completed   0             31m
rook-ceph-osd-prepare-k8s-node02-ncvww                   0/1     Completed   0             31m
rook-ceph-tools-897d6797f-vdzqf                          1/1     Running     0             29m
rook-discover-2gqcb                                      1/1     Running     0             46m
rook-discover-67r4w                                      1/1     Running     0             46m
rook-discover-7xsvk                                      1/1     Running     0             46m
rook-discover-9szts                                      1/1     Running     0             46m
rook-discover-v89t6                                      1/1     Running     0             46m
```

## 2.2.rook-ceph客户端验证

```bash
# 1.sdb已经被ceph接管
lsblk -f
sda                                                                             
├─sda1 vfat                 84D2-8BF5                               511M     0% /boot/efi
├─sda2                                                                          
└─sda5 ext4                 d68b64c3-e307-49cc-98fb-7cf632a021ba     73G    20% /
sdb    ceph_bluestore

# 2.toolbox 命令行工具
kubectl create -f toolbox.yaml
kubectl exec -it $(kubectl -n rook-ceph get pod -l "app=rook-ceph-tools" -o jsonpath='{.items[0].metadata.name}') -n rook-ceph -- bash
#查看ceph状态
ceph status
  cluster:
    id:     3d3ccaed-487f-49ee-a7b6-74eaf4fe704b
    health: HEALTH_OK

  services:
    mon: 3 daemons, quorum a,b,c (age 10m)
    mgr: a(active, since 10m)
    osd: 5 osds: 5 up (since 94s), 5 in (since 13m)

  data:
    pools:   1 pools, 1 pgs
    objects: 0 objects, 0 B
    usage:   5.0 GiB used, 245 GiB / 250 GiB avail
    pgs:     1 active+clean

#查看osd状态
ceph osd status
ID  HOST           USED  AVAIL  WR OPS  WR DATA  RD OPS  RD DATA  STATE
 0  k8s-node02    1030M  48.9G      0        0       0        0   exists,up
 1  k8s-node01    1030M  48.9G      0        0       0        0   exists,up
 2  k8s-master02  1030M  48.9G      0        0       0        0   exists,up
 3  k8s-master03  1030M  48.9G      0        0       0        0   exists,up
 4  k8s-master01  1030M  48.9G      0        0       0        0   exists,up

# 查看ceph配置文件
cat /etc/ceph/ceph.conf
[global]
mon_host = 10.98.145.93:6789,10.104.33.104:6789,10.98.146.16:6789

[client.admin]
keyring = /etc/ceph/keyring
```

常用命令：

```bash
ceph status
ceph osd status
ceph df 
rados df
```

## 2.3.配置ceph dashboard

```bash
# 创建dashboard
kubectl apply -f dashboard-external-https.yaml

# 查看svc
kubectl get svc -n rook-ceph|grep dashboard
rook-ceph-mgr-dashboard                  ClusterIP   10.254.253.104   <none>        7000/TCP            53s
rook-ceph-mgr-dashboard-external-https   NodePort    10.254.241.157   <none>        8443:30857/TCP      12h

# 登录，这里我们使用node节点ip加端口
https://192.168.80.45:30802/#/login?returnUrl=%2Fdashboard
# 用户admin，密码下面命令结果
kubectl -n rook-ceph get secret rook-ceph-dashboard-password -o jsonpath="{['data']['password']}"|base64 --decode && echo
```

页面：

![img](https://img2023.cnblogs.com/blog/1740081/202306/1740081-20230609214644590-1767150554.png)

## 2.4.处理dashboard警告

页面显示HEALTH_WARN告警

进入 ceph-tools 执行以下命令：

```bash
kubectl exec -it $(kubectl -n rook-ceph get pod -l "app=rook-ceph-tools" -o jsonpath='{.items[0].metadata.name}') -n rook-ceph -- bash
ceph config set mon auth_allow_insecure_global_id_reclaim false
# 刷新一下界面
```

参考：https://zhuanlan.zhihu.com/p/387531212，https://blog.csdn.net/networken/article/details/85772418

# 3.使用ceph创建StorageClass

参考文档：https://rook.io/docs/rook/v1.6/ceph-filesystem.html

## 3.1.创建文件系统

链接：https://github.com/rook/rook/blob/v1.6.3/cluster/examples/kubernetes/ceph/filesystem.yaml

```bash
cat > filesystem.yaml <<EOF
apiVersion: ceph.rook.io/v1
kind: CephFilesystem
metadata:
  name: myfs
  namespace: rook-ceph
spec:
  metadataPool:
    replicated:
      size: 3
  dataPools:
    - replicated:
        size: 3
  preserveFilesystemOnDelete: true
  metadataServer:
    activeCount: 1
    activeStandby: true
EOF
kubectl create -f filesystem.yaml

# 或者直接执行源文件
kubectl create -f rook-1.6.3/cluster/examples/kubernetes/ceph/filesystem.yaml

# 查看pod执行情况
kubectl -n rook-ceph get pod -l app=rook-ceph-mds
```

## 3.2.StorageClass资源的创建

链接：https://github.com/rook/rook/blob/v1.6.3/cluster/examples/kubernetes/ceph/csi/cephfs/storageclass.yaml

```bash
cat > storageclass.yaml <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-cephfs
# Change "rook-ceph" provisioner prefix to match the operator namespace if needed
provisioner: rook-ceph.cephfs.csi.ceph.com
parameters:
  # clusterID is the namespace where operator is deployed.
  clusterID: rook-ceph

  # CephFS filesystem name into which the volume shall be created
  fsName: myfs

  # Ceph pool into which the volume shall be created
  # Required for provisionVolume: "true"
  pool: myfs-data0

  # The secrets contain Ceph admin credentials. These are generated automatically by the operator
  # in the same namespace as the cluster.
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-cephfs-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/controller-expand-secret-name: rook-csi-cephfs-provisioner
  csi.storage.k8s.io/controller-expand-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-cephfs-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph

reclaimPolicy: Delete
EOF

kubectl create -f storageclass.yaml

# 源文件地址
kubectl create -f cluster/examples/kubernetes/ceph/csi/cephfs/storageclass.yaml

# 查看storageclass创建情况
kubectl get sc rook-cephfs
NAME          PROVISIONER                     RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
rook-cephfs   rook-ceph.cephfs.csi.ceph.com   Delete          Immediate           true                   10m
```

## 3.3.创建实例

链接：https://github.com/rook/rook/blob/v1.6.3/cluster/examples/kubernetes/ceph/csi/cephfs/kube-registry.yaml

```bash
cat > kube-registry.yaml <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cephfs-pvc
  namespace: kube-system
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  storageClassName: rook-cephfs
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kube-registry
  namespace: kube-system
  labels:
    k8s-app: kube-registry
    kubernetes.io/cluster-service: "true"
spec:
  replicas: 3
  selector:
    matchLabels:
      k8s-app: kube-registry
  template:
    metadata:
      labels:
        k8s-app: kube-registry
        kubernetes.io/cluster-service: "true"
    spec:
      containers:
      - name: registry
        image: registry:2
        imagePullPolicy: Always
        resources:
          limits:
            cpu: 100m
            memory: 100Mi
        env:
        # Configuration reference: https://docs.docker.com/registry/configuration/
        - name: REGISTRY_HTTP_ADDR
          value: :5000
        - name: REGISTRY_HTTP_SECRET
          value: "Ple4seCh4ngeThisN0tAVerySecretV4lue"
        - name: REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY
          value: /var/lib/registry
        volumeMounts:
        - name: image-store
          mountPath: /var/lib/registry
        ports:
        - containerPort: 5000
          name: registry
          protocol: TCP
        livenessProbe:
          httpGet:
            path: /
            port: registry
        readinessProbe:
          httpGet:
            path: /
            port: registry
      volumes:
      - name: image-store
        persistentVolumeClaim:
          claimName: cephfs-pvc
          readOnly: false
EOF

kubectl create -f kube-registry.yaml

# 或者执行源文件
kubectl create -f cluster/examples/kubernetes/ceph/csi/cephfs/kube-registry.yaml

# 查看pvc
kubectl get pvc -n kube-system
NAME         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
cephfs-pvc   Bound    pvc-22e7f04a-1dbe-4590-ab72-318111b776ac   1Gi        RWX            rook-cephfs    8m6s

# 查看pod启动
kubectl get pod -n kube-system  | grep registry
kube-registry-74d7b9999c-7tgph             1/1     Running   0               10m
kube-registry-74d7b9999c-vxhdj             1/1     Running   0               10m
kube-registry-74d7b9999c-z65jd             1/1     Running   0               10m

# 清理实例
kubectl delete -f kube-registry.yaml
```

 