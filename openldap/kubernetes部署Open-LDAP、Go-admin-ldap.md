# kubernetes部署Open-LDAP、Go-admin-ldap

前言：这里主要是根据[go-ldap-admin官方](http://ldapdoc.eryajf.net/)的`docker-compose`去进行改造的，yaml内的环境变量也是🐵🙊🙉🙈。

```bash
git clone https://github.com/eryajf/go-ldap-admin.git
cd go-ldap-admin
```

**注：所有需要的源头文件都在这个仓库内。**

参考docker-compose.yaml去进行改造🤔：

```bash
version: '3'

networks:
  go-ldap-admin:
    driver: bridge

services:
  mysql:
    image: dockerproxy.com/mysql/mysql-server:5.7
    container_name: go-ldap-admin-mysql # 指定容器名称，如果不设置此参数，则由系统自动生成
    hostname: go-ldap-admin-mysql
    restart: always # 设置容器自启模式
    ports:
      - '3307:3306'
    environment:
      TZ: Asia/Shanghai # 设置容器时区与宿主机保持一致
      MYSQL_ROOT_PASSWORD: 123456 # 设置root密码
      MYSQL_ROOT_HOST: "%"
      MYSQL_DATABASE: go_ldap_admin
    volumes:
      # 数据挂载目录自行修改哦！
      - /etc/localtime:/etc/localtime:ro # 设置容器时区与宿主机保持一致
      - ./data/mysql:/var/lib/mysql/data # 映射数据库保存目录到宿主机，防止数据丢失
      - ./config/my.cnf:/etc/mysql/my.cnf # 映射数据库配置文件
    command: --default-authentication-plugin=mysql_native_password #解决外部无法访问
    networks:
      - go-ldap-admin

  openldap:
    image: dockerproxy.com/osixia/openldap:1.4.0
    container_name: go-ldap-admin-openldap
    hostname: go-ldap-admin-openldap
    restart: always
    environment:
      TZ: Asia/Shanghai
      LDAP_ORGANISATION: "eryajf.net"
      LDAP_DOMAIN: "eryajf.net"
      LDAP_ADMIN_PASSWORD: "123456"
    command: [ '--copy-service' ]
    volumes:
      - ./data/openldap/database:/var/lib/ldap
      - ./data/openldap/config:/etc/ldap/slapd.d
      - ./config/init.ldif:/container/service/slapd/assets/config/bootstrap/ldif/custom/init.ldif
    ports:
      - 388:389
    networks:
      - go-ldap-admin

  phpldapadmin:
    image: dockerproxy.com/osixia/phpldapadmin:0.9.0
    container_name: go-ldap-admin-phpldapadmin
    hostname: go-ldap-admin-phpldapadmin
    restart: always
    environment:
      TZ: Asia/Shanghai # 设置容器时区与宿主机保持一致
      PHPLDAPADMIN_HTTPS: "false" # 是否使用https
      PHPLDAPADMIN_LDAP_HOSTS: go-ldap-admin-openldap # 指定LDAP容器名称
    ports:
      - 8091:80
    volumes:
      - ./data/phpadmin:/var/www/phpldapadmin
    depends_on:
      - openldap
    links:
      - openldap:go-ldap-admin-openldap # ldap容器的 service_name:container_name
    networks:
      - go-ldap-admin

  go-ldap-admin-server:
    image: dockerproxy.com/eryajf/go-ldap-admin-server
    container_name: go-ldap-admin-server
    hostname: go-ldap-admin-server
    restart: always
    environment:
      TZ: Asia/Shanghai
      WAIT_HOSTS: mysql:3306, openldap:389
    ports:
      - 8888:8888
    # volumes:  # 可按需打开此配置，将配置文件挂载到本地 可在服务运行之后，执行 docker cp go-ldap-admin-server:/app/config.yml ./config 然后再取消该行注释
    #   - ./config/config.yml:/app/config.yml
    depends_on:
      - mysql
      - openldap
    links:
      - mysql:go-ldap-admin-mysql # ldap容器的 service_name:container_name
      - openldap:go-ldap-admin-openldap # ldap容器的 service_name:container_name
    networks:
      - go-ldap-admin

  go-ldap-admin-ui:
    image: dockerproxy.com/eryajf/go-ldap-admin-ui
    container_name: go-ldap-admin-ui
    hostname: go-ldap-admin-ui
    restart: always
    environment:
      TZ: Asia/Shanghai
    ports:
      - 8090:80
    depends_on:
      - go-ldap-admin-server
    links:
      - go-ldap-admin-server:go-ldap-admin-server
    networks:
      - go-ldap-admin
```

# 1.mysql:5.7

镜像地址🫠：https://registry.hub.docker.com/r/mysql/mysql-server/tags

```bash
kubectl create namespace open-ldap
```

## 1.1.创建secret资源

```bash
echo -n "1qazZSE$" | base64
MXFhelpTRSQ=

# secret内容
cat > secret.yml <<EOF
---
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secrets
  namespace: open-ldap
type: Opaque
data:
  ROOT_PASSWORD: MXFhelpTRSQ=
EOF

# 创建secret
kubectl apply -f secret.yml
kubectl get secret -n open-ldap
kubectl describe secret mysql-secrets -n open-ldap
```

## 1.2.创建my.cnf

```bash
cat > mysql-config-map.yaml <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-config-map
  namespace: open-ldap
data:
  my.cnf: |
    [client]
    port = 3306
    socket = /var/lib/mysql/data/mysql.sock
    [mysqld]
     # 针对5.7版本执行group by字句出错问题解决
    sql_mode='STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION'
     # 一般配置选项
    basedir = /var/lib/mysql
    datadir = /var/lib/mysql/data
    port = 3306
    socket = /var/lib/mysql/data/mysql.sock
    lc-messages-dir = /usr/share/mysql # 务必配置此项，否则执行sql出错时，只能显示错误代码而不显示具体错误消息
    character-set-server=utf8mb4
    back_log = 300
    default_authentication_plugin=mysql_native_password
    max_connections = 3000
    max_connect_errors = 50
    table_open_cache = 4096
    max_allowed_packet = 32M
    #binlog_cache_size = 4M
    max_heap_table_size = 128M
    read_rnd_buffer_size = 16M
    sort_buffer_size = 16M
    join_buffer_size = 16M
    thread_cache_size = 16
    query_cache_size = 64M
    query_cache_limit = 4M
    ft_min_word_len = 8
    thread_stack = 512K
    #tx_isolation = READ-COMMITTED
    tmp_table_size = 64M
    #log-bin=mysql-bin
    long_query_time = 6
    server_id=1
    innodb_buffer_pool_size = 1024M
    innodb_thread_concurrency = 16
    innodb_log_buffer_size = 16M
    wait_timeout= 31536000
    interactive_timeout= 31536000
    lower_case_table_names = 1
    bind-address = 0.0.0.0
EOF
```

创建资源：

```bash
kubectl apply -f mysql-config-map.yaml
```

## 1.3.创建持久卷声明资源

```bash
cat > persistentVolumeClaim.yml <<EOF
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-data-disk
  namespace: open-ldap
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
EOF

# 创建pvc并自动挂载sc
kubectl apply -f persistentVolumeClaim.yml
kubectl get persistentvolumeclaim mysql-data-disk -n open-ldap
```

## 1.4.创建deployment资源

```bash
cat > deployment.yml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql-deployment
  namespace: open-ldap
  labels:
    app: mysql
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:5.7
          ports:
            - containerPort: 3306
          volumeMounts:
            - mountPath: "/var/lib/mysql"
              subPath: "mysql"
              name: mysql-data
            - name: mysql-config-map
              mountPath: "/etc/mysql/my.cnf"
              subPath: my.cnf
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secrets
                  key: ROOT_PASSWORD
            - name: MYSQL_ROOT_HOST
              value: "%"
            - name: TZ
              value: "Asia/Shanghai"
      volumes:
        - name: mysql-data
          persistentVolumeClaim:
            claimName: mysql-data-disk
        - name: mysql-config-map
          configMap:
            name: mysql-config-map
            items:
            - key: my.cnf
              path: my.cnf
EOF

# 创建deployment资源
kubectl apply -f deployment.yml
kubectl get pod -n open-ldap
```

## 1.5.创建mysql的svc资源

```bash
cat > service.yml <<EOF
---
apiVersion: v1
kind: Service
metadata:
  name: mysql-service
  namespace: open-ldap
spec:
  selector:
    app: mysql
  ports:
  - protocol: TCP
    port: 3306
    targetPort: 3306
EOF

# 创建svc
kubectl apply -f service.yml
kubectl get svc -n open-ldap
```

## 1.6.测试MySQL登录

```bash
kubectl get svc -n open-ldap
NAME            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
mysql-service   ClusterIP   10.98.53.166   <none>        3306/TCP   4m42s


mysql -u root -h `kubectl get svc -n open-ldap mysql-service -o json | jq .spec.clusterIP | bc -l` -p
Enter password: 1qazZSE$
```

## 1.7.创建数据库

```bash
CREATE USER 'ldap'@'%' IDENTIFIED BY 'Eryajf@123';
CREATE DATABASE IF NOT EXISTS `go_ldap_admin` CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
GRANT ALL PRIVILEGES on `go_ldap_admin`.* to 'ldap'@'%';
FLUSH privileges;
```

## 1.8.设置环境变量

后面的配置文件会用到service的ip地址去连接，配置文件直接填写环境变量。

```bash
MYSQL_IP=`kubectl get svc -n open-ldap mysql-service -o json | jq .spec.clusterIP | bc -l`
echo $MYSQL_IP
```

# 2.openLDAP

## 2.1.创建configmap资源

```bash
cat > init.ldif <<EOF
dn: ou=people,dc=eryajf,dc=net
ou: people
description: 用户根目录
objectClass: organizationalUnit

dn: ou=dingtalkroot,dc=eryajf,dc=net
ou: dingtalkroot
description: 钉钉根部门
objectClass: top
objectClass: organizationalUnit

dn: ou=wecomroot,dc=eryajf,dc=net
ou: wecomroot
description: 企业微信根部门
objectClass: top
objectClass: organizationalUnit

dn: ou=feishuroot,dc=eryajf,dc=net
ou: feishuroot
description: 飞书根部门
objectClass: top
objectClass: organizationalUnit
EOF
```

- 生成configmap

```bash
kubectl create configmap openldap-init --from-file=./init.ldif -n open-ldap
```

## 2.2.创建pvc资源

```bash
cat > open-ldap-pvc.yaml << EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ldap-data-pvc
  namespace: open-ldap
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: nfs-storage
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ldap-config-pvc
  namespace: open-ldap
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: nfs-storage
EOF

# 创建资源
kubectl apply -f open-ldap-pvc.yaml
kubectl get pvc -n open-ldap
```

## 2.3.创建Deployment资源

镜像地址🤔：https://registry.hub.docker.com/r/osixia/openldap/tags

```bash
cat > ldap-deployment.yaml <<EOF
kind: Deployment
apiVersion: apps/v1
metadata:
  name: openldap
  namespace: open-ldap
  labels:
    app: openldap
  annotations:
    app.kubernetes.io/alias-name: LDAP
    app.kubernetes.io/description: 认证中心
spec:
  replicas: 1
  selector:
    matchLabels:
      app: openldap
  template:
    metadata:
      labels:
        app: openldap
    spec:
      containers:
        - name: go-ldap-admin-openldap
          args:
            - --copy-service
          image: 'osixia/openldap:1.5.0'    
          ports:
            - name: tcp-389
              containerPort: 389
              protocol: TCP
            - name: tcp-636
              containerPort: 636
              protocol: TCP
          env:
            - name: TZ
              value: Asia/Shanghai
            - name: LDAP_ORGANISATION
              value: "eryajf.net"
            - name: LDAP_DOMAIN
              value: "eryajf.net"
            - name: LDAP_ADMIN_PASSWORD
              value: "123456"
            - name: LDAP_BACKEND
              value: mdb
          resources:
            limits:
              cpu: 500m
              memory: 500Mi
            requests:
              cpu: 100m
              memory: 100Mi
          volumeMounts:
            - name: ldap-config-pvc
              mountPath: /etc/ldap/slapd.d
            - name: ldap-data-pvc
              mountPath: /var/lib/ldap
            - name: openldap-init
              mountPath: /container/service/slapd/assets/config/bootstrap/ldif/custom/init.ldif
              subPath: init.ldif
      volumes:
        - name: ldap-config-pvc
          persistentVolumeClaim:
            claimName: ldap-config-pvc
        - name: ldap-data-pvc
          persistentVolumeClaim:
            claimName: ldap-data-pvc
        - name: openldap-init
          configMap:
            name: openldap-init            
---
apiVersion: v1
kind: Service
metadata:
  name: openldap-svc
  namespace: open-ldap
  labels:
    app: openldap-svc
spec:
  ports:
  - name: tcp-389
    port: 389
    protocol: TCP
    targetPort: 389
  - name: tcp-636
    port: 636
    protocol: TCP
    targetPort: 636
  selector:
    app: openldap
EOF
kubectl apply -f ldap-deployment.yaml
```

## 2.4.设置环境变量

```bash
OPENLDAP_IP=`kubectl get svc -n open-ldap openldap-svc -o json | jq .spec.clusterIP | bc -l`
echo $OPENLDAP_IP
```

# 3.phpldapadmin

## 3.1.创建Deployment资源

镜像地址🙃：https://registry.hub.docker.com/r/osixia/phpldapadmin/tags

Client端参考地址： https://github.com/osixia/docker-phpLDAPadmin

```bash
cat >  ldap-phpldapadmin.yaml << EOF
kind: Deployment
apiVersion: apps/v1
metadata:
  name: ldap-phpldapadmin
  namespace: open-ldap
  labels:
    app: ldap-phpldapadmin
  annotations:
    app.kubernetes.io/alias-name: LDAP
    app.kubernetes.io/description: LDAP在线工具
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ldap-phpldapadmin
  template:
    metadata:
      labels:
        app: ldap-phpldapadmin
    spec:
      containers:
        - name: go-ldap-admin-phpldapadmin
          image: 'osixia/phpldapadmin:0.9.0'
          ports:
            - name: tcp-80
              containerPort: 80
              protocol: TCP
          env:
            - name: TZ
              value: Asia/Shanghai
            - name: PHPLDAPADMIN_HTTPS
              value: 'false'
            # openldap的svc
            - name: PHPLDAPADMIN_LDAP_HOSTS
              value: $OPENLDAP_IP
          resources:
            limits:
              cpu: 500m
              memory: 500Mi
            requests:
              cpu: 10m
              memory: 10Mi
---
apiVersion: v1
kind: Service
metadata:
  name: ldap-phpldapadmin-svc
  namespace: open-ldap
  labels:
    app: ldap-phpldapadmin-svc
spec:
  ports:
  - name: tcp-80
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: ldap-phpldapadmin
EOF
kubectl apply -f ldap-phpldapadmin.yaml
```

# 4.go-ldap-admin-server

官网：[http://ldapdoc.eryajf.net](http://ldapdoc.eryajf.net/)

参考博客：https://juejin.cn/post/7162859463421100040#heading-4

镜像地址🤐：https://hub.docker.com/r/eryajf/go-ldap-admin-server/tags

## 4.1.config.yml

```bash
cat > config.yml <<EOF
# delelopment
system:
  # 设定模式(debug/release/test,正式版改为release)
  mode: debug
  # url前缀
  url-path-prefix: api
  # 程序监听端口
  port: 8888
  # 是否初始化数据(没有初始数据时使用, 已发布正式版改为false)
  init-data: true
  # rsa公钥文件路径(config.yml相对路径, 也可以填绝对路径)
  rsa-public-key: go-ldap-admin-pub.pem
  # rsa私钥文件路径(config.yml相对路径, 也可以填绝对路径)
  rsa-private-key: go-ldap-admin-priv.pem

logs:
  # 日志等级(-1:Debug, 0:Info, 1:Warn, 2:Error, 3:DPanic, 4:Panic, 5:Fatal, -1<=level<=5, 参照zap.level源码)
  level: -1
  # 日志路径
  path: logs
  # 文件最大大小, M
  max-size: 50
  # 备份数
  max-backups: 100
  # 存放时间, 天
  max-age: 30
  # 是否压缩
  compress: false

database:
  # 数据库类型 mysql sqlite3
  driver: mysql
  # 数据库连接sqlite3数据文件的路径
  source: go-ldap-admin.db

mysql:
  # 用户名
  username: ldap
  # 密码
  password: Eryajf@123
  # 数据库名
  database: go_ldap_admin
  # 主机地址
  host: $MYSQL_IP
  # 端口
  port: 3306
  # 连接字符串参数
  query: parseTime=True&loc=Local&timeout=10000ms
  # 是否打印日志
  log-mode: true
  # 数据库表前缀(无需再末尾添加下划线, 程序内部自动处理)
  table-prefix: tb
  # 编码方式
  charset: utf8mb4
  # 字符集(utf8mb4_general_ci速度比utf8mb4_unicode_ci快些)
  collation: utf8mb4_general_ci

# casbin配置
casbin:
  # 模型配置文件, config.yml相对路径
  model-path: 'rbac_model.conf'

# jwt配置
jwt:
  # jwt标识
  realm: test jwt
  # 服务端密钥
  key: secret key
  # token过期时间, 小时
  timeout: 12000
  # 刷新token最大过期时间, 小时
  max-refresh: 12000

# 令牌桶限流配置
rate-limit:
  # 填充一个令牌需要的时间间隔,毫秒
  fill-interval: 50
  # 桶容量
  capacity: 200

# email configuration
email:
  port: '465'
  user: 'w1354586675@163.com'
  from: 'go-ldap-admin后台'
  host: 'smtp.163.com'
  # is-ssl: true
  pass: 'FRURSIIFAARNZJTR'

# # ldap 配置
ldap:
  # ldap服务器地址
  url: ldap://$OPENLDAP_IP:389
  # ladp最大连接数设置
  max-conn: 10
  # ldap服务器基础DN
  base-dn: "dc=eryajf,dc=net"
  # ldap管理员DN
  admin-dn: "cn=admin,dc=eryajf,dc=net"
  # ldap管理员密码
  admin-pass: "123456"
  # ldap用户OU
  user-dn: "ou=people,dc=eryajf,dc=net"
  # ldap用户初始默认密码
  user-init-password: "123456"
  # 是否允许更改分组DN
  group-name-modify: false
  # 是否允许更改用户DN
  user-name-modify: false
# 📢 即便用不到如下三段配置信息，也不要删除，否则会有一些奇怪的错误出现
dingtalk:
  # 配置获取详细文档参考： http://ldapdoc.eryajf.net/pages/94f43a/
  flag: "dingtalk" # 作为钉钉在平台的标识
  app-key: "xxxxxxxxxxxxxxx" # 应用的key
  app-secret: "xxxxxxxxxxxxxxxxxxxxxxxxxxxx" # 应用的secret
  agent-id: "12121212" # 目前agent-id未使用到，可忽略
  enable-sync: false  # 是否开启定时同步钉钉的任务
  dept-sync-time: "0 30 2 * * *" # 部门同步任务的时间点 * * * * * * 秒 分 时 日 月 周, 请把时间设置在凌晨 1 ~ 5 点
  user-sync-time: "0 30 3 * * *" # 用户同步任务的时间点 * * * * * * 秒 分 时 日 月 周, 请把时间设置在凌晨 1 ~ 5 点,注意请把用户同步的任务滞后于部门同步时间,比如部门为2点,则用户为3点
wecom:
  # 配置获取详细文档参考：http://ldapdoc.eryajf.net/pages/cf1698/
  flag: "wecom" # 作为微信在平台的标识
  corp-id: "xxxx" # 企业微信企业ID
  agent-id: 1000003 # 企业微信中创建的应用ID
  corp-secret: "xxxxx" # 企业微信中创建的应用secret
  enable-sync: false # 是否开启定时同步企业微信的任务
  dept-sync-time: "0 30 2 * * *" # 部门同步任务的时间点 * * * * * * 秒 分 时 日 月 周, 请把时间设置在凌晨 1 ~ 5 点
  user-sync-time: "0 30 3 * * *" # 用户同步任务的时间点 * * * * * * 秒 分 时 日 月 周, 请把时间设置在凌晨 1 ~ 5 点,注意请把用户同步的任务滞后于部门同步时间,比如部门为2点,则用户为3点
feishu:
  # 配置获取详细文档参考：http://ldapdoc.eryajf.net/pages/83c90b/
  flag: "feishu" # 作为飞书在平台的标识
  app-id: "xxxxxxx" # 飞书的app-id
  app-secret: "xxxxxxxxxxx" # 飞书的app-secret
  enable-sync: false  # 是否开启定时同步飞书的任务
  dept-sync-time: "0 20 0 * * *" # 部门同步任务的时间点 * * * * * * 秒 分 时 日 月 周, 请把时间设置在凌晨 1 ~ 5 点
  user-sync-time: "0 40 0 * * *" # 用户同步任务的时间点 * * * * * * 秒 分 时 日 月 周, 请把时间设置在凌晨 1 ~ 5 点,注意请把用户同步的任务滞后于部门同步时间,比如部门为2点,则用户为3点
  dept-list:      #部门列表，不设置则使用公司根部门，如果不希望添加子部门，在开头加上^
    # - "^od-xxx"
    # - "od-xxx"
  enable-bot-inform: false
EOF
```

- 生成configmap

```bash
kubectl create configmap config --from-file=./config.yml -n open-ldap
```

## 4.2.创建Deployment资源

```bash
cat > go-ldap-admin.yaml <<EOF
kind: Deployment
apiVersion: apps/v1
metadata:
  name: go-ldap-admin
  namespace: open-ldap
  labels:
    app: go-ldap-admin
  annotations:
    app.kubernetes.io/alias-name: go-ldap-admin
spec:
  replicas: 1
  selector:
    matchLabels:
      app: go-ldap-admin
  template:
    metadata:
      labels:
        app: go-ldap-admin
    spec:
      volumes:
        - name: config
          configMap:
            name: config    
      containers:
        - name: go-ldap-admin-server
          image: eryajf/go-ldap-admin-server
          env:
          # mysql的svc, openldap的svc，记得telnet下端口
          - name: WAIT_HOSTS
            value: $MYSQL_IP:3306, $OPENLDAP_IP:389
          volumeMounts:
            - name: config
              mountPath: /app/config.yml
              subPath: config.yml
          ports:
            - name: tcp-8888
              containerPort: 8888
              protocol: TCP
          resources:
            limits:
              cpu: 500m
              memory: 500Mi
            requests:
              cpu: 10m
              memory: 10Mi
---
apiVersion: v1
kind: Service
metadata:
  name: go-ldap-admin-server 
  namespace: open-ldap
  labels:
    app: go-ldap-admin-server
spec:
  ports:
  - name: tcp-8888
    port: 8888
    protocol: TCP
    targetPort: 8888
  selector:
    app: go-ldap-admin
EOF
kubectl apply -f go-ldap-admin.yaml
```

# 5.go-ldap-admin-ui

镜像地址🫤：https://hub.docker.com/r/eryajf/go-ldap-admin-ui/tags

## 5.1.创建Deployment资源

```bash
cat > go-ldap-admin-ui.yaml <<EOF
kind: Deployment
apiVersion: apps/v1
metadata:
  name: go-ldap-admin-ui
  namespace: open-ldap
  labels:
    app: go-ldap-admin-ui
  annotations:
    app.kubernetes.io/alias-name: go-ldap-admin-ui
spec:
  replicas: 1
  selector:
    matchLabels:
      app: go-ldap-admin-ui
  template:
    metadata:
      labels:
        app: go-ldap-admin-ui
    spec:
      containers:
        - name: go-ldap-admin-ui
          image: eryajf/go-ldap-admin-ui
          ports:
            - name: tcp-80
              containerPort: 80
              protocol: TCP
          env:
          - name: TZ
            value: Asia/Shanghai
          resources:
            limits:
              cpu: 500m
              memory: 500Mi
            requests:
              cpu: 10m
              memory: 10Mi
---
apiVersion: v1
kind: Service
metadata:
  name: go-ldap-admin-ui-svc
  namespace: open-ldap
  labels:
    app: go-ldap-admin-ui-svc
spec:
  type: ClusterIP 
  ports:
  - name: tcp-80
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: go-ldap-admin-ui
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/service-weight: ""
  name: openldap-ingress
  namespace: open-ldap
spec:
  ingressClassName: nginx
  rules:
  - host: openldap.cloud.com
    http:
      paths:
      - backend:
          service:
            name: go-ldap-admin-ui-svc
            port:
              number: 80
        path: /
        pathType: Prefix
EOF
kubectl create -f go-ldap-admin-ui.yaml
```

结尾：如果出现502报错，需要看下数据库是否创建，连接是否配置正确。

账号：`admin`，密码：`123456`

![img](https://img2023.cnblogs.com/blog/1740081/202305/1740081-20230528153025666-1198127967.png)

![img](https://img2023.cnblogs.com/blog/1740081/202305/1740081-20230528153129874-764390654.png)