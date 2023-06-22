前言：部署方式：本地部署，采用高可用去访问harbor

1. 两台机器离线部署harbor
2. 两台机器部署keepalived
3. harbor仓库配置镜像同步操作
4. 上传镜像，测试三个地址（node01、node02、vip）

项目地址：https://github.com/goharbor/harbor、官网：https://goharbor.io/

规划表格：

| host-name | ip address     |
| --------- | -------------- |
| vip       | 192.168.80.200 |
| node01    | 192.168.80.46  |
| node02    | 192.168.80.47  |

**注意**：**不适合在k8s 1.25版本的集群上部署，因为可能本地容器运行使用的是Containerd，如果要本地离线部署请用安装好docker的机器。**

**架构图如下：**

**![img](https://img2022.cnblogs.com/blog/1740081/202211/1740081-20221108151158397-1847784427.png)**

# 1. 下载jar包并安装（两台机器一样）

- node01、node02机器同时操作（**注意ip地址**）

```bash
# 1.下载离线包
# 因为网络问题我们通过找到代理下载
wget https://ghproxy.com/https://github.com/goharbor/harbor/releases/download/v2.6.1/harbor-offline-installer-v2.6.1.tgz
cd harbor/
ls
common.sh  harbor.v2.6.1.tar.gz  harbor.yml.tmpl  install.sh  LICENSE  prepare
cp harbor.yml.tmpl harbor.yml
# 2.修改配置文件
vim harbor.yml
# hostname需要修改成本机的地址
hostname: 192.168.80.64
# 密码修改，我在这里就不进行修改了
harbor_admin_password:Harbor12345
# 修改存储磁盘，默认是data，具体根据你服务器来修改，
# 应该放置到最大目录下，我在这里就不修改了
data_volume: /data
# harbordb的密码，我在这里也不进行修改了
password: root123
# 把有关https的全部注释掉，否则会报错
# https:
  # https port for harbor, default is 443
  # port: 443
  # The path of cert and key files for nginx
  #certificate: /your/certificate/path
  #private_key: /your/private/key/path

# 3.安装
root@k8s-node01:~/harbor# pwd
/root/harbor
root@k8s-node01:~/harbor# ./install.sh
```

访问地址：http://192.168.80.64、http://192.168.80.65

User：admin Password：Harbor12345

![img](https://img2022.cnblogs.com/blog/1740081/202211/1740081-20221108145045122-819886149.png)

# 2. keepalived配置

## 2.1 主一节点

- 192.168.80.46操作

```bash
apt install keepalived -y

#cp /etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf.bak
cat > /etc/keepalived/keepalived.conf << EOF
! Configuration File for keepalived

global_defs {
    router_id LVS_DEVEL
}
vrrp_script chk_apiserver {
    script "/etc/keepalived/check_harbor.sh"
    interval 5 
    weight -5
    fall 2
    rise 1
}
vrrp_instance VI_1 {
    state MASTER
    # 注意网卡名
    interface ens33
    mcast_src_ip 192.168.80.46
    virtual_router_id 51
    priority 100
    nopreempt
    advert_int 2
    authentication {
        auth_type PASS
        auth_pass K8SHA_KA_AUTH
    }
    virtual_ipaddress {
        192.168.80.200
    }
    track_script {
      chk_apiserver 
} }

EOF
```

## 2.2 主二节点

- 192.168.80.47操作

```bash
apt install keepalived -y

#cp /etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf.bak
cat > /etc/keepalived/keepalived.conf << EOF
! Configuration File for keepalived

global_defs {
    router_id LVS_DEVEL
}
vrrp_script chk_apiserver {
    script "/etc/keepalived/check_harbor.sh"
    interval 5 
    weight -5
    fall 2
    rise 1
}
vrrp_instance VI_1 {
    state MASTER
    # 注意网卡名
    interface ens33
    mcast_src_ip 192.168.80.47
    virtual_router_id 51
    priority 100
    nopreempt
    advert_int 2
    authentication {
        auth_type PASS
        auth_pass K8SHA_KA_AUTH
    }
    virtual_ipaddress {
        192.168.80.200
    }
    track_script {
      chk_apiserver 
} }

EOF
```

## 2.3 检查脚本

- 两台主机操作

```bash
cat > /etc/keepalived/check_harbor.sh << EOF
#!/bin/bash
#count=$(docker-compose -f /opt/harbor/docker-compose.yml ps -a|grep healthy|wc -l)
# 不能频繁调用docker-compose 否则会有非常多的临时目录被创建：/tmp/_MEI*
count=$(docker ps |grep goharbor|grep healthy|wc -l)
status=$(ss -tlnp|grep -w 443|wc -l)
if [ $count -ne 9 -a  ];then
   exit 8
elif [ $status -lt 2 ];then
   exit 9
else
   exit 0
fi
EOF
```

## 2.4 脚本授权

```bash
chmod +x /etc/keepalived/check_harbor.sh
sudo useradd keepalived_script
sudo passwd keepalived_script
sudo chown -R keepalived_script:keepalived_script /etc/keepalived/check_harbor.sh

systemctl daemon-reload
systemctl enable --now keepalived
systemctl start keepalived
```

# 3. 配置双主策略

- 两台机器互相pull images策略

## 3.1 创建仓库

- 两个harbor统一操作

![img](https://img2022.cnblogs.com/blog/1740081/202211/1740081-20221108145252271-1938765803.png)

## 3.2 创建仓库管理

- 两个harbor统一操作（**注意地址**）

![img](https://img2022.cnblogs.com/blog/1740081/202211/1740081-20221108145305626-610976168.png)

## 3.3 创建用户（非必须）

- 两个harbor统一操作

 ![img](https://img2022.cnblogs.com/blog/1740081/202211/1740081-20221108145332459-1496554406.png)

将用户变更为仓库成员，权限自定义：

![img](https://img2022.cnblogs.com/blog/1740081/202211/1740081-20221108145346485-666353926.png)

## 3.4 设置复制规则

将规则修改为push的方式

![img](https://img2022.cnblogs.com/blog/1740081/202211/1740081-20221108145404616-654968790.png)

 

# 4. 测试登录harbor

## 4.1 修改dockers启动配置文件

**注：****如果docker服务是在k8s集群内的话请思考下能不能随便重启docker，或者考虑换个有docker的机器。**

```bash
# 1.查找文件
find / -name docker.service -type f
vim /usr/lib/systemd/system/docker.service

# 2.修改配置
# 查找：ExecStart=/usr/bin/dockerd 在其后面添加 --insecure-registry=192.168.80.200 配置
# 添加后变成：ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock --insecure-registry=192.168.80.200
systemctl daemon-reload
systemctl restart docker

# 3.重启harbor
docker-compose start

# 4.登录

docker login 192.168.80.200  -u admin -p Harbor12345
WARNING! Using --password via the CLI is insecure. Use --password-stdin.
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
```

结尾：因为只是测试只有vip 地址漂移，`harbor`没有配备高可用服务，如果有需要可以考虑 `haproxy`服务做负载均衡。🐳

 