前言：本地使用nfs服务作为挂载

动态挂载文档：https://kubernetes.io/zh-cn/docs/concepts/storage/storage-classes/#nfs

GitHub：https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner

| hostname     | ip address    | service    |
| ------------ | ------------- | ---------- |
| k8s-master01 | 192.168.80.45 | nfs 服务端 |
| k8s-node01   | 192.168.80.46 | nfs 客户端 |
| k8s-node02   | 192.168.80.47 | nfs 客户端 |

- 挂载目录：/nfs

# 1. nfs服务安装

- k8s-master01操作

```bash
apt install nfs-kernel-server nfs-common -y
mkdir /nfs # 挂在目录
echo "/nfs *(rw,sync,no_root_squash,no_subtree_check)" >> /etc/exports

#重启
exportfs -a
systemctl restart nfs-kernel-server
systemctl enable nfs-kernel-server
---
```

解释：

```bash
各字段解析如下：
/nfs: 要共享的目录
：指定可以访问共享目录的用户 ip, * 代表所有用户。192.168.3.　指定网段。192.168.3.29 指定 ip。
rw：可读可写。如果想要只读的话，可以指定 ro。
sync：文件同步写入到内存与硬盘中。
async：文件会先暂存于内存中，而非直接写入硬盘。
no_root_squash：登入 nfs 主机使用分享目录的使用者，如果是 root 的话，那么对于这个分享的目录来说，他就具有 root 的权限！这个项目『极不安全』，不建议使用！但如果你需要在客户端对 nfs 目录进行写入操作。你就得配置 no_root_squash。方便与安全不可兼得。
root_squash：在登入 nfs 主机使用分享之目录的使用者如果是 root 时，那么这个使用者的权限将被压缩成为匿名使用者，通常他的 UID 与 GID 都会变成 nobody 那个系统账号的身份。
subtree_check：强制 nfs 检查父目录的权限（默认）
no_subtree_check：不检查父目录权限 作者：孤风孤影 https://www.bilibili.com/read/cv14041453/ 出处：bilibili
```

- k8s-node01、k8s-node02挂载

```bash
apt install nfs-common -y
mkdir -p /nfs/
mount -t nfs 192.168.80.45:/nfs/ /nfs/
df -h

# 设置启动自动挂载
echo "192.168.80.45:/nfs /nfs         nfs     defaults                 0       0"  >> /etc/fstab
```

centos安装方式：

```bash
yum install nfs-utils -y
systemctl start nfs
systemctl enable nfs.service
```

# 2. 创建存储

文件参考：[文件内容地址](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner/tree/master/deploy)

```bash
mkdir -p ~/nfs-storageclass && cd ~/nfs-storageclass
cat > nfs-storage.yaml << EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-storage
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: k8s-sigs.io/nfs-subdir-external-provisioner
parameters:
  archiveOnDelete: "false"  ## 删除pv的时候，pv的内容是否要备份


---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-client-provisioner
  labels:
    app: nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: default
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: nfs-client-provisioner
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          #image: k8s.gcr.io/sig-storage/nfs-subdir-external-provisioner:v4.0.2
          image: registry.cn-hangzhou.aliyuncs.com/image-storage/nfs-subdir-external-provisioner:v4.0.2
          # resources:
          #    limits:
          #      cpu: 10m
          #    requests:
          #      cpu: 10m
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: k8s-sigs.io/nfs-subdir-external-provisioner
            - name: NFS_SERVER
              value: 192.168.80.45 ## 指定自己nfs服务器地址
            - name: NFS_PATH
              value: /nfs/  ## nfs服务器共享的目录
      volumes:
        - name: nfs-client-root
          nfs:
            server: 192.168.80.45 #注意这里的地址
            path: /nfs/
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: default
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-client-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    # replace with namespace where provisioner is deployed
    namespace: default
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: default
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: default
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    # replace with namespace where provisioner is deployed
    namespace: default
roleRef:
  kind: Role
  name: leader-locking-nfs-client-provisioner
  apiGroup: rbac.authorization.k8s.io
EOF
```

启动容器：

```bash
# 创建默认存储 
kubectl apply -f nfs-storage.yaml

# 查看创建情况
kubectl get storageclasses.storage.k8s.io
NAME                    PROVISIONER                                   RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
nfs-storage (default)   k8s-sigs.io/nfs-subdir-external-provisioner   Delete          Immediate           false                  33m
```

# 3. 创建pvc

测试pvc文件内容：

```bash
cat > pvc.yaml << EOF
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: nginx-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 200Mi
EOF
```

 创建、查看：

```bash
# 创建pvc
kubectl apply -f pvc.yaml

# 查看pv
kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM               STORAGECLASS   REASON   AGE
pvc-242d1210-fb9f-4c53-b56d-88a8081305f6   200Mi      RWX            Delete           Bound    default/nginx-pvc   nfs-storage             31m
```