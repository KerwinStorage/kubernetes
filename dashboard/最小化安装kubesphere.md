版本：

- KubeSphere 3.3
- kubernetes 1.22.10

链接：

- https://v2-1.docs.kubesphere.io/docs/zh-cn/installation/install-on-k8s
- https://github.com/kubesphere/kubesphere

**以下操作参考[官方文档](https://kubesphere.io/zh/docs/v3.3/installing-on-kubernetes/introduction/overview/)就行。**

## 1. 准备工作

1. 在集群节点中运行 `kubectl version`，确保 Kubernetes 版本可兼容。输出如下所示：

```bash
kubectl version
Client Version: version.Info{Major:"1", Minor:"22", GitVersion:"v1.22.10", GitCommit:"eae22ba6238096f5dec1ceb62766e97783f0ba2f", GitTreeState:"clean", BuildDate:"2022-05-24T12:56:35Z", GoVersion:"go1.16.15", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"22", GitVersion:"v1.22.10", GitCommit:"eae22ba6238096f5dec1ceb62766e97783f0ba2f", GitTreeState:"clean", BuildDate:"2022-05-24T12:50:52Z", GoVersion:"go1.16.15", Compiler:"gc", Platform:"linux/amd64"}
```

1. 检查集群中的可用资源是否满足最低要求。

```bash
$ free -g

            total        used        free      shared  buff/cache   available
Mem:              16          4          10           0           3           2
Swap:             0           0           0
```

1. 检查集群中是否有**默认** StorageClass（准备默认 StorageClass 是安装 KubeSphere 的前提条件）。

```bash
kubectl get sc
NAME                    PROISIONER                                   RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
nfs-storage (default)   k8s-sigs.io/nfs-subdir-external-provisioner   Delete          Immediate           false                  30d
```

**注：使用nfs方式存储。**

## 2. 部署 KubeSphere

确保现有的 Kubernetes 集群满足所有要求之后，您可以使用 kubectl 以默认最小安装包来安装 KubeSphere。

1. 执行以下命令以开始安装：

```bash
kubectl apply -f https://github.com/kubesphere/ks-installer/releases/download/v3.3.1/kubesphere-installer.yaml
kubectl apply -f https://github.com/kubesphere/ks-installer/releases/download/v3.3.1/cluster-configuration.yaml
```

1. 检查安装日志：

```bash
kubectl logs -n kubesphere-system $(kubectl get pod -n kubesphere-system -l 'app in (ks-install, ks-installer)' -o jsonpath='{.items[0].metadata.name}') -f
Collecting installation results ...
#####################################################
###              Welcome to KubeSphere!           ###
#####################################################

Console: http://192.168.80.45:30880 #登录的地点
Account: admin #账户
Password: P@88w0rd #密码
NOTES：
  1. After you log into the console, please check the
     monitoring status of service components in
     "Cluster Management". If any service is not
     ready, please wait patiently until all components
     are up and running.
  2. Please change the default password after login.

#####################################################
https://kubesphere.io             2023-01-18 00:27:10
#####################################################
```

1. 使用 `kubectl get pod --all-namespaces` 查看所有 Pod 在 KubeSphere 相关的命名空间是否正常运行。如果是正常运行，请通过以下命令来检查控制台的端口（默认为 30880）：

```bash
kubectl get svc/ks-console -n kubesphere-system
NAME         TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
ks-console   NodePort   10.105.105.226   <none>        80:30880/TCP   71m
```

http://192.168.80.45:30880

![img](https://img2023.cnblogs.com/blog/1740081/202301/1740081-20230118182529943-963187280.png)

 

## 3. 启用可插拔组件（可选）

### 3.1 在 Kubernetes 上安装

- [启用可插拔组件 (kubesphere.io)](https://kubesphere.io/zh/docs/v3.3/pluggable-components/) [KubeSphere 应用商店](https://kubesphere.io/zh/docs/v3.3/pluggable-components/app-store/)

1. 下载配置文件

```bash
mkdir -p /root/kubesphere && cd /root/kubesphere
wget https://github.com/kubesphere/ks-installer/releases/download/v3.3.1/cluster-configuration.yaml
vim cluster-configuration.yaml
# 在 cluster-configuration.yaml 文件中，搜索 openpitrix，并将 enabled 的 false 改为 true。完成后保存文件
openpitrix:
  store:
    enabled: true # 将“false”更改为“true”。
```

1. 执行以下命令开始安装：

```bash
kubectl apply -f https://github.com/kubesphere/ks-installer/releases/download/v3.3.1/kubesphere-installer.yaml
kubectl apply -f cluster-configuration.yaml
```

http://192.168.80.45:30880/apps

![img](https://img2023.cnblogs.com/blog/1740081/202301/1740081-20230118182710691-1435507783.png)

### 3.2 在安装后启用应用商店

1. 使用 `admin` 用户登录控制台，点击左上角的**平台管理**，选择**集群管理**。
2. 点击**定制资源定义**，在搜索栏中输入 `clusterconfiguration`，点击结果查看其详细页面。
3. 在**自定义资源**中，点击 `ks-installer` 右侧的 ![img](https://kubesphere.io/images/docs/v3.3/zh-cn/enable-pluggable-components/kubesphere-app-store/three-dots.png)，选择**编辑 YAML**。
4. 在该 YAML 文件中，搜索 `openpitrix`，将 `enabled` 的 `false` 改为 `true`。完成后，点击右下角的**确定**，保存配置。

```bash
openpitrix:
  store:
    enabled: true # 将“false”更改为“true”。
```

5. 在 kubectl 中执行以下命令检查安装过程：

```bash
kubectl logs -n kubesphere-system $(kubectl get pod -n kubesphere-system -l 'app in (ks-install, ks-installer)' -o jsonpath='{.items[0].metadata.name}') -f
```

 