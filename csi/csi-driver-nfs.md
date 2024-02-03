# 													csi-driver-nfs持久化

# 1.简介

csi-driver-nfs 是一个用于 Kubernetes 的 NFS CSI 驱动程序，它可以让 Kubernetes 访问 Linux 节点上的 NFS 服务器。它的 CSI 插件名称是 nfs.csi.k8s.io。这个驱动程序需要已经存在并配置好的 NFSv3 或 NFSv4 服务器，它支持通过创建 NFS 服务器下的新子目录来动态分配持久卷（Persistent Volumes）。这个驱动程序的项目状态是 GA（正式发布）。

这个驱动程序的主要功能和特点有：

- 支持 NFSv3 和 NFSv4 协议
- 支持快照（Snapshot）和卷克隆（Volume cloning）
- 支持 fsGroupPolicy，可以在 Pod 中设置 fsGroup
- 支持多种安装方式，包括 helm charts 和 kubectl
- 支持多种参数设置，包括 mountOptions，server，share，subPath，readOnly 等
- 支持 Kubernetes 1.21+ 版本

# 2.部署nfs

```shell
apt install nfs-kernel-server nfs-common -y
mkdir -p /data
echo "/data *(rw,sync,no_root_squash,no_subtree_check)" >> /etc/exports

#重启
exportfs -a
systemctl restart nfs-kernel-server.service
systemctl enable nfs-kernel-server.service
```

容器部署：[链接](https://github.com/kubernetes-csi/csi-driver-nfs/blob/master/deploy/example/nfs-provisioner/nfs-server.yaml)

# 3.安装csi驱动

下载包地址：[链接](https://github.com/kubernetes-csi/csi-driver-nfs/tags)

部署参考链接：[链接](https://github.com/kubernetes-csi/csi-driver-nfs/blob/master/docs/install-csi-driver-v4.6.0.md)

```shell
wget https://github.com/kubernetes-csi/csi-driver-nfs/archive/refs/tags/v4.6.0.tar.gz
tar zxvf v4.6.0.tar.gz && cd csi-driver-nfs-4.6.0

# 查看镜像
cd csi-driver-nfs-4.6.0/deploy/v4.6.0

# 创建资源
./deploy/install-driver.sh v4.6.0 local

# 查看启动状态
kubectl -n kube-system get pod -o wide -l app=csi-nfs-controller
kubectl -n kube-system get pod -o wide -l app=csi-nfs-node
```

镜像列表：

```shell
# 国外镜像
grep -nr "registry.k8s.io" | awk '{print $3}'
registry.k8s.io/sig-storage/nfsplugin:v4.6.0
registry.k8s.io/sig-storage/livenessprobe:v2.11.0
registry.k8s.io/sig-storage/csi-node-driver-registrar:v2.9.1
registry.k8s.io/sig-storage/snapshot-controller:v6.3.2
registry.k8s.io/sig-storage/csi-provisioner:v3.6.2
registry.k8s.io/sig-storage/csi-snapshotter:v6.3.2

# 自己推送的镜像
registry.cn-hangzhou.aliyuncs.com/image-storage/nfspluginnfsplugin:v4.6.0
registry.cn-hangzhou.aliyuncs.com/image-storage/livenessprobe:v2.11.0
registry.cn-hangzhou.aliyuncs.com/image-storage/csi-node-driver-registrar:v2.9.1
registry.cn-hangzhou.aliyuncs.com/image-storage/snapshot-controller:v6.3.2
registry.cn-hangzhou.aliyuncs.com/image-storage/csi-provisioner:v3.6.2
registry.cn-hangzhou.aliyuncs.com/image-storage/csi-snapshotter:v6.3.2
```

# 4.storageclass创建

```shell
cd csi-driver-nfs-4.6.0/deploy/example

# storageclass文件内容
cat > storageclass-nfs.yaml <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-csi
  annotations:
    # 此操作是1.25的以上的一个alpha的新功能，是将此storageclass设置为默认，这个在前面文章有讲过
    storageclass.kubernetes.io/is-default-class: "true"
# 此处指定了csidrivers的名称
provisioner: nfs.csi.k8s.io
parameters:
  # NFS的Server
  server: 192.168.80.45
  # NFS的存储路径
  share: /data
  # csi.storage.k8s.io/provisioner-secret is only needed for providing mountOptions in DeleteVolume
  # csi.storage.k8s.io/provisioner-secret-name: "mount-options"
  # csi.storage.k8s.io/provisioner-secret-namespace: "default"
reclaimPolicy: Delete
volumeBindingMode: Immediate
mountOptions:
  # 这里不只可以配置nfs的版本
  - nfsvers=4.1
EOF
```

|      参数名称      |                             意义                             |            示例            | 强制参数 |    默认值    |
| :----------------: | :----------------------------------------------------------: | :------------------------: | :------: | :----------: |
|      `server`      |                        NFS的服务地址                         |         10.0.0.11          |   YES    |              |
|      `share`       |                        NFS共享的路径                         |             /              |   YES    |              |
|      `subDir`      |                      NFS 共享下的子目录                      |                            |    NO    |              |
| `mountPermissions` | 挂载的文件夹权限。默认值为，如果设置为非零，驱动程序将在挂载后执行`0` `chmod` |                            |    NO    | 不存在则创建 |
|     `onDelete`     |                 删除卷时，如果目录是`retain`                 | delete`（默认值），`retain |    NO    |    delete    |

创建资源：

```shell
kubectl apply -f storageclass-nfs.yaml
# 查看sc
kubectl get sc |  grep nfs
nfs (default)           nfs.csi.k8s.io         Delete          Immediate              false                  29s
```

# 5.测试部分

文件内容：

```shell
cat > pvc-dynamic.yml <<EOF
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-nfs-dynamic
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  storageClassName: nfs-csi
EOF
```

创建资源：

```shell
kubectl apply -f pvc-dynamic.yml

kubectl get pvc pvc-nfs-dynamic
NAME              STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pvc-nfs-dynamic   Bound    pvc-e4c88c38-4f43-4813-94ab-110d248bcd84   1Gi        RWX            nfs-csi        37s
```

参考文章：

- [链接](https://www.cnblogs.com/layzer/articles/nfs-csi-use.html)