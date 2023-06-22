# 更新阿里epel源

```bash
curl -o /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo    --- 扩展源信息
yum  makecache   --更新yum源信息
```

# 安装inoutfy-tools

```bash
yum install -y inotify-tools
[root@nfs01 ~]# rpm   -ql    inotify-tools

/usr/bin/inotifywait <--- 实现对数据目录信息变化监控（重点了解的命令）
/usr/bin/inotifywatch <--- 监控数据信息变化，对变化的数据进行统计  
```

# 文件目录信息

```bash
[root@web01 inotify]# ls
max_queued_events  max_user_instances  max_user_watches


max_user_watches: 设置inotifywait或inotifywatch命令可以监视的文件数量（单进程）
默认只能监控8192个文件

max_user_instances: 设置每个用户可以运行的inotifywait或inotifywatch命令的进程数
默认每个用户可以开启inotify服务128个进程

max_queued_events: 设置inotify实例事件（event）队列可容纳的事件数量
默认监控事件队列长度为16384
```

# 部署rsync服务

客户端、服务端安装rsync

```bash
# 1.安装
yum install -y rsync

# 2.服务端页面配置文件
uid = rsync
gid = rsync
port = 873
fake super = yes
use chroot = no
max connections = 200
timeout = 600
ignore errors
read only = false
list = false
auth users = rsync_backup
secrets file = /etc/rsync.passwd
log file = /var/log/rsyncd.log
#####################################
[zls]
comment = welcome to oldboyedu backup!
path = /backup


# 2.接着创建用户
[root@backup ~]# id rsync               #检查用户是否存在
id: rsync: no such user
#创建用户（不允许登录，不创建家目录）
[root@backup ~]# useradd rsync -s /sbin/nologin -M

# 3.创建一个备份目录
[root@backup ~]# mkdir /backup          #创建目录backup
#授权rsync用户
[root@backup ~]# chown -R rsync.rsync /backup/

# 4.创建虚拟用户及密码文件
[root@backup ~]# vim /etc/rsync.passwd
rsync_backup:123456                     #用户名:密码
[root@backup ~]# chmod 600 /etc/rsync.passwd
创建虚拟用户及密码文件
vim /etc/rsync.passwd
rsync_backup:123456

chmod 600 /etc/rsync.passwd    #授权

# 5.启动rsync添加开机自启
ll /usr/lib/systemd/system/rsyncd.service
-rw-r--r-- 1 root root 237 Apr 26 01:17 /usr/lib/systemd/system/rsyncd.service

systemctl start rsyncd
systemctl enable rsyncd

# 6.检查rsync 873 端口是否处于监听状态
netstat -antup | grep 873

# 7.rsync 客户端仅需配置虚拟用户的密码,并授权为600安全权限
yum install rsync -y
echo " 123456 " >  /etc/rsync.pass
chmod 600 /etc/rsync.pass
```

### rsync服务端部署

- 检查rsync软件是否已经安装
- 编写rsync软件主配置文件
- 创建备份目录管理用户
- 创建备份目录，并进行授权
- 创建认证文件，编写认证用户和密码信息，设置文件权限为600
- 启动rsync守护进程服务

**rsync客户端部署**

- 检查rsync软件是否已经安装
- 创建认证文件，编写认证用户密码信息即可，设置文件权限为600
- 利用客户端进行数据同步测试

```bash
rsync -avz /backup  rsync_backup@172.16.1.41::backup/`hostname  -i` --passworld-file=/etc/rsync.passworld
```

**inotify软件应用命令：**

inotifywait

- -m|--monitor：始终保持事件监听状态
- -r：进行递归监控
- -q|--quiet：将无用的输出信息，不进行显示
- --timefmt <fmt>：设定日期的格式
- man strftime：获取更多时间参数信息
- --format <fmt>：命令执行过程中，输出的信息格式
- -e：指定监控的事件信息

man inotifywait 查看所有参数说明和所有可以监控的事件信息

**总结主要用到的事件信息：**

**create创建、delete删除、moved_to移入、close_write修改**

inotifywait -mrq --timefmt "%F" --format "%T %w%f 事件信息：%e" /data <-- 相对完整的命令应用

inotifywait -mrq --timefmt "%F" --format "%T %w%f 事件信息：%e" -e create /data <-- 指定监控什么事件信息

inotifywait -mrq --format "%w%f" -e create,delete,moved_to,close_write /data

**以上为实现实时同步过程，所需要的重要监控命令**

```bash
cat <<"END" > /root/sync.sh
#!/bin/bash
Monitor="inotifywait -mrq -e modify,create,attrib,move,delete /data/scripts"
Sync="rsync -azH --delete /data/scripts/ root@repo:/data/scripts/"
$Monitor | while read DIRECTORY EVERT FILE
do
        $Sync
done
END

chmod +x /root/sync.sh
```

**注：两台主机之间已经实现了免密登录所以没有使用密码参数。** 

运行脚本

```bash
echo "/root/sync.sh" >> /etc/rc.local #开机自启
nohup /root/sync.sh & #后台运行脚本
```

 验证

```bash
cd /data/scripts/
echo "test" > 1.txt
ls
# 查看是否同步
```

 