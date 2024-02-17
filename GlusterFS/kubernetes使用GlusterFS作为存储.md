# 			kubernetes对接kadalu使用GlusterFS作为存储

# 1.安装glusterfs

gluster官网：https://www.gluster.org

部署参考：https://cloud-atlas.readthedocs.io/zh-cn/latest/kubernetes/storage/k8s_gluster.html

前期需要准备3个节点作为glusterfs集群slave，并且每个节点至少需要个提供1块磁盘。

| 节点名称     | ip地址        | 磁盘     |
| ------------ | ------------- | -------- |
| k8s-master01 | 192.168.80.45 | /dev/sdc |
| k8s-node01   | 192.168.80.46 | /dev/sdc |
| k8s-node02   | 192.168.80.47 | /dev/sdc |

## 1.1.节点免密

```shell
apt install -y sshpass
ssh-keygen -f /root/.ssh/id_rsa -P ''
export IP="k8s-master01 k8s-node01 k8s-node02"
export SSHPASS=1qazZSE$
for HOST in $IP;do
     sshpass -e ssh-copy-id -o StrictHostKeyChecking=no $HOST
done
```

## 1.2.部署服务

```shell
# https://docs.gluster.org/en/v3/Install-Guide/Install/
apt install software-properties-common
sudo add-apt-repository ppa:gluster/glusterfs-9
sudo apt-get update

# 安装 glusterfs 组件
apt install glusterfs-server glusterfs-client glusterfs-common -y

## 创建 glusterfs 目录
mkdir -p /opt/glusterd

## 修改 glusterd 目录
sed -i 's/var\/lib/opt/g' /etc/glusterfs/glusterd.vol

# 启动 glusterfs
systemctl start glusterd.service

# 设置开机启动
systemctl enable glusterd.service

# 查看状态
systemctl status glusterd.service

# 检查版本
glusterfs -V
# glusterfs 10.1
```

## 1.3.加载内核模块

```shell
echo dm_thin_pool | sudo tee -a /etc/modules
echo dm_snapshot | sudo tee -a /etc/modules
echo dm_mirror | sudo tee -a /etc/modules
apt-get -y install thin-provisioning-tools
```

## 1.4.配置 glusterfs

```shell
# 加入主机解析
vim /etc/hosts
192.168.80.45 glusterfs01
192.168.80.46 glusterfs02
192.168.80.47 glusterfs03

# 创建存储目录
mkdir -p /opt/gfs_data

# 添加节点到集群（glusterfs01机器上执行，本机不需要加入集群操作）
gluster peer probe k8s-node01
gluster peer probe k8s-node02

# 查看集群状态
gluster peer status
Number of Peers: 2

Hostname: k8s-node01
Uuid: 261672d6-bb61-4ead-9de0-4405c365bd62
State: Peer in Cluster (Connected)

Hostname: k8s-node02
Uuid: 6d7754ed-6219-4402-93bb-c8128933fc24
State: Peer in Cluster (Connected)
```

## 1.5.添加磁盘

```shell
mkfs.ext4 /dev/sdc
mkdir -p /glusterfs_date
mount /dev/sdc /glusterfs_date
# 只用于演示，这里就不挂载了，以免后期重启系统失败。
# echo "/dev/sdc  /glusterfs_date  ext4  defaults 0 0" >> /etc/fstab
mount -a
```

## 1.6.创建卷

GlusterFS客户端常用命令：

| **命令**                           | **功能**   |
| :--------------------------------- | ---------- |
| gluster peer probe                 | 添加节点   |
| gluster peer detach                | 移除节点   |
| gluster volume create              | 创建卷     |
| gluster volume start $VOLUME_NAFME | 启动卷     |
| gluster volume stop $VOLUME_NAME   | 停止卷     |
| gluster volume delete $VOlUME_NAME | 删除卷     |
| gluster volume quota enable        | 开启卷配额 |
| gluster volume quota disable       | 关闭卷配额 |
| gluster volume quota limitusage    | 设定卷配额 |

