# 					helm部署MySQL主从架构8.0

# 1.准备原文件

## 1.1.更新chart仓库

首先，添加存储库并更新Helm存储库：

```shell
# 更新chart仓库
helm repo add aliyun https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts
helm repo add stable http://mirror.azure.cn/kubernetes/charts
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# 搜索MySQL版本
helm search repo mysql

# 将包下载到本地
helm pull bitnami/mysql --version=9.19.1
tar zxvf mysql-9.19.1.tgz && cp mysql/values.yaml mysql-values.yaml
```

## 1.2.修改values配置

```shell
# 选择
global:
  storageClass: "nfs-csi"
# 指定MySQL镜像版本
image:
  registry: docker.io
  repository: bitnami/mysql
  tag: 8.0.36-debian-11-r4

# 开启主从模式
architecture: replication

# 修改MySQL密码root密码
auth:
  rootPassword: "1qazZSE$"
  createDatabase: true
  database: "my_database"
  username: "kerwin"
  password: "1qazZSE$"
  replicationUser: replicator
  replicationPassword: "1qazZSE$"
```

镜像地址：https://hub.docker.com/r/bitnami/mysql/tags

### 1.2.1主节点

```shell
primary:
  name: primary
  command: []
  args: []
  lifecycleHooks: {}
  hostAliases: []
  configuration: |-
    [mysqld]
    default_authentication_plugin=mysql_native_password
    skip-name-resolve
    explicit_defaults_for_timestamp
    basedir=/opt/bitnami/mysql
    plugin_dir=/opt/bitnami/mysql/lib/plugin
    port=3306
    socket=/opt/bitnami/mysql/tmp/mysql.sock
    datadir=/bitnami/mysql/data
    tmpdir=/opt/bitnami/mysql/tmp
    max_allowed_packet=16M
    bind-address=*
    # 作为临时解决方案，您可以禁止显示来自mysqld.log的警告消息。https://knowledge.broadcom.com/external/article/271693/latest-security-patch-generates-mysql-w.html
    log_error_suppression_list='MY-013360'
    pid-file=/opt/bitnami/mysql/tmp/mysqld.pid
    log-error=/opt/bitnami/mysql/logs/mysqld.log
    character-set-server=UTF8
    collation-server=utf8_general_ci
    slow_query_log=0
    slow_query_log_file=/opt/bitnami/mysql/logs/mysqld.log
    long_query_time=10.0
 
    [client]
    port=3306
    socket=/opt/bitnami/mysql/tmp/mysql.sock
    default-character-set=UTF8
    plugin_dir=/opt/bitnami/mysql/lib/plugin
 
    [manager]
    port=3306
    socket=/opt/bitnami/mysql/tmp/mysql.sock
    pid-file=/opt/bitnami/mysql/tmp/mysqld.pid
  existingConfigmap: ""
  updateStrategy:
    type: RollingUpdate
  persistence:
    enabled: true
    existingClaim: ""
    subPath: ""
    storageClass: "managed-nfs-storage"
    annotations: {}
    accessModes:
      - ReadWriteOnce
    size: 200Gi
    selector: {}
  extraVolumes: []
```

### 1.2.2从节点

```shell
secondary:
  name: secondary
  replicaCount: 2
  hostAliases: []
  command: []
  args: []
  lifecycleHooks: {}
  configuration: |-
    [mysqld]
    default_authentication_plugin=mysql_native_password
    skip-name-resolve
    explicit_defaults_for_timestamp
    basedir=/opt/bitnami/mysql
    plugin_dir=/opt/bitnami/mysql/lib/plugin
    port=3306
    socket=/opt/bitnami/mysql/tmp/mysql.sock
    datadir=/bitnami/mysql/data
    tmpdir=/opt/bitnami/mysql/tmp
    max_allowed_packet=16M
    bind-address=*
    # 作为临时解决方案，您可以禁止显示来自mysqld.log的警告消息。https://knowledge.broadcom.com/external/article/271693/latest-security-patch-generates-mysql-w.html
    log_error_suppression_list='MY-013360'
    pid-file=/opt/bitnami/mysql/tmp/mysqld.pid
    log-error=/opt/bitnami/mysql/logs/mysqld.log
    character-set-server=UTF8
    collation-server=utf8_general_ci
    slow_query_log=0
    slow_query_log_file=/opt/bitnami/mysql/logs/mysqld.log
    long_query_time=10.0
 
    [client]
    port=3306
    socket=/opt/bitnami/mysql/tmp/mysql.sock
    default-character-set=UTF8
    plugin_dir=/opt/bitnami/mysql/lib/plugin
 
    [manager]
    port=3306
    socket=/opt/bitnami/mysql/tmp/mysql.sock
    pid-file=/opt/bitnami/mysql/tmp/mysqld.pid
  existingConfigmap: ""
  persistence:
    enabled: true
    existingClaim: ""
    subPath: ""
    storageClass: "managed-nfs-storage"
    annotations: {}
    accessModes:
      - ReadWriteOnce
    size: 8Gi
    selector: {}
  extraVolumes: []
```

# 2.helm部署

## 2.1.创建资源

