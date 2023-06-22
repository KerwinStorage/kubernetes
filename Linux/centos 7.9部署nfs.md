## 一、 前言

本篇文章是为了研究`StorageClass`在`centos`上使用`nfs`方式同步，所有node节点都需要挂载目录。🧑‍💻🧑‍💻🧑‍💻

节点：

| ip address     | hostname | 系统                                 | nfs服务 |
| -------------- | -------- | ------------------------------------ | ------- |
| 192.168.80.100 | nfs01    | CentOS Linux release 7.9.2009 (Core) | 服务端  |
| 192.168.80.200 | nfs02    | CentOS Linux release 7.9.2009 (Core) | 客户端  |

## 二、安装nfs

- **所有主机操作**

```bash
# 1.查看系统版本
cat /etc/redhat-release
CentOS Linux release 7.9.2009 (Core)
# 2.查看并关闭防火墙
systemctl status firewalld
systemctl stop firewalld
systemctl disable firewalld
# 3.永久关闭
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=disabled/' /etc/selinux/config
```

安装 NFS 和 RPC

- **所有主机操作**

```bash
rpm -qa | grep nfs
rpm -qa | grep rpcbind
yum install -y nfs-utils rpcbind

rpm -qa | grep nfs
nfs4-acl-tools-0.3.3-21.el7.x86_64
nfs-utils-1.3.0-0.68.el7.2.x86_64
libnfsidmap-0.25-19.el7.x86_64
rpm -qa | grep rpcbind
rpcbind-0.2.0-49.el7.x86_64
```

## 三、服务端配置

- `192.168.80.100`

```bash
mkdir -p /root/data/k8s
echo "/root/data/k8s *(rw,sync,no_root_squash,fsid=0)" >> /etc/exports
# 1.生效配置
exportfs -r
# 2.启动
systemctl enable rpcbind && systemctl start rpcbind
systemctl enable nfs && systemctl restart nfs
# 3.服务器注册端口：111
rpcinfo -p
# 4.查看是否成功
showmount -e localhost
Export list for localhost:
/root/data/k8s *

# 5.查看启动挂载
cat /var/lib/nfs/etab
/root/data/k8s       *(rw,sync,wdelay,hide,nocrossmnt,secure,no_root_squash,no_all_squash,no_subtree_check,secure_locks,acl,no_pnfs,fsid=0,anonuid=65534,anongid=65534,sec=sys,rw,secure,no_root_squash,no_all_squash)
```

## 四、客户端

- `192.168.80.200`

```bash
# 1.创建挂载目录
mkdir -p /root/data/k8s
# 2.查看挂载ip
showmount -e 192.168.80.100
# 3.挂载
mount -t nfs 192.168.80.100:/root/data/k8s /root/data/k8s
# 4.查看磁盘
df -h
Filesystem                Size  Used Avail Use% Mounted on
devtmpfs                  471M     0  471M   0% /dev
tmpfs                     487M     0  487M   0% /dev/shm
tmpfs                     487M  8.4M  478M   2% /run
tmpfs                     487M     0  487M   0% /sys/fs/cgroup
/dev/sda3                  78G  5.1G   73G   7% /
/dev/sda1                 297M  152M  145M  52% /boot
tmpfs                      98M   12K   98M   1% /run/user/42
tmpfs                      98M     0   98M   0% /run/user/0
192.168.80.200:/root/nfs/data   78G  5.2G   73G   7% /root/nfs/data
# 5.启动挂载
cat /etc/fstab
192.168.80.100:/root/nfs/data            /root/nfs/data           nfs     defaults        1 1
```

## 五、测试同步

```bash
# 1.客户端
cd /root/nfs/data && touch 1.txt
# 2.服务端
cd /root/nfs/data && ls 1.txt
```

 