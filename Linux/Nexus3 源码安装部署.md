## Nexus介绍

**参考地址：**http://www.tinygroup.org/docs/63dd905967cf4474afee222368da9a84

[Nexus](http://nexus.sonatype.org/) 是Maven仓库管理器，如果你使用Maven，你可以从[Maven中央仓库](http://repo1.maven.org/maven2/) 下载所需要的构件（artifact），但这通常不是一个好的做法，你应该在本地架设一个Maven仓库服务器，在代理远程仓库的同时维护本地仓库，以节省带宽和时间，Nexus就可以满足这样的需要。此外，他还提供了强大的仓库管理功能，构件搜索功能，它基于REST，友好的UI是一个extjs的REST客户端，它占用较少的内存，基于简单文件系统而非数据库。这些优点使其日趋成为最流行的Maven仓库管理器。

### 私服简介

私服是指私有服务器，是架设在局域网的一种特殊的远程仓库，目的是代理远程仓库及部署第三方构建。有了私服之后，当 Maven 需要下载构件时，直接请求私服，私服上存在则下载到本地仓库；否则，私服请求外部的远程仓库，将构件下载到私服，再提供给本地仓库下载。

![img](https://img2020.cnblogs.com/blog/1740081/202005/1740081-20200514202637567-1969408652.png)

 

### 安装jdk

**注：nexus由于是java语言开发的，所以依赖java环境，要先确认主机上有java环境。**

**jdk8下载地址：**[点我点我](https://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html) **这里我们使用jdk-8u241.rpm包二进制安装**

```
yum localinstall jdk-8u241-linux-x64.rpm -y  #安装

java -version  #查看版本
java version "1.8.0_241"
Java(TM) SE Runtime Environment (build 1.8.0_241-b07)
Java HotSpot(TM) 64-Bit Server VM (build 25.241-b07, mixed mode)
```

### 安装maven

**maven下载地址：**[点我点我](https://mirrors.tuna.tsinghua.edu.cn/apache/maven/maven-3/3.3.9/)

```
wget https://mirrors.tuna.tsinghua.edu.cn/apache/maven/maven-3/3.6.3/binaries/apache-maven-3.6.3-bin.tar.gz
tar xf apache-maven-3.6.3-bin.tar.gz -C /usr/local
ll /usr/local/apache-maven-3.6.3/
ln -s /usr/local/apache-maven-3.6.3/ /usr/local/maven
ll /usr/local/maven/
vim /etc/profile
#文件结尾添加两行
export M2_HOME=/usr/local/maven
export PATH=${M2_HOME}/bin:$PATH

source /etc/profile
mvn -v    #查看版本 相对应的java maven 和内核信息

常用命令
#这种方式打出的包默认是 jar包，默认下载maven的中央仓库下载依赖和插件等,这里的速度会有点慢，因为是访问的国外的地址拉倒本地。后续需要调整为私服方式，如果java代码多的情况下
mvn package  
mvn clean
mvn test
mvn install
```

**更改maven源**

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

## 源码安装nexus3

官方地址：[点我点我](https://blog.sonatype.com/) 下载地址：[点我点我](https://www.sonatype.com/oss-thank-you-tar.gz)

**注：这里部署官网无法下载我们需要的包，从网上找的包。** **网盘地址：**[点击这里](https://pan.baidu.com/s/1PyJ5a7IDrERKBDghdxxMig) **提取码：**pl1b

> **在`/usr/local/nexus-3.18.1-01/etc`的目录下有一个`nexus-default.properties`文件配置服务端口**
>
> **默认：8081**

**安装**

```
tar xf nexus-3.18.1-01-unix.tar.gz  -C /usr/local/  #解压


[root@docker01 local]# ll nexus-3.18.1-01/
total 72
drwxr-xr-x  3 root root    73 May 14 15:18 bin
drwxr-xr-x  2 root root    26 May 14 15:18 deploy
drwxr-xr-x  7 root root   104 May 14 15:23 etc
drwxr-xr-x  4 root root   184 May 14 15:18 lib
-rw-r--r--  1 root root   395 Aug  7  2019 NOTICE.txt
-rw-r--r--  1 root root 17321 Aug  7  2019 OSS-LICENSE.txt
-rw-r--r--  1 root root 39222 Aug  7  2019 PRO-LICENSE.txt
drwxr-xr-x  3 root root  4096 May 14 15:18 public
drwxr-xr-x 20 root root  4096 May 14 15:18 system
 
[root@docker01 local]# ll sonatype-work/
total 0
drwxr-xr-x 15 root root 253 May 14 16:07 nexus3
```

### 配置开机自启动

**进入`/etc/init.d`目录，新建脚本文件nexus：**

**注意：java的指定目录使用`which java`查看，这里二进制安装是默认目录**

```
##进入/etc/init.d
[root@docker01 ~]# cd /etc/init.d/

[root@docker01 init.d]# vim nexus
#脚本内容：
#!/bin/bash
#chkconfig:2345 20 90
#description:nexus
#processname:nexus

export JAVA_HOME=/usr/bin/java

case $1 in
        start) su root /usr/local/nexus-3.18.1-01/bin/nexus start;;
        stop) su root /usr/local/nexus-3.18.1-01/bin/nexus stop;;
        status) su root /usr/local/nexus-3.18.1-01/bin/nexus status;;
        restart) su root /usr/local/nexus-3.18.1-01/bin/nexus restart;;
        dump) su root /usr/local/nexus-3.18.1-01/bin/nexus dump;;
        console) su root /usr/local/nexus-3.18.1-01/bin/nexus console;;
        *) echo "Usage: nexus {start|stop|run|run-redirect|status|restart|force-reload}"
esac
```

**设置脚本权限：**

```
[root@docker01 ~]# chmod +x /etc/init.d/nexus
```

**使用service命令使用nexus：**

```
[root@docker01 ~]# service nexus status
WARNING: ************************************************************
WARNING: Detected execution as "root" user.  This is NOT recommended!
WARNING: ************************************************************
nexus is running.
```

**添加到开机启动：**

```
[root@docker01 ~]# chkconfig nexus on
```

**查看nexus开机启动：**

```
[root@docker01 ~]# chkconfig --list nexus

Note: This output shows SysV services only and does not include native
      systemd services. SysV configuration data might be overridden by native
      systemd configuration.

      If you want to list systemd services use 'systemctl list-unit-files'.
      To see services enabled on particular target use
      'systemctl list-dependencies [target]'.

nexus              0:off    1:off    2:on    3:on    4:on    5:on    6:off
```

**设置防火墙：**

```
[root@docker01 ~]# cd /etc/firewalld/zones/
[root@docker01 ~]# vim public.xml
添加以下放开端口内容, 其它不变
  <rule family="ipv4">
  <!-- 开放8081端口给任意ip  -->
　　<port protocol="tcp" port="8081"/>
　　<accept/>
  </rule>
```

### 查看端口

```
[root@docker01 ~]# netstat -lntup | grep 8081
tcp        0      0 0.0.0.0:8081            0.0.0.0:*               LISTEN      104016/java
```

## web页面操作

**登录：**ip地址:8081

![img](https://img2020.cnblogs.com/blog/1740081/202005/1740081-20200514203205416-2035728480.png)

```
cat /usr/local/sonatype-work/nexus3/admin.password
6977e617-8a13-4a84-bcd3-586c42bc06a9  ##admin的密码
```

初始密码：

![img](https://img2020.cnblogs.com/blog/1740081/202005/1740081-20200514203857341-1232140399.png)

### 修改密码

![img](https://img2020.cnblogs.com/blog/1740081/202005/1740081-20200514204010661-562306747.png)

![img](https://img2020.cnblogs.com/blog/1740081/202005/1740081-20200514204036787-1456477264.png)

 