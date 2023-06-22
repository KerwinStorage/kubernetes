前言：**此文档是跟着课程来的，主要是为了熟悉SpringCloud 和 kubernetes是怎么结合的，后续用在测试cicd流水线上**。

相关文档：

- 中文文档：[https://www.springcloud.cc](https://www.springcloud.cc/) https://www.springcloud.cc/spring-cloud-greenwich.html

# 1.基础环境

## 1.1.java环境配置

JDK8 链接: [下载](https://pan.baidu.com/s/1QW66-QJURanKzSRnWPlPCA?pwd=j3ie)

```
mkdir -p /usr/local/src/jdk ;  cd /usr/local/src/jdk
tar -zxvf jdk-8u221-linux-x64.tar.gz -C /usr/local
vim /etc/profile
export JAVA_HOME=/usr/local/jdk1.8.0_221
export PATH=$JAVA_HOME/bin:$PATH
export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tool.jar
source /etc/profile
# 查看java版本
java -version
```

## 1.2.maven环境配置

下载：

```
wget https://mirrors.tuna.tsinghua.edu.cn/apache/maven/maven-3/3.6.3/binaries/apache-maven-3.6.3-bin.tar.gz
tar xf apache-maven-3.6.3-bin.tar.gz -C /usr/local
ln -s /usr/local/apache-maven-3.6.3/ /usr/local/maven
vim /etc/profile
#文件结尾添加两行
export M2_HOME=/usr/local/maven
export PATH=${M2_HOME}/bin:$PATH
source /etc/profile
mvn -v
```

更改maven源：

```
vim  /usr/local/maven/conf/settings.xml
#将所有内容复制到<mirrors>之间
<mirror>
    <id>nexus-aliyun</id>  
    <mirrorOf>central</mirrorOf>    
    <name>Nexus aliyun</name>  
    <url>http://maven.aliyun.com/nexus/content/groups/public</url>  
</mirror>
```

# 2.测试源码

链接：[下载](https://pan.baidu.com/s/1MWQPgCM6iRF1FxiZeOq1Og?pwd=cb7x)

```
# 直接放在root目录下
unzip -d /root microservic-test.zip
```

# 3.连接初始化

注：这里是使用二进制部署的MySQL 8.0版本，数据授权会于5.*版本操作不一样。

创建数据库 tb_order、tb_product、tb_stock

```
# 创建数据库
create database tb_product;
create database tb_stock;
create database tb_order;

# 导入表
use tb_order;
source /root/microservic-test/db/order.sql

use tb_stock;
source /root/microservic-test/db/stock.sql

use tb_product;
source /root/microservic-test/db/product.sql

# 对 MySQL 数据库授权
create user 'root'@'%' identified by '1qazZSE$';
create user 'root'@'192.168.%.%' identified by '1qazZSE$';
create user 'root'@'10.196.%.%' identified by '1qazZSE$';
grant all privileges on *.* to 'root'@'%' with grant option;
grant all privileges on *.* to 'root'@'192.168.%.%' with grant option;
grant all privileges on *.* to 'root'@'10.196.%.%' with grant option;
flush privileges;
```

注：不授权会看不到数据。

## 3.1.修改源代码，更改数据库连接地址

```
# 改成自己的地址，账户，密码
vim /root/microservic-test/stock-service/stock-service-biz/src/main/resources/application-fat.yml
spring:
  datasource:
    url: jdbc:mysql://192.168.80.45:3306/tb_stock?characterEncoding=utf-8
    username: root
    password: 1qazZSE$
    driver-class-name: com.mysql.jdbc.Driver

eureka:
  instance:
    prefer-ip-address: true
  client:
    register-with-eureka: true
    fetch-registry: true
    service-url:
      defaultZone: http://eureka-0.eureka.ms:8888/eureka,http://eureka-1.eureka.ms:8888/eureka,http://eureka-2.eureka.ms:8888/eureka
      
      
vim /root/microservic-test/product-service/product-service-biz/src/main/resources/application-fat.yml
spring:
  datasource:
    url: jdbc:mysql://192.168.80.45:3306/tb_product?characterEncoding=utf-8
    username: root
    password: 1qazZSE$
    driver-class-name: com.mysql.jdbc.Driver

eureka:
  instance:
    prefer-ip-address: true
  client:
    register-with-eureka: true
    fetch-registry: true
    service-url:
      defaultZone: http://eureka-0.eureka.ms:8888/eureka,http://eureka-1.eureka.ms:8888/eureka,http://eureka-2.eureka.ms:8888/eureka

vim /root/microservic-test/order-service/order-service-biz/src/main/resources/application-fat.yml
spring:
  datasource:
    url: jdbc:mysql://192.168.80.45:3306/tb_order?characterEncoding=utf-8
    username: root
    password: 1qazZSE$
    driver-class-name: com.mysql.jdbc.Driver

eureka:
  instance:
    prefer-ip-address: true
  client:
    register-with-eureka: true
    fetch-registry: true
    service-url:
      defaultZone: http://eureka-0.eureka.ms:8888/eureka,http://eureka-1.eureka.ms:8888/eureka,http://eureka-2.eureka.ms:8888/eureka

# 过滤下
grep -nr '1qazZSE\$' /root/microservic-test
grep -nr '192.168.80.45' /root/microservic-test
```

# 4.Maven编译

```
cd /root/microservic-test
mvn clean package -D maven.test.skip=true
find /root/microservic-test -name "target"
```

# 5.构建镜像

注：这里用的是`java:openjdk-8u111-jdk-alpine`镜像。

## 5.1.创建命名空间

```
kubectl create ns ms
# 如果使用harbor请添加凭据
kubectl create secret docker-registry registry-pull-secret --docker-server=192.168.80.45 --docker-username=admin --docker-password=Harbor12345 -n ms
```

## 5.1.eureka-service

java镜像：[点击](https://hub.docker.com/_/java/tags)

构建eureka镜像

```
cd /root/microservic-test/eureka-service
docker build -t 192.168.80.45:5000/microservice/eureka:v1 .
docker push 192.168.80.45:5000/microservice/eureka:v1
```

部署服务：

```
cd /root/microservic-test/k8s
vim eureka.yaml
# 把镜像换成自己的，如果不需要凭据，这个可以去掉：imagePullSecrets
# 开始部署
kubectl apply -f eureka.yaml
kubectl get pod -n ms -owide  | grep eureka
eureka-0                   1/1     Running   0             12h   172.16.58.218   k8s-node02     <none>           <none>
eureka-1                   1/1     Running   1 (12h ago)   12h   172.16.85.212   k8s-node01     <none>           <none>
eureka-2                   1/1     Running   0             12h   172.16.58.228   k8s-node02     <none>           <none>
```

## 5.3.Gateway

构建Gateway镜像

```
cd /root/microservic-test/gateway-service
docker build -t 192.168.80.45:5000/microservice/gateway:v1 .
docker push 192.168.80.45:5000/microservice/gateway:v1
```

部署服务

```
cd /root/microservic-test/k8s
vim gateway.yaml
kubectl apply -f gateway.yaml
kubectl get pod -n ms -owide  | grep gateway
gateway-56b4f9995f-ncqj5   1/1     Running   0             11h   172.16.58.223   k8s-node02     <none>           <none>
gateway-56b4f9995f-p7ltg   1/1     Running   0             11h   172.16.85.211   k8s-node01     <none>           <none>
```

## 5.4.portal

构建portal镜像

```
cd /root/microservic-test/portal-service
docker build -t 192.168.80.45:5000/microservice/portal:v1 .
docker push 192.168.80.45:5000/microservice/portal:v1
```

部署服务

```
cd /root/microservic-test/k8s
vim portal.yaml
kubectl apply -f portal.yaml
kubectl get pod -n ms -owide  | grep portal
portal-59fdbffbc-7blk5     1/1     Running   0             94m   172.16.32.153   k8s-master01   <none>           <none>
```

## 5.5.order订单服务

构建order镜像

```
cd /root/microservic-test/order-service/order-service-biz
docker build -t 192.168.80.45:5000/microservice/order:v1 .
docker push 192.168.80.45:5000/microservice/order:v1
```

部署服务

```
cd /root/microservic-test/k8s
vim  order.yaml
kubectl apply -f order.yaml
kubectl get pod -n ms -owide  | grep order
order-6d7d8d56d5-l67ph     1/1     Running   0             59m   172.16.32.154   k8s-master01   <none>           <none>
```

## 5.6.product

构建product镜像

```
cd /root/microservic-test/product-service/product-service-biz
docker build -t 192.168.80.45:5000/microservice/product:v1 .
docker push 192.168.80.45:5000/microservice/product:v1
```

部署服务

```
cd /root/microservic-test/k8s
vim product.yaml
kubectl apply -f product.yaml
kubectl get pod -n ms -owide  | grep product
product-779d6b44fc-lpxwg   1/1     Running   0             55m   172.16.32.155   k8s-master01   <none>           <none>
```

## 5.7.库存 stock 服务

构建stock镜像

```
cd /root/microservic-test/stock-service/stock-service-biz
docker build -t 192.168.80.45:5000/microservice/stock:v1 .
docker push 192.168.80.45:5000/microservice/stock:v1
```

部署服务

```
cd /root/microservic-test/k8s
vim stock.yaml
kubectl apply -f stock.yaml
kubectl get pod -n ms -owide  | grep stock
kubectl get pod -n ms -owide  | grep stock
stock-7794d86576-fqhn4     1/1     Running   0             47m   172.16.32.156   k8s-master01   <none>           <none>
```

# 6.hosts解析

```
192.168.80.66 eureka.ctnrs.com
192.168.80.66 gateway.ctnrs.com
192.168.80.66 portal.ctnrs.com
```

这里ingress做了高可用和vip漂移，所以指向的地址都是一个。vip地址指向后台ingress的svc和端口（做了nodePort操作）每台node机器都会有一个ingress容器在上面所以访问谁都能访问到，然后创建ingress，访问域名就会访问后台ingress的svc和端口，然后分发到服务的svc ip和端口，服务的svc就会访问pod的地址和端口。

[http://eureka.ctnrs.com](http://eureka.ctnrs.com/)

![img](https://img2023.cnblogs.com/blog/1740081/202305/1740081-20230505110319307-1219033192.png)

http://portal.ctnrs.com

![img](https://img2023.cnblogs.com/blog/1740081/202305/1740081-20230505110507130-133981282.png)

![img](https://img2023.cnblogs.com/blog/1740081/202305/1740081-20230505110553143-67283874.png)

[http://gateway.ctnrs.com](http://gateway.ctnrs.com/)

![img](https://img2023.cnblogs.com/blog/1740081/202305/1740081-20230505110526090-1464335364.png)