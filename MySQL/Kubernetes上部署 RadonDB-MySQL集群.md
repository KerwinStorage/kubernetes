# 1.mysql部署

部署参考文档：https://radondb.com/docs/mysql/v2.2.0/installation/on_kubernetes/#content

参数：https://github.com/radondb/radondb-mysql-kubernetes/blob/main/docs/zh-cn/config_para.md

官网：[https://radondb.com](https://radondb.com/)

```bash
helm repo add radondb https://radondb.github.io/radondb-mysql-kubernetes
helm search repo
# 部署
helm install demo radondb/mysql-operator
NAME: demo
LAST DEPLOYED: Tue May 23 17:39:06 2023
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Welcome to RadonDB MySQL Kubernetes!

> Create MySQLCluster:

kubectl apply -f https://github.com/radondb/radondb-mysql-kubernetes/releases/latest/download/mysql_v1alpha1_mysqlcluster.yaml

Create Users:

kubectl apply -f https://github.com/radondb/radondb-mysql-kubernetes/releases/latest/download/mysql_v1alpha1_mysqluser.yaml

Connect to the database:

kubectl exec -it svc/sample-leader -c mysql -- mysql -usuper_usr -pRadonDB@123

Change password:

kubectl patch secret sample-user-password --patch="{\"data\": { \"super_usr\": \"$(echo -n <yourpass> |base64 -w0)\" }}" -oyaml

Github: https://github.com/radondb/radondb-mysql-kubernetes
# 部署 RadonDB MySQL 集群
kubectl apply -f https://github.com/radondb/radondb-mysql-kubernetes/releases/latest/download/mysql_v1alpha1_mysqlcluster.yaml
# 效验
kubectl get deployment,svc
NAME                                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/demo-mysql-operator      1/1     1            1           2m51s

NAME                             TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE
service/mysql-operator-metrics   ClusterIP   10.107.167.19    <none>        8443/TCP            37m
service/radondb-mysql-webhook    ClusterIP   10.97.244.14     <none>        443/TCP             37m
service/sample-follower          ClusterIP   10.100.241.1     <none>        3306/TCP,8082/TCP   35m
service/sample-leader            ClusterIP   10.108.130.159   <none>        3306/TCP,8082/TCP   35m
service/sample-mysql             ClusterIP   None             <none>        3306/TCP,8082/TCP   35m

# 效验crd
kubectl get crd | grep mysql.radondb.com
backups.mysql.radondb.com                             2023-05-23T09:39:04Z
mysqlclusters.mysql.radondb.com                       2023-05-23T09:39:04Z
mysqlusers.mysql.radondb.com                          2023-05-23T09:39:04Z

# 安装客户端
apt-get install mysql-client
# 或者
apt-get install default-mysql-client
# 登录数据库
mysql -h 10.108.130.159  -P 3306 -u radondb_usr -p
Enter password: RadonDB@123 #默认密码，社区文档里面有显示。

#卸载 Operator
helm delete demo

#卸载集群
kubectl delete mysqlclusters.mysql.radondb.com sample


###
```

RadonDB MySQL 提供 Leader 和 Follower 两种服务，分别用于客户端访问主从节点。Leader 服务始终指向主节点（可读写），Follower 服务始终指向从节点（只读）。

解决无法使用root用户去创建仓库：[解决方法](https://github.com/radondb/radondb-mysql-kubernetes/issues/104)

```bash
# 进入leader节点中的xenon容器
kubectl exec -it sample-mysql-0  -c xenon -- bash
# 增加一个superuser
xenoncli mysql createsuperuser ldap % Eryajf@123 NO
# 授权root账户创建仓库
CREATE USER 'ldap'@'%' IDENTIFIED BY 'Eryajf@123';
CREATE DATABASE IF NOT EXISTS `go_ldap_admin` CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
GRANT ALL PRIVILEGES on `go_ldap_admin`.* to 'ldap'@'%';
FLUSH privileges;
```

 