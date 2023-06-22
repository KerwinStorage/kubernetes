## 查看系统版本

```
root@ubuntu:~# lsb_release -a
No LSB modules are available.
Distributor ID:    Ubuntu
Description:    Ubuntu 18.04.4 LTS
Release:    18.04
Codename:    bionic
```

## 修改配置文件

**注意：配置文件yaml语法格式，否则netplan命令无法生效**

```
nano /etc/netplan/01-network-manager-all.yaml
# Let NetworkManager manage all devices on this system
network:
  version: 2
  #renderer: NetworkManager
  ethernets:
        ens33:
            addresses: [192.168.74.128/24]
            gateway4: 192.168.74.2
            nameservers:
                addresses: [192.168.0.1,223.5.5.5]
```

- ens33：网卡名称
- address：自定义的ip地址
- gatway4：ip地址网关（**如果是虚拟机，需要和虚拟机设置的网关一致**）
- namserver：DNS解析（**这里如果需要多个DNS解析,需要用`,`号隔开。**）

然后使用下面的命令生效：

```
netplan apply
#或者
systemctl restart   systemd-networkd.service
```

注意以下几点：

1. 普通用户需要sudo去修改
2. 将renderer: NetworkManager注释掉
3. 配置文件需要使用yaml语法格式，每个配置项使用空格缩进表示层级
4. 对应配置项后面跟着冒号，之后要接个空格，否则netplan命令会报错
5. 之前版本配置网卡使用`/etc/network/interfaces`进行配置

```
auto ens33
iface ens33 inet static
address 192.168.74.128
netmask 255.255.255.0
gateway 192.168.74.2

#重启网卡
service restart network
```

　　6. 其他版本的配置文件可能不一样，文件命名为`50-cloud-init.yaml`，同一路径下。

## 配置DNS

方法一：临时文件

```bash
sudo vim /etc/resolv.conf
nameserver 8.8.8.8 //google的域名解析服务器 
nameserver 114.114.114.114 //联通的域名解析服务器
```

不需要重启就可以生效

方法二：永久生效

```bash
sudo apt install resolvconf
sudo systemctl enable resolvconf.service

sudo echo "nameserver 8.8.8.8" >>  /etc/resolvconf/resolv.conf.d/head

sudo systemctl start resolvconf.service
sudo systemctl restart resolvconf.service
sudo systemctl restart systemd-resolved.service
sudo systemctl status resolvconf.service
```

> 重启机器，默认会将dns地址写入 /etc/resolv.conf文件内