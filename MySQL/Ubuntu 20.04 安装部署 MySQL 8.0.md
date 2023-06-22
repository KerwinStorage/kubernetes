# 1. 规划

官网：[MySQL 社区官网](https://dev.mysql.com/downloads/mysql/8.0.html)，[下载包](https://downloads.mysql.com/archives/community/)需要创建oracle账户，本地使用的是Linux 通用的二进制包`mysql-8.0.31-linux-glibc2.12-x86_64.tar` md5：`89e902edeb75216c366e878f3c9e85be`

| hostname | IP address     | 系统版本       |
| -------- | -------------- | -------------- |
| db01     | 192.168.80.100 | Ubuntu 20.04.5 |

```bash
wget https://downloads.mysql.com/archives/get/p/23/file/mysql-8.0.31-linux-glibc2.12-x86_64.tar.xz
```

# 2. MySQL 8.0.x安装过程

## 2.1 安装准备

```bash
groupadd mysql
useradd -r -g mysql -s /bin/false mysql # 用户不登录

# 数量路径
mkdir -p /data/mysql/data_3306

# 日志路径
mkdir -p /data/mysql/binlog_3306
cd /usr/local && tar xf mysql-8.0.31-linux-glibc2.12-x86_64.tar.xz
ln -s mysql-8.0.31-linux-glibc2.12-x86_64 mysql

# 环境变量
vim /etc/profile
export PATH=$PATH:/usr/local/mysql/bin
source /etc/profile

# 安装依赖
apt-get install libtinfo5

which mysql
/usr/local/mysql/bin/mysql
# 授权
chown -R mysql.mysql /data/ /usr/local/mysql/
```

## 2.2 初始化

```bash
apt-get install libaio-dev -y
# 初始化数据库
mysqld --initialize-insecure --user=mysql --basedir=/usr/local/mysql --datadir=/data/mysql/data_3306
# 创建临时配置文件
cat > /etc/my.cnf <<EOF 
[mysqld]
user=mysql
basedir=/usr/local/mysql
datadir=/data/mysql/data_3306
socket=/tmp/mysql.sock
server_id=6
log_bin=/data/mysql/binlog_3306
port=3306
[mysql]
socket=/tmp/mysql.sock
EOF
```

`--initialize-insecure`：不安全的 , 没有密码,没有密码策略

`--initialize`：安全的模式,没次初始化完成都会生成临时密码,只能登陆数据库,需要人为修改后才能正常管理数据库。

## 2.3 准备启动文件

```bash
cp /usr/local/mysql/support-files/mysql.server  /etc/init.d/mysql
cat > /usr/lib/systemd/system/mysql.service <<EOF
#内容如下：
[Unit]
Description=MySQL
SourcePath=/etc/init.d/mysql
Before=shutdown.target

[Service]
User=mysql
Type=forking
TimeoutSec=0
PermissionsStartOnly=true
ExecStart=/usr/local/mysql/bin/mysqld --defaults-file=/etc/my.cnf --user=mysql --daemonize
ExecStop=/etc/init.d/mysql stop
LimitNOFILE = 65535
Restart=on-failure
RestartSec=10
RestartPreventExitStatus=1
PrivateTmp=false

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl start mysql.service
systemctl enable mysql.service
```

几个systemctl常用的命令：

```bash
# 列出所有可用单元
systemctl list-unit-files

# 从可用单元中查看是否有mysql
systemctl list-unit-files | grep mysql

# 启动mysql服务
systemctl start mysql.service

# 查看mysql服务状态active即为启动状态，inactive是非启动状态
systemctl status mysql.service


# 重启mysql服务
systemctl restart mysql.service

# 停止mysql服务
systemctl stop mysql.service
```

附录：

```bash
# 设置密码
mysql
use mysql;
alter user 'root'@'localhost' identified by '1qazZSE$';
flush privileges;
#退出重新登录
mysql -uroot -p1qazZSE$
```

 