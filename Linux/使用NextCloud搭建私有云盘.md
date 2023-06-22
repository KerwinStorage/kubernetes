## NextCloud概述

------

 

官网：https://nextcloud.com/

![img](https://img2020.cnblogs.com/blog/1740081/202004/1740081-20200417133118926-555988005.png)

NextCloud 一款文件主机服务软件，就是我们平时使用的云存储，但是是在搭建自己的私有云。本项目是基于PHP和 SQLite，MySQL，Oracle 戒 PostgreSQL 数据库，所以它可以运行在所有的平台上。

nextcloud 是一个开源免费专业的私有云存储项目，它能帮你快速在个人电脑戒服务器上架设一套专属的私有云文件同步网盘，可以像 百度云盘那样实现文件跨平台同步、共享、版本控制、团队协作等等。

nextcloud 能让你将所有的文件掌握在自己的手中，只要你的设备性能和空间充足，那么用起来几乎没有任何限制。

 

 

## LNMP环境搭建

------

 

### 环境准备

注：准备环境请看[官网文档](https://docs.nextcloud.com/server/15/admin_manual/installation/system_requirements.html)

```
yum install -y epel-release yum-utils unzip curl wget bash-completion policycoreutils-python mlocate bzip2
```

### 安装web服务和数据库服务

```
yum install -y httpd mariadb-server mariadb sqlite
```

### 安装php7.2

```
# 下载源
rpm -Uvh https://mirror.webtatic.com/yum/el7/webtatic-release.rpm
#php 和 nextcloud 需要的功能模块
yum install -y php72w php72w-cli php72w-common php72w-curl php72w-gd php72w-mbstring php72w-mysqlnd php72w-process php72w-xml php72w-zip php72w-opcache php72w-pecl-apcu php72w-intl php72w-pecl-redis
```

 

## 初始化LAMP网站架构

------

 

### 启动LAMP服务

```
systemctl start httpd.service
systemctl start mariadb.service
```

### 关闭防护墙和selinux

```
[root@ntp ~]# iptables -F
[root@ntp ~]# getenforce
Disabled
```

###  初始化mariadb数据库密码

```
mysqladmin -uroot  password "123456"
```

 

## 部署Nextcloud 私有云盘

------

 

软件下载地址：[https://nextcloud.com/install/#](https://nextcloud.com/install/)

### 网站目录

注：下载server端服务

```
wget https://download.nextcloud.com/server/releases/nextcloud-18.0.3.zip
unzip nextcloud-18.0.3.zip
# 拷贝 nextcloud 项目文件到网站目录
cp -r nextcloud/* /var/www/html/
# 创建数据目录
mkdir /var/www/html/data
# 授权
chown -R apache.apache /var/www/html/
```

### 创建数据库

```
mysql -uroot -p123456
create database nextcloud;
```

### web界面安装nextcloud

登录：http://10.0.0.88/index.php

扩展如果想了解其他类似软件请看这个链接：https://www.kancloud.cn/websoft9/cloudbox-practice/647724

![img](https://img2020.cnblogs.com/blog/1740081/202004/1740081-20200417133013134-1893856781.png)

页面展示：

![img](https://img2020.cnblogs.com/blog/1740081/202004/1740081-20200417133040614-1366170156.png)