```sh
kubectl create namespace mysql
helm install mysql-cluster  mysql-9.19.1.tgz --namespace mysql --set useBundledSystemChart=true --set volumePermissions.enabled=true -f mysql-values.yaml
NAME: mysql-cluster
LAST DEPLOYED: Mon Feb  5 23:10:52 2024
NAMESPACE: mysql
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
CHART NAME: mysql
CHART VERSION: 9.19.1
APP VERSION: 8.0.36

** Please be patient while the chart is being deployed **

Tip:

  Watch the deployment status using the command: kubectl get pods -w --namespace mysql

Services:

  echo Primary: mysql-cluster-primary.mysql.svc.cluster.local:3306
  echo Secondary: mysql-cluster-secondary.mysql.svc.cluster.local:3306

Execute the following to get the administrator credentials:

  echo Username: root
  MYSQL_ROOT_PASSWORD=$(kubectl get secret --namespace mysql mysql-cluster -o jsonpath="{.data.mysql-root-password}" | base64 -d)

To connect to your database:

  1. Run a pod that you can use as a client:

      kubectl run mysql-cluster-client --rm --tty -i --restart='Never' --image  docker.io/bitnami/mysql:5.7.43 --namespace mysql --env MYSQL_ROOT_PASSWORD=$MYSQL_ROOT_PASSWORD --command -- bash

  2. To connect to primary service (read/write):

      mysql -h mysql-cluster-primary.mysql.svc.cluster.local -uroot -p"$MYSQL_ROOT_PASSWORD"

  3. To connect to secondary service (read-only):

      mysql -h mysql-cluster-secondary.mysql.svc.cluster.local -uroot -p"$MYSQL_ROOT_PASSWORD"
```

## 2.2.遇到的问题汇总

### 2.2.1创建目录失败

问题描述：容器无法创建目录

```shell
mysql 15:30:40.61
mysql 15:30:40.62 Welcome to the Bitnami mysql container
mysql 15:30:40.63 Subscribe to project updates by watching https://github.com/bitnami/containers
mysql 15:30:40.65 Submit issues and feature requests at https://github.com/bitnami/containers/issues
mysql 15:30:40.70
mysql 15:30:40.74 INFO  ==> ** Starting MySQL setup **
mysql 15:30:40.90 INFO  ==> Validating settings in MYSQL_*/MARIADB_* env vars
mysql 15:30:41.12 INFO  ==> Initializing mysql database
mkdir: cannot create directory '/bitnami/mysql/data': Permission denied # 无法创建目录，权限被拒绝
```

问题链接：

- [stackoverflow](https://stackoverflow.com/questions/69795804/mariadb-galera-on-minikube-mkdir-cannot-create-directory-bitnami-mariadb-dat)

解决方案：helm部署时加入`--set volumePermissions.enabled=true`的参数。[链接](https://stackoverflow.com/questions/70676003/specify-mysqld-in-kubernetes-deployment)

### 2.2.2身份验证插件在MySQL 8.0中已被弃用

```shell
2024-02-08T07:45:24.876894Z 15 [Warning] [MY-013360] [Server] Plugin mysql_native_password reported: ''mysql_native_password' is deprecated and will be removed in a future release. Please use caching_sha2_password instead
```

问题链接：

- [support.cpanel.net](https://support.cpanel.net/hc/en-us/articles/16550190886935-MySQL-log-warning-mysql-native-password-is-deprecated-and-will-be-removed-in-a-future-release)
- [kn007博客](https://kn007.net/topics/mysql-8-0-34-deprecated-mysql-native-password/)
- [knowledge.broadcom.com](https://knowledge.broadcom.com/external/article/271693/latest-security-patch-generates-mysql-w.html)

解决方案：configuration内加入`log_error_suppression_list='MY-013360'`，忽略报错。

# 3.验证主从同步

## 3.1.安装客户端

```shell
apt install mysql-client -y
```

## 3.2.验证主节点

```shell
# 登录主容器svc
mysql -uroot -h10.107.166.168 -p1qazZSE$

# 查看主从状态
# 查看File和Position的值，在从库配置中会显示。
show master status\G;
*************************** 1. row ***************************
             File: mysql-bin.000007
         Position: 157
     Binlog_Do_DB:
 Binlog_Ignore_DB:
Executed_Gtid_Set:
1 row in set (0.00 sec)

ERROR:
No query specified
```

## 3.3.验证从节点

```shell
# 登录从容器svc
mysql -uroot -h10.101.7.132 -p1qazZSE$

show slave status\G;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for source to send event
                  Master_Host: mysql-cluster-primary
                  Master_User: replicator
                  Master_Port: 3306
                Connect_Retry: 10
              Master_Log_File: mysql-bin.000007
          Read_Master_Log_Pos: 157
               Relay_Log_File: mysql-relay-bin.000016
                Relay_Log_Pos: 373
        Relay_Master_Log_File: mysql-bin.000007
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB:
          Replicate_Ignore_DB:
           Replicate_Do_Table:
       Replicate_Ignore_Table:
      Replicate_Wild_Do_Table:
  Replicate_Wild_Ignore_Table:
                   Last_Errno: 0
                   Last_Error:
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 157
              Relay_Log_Space: 799
              Until_Condition: None
               Until_Log_File:
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File:
           Master_SSL_CA_Path:
              Master_SSL_Cert:
            Master_SSL_Cipher:
               Master_SSL_Key:
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error:
               Last_SQL_Errno: 0
               Last_SQL_Error:
  Replicate_Ignore_Server_Ids:
             Master_Server_Id: 841
                  Master_UUID: e3278223-c655-11ee-806a-fe38b711638f
             Master_Info_File: mysql.slave_master_info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Replica has read all relay log; waiting for more updates
           Master_Retry_Count: 86400
                  Master_Bind:
      Last_IO_Error_Timestamp:
     Last_SQL_Error_Timestamp:
               Master_SSL_Crl:
           Master_SSL_Crlpath:
           Retrieved_Gtid_Set:
            Executed_Gtid_Set:
                Auto_Position: 0
         Replicate_Rewrite_DB:
                 Channel_Name:
           Master_TLS_Version:
       Master_public_key_path:
        Get_master_public_key: 0
            Network_Namespace:
1 row in set, 1 warning (0.00 sec)

ERROR:
No query specified
```

