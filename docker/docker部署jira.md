## docker服务部署

### MySQL数据库部署

文章参考：[http://ghang.top/2019/08/09/Docker-%E9%83%A8%E7%BD%B2Jira8-1-0/](http://ghang.top/2019/08/09/Docker-部署Jira8-1-0/)

数据库文献参考：https://confluence.atlassian.com/adminjiraserver/connecting-jira-applications-to-mysql-5-7-966063305.html

安装步骤文献参考：https://confluence.atlassian.com/adminjiraserver080/running-the-setup-wizard-967896939.html

1. 起数据库

```
docker run --name mysql \
    --restart always \
    -p 3306:3306 \
    -e MYSQL_ROOT_PASSWORD=jira2019 \
    -v mysql_data:/var/lib/mysql \
    -v mysql_conf:/etc/mysql/conf.d \
    -v mysql_backup:/backup \
    -d mysql:5.7
```

2. 修改配置文件

```
vim /etc/mysql/conf.d/mysql.cnf
[client]
default-character-set = utf8

[mysql]
default-character-set = utf8

[mysqld]
character_set_server = utf8
collation-server = utf8_bin
transaction_isolation = READ-COMMITTED
max_allowed_packet = 100M
innodb_log_file_size=256M
```

3. 创建jira数据库

```
#删除数据库
drop database jira;  <----会删除之前留下的数据
create database jira character set 'UTF8';
create user jira identified by 'jira';
grant  all privileges on `jira`.* to  'jira'@'172.%' identified by 'jira' with grant option;
grant all privileges on `jira`.* to 'jira'@'localhost' identified by 'jira' with grant option;
<----这里可以根据自己的网段加上--->
flush privileges;
alter database jira character set utf8 collate utf8_bin;
alter database confluence character set utf8 collate utf8_bin;
-- confluence要求设置事务级别为READ-COMMITTED
set global tx_isolation='READ-COMMITTED';
```

4. 进入容器重启mysql

```
/etc/init.d/mysql stop 
/etc/init.d/mysql start
```

## jira服务部署

### docker起jira

官方地址：https://www.atlassian.com/software/jira

镜像官方地址：https://hub.docker.com/search?q=jira&type=image

**注：这里网页登录是映射到宿主机85端口。**

```
docker run --name jira \
    --restart always \
    --link mysql:mysql \
    -e ATL_PROXY_NAME=192.168.74.128:85 \
    -e ATL_PROXY_PORT=85 \
    -e ATL_TOMCAT_SCHEME=http \
    -e JVM_SUPPORT_RECOMMENDED_ARGS=-Duser.timezone=Asia/Shanghai \
    -p 85:8080 \
    -v jira_data:/var/atlassian/application-data/jira \
    -v jira_opt:/opt/atlassian/jira \
    -d atlassian/jira-software:8.0.3
```

**注：这里的配置的端口和自己映射宿主机的端口要一致，否则会出现创建项目时出错，XSRF检查失败。**

 ![img](https://img2020.cnblogs.com/blog/1740081/202010/1740081-20201020083136880-1615481386.png)

登录：宿主机ip:85

> 这里破解跟confluence略有不同，jira破解需要先按照提示去官方获取试用授权码，这样才可以进入系统，然后进行破解包的替换，重启服务即可。

后续更新安装和破解

 破解软件地址：https://gitee.com/pengzhile/atlassian-agent

- 直接下载本项目[release](https://gitee.com/pengzhile/atlassian-agent/releases)包。

需要破解的id：

 ![img](https://img2020.cnblogs.com/blog/1740081/202010/1740081-20201023135242496-1415337667.png)

破解 

![img](https://img2020.cnblogs.com/blog/1740081/202010/1740081-20201023135359104-730385148.png)

正在破解

![img](https://img2020.cnblogs.com/blog/1740081/202010/1740081-20201023135436988-678767241.png)

设置管理员账户和密码

![img](https://img2020.cnblogs.com/blog/1740081/202010/1740081-20201023135758029-1930379103.png)

https://www.jianshu.com/p/b95ceabd3e9d