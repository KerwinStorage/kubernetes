# 								kubeblocks的使用

介绍：它是基于 Kubernetes 的云原生数据基础设施，为用户提供了关系型数据库、NoSQL 数据库、向量数据库以及流计算系统的管理控制功能。可以使用提供的命令轻松部署处理数据库实例。

github：https://github.com/apecloud/kubeblocks

官网：https://kubeblocks.io

## 1.初步使用

安装kbcli：

官网说明：https://kubeblocks.io/docs/release-0.7/user_docs/installation/install-with-kbcli/install-kbcli

```shell
curl -fsSL https://kubeblocks.io/installer/install_cli.sh | bash
# 命令补全
kbcli completion zsh -h
echo "autoload -U compinit; compinit" >> ~/.zshrc
echo "source <(kbcli completion zsh); compdef _kbcli kbcli" >> ~/.zshrc
```

通过 kbcli 安装 KubeBlocks：

官网说明：https://kubeblocks.io/docs/release-0.7/user_docs/installation/install-with-kbcli/install-kubeblocks-with-kbcli

```shell
# kbcli kubeblocks install
KubeBlocks will be installed to namespace "kb-system"
Kubernetes version 1.26.5
kbcli version 0.7.1
Collecting data from cluster                       OK
Kubernetes cluster preflight                       OK
Add and update repo kubeblocks                     OK
Install KubeBlocks 0.7.1                           OK
Wait for addons to be enabled
  apecloud-mysql                                   OK
  kafka                                            OK
  mongodb                                          OK
  mysql                                            OK
  postgresql                                       OK
  pulsar                                           OK
  redis                                            OK
  snapshot-controller                              OK

KubeBlocks 0.7.1 installed to namespace kb-system SUCCESSFULLY!

-> Basic commands for cluster:
    kbcli cluster create -h     # help information about creating a database cluster
    kbcli cluster list          # list all database clusters
    kbcli cluster describe <cluster name>  # get cluster information

-> Uninstall KubeBlocks:
    kbcli kubeblocks uninstall


# 查看启动容器
# kubectl get pod -n kb-system
NAME                                            READY   STATUS    RESTARTS   AGE
kb-addon-snapshot-controller-8484bbd44c-lc69m   1/1     Running   0          102s
kubeblocks-69b7c6db64-xskrj                     1/1     Running   0          2m22s
kubeblocks-dataprotection-67f46457c7-7s7kv      1/1     Running   0          2m22s

# 查看kubeblocks的状态
kbcli kubeblocks status
KubeBlocks is deployed in namespace: kb-system,version: 0.7.1

KubeBlocks Workloads:
NAMESPACE   KIND         NAME                           READY PODS   CPU(CORES)   MEMORY(BYTES)   CREATED-AT
kb-system   Deployment   kb-addon-snapshot-controller   1/1          N/A          N/A             Dec 03,2023 14:12 UTC+0800
kb-system   Deployment   kubeblocks                     1/1          N/A          N/A             Dec 03,2023 14:11 UTC+0800
kb-system   Deployment   kubeblocks-dataprotection      1/1          N/A          N/A             Dec 03,2023 14:11 UTC+0800

KubeBlocks Addons:
NAME                           STATUS     TYPE   PROVIDER
alertmanager-webhook-adaptor   Disabled   Helm   apecloud
apecloud-mysql                 Enabled    Helm   apecloud
apecloud-otel-collector        Disabled   Helm   apecloud
aws-load-balancer-controller   Disabled   Helm   N/A
bytebase                       Disabled   Helm   community
cert-manager                   Disabled   Helm   community
csi-hostpath-driver            Disabled   Helm   community
csi-s3                         Disabled   Helm   community
elasticsearch                  Disabled   Helm   community
external-dns                   Disabled   Helm   N/A
fault-chaos-mesh               Disabled   Helm   community
foxlake                        Disabled   Helm   community
grafana                        Disabled   Helm   community
greptimedb                     Disabled   Helm   community
jupyter-hub                    Disabled   Helm   community
jupyter-notebook               Disabled   Helm   community
kafka                          Enabled    Helm   community
kubebench                      Disabled   Helm   community
kubeblocks-csi-driver          Disabled   Helm   N/A
llm                            Disabled   Helm   community
loki                           Disabled   Helm   community
mariadb                        Disabled   Helm   community
migration                      Disabled   Helm   community
milvus                         Disabled   Helm   community
minio                          Disabled   Helm   community
mongodb                        Enabled    Helm   community
mysql                          Enabled    Helm   community
nebula                         Disabled   Helm   community
neon                           Disabled   Helm   community
nvidia-gpu-exporter            Disabled   Helm   community
nyancat                        Disabled   Helm   apecloud
opensearch                     Disabled   Helm   community
oracle-mysql                   Disabled   Helm   ApeCloud
orioledb                       Disabled   Helm   apecloud
polardbx                       Disabled   Helm   community
postgresql                     Enabled    Helm   community
prometheus                     Disabled   Helm   community
pulsar                         Enabled    Helm   community
pyroscope-server               Disabled   Helm   community
qdrant                         Disabled   Helm   community
redis                          Enabled    Helm   community
risingwave                     Disabled   Helm   community
snapshot-controller            Enabled    Helm   community
starrocks                      Disabled   Helm   community
tdengine                       Disabled   Helm   community
victoria-metrics-agent         Disabled   Helm   community
weaviate                       Disabled   Helm   community
xinference                     Disabled   Helm   community
zookeeper                      Disabled   Helm   community
```

## 2.创建MySQL

官网：https://kubeblocks.io/docs/release-0.7/user_docs/kubeblocks-for-mysql/cluster-management/create-and-connect-a-mysql-cluster

```shell
# 创建单实例
kbcli cluster create mysql mycluster
Info: --version is not specified, ac-mysql-8.0.30 is applied by default.
Cluster mycluster created

# 创建集群实例
kbcli cluster create mysql --mode raftGroup --availability-policy none  mysql-cluster
Info: --version is not specified, ac-mysql-8.0.30 is applied by default.
Cluster mysql-cluster created

# 连接MySQL
kbcli cluster connect mysql-cluster
Connect to instance mysql-cluster-mysql-1: out of mysql-cluster-mysql-1(leader), mysql-cluster-mysql-2(follower), mysql-cluster-mysql-0(follower)
Defaulted container "mysql" out of: mysql, metrics, vttablet, kb-checkrole, config-manager
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 58
Server version: 8.0.30 WeSQL Server - GPL, Release 5, Revision 4ca1eb8

Copyright (c) 2000, 2022, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql>
```

注：具体操作在官网已经写得很清楚，这里不在操作，可以根据官网进行容器数量的修改和内存和cpu的修改。