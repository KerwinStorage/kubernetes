## 1. 规划

升级前：

| 组件名称                | 位置                             | 版本     |
| ----------------------- | -------------------------------- | -------- |
| kube-apiserver          | /usr/bin/kube-apiserver          | v1.21.13 |
| etcd、etcdctl           | /usr/bin/etcd、/usr/bin/etcdctl  | 3.5.4    |
| kube-controller-manager | /usr/bin/kube-controller-manager | v1.21.13 |
| kube-scheduler          | /usr/bin/kube-scheduler          | v1.21.13 |
| docker                  | /usr/bin/docker                  | 20.10.21 |
| kubelet                 | /usr/bin/kubelet                 | v1.21.13 |
| kube-proxy              | /usr/bin/kube-proxy              | v1.21.13 |
| cni                     | /opt/cni/bin/                    | v1.1.1   |
| Calico                  | 容器部署                         | v3.22    |
| coredns                 | 容器部署                         | 1.6.2    |

升级后：

| 组件名称                | 位置                             | 版本     |
| ----------------------- | -------------------------------- | -------- |
| kube-apiserver          | /usr/bin/kube-apiserver          | v1.22.10 |
| etcd、etcdctl           | /usr/bin/etcd、/usr/bin/etcdctl  | 3.5.4    |
| kube-controller-manager | /usr/bin/kube-controller-manager | v1.22.10 |
| kube-scheduler          | /usr/bin/kube-scheduler          | v1.22.10 |
| docker                  | /usr/bin/docker                  | 20.10.21 |
| kubelet                 | /usr/bin/kubelet                 | v1.22.10 |
| kube-proxy              | /usr/bin/kube-proxy              | v1.22.10 |
| cni                     | /opt/cni/bin/                    | v1.1.1   |
| Calico                  | 容器部署                         | v3.22    |
| coredns                 | 容器部署                         | 1.6.2    |

> **这里主要升级kubernetes主要组件：`kube-apiserver`、`kube-controller-manager`、`kube-scheduler`、`kubelet`、`kube-proxy`，其余的不需要进行升级，在升级时最好是一个高可用的集群，master01升级时api-server迁移到其他master节点进行调度。**

## 2. master节点

```bash
mkdir -p $HOME/k8s-install && cd $HOME/k8s-install

# 准备二进制包
K8S_VERSION="v1.22.10"
wget https://dl.k8s.io/$K8S_VERSION/kubernetes-server-linux-amd64.tar.gz
tar zxvf kubernetes-server-linux-amd64.tar.gz

# 停掉master01中的 kube-apiserver kube-scheduler kube-controller-manager kubectl kubelet kube-proxy
systemctl stop kube-apiserver
systemctl stop kube-scheduler
systemctl stop kube-controller-manager
systemctl stop kubelet
systemctl stop kube-proxy



cd kubernetes/server/bin
cp kube-apiserver kube-scheduler kube-controller-manager kubectl kubelet kube-proxy /usr/bin

# 启动服务
systemctl restart kube-apiserver
systemctl restart kube-scheduler
systemctl restart kube-controller-manager
systemctl restart kubelet
systemctl restart kube-proxy

# 查看启动情况
# systemctl status kube-apiserver
# systemctl status kube-scheduler
# systemctl status kube-controller-manager
# systemctl status kubelet
# systemctl status kube-proxy
```

## 3. node节点

```bash
# node节点操作
systemctl stop kubelet
systemctl stop kube-proxy

# master01节点操作
scp kubectl kubelet kube-proxy root@k8s-node01:/usr/bin
scp kubectl kubelet kube-proxy root@k8s-node02:/usr/bin

# node节点重启
systemctl restart kubelet
systemctl restart kube-proxy

kubectl get node
NAME           STATUS   ROLES    AGE   VERSION
k8s-master01   Ready    master   36h   v1.22.10
k8s-node01     Ready    node     21h   v1.22.10
k8s-node02     Ready    node     21h   v1.22.10
```

 