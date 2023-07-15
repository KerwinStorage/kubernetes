# 										MinIO 搭建

官网：https://min.io
中文官网：http://www.minio.org.cn，http://dl.minio.org.cn
GitHub：https://github.com/minio

对象存储服务OSS（Object Storage Service）是一种海量、安全、低成本、高可靠的云存储服务，**适合存放任意类型的文件**。容量和处理能力弹性扩展，多种存储类型供选择，全面优化存储成本。

# 1.helm部署

**前置条件**：[nfs作为存储插件](https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner)

注：主要是使用nfs命令将集群本地的资源共享起来，所有机器都能访问到，做成一个sc。

## 1.1.helm部署资源

微软仓库：http://mirror.azure.cn/kubernetes/charts

阿里云仓库：https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts

官方仓库：https://hub.kubeapps.com/charts/incubator

```shell
# 创建名称空间
kubectl create ns minio

# 搜索可用的version
helm search repo minio/minio
NAME            CHART VERSION   APP VERSION     DESCRIPTION
minio/minio     8.0.10          master          High Performance, Kubernetes Native Object Storage

# 添加仓库
helm repo add minio https://helm.min.io

# 下载chart
mkdir -p ~/minio && cd ~/minio
helm fetch minio/minio
tar zxvf minio-8.0.10.tgz
cd minio
```

修改 values.yaml

```shell
accessKey: 'minio'
secretKey: 'minio123'
persistence:
  enabled: true
  storageCalss: 'nfs-storage' # 自己使用nfs插件创建的存储：kubectl get sc
  VolumeName: ''
  accessMode: ReadWriteOnce
  size: 50Gi

service:
  type: NodePort
  clusterIP: ~
  port: 9000
  nodePort: 32000

resources:
  requests:
    memory: 128Mi
```

**注：如果镜像不好下载，这里`registry.cn-hangzhou.aliyuncs.com/image-storage/minio:RELEASE.2021-02-14T04-01-33Z`，是推送到阿里镜像仓库的地址可以进行代替，具体看`values.yaml`文件。**

helm 安装：

```sh
helm install -f values.yaml minio  minio/minio -n minio
```

查看资源：

```shell
kubectl get pod -n minio
NAME                    READY   STATUS    RESTARTS   AGE
minio-fc58db647-h728b   1/1     Running   0          19s

kubectl get svc  -n minio
NAME    TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
minio   NodePort   10.111.43.35   <none>        9000:32000/TCP   40s
# 这里之间暴露主机的32000端口去访问

kubectl get pv -n minio
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM         STORAGECLASS   REASON   AGE
pvc-3c105cb9-307d-4859-93d1-e33e74fa3ee3   50Gi       RWO            Delete           Bound    minio/minio   nfs-storage             2m57s
```

http://192.168.80.45:32000/minio/login

accessKey: '`minio`'，secretKey: '`minio123`'。

![img](https://img2023.cnblogs.com/blog/1740081/202307/1740081-20230715110440273-2138696710.png)

登录：

![img](https://img2023.cnblogs.com/blog/1740081/202307/1740081-20230715110457652-2099689466.png)

