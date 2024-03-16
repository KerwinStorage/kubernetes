# 		 			探索在openebs中使用lvm做持久化

# 1.部署

官网：https://openebs.io

lvm项目地址：https://github.com/openebs/lvm-localpv

CNCF沙盒项目：https://www.cncf.io/sandbox-projects

## 1.1.先决条件

**验证是否配置了 iSCSI 服务**

参考：https://openebs.io/docs/user-guides/prerequisites

```shell
# Install iSCSI tools
sudo apt-get update
sudo apt-get install open-iscsi
sudo cat /etc/iscsi/initiatorname.iscsi
systemctl status iscsid
# 开机自启
sudo systemctl enable --now iscsid
```

## 1.2.本地创建vg

```shell
apt install lvm2 -y
lsblk
# 创建pv和vg
sudo pvcreate /dev/loop0
sudo vgcreate lvmvg /dev/loop0 
```

注意：这里根据自己需求看是否全部的node节点都需要使用lvm做本地存储，也可以选择部分节点，这里就是根据所有节点都需要去配置。

## 1.3.OpenEBS LVM安装

最新配置文件：https://openebs.github.io/charts/lvm-operator.yaml

```shell
kubectl apply -f https://openebs.github.io/charts/lvm-operator.yaml
```

注：因为镜像在国外，这里将如何获取的两种办法写在了下面，其他镜像可能是有些慢

**第一种：国外地址加速**

```shell
cat lvm-operator.yaml | grep image:
image: registry.k8s.io/sig-storage/csi-resizer:v1.8.0
image: registry.k8s.io/sig-storage/csi-snapshotter:v6.2.2
image: registry.k8s.io/sig-storage/snapshot-controller:v6.2.2
image: registry.k8s.io/sig-storage/csi-provisioner:v3.5.0
image: openebs/lvm-driver:1.3.0
image: registry.k8s.io/sig-storage/csi-node-driver-registrar:v2.8.0
image: openebs/lvm-driver:1.3.0
```

镜像pull不到本地请参考这个：https://github.com/anjia0532/gcr.io_mirror/issues/3341

**第二种：我自己的阿里镜像仓库**

```shell
registry.cn-hangzhou.aliyuncs.com/image-storage/csi-snapshotter:v6.2.2
registry.cn-hangzhou.aliyuncs.com/image-storage/snapshot-controller:v6.2.2
registry.cn-hangzhou.aliyuncs.com/image-storage/csi-resizer:v1.8.0
registry.cn-hangzhou.aliyuncs.com/image-storage/csi-node-driver-registrar:v2.8.0
registry.cn-hangzhou.aliyuncs.com/image-storage/csi-provisioner:v3.5.0
registry.cn-hangzhou.aliyuncs.com/image-storage/tests-fio:latest
```

查看pod启动状态：

```shell
kubectl get pods -n kube-system -l role=openebs-lvm
NAME                       READY   STATUS    RESTARTS      AGE
openebs-lvm-controller-0   5/5     Running   6 (28m ago)   23h
openebs-lvm-node-cnzfc     2/2     Running   0             23h
openebs-lvm-node-zr8fz     2/2     Running   0             23h
```

## 1.4.使用lvm作为持久化

### 1.4.1.创建storage

默认模版：

```sh
cat > sc.yaml <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: openebs-lvmpv
parameters:
  storage: "lvm"
  volgroup: "lvmvg"
provisioner: local.csi.openebs.io
EOF
```

选择节点去使用lvm：

```shell
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: openebs-lvmpv
allowVolumeExpansion: true
parameters:
  storage: "lvm"
  volgroup: "lvmvg"
provisioner: local.csi.openebs.io
allowedTopologies:
- matchLabelExpressions:
  - key: kubernetes.io/hostname
    values:
      - lvmpv-node1
      - lvmpv-node2
```

### 1.4.2.创建pvc

```shell
cat > pvc.yaml <<EOF
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: csi-lvmpv
spec:
  storageClassName: openebs-lvmpv
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 4Gi
EOF
```

### 1.4.3.部署应用程序

```shell
cat > fio.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: fio
spec:
  restartPolicy: Never
  containers:
  - name: perfrunner
    image: openebs/tests-fio
    command: ["/bin/bash"]
    args: ["-c", "while true ;do sleep 50; done"]
    volumeMounts:
       - mountPath: /datadir
         name: fio-vol
    tty: true
  volumes:
  - name: fio-vol
    persistentVolumeClaim:
      claimName: csi-lvmpv
EOF
```

查看pod和挂载状态：

```shell
$ kubectl get pod
NAME                                     READY   STATUS    RESTARTS        AGE
fio                                      1/1     Running   0               22h

$ kubectl get pvc
NAME                             STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS    AGE
csi-lvmpv                        Bound    pvc-8765653e-2767-4fc1-a4de-aea96f538721   4Gi        RWO            openebs-lvmpv   22h

$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS     CLAIM
pvc-8765653e-2767-4fc1-a4de-aea96f538721   4Gi        RWO            Delete           Bound      default/csi-lvmpv                        openebs-lvmpv            22h

$ lvs
  LV                                       VG            Attr       LSize Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  pvc-8765653e-2767-4fc1-a4de-aea96f538721 carina-vg-ssd -wi-ao---- 4.00g

$ vgs
  VG            #PV #LV #SN Attr   VSize    VFree
  carina-vg-ssd   1   1   0 wz--n- <200.00g <196.00g

$ pvs
  PV         VG            Fmt  Attr PSize    PFree
  /dev/sdb   carina-vg-ssd lvm2 a--  <200.00g <196.00g
```