关于卷的类型：[官网链接](https://docs.gluster.org/en/latest/Administrator-Guide/Setting-Up-Volumes/#arbiter-configuration-for-replica-volumes) 可以查看这个链接，本次创建卷只用于演示。后期我们会使用kadalu操作容器自动对本地磁盘进行持久化操作。

```shell
# 创建分布式卷，卷名是gv1
gluster volume create gv1 k8s-master01:/glusterfs_date k8s-node01:/glusterfs_date k8s-node02:/glusterfs_date force
volume create: gv1: success: please start the volume to access data

# 查看gv1卷的状态
gluster volume info gv1
Volume Name: gv1 # 卷名：gv1
Type: Distribute # 类型：分布式卷
Volume ID: 3a5ae61c-3716-42c3-81f6-03f3dea2ec39 # 卷的ID
Status: Created  # 状态：创建
Snapshot Count: 0 # 快照计数：0
Number of Bricks: 3 # 块的数量：3
Transport-type: tcp # 传输类型：tcp协议
Bricks: # 以下是哪些主机块的信息
Brick1: k8s-master01:/glusterfs_date 
Brick2: k8s-node01:/glusterfs_date
Brick3: k8s-node02:/glusterfs_date
Options Reconfigured: # 选项配置
storage.fips-mode-rchecksum: on
transport.address-family: inet
nfs.disable: on # nfs禁用：开启

# 启用卷
gluster volume start gv1
```

## 1.7.验证

```shell
mkdir -p /data/gv1
mount.glusterfs   k8s-node01:/gv1 /data/gv1

# 查看磁盘挂载情况
df -h
k8s-node01:/gv1  883G  8.9G  838G   2% /data/gv1

# 创建文件检验
cd  /data/gv1
touch {1..5}

# node主机查看
root@k8s-node01:/glusterfs_date# ls
1  5  lost+found
root@k8s-master01:/glusterfs_date# ls
2  3  4  lost+found
root@k8s-node02:/glusterfs_date# ls
lost+found
```

# 2.Kubernets对接kadalu

官网：https://kadalu.tech

Github：https://github.com/kadalu/kadalu

注：强烈建议在参考部署前看下这篇文档：https://thoughtexpo.com/exploring-kadalu-storage-in-k3d-cluster-glusterfs，因为后续部署都是参考这篇文档完成的。

## 2.1.前期准备

查看节点：

```shell
kubectl get  nodes
NAME           STATUS   ROLES                  AGE   VERSION
k8s-master01   Ready    control-plane,master   19d   v1.26.0
k8s-node01     Ready    <none>                 19d   v1.26.0
k8s-node02     Ready    <none>                 19d   v1.26.0
```

## 2.2.Kadalu 设置

### 2.2.1创建秘钥

在部署 kadalu 之前，我们将创建一个秘钥，以便在文章末尾与外部 gluster 一起使用。

```shell
kubectl create namespace kadalu

# 这里在前文我们已经将秘钥创建完成
kubectl create secret generic glusterquota-ssh-secret --from-literal=glusterquota-ssh-username=root --from-file=ssh-privatekey=/root/.ssh/id_rsa -n kadalu

kubectl config set-context --current --namespace=kadalu

kubectl get all

kubectl get csidrivers
```

### 2.2.2创建kadalu-operator

```shell
curl -s https://raw.githubusercontent.com/kadalu/kadalu/devel/manifests/kadalu-operator.yaml | sed 's/"no"/"yes"/' | kubectl apply -f -
```

文件内容：

```shell
---
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: kadalustorages.kadalu-operator.storage
spec:
  group: kadalu-operator.storage
  names:
    kind: KadaluStorage
    listKind: KadaluStorageList
    plural: kadalustorages
    singular: kadalustorage
    shortNames:
      - kadalu
      - kds
  scope: Namespaced
  versions:
    - name: v1alpha1
      storage: true
      served: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              x-kubernetes-preserve-unknown-fields: true
              type: object
              properties:
                disperse:
                  type: object
                  properties:
                    data:
                      type: integer
                    redundancy:
                      type: integer
                type:
                  type: string
                pvReclaimPolicy:
                  type: string
                  default: delete
                volume_id:
                  type: string
                kadalu_format:
                  type: string
                single_pv_per_pool:
                  type: boolean
                  default: false
                # Refer https://github.com/kubernetes-client/python/blob/da6076/kubernetes/docs/V1Toleration.md
                tolerations:
                  type: array
                  items:
                    type: object
                    properties:
                      effect:
                        type: string
                      key:
                        type: string
                      operator:
                        type: string
                      tolerationSeconds:
                        format: int64
                        type: integer
                      value:
                        type: string
                options:
                  type: array
                  items:
                    type: object
                    properties:
                      key:
                        type: string
                      value:
                        type: string
                storage:
                  type: array
                  items:
                    type: object
                    properties:
                      node:
                        type: string
                      device:
                        type: string
                      path:
                        type: string
                      pvc:
                        type: string
                      decommissioned:
                        type: string
                tiebreaker:
                  type: object
                  properties:
                    deployment:
                      type: string
                    node:
                      type: string
                    path:
                      type: string
                    port:
                      type: integer
                details:
                  type: object
                  properties:
                    gluster_host:
                      type: string
                    gluster_hosts:
                      type: array
                      items:
                        type: string
                    gluster_volname:
                      type: string
                    gluster_options:
                      type: string

---
kind: Namespace
apiVersion: v1
metadata:
  name: kadalu
---
# Source: kadalu/charts/operator/templates/serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kadalu-operator
  namespace: kadalu
---
# Source: kadalu/charts/operator/templates/serviceaccount.yaml
kind: ServiceAccount
apiVersion: v1
metadata:
  name: kadalu-csi-nodeplugin
  namespace: kadalu
---
# Source: kadalu/charts/operator/templates/serviceaccount.yaml
kind: ServiceAccount
apiVersion: v1
metadata:
  name: kadalu-csi-provisioner
  namespace: kadalu
---
# Source: kadalu/charts/operator/templates/serviceaccount.yaml
kind: ServiceAccount
apiVersion: v1
metadata:
  name: kadalu-server-sa
  namespace: kadalu
---
# Source: kadalu/charts/operator/templates/rbac.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: kadalu-operator
rules:
  - apiGroups:
      - storage.k8s.io
      - csi.storage.k8s.io
    resources:
      - csidrivers
      - storageclasses
    verbs:
      - create
      - delete
      - get
      - list
      - update
  - apiGroups:
      - kadalu-operator.storage
    resources:
      - kadalustorages
    verbs:
      - watch
  - apiGroups:
      - ""
    resources:
      - persistentvolumes
    verbs:
      - get
      - list
---
# Source: kadalu/charts/operator/templates/rbac.yaml
# CSI External Attacher
# https://github.com/kubernetes-csi/external-attacher/blob/master/deploy/kubernetes/rbac.yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: kadalu-csi-external-attacher
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "patch"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["csinodes"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["volumeattachments"]
    verbs: ["get", "list", "watch", "patch"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["volumeattachments/status"]
    verbs: ["patch"]
---
# Source: kadalu/charts/operator/templates/rbac.yaml
# CSI External Provisioner
# https://github.com/kubernetes-csi/external-provisioner/blob/master/deploy/kubernetes/rbac.yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: kadalu-csi-external-provisioner
rules:
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
    verbs: ["list", "watch", "create", "update", "patch"]
  - apiGroups: ["snapshot.storage.k8s.io"]
    resources: ["volumesnapshots"]
    verbs: ["get", "list"]
  - apiGroups: ["snapshot.storage.k8s.io"]
    resources: ["volumesnapshotcontents"]
    verbs: ["get", "list"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["csinodes"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list", "watch"]
  # PUBLISH_UNPUBLISH_VOLUME capability
  - apiGroups: ["storage.k8s.io"]
    resources: ["volumeattachments"]
    verbs: ["get", "list", "watch"]
---
# Source: kadalu/charts/operator/templates/rbac.yaml
# CSI External Resizer
# https://github.com/kubernetes-csi/external-resizer/blob/master/deploy/kubernetes/rbac.yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: kadalu-csi-external-resizer
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "patch"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims/status"]
    verbs: ["patch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["list", "watch", "create", "update", "patch"]
---
# Source: kadalu/charts/operator/templates/rbac.yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: kadalu-operator
subjects:
  - kind: ServiceAccount
    name: kadalu-operator
    namespace: kadalu
roleRef:
  kind: ClusterRole
  name: kadalu-operator
  apiGroup: rbac.authorization.k8s.io
---
# Source: kadalu/charts/operator/templates/rbac.yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: kadalu-csi-external-attacher
subjects:
  - kind: ServiceAccount
    name: kadalu-csi-provisioner
    namespace: kadalu
roleRef:
  kind: ClusterRole
  name: kadalu-csi-external-attacher
  apiGroup: rbac.authorization.k8s.io
---
# Source: kadalu/charts/operator/templates/rbac.yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: kadalu-csi-external-provisioner
subjects:
  - kind: ServiceAccount
    name: kadalu-csi-provisioner
    namespace: kadalu
roleRef:
  kind: ClusterRole
  name: kadalu-csi-external-provisioner
  apiGroup: rbac.authorization.k8s.io
---
# Source: kadalu/charts/operator/templates/rbac.yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: kadalu-csi-external-resizer
subjects:
  - kind: ServiceAccount
    name: kadalu-csi-provisioner
    namespace: kadalu
roleRef:
  kind: ClusterRole
  name: kadalu-csi-external-resizer
  apiGroup: rbac.authorization.k8s.io
---
# Source: kadalu/charts/operator/templates/rbac.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: kadalu-operator
  namespace: kadalu
rules:
  - apiGroups: [""]
    resources:
      - configmaps
      - persistentvolumes
      - pods
      - pods/exec
      - services
    verbs:
      - create
      - delete
      - get
      - list
      - patch
  - apiGroups:
      - ""
      - apiextensions.k8s.io
    resources:
      - customresourcedefinitions
    verbs:
      - create
      - update
  - apiGroups:
      - apps
    resources:
      - daemonsets
      - statefulsets
    verbs:
      - create
      - delete
      - get
      - patch
---
# Source: kadalu/charts/operator/templates/rbac.yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: kadalu-operator
  namespace: kadalu
subjects:
  - kind: ServiceAccount
    name: kadalu-operator
    namespace: kadalu
roleRef:
  kind: Role
  name: kadalu-operator
  apiGroup: rbac.authorization.k8s.io
---
# Source: kadalu/charts/operator/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: operator
  namespace: kadalu
  labels:
    app.kubernetes.io/part-of: kadalu
    app.kubernetes.io/name: kadalu-operator
    app.kubernetes.io/component: operator
spec:
  replicas: 1
  selector:
    matchLabels:
      name: kadalu
  template:
    metadata:
      labels:
        name: kadalu
        app.kubernetes.io/part-of: kadalu
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8050"
    spec:
      serviceAccountName: kadalu-operator
      containers:
        - name: kadalu-operator
          securityContext:
            capabilities: {}
            privileged: true
          image: docker.io/kadalu/kadalu-operator:devel
          imagePullPolicy: IfNotPresent
          env:
            - name: WATCH_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: OPERATOR_NAME
              value: "kadalu-operator"
            - name: IMAGES_HUB
              value: "docker.io"
            - name: DOCKER_USER
              value: "kadalu"
            - name: KADALU_VERSION
              value: "devel"
            - name: KADALU_NAMESPACE
              value: "kadalu"
            - name: KUBELET_DIR
              value: "/var/lib/kubelet"
            - name: K8S_DIST
              value: "kubernetes"
            - name: VERBOSE
              value: "no"
```

### 2.2.3csi-nodeplugin

nodeplugin 被部署为守护进程集，它将在每个节点上运行。

```shell
curl -s https://raw.githubusercontent.com/kadalu/kadalu/devel/manifests/csi-nodeplugin.yaml | sed 's/"no"/"yes"/' | kubectl apply -f -
```

文件内容：

```shell
---
# Source: kadalu/charts/csi-nodeplugin/templates/clusterrole.yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: kadalu-csi-nodeplugin
rules:
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
---
# Source: kadalu/charts/csi-nodeplugin/templates/clusterrolebinding.yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: kadalu-csi-nodeplugin
  namespace: kadalu
subjects:
  - kind: ServiceAccount
    name: kadalu-csi-nodeplugin
    namespace: kadalu
roleRef:
  kind: ClusterRole
  name: kadalu-csi-nodeplugin
  apiGroup: rbac.authorization.k8s.io
---
# Source: kadalu/charts/csi-nodeplugin/templates/daemonset.yaml
kind: DaemonSet
apiVersion: apps/v1
metadata:
  name: kadalu-csi-nodeplugin
  namespace: kadalu
  labels:
    app.kubernetes.io/part-of: kadalu
    app.kubernetes.io/name: kadalu-csi-nodeplugin
    app.kubernetes.io/component: csi-driver
spec:
  selector:
    matchLabels:
      app.kubernetes.io/part-of: kadalu
      app.kubernetes.io/name: kadalu-csi-nodeplugin
      app.kubernetes.io/component: csi-driver
  template:
    metadata:
      labels:
        app.kubernetes.io/part-of: kadalu
        app.kubernetes.io/name: kadalu-csi-nodeplugin
        app.kubernetes.io/component: csi-driver
      namespace: kadalu
    spec:
      serviceAccountName: kadalu-csi-nodeplugin
      containers:
        - name: csi-node-driver-registrar
          image: docker.io/raspbernetes/csi-node-driver-registrar:2.0.1
          args:
            - "--v=5"
            - "--csi-address=$(ADDRESS)"
            - "--kubelet-registration-path=$(DRIVER_REG_SOCK_PATH)"
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "rm -rf /registration/kadalu /registration/kadalu-reg.sock"]
          env:
            - name: ADDRESS
              value: /plugin/csi.sock
            - name: DRIVER_REG_SOCK_PATH
              value: /var/lib/kubelet/plugins/kadalu/csi.sock
            - name: KUBE_NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
          volumeMounts:
            - name: plugin-dir
              mountPath: /plugin
            - name: registration-dir
              mountPath: /registration
        - name: kadalu-nodeplugin
          securityContext:
            privileged: true
            capabilities:
              add: ["SYS_ADMIN"]
            allowPrivilegeEscalation: true
          image: docker.io/kadalu/kadalu-csi:devel
          env:
            - name: NODE_ID
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
            - name: CSI_ENDPOINT
              value: unix://plugin/csi.sock
            - name: KADALU_VERSION
              value: "devel"
            - name: K8S_DIST
              value: "kubernetes"
            - name: VERBOSE
              value: "no"
            - name: CSI_ROLE
              value: "nodeplugin"
          volumeMounts:
            - name: plugin-dir
              mountPath: /plugin
            - name: pods-mount-dir
              mountPath: /var/lib/kubelet/pods
              mountPropagation: "Bidirectional"
            - name: glusterfsd-volfilesdir
              mountPath: "/var/lib/gluster"
            - name: gluster-dev
              mountPath: "/dev"
            - name: varlog
              mountPath: /var/log/gluster
            - name: csi-dir
              mountPath: /var/lib/kubelet/plugins/kubernetes.io/csi
              mountPropagation: "Bidirectional"
        - name: kadalu-logging
          image: docker.io/library/busybox
          command: ["/bin/sh"]
          args: ["-c", "while true; do logcnt=$(/bin/ls /var/log/gluster/ | wc -l); if [ ${logcnt} -gt 0 ]; then break; fi; sleep 5; done; tail -F /var/log/gluster/*.log"]
          volumeMounts:
            - name: varlog
              mountPath: "/var/log/gluster"
      volumes:
        - name: plugin-dir
          hostPath:
            path: /var/lib/kubelet/plugins/kadalu
            type: DirectoryOrCreate
        - name: pods-mount-dir
          hostPath:
            path: /var/lib/kubelet/pods
            type: Directory
        - name: registration-dir
          hostPath:
            path: /var/lib/kubelet/plugins_registry/
            type: Directory
        - name: glusterfsd-volfilesdir
          configMap:
            name: "kadalu-info"
        - name: gluster-dev
          hostPath:
            path: "/dev"
        - name: varlog
          emptyDir: {}
        - name: csi-dir
          hostPath:
            path: /var/lib/kubelet/plugins/kubernetes.io/csi/
            type: DirectoryOrCreate
```

### 2.2.4查看资源部署情况

```shell
kubectl get csidriver
NAME                   ATTACHREQUIRED   PODINFOONMOUNT   STORAGECAPACITY   TOKENREQUESTS   REQUIRESREPUBLISH   MODES                  AGE
kadalu                 false            true             false             <unset>         false               Persistent             16h

kubectl describe cm kadalu-info
Name:         kadalu-info
Namespace:    kadalu
Labels:       <none>
Annotations:  <none>

Data
====
replica3.info:
----
{"namespace": "kadalu", "kadalu_version": "devel", "volname": "replica3", "volume_id": "d3c51400-cd5d-11ee-ad85-be1f20783003", "single_pv_per_pool": false, "type": "Replica3", "pvReclaimPolicy": "delete", "bricks": [{"brick_path": "/bricks/replica3/data/brick", "kube_hostname": "k8s-master01", "node": "server-replica3-0-0.replica3", "node_id": "node-0", "host_brick_path": "", "brick_device": "/dev/sdc", "pvc_name": "", "brick_device_dir": "", "decommissioned": "", "brick_index": 0}, {"brick_path": "/bricks/replica3/data/brick", "kube_hostname": "k8s-node01", "node": "server-replica3-1-0.replica3", "node_id": "node-1", "host_brick_path": "", "brick_device": "/dev/sdc", "pvc_name": "", "brick_device_dir": "", "decommissioned": "", "brick_index": 1}, {"brick_path": "/bricks/replica3/data/brick", "kube_hostname": "k8s-node02", "node": "server-replica3-2-0.replica3", "node_id": "node-2", "host_brick_path": "", "brick_device": "/dev/sdc", "pvc_name": "", "brick_device_dir": "", "decommissioned": "", "brick_index": 2}], "disperse": {"data": 0, "redundancy": 0}, "options": {}}
uid:
----
0cf6f175-9982-446f-b365-64ad28237c6f
volumes:
----


BinaryData
====

Events:  <none>

kubectl get sc
No resources found

kubectl get kds
No resources found in kadalu namespace.
```

### 2.2.5KadaluStorage

```shell
cat > storage-config-device.yaml <<EOF
apiVersion: kadalu-operator.storage/v1alpha1
kind: KadaluStorage
metadata:
  name: replica3
spec:
  type: Replica3
  storage:
    - node: k8s-master01 # 主机名
      device: /dev/sdc # 之前准备的磁盘
    - node: k8s-node01
      device: /dev/sdc
    - node: k8s-node02
      device: /dev/sdc
EOF
```

创建资源：

```shell
kubectl apply -f storage-config-device.yaml

kubectl get all -l app.kubernetes.io/component=server -o wide
NAME                      READY   STATUS    RESTARTS   AGE   IP              NODE           NOMINATED NODE   READINESS GATES
pod/server-replica3-0-0   1/1     Running   0          23m   10.244.32.177   k8s-master01   <none>           <none>
pod/server-replica3-1-0   1/1     Running   0          23m   10.244.85.221   k8s-node01     <none>           <none>
pod/server-replica3-2-0   1/1     Running   0          23m   10.244.58.212   k8s-node02     <none>           <none>

NAME               TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)     AGE   SELECTOR
service/replica3   ClusterIP   None         <none>        24007/TCP   23m   app.kubernetes.io/component=server,app.kubernetes.io/name=server,app.kubernetes.io/part-of=kadalu

NAME                                 READY   AGE   CONTAINERS   IMAGES
statefulset.apps/server-replica3-0   1/1     23m   server       docker.io/kadalu/kadalu-server:devel
statefulset.apps/server-replica3-1   1/1     23m   server       docker.io/kadalu/kadalu-server:devel
statefulset.apps/server-replica3-2   1/1     23m   server       docker.io/kadalu/kadalu-server:devel


kubectl describe svc
Name:              replica3
Namespace:         kadalu
Labels:            app.kubernetes.io/component=server
                   app.kubernetes.io/name=replica3-service
                   app.kubernetes.io/part-of=kadalu
Annotations:       <none>
Selector:          app.kubernetes.io/component=server,app.kubernetes.io/name=server,app.kubernetes.io/part-of=kadalu
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                None
IPs:               None
Port:              brickport  24007/TCP
TargetPort:        24007/TCP
Endpoints:         10.244.32.177:24007,10.244.58.212:24007,10.244.85.221:24007
Session Affinity:  None
Events:            <none>


for i in 0 1 2; do kubectl exec -it server-replica3-$i-0 -- sh -c 'hostname; df -hT | grep bricks; ls -lR /bricks/replica3/data'; done;


kubectl exec -it kadalu-csi-provisioner-0 -c kadalu-provisioner -- sh -c 'df -h | grep -P secret'
tmpfs            7.7G  8.0K  7.7G   1% /etc/secret-volume
tmpfs            7.7G   12K  7.7G   1% /run/secrets/kubernetes.io/serviceaccount
```

# 3.卷操作

查看本地storage相关资源：

```shell
kubectl get kds,sc
NAME                                             AGE
kadalustorage.kadalu-operator.storage/replica3   25m

NAME                                                PROVISIONER            RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
storageclass.storage.k8s.io/kadalu.replica3         kadalu                 Delete          Immediate              true                   25m
```

## 3.1.创建pvc

```shell
cat > sample-pvc.yaml <<EOF
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: replica3-pvc
spec:
  storageClassName: kadalu.replica3
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
EOF
```

创建pvc：

```shell
kubectl apply -f sample-pvc.yaml

kubectl get pv,pvc
NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                  STORAGECLASS      REASON   AGE
persistentvolume/pvc-64893704-138a-4efb-9c6f-b9b14aaa4669   1Gi        RWX            Delete           Bound    kadalu/replica3-pvc                    kadalu.replica3            23m

NAME                                 STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      AGE
persistentvolumeclaim/replica3-pvc   Bound    pvc-64893704-138a-4efb-9c6f-b9b14aaa4669   1Gi        RWX            kadalu.replica3   23m
```

## 3.2.查看csi-provisioner

```shell
kubectl exec -it kadalu-csi-provisioner-0 -c kadalu-provisioner -- bash

df -hT | grep kadalu
kadalu:replica3 fuse.glusterfs  295G  3.0G  280G   2% /mnt/replica3

ls /mnt/replica3/
info  stat.db  subvol

ls /mnt/replica3/subvol/3e/db/pvc-64893704-138a-4efb-9c6f-b9b14aaa4669/
cat /mnt/replica3/info/subvol/3e/db/pvc-64893704-138a-4efb-9c6f-b9b14aaa4669.json
{"size": 1073741824, "path_prefix": "subvol/3e/db"}
```

参考文档：

- https://zhuanlan.zhihu.com/p/405812023
- https://thoughtexpo.com/exploring-kadalu-storage-in-k3d-cluster-glusterfs