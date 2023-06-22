## Gitlab介绍

[GitLab](https://link.jianshu.com/?t=https://about.gitlab.com/)是一个利用Ruby on Rails开发的开源应用程序，实现一个自托管的Git项目仓库，可通过Web界面进行访问公开的或者私人项目。

![img](https://img2020.cnblogs.com/blog/1740081/202005/1740081-20200513222918379-1198265679.jpg)

 

它拥有与Github类似的功能，能够浏览源代码，管理缺陷和注释。可以管理团队对仓库的访问，它非常易于浏览提交过的版本并提供一个文件历史库。团队成员可以利用内置的简单聊天程序(Wall)进行交流。它还提供一个代码片段收集功能可以轻松实现代码复用，便于日后有需要的时候进行查找。开源中国代码托管平台git.oschina.net就是基于GitLab项目搭建。

## docker介绍

**注：本次教程是直接从docker去pull别人已经准备好的镜像，后期还会有如何自己制作镜像。**

### docker的优势

docker五大优势：持续集成、版本控制、可移植性、隔离性和安全性

**对比传统虚拟机总结**

 

| 特性       | 容器               | 虚拟机     |
| ---------- | ------------------ | ---------- |
| 启动       | 秒级               | 分钟级     |
| 硬盘使用   | 一般为`MB`         | 一般为`GB` |
| 性能       | 接近原生           | 弱于       |
| 系统支持量 | 单机支持上千个容器 | 一般几十个 |

 

### docker系统架构

docker使用客户端-服务端（C/S）架构模式，使用远程API来管理和创建docker容器。

docker容器通过docker镜像来创建。

容器于镜像的管理类似于面向对象编程中的对象于类。

![img](https://img2020.cnblogs.com/blog/1740081/202005/1740081-20200513223159644-162103972.png)

 

| 标题              | 说明                                                         |
| ----------------- | ------------------------------------------------------------ |
| 镜像（images）    | Docker镜像是用于创建Docker容器的模板。                       |
| 容器（Container） | 容器是独立运行的一个或一组应用。                             |
| 客户端（Client）  | docker客户端通过命令行或者其他工具使用[Docker API](https://docs.docker.com/engine/api/)与Docker的守护进程通信。 |
| 主机（Host）      | 一个物理或者虚拟机的机器用于执行Docker守护进程和容器。       |
| 仓库（Registry）  | Docker仓库用来保存镜像，可以理解为代码控制中的代码仓库。[Docker Hub](https://hub.docker.com)提供了庞大的镜像集合供使用。 |
| Docker Machine    | Docker Machine是一个简化Docker安装命令的命令行工具，通过一个简单的命令行即可在相应的平台上安装Docker，比如VirtualBox、Digital Ocean、Microsoft Azure。 |

## 安装前准备工作

###  安装docker-ce

**环境规划：**

 

| 主机名   | 内存 | ip        |
| -------- | ---- | --------- |
| docker01 | 3G   | 10.0.0.11 |

**安装docker**

***\*注意：这里我们是从[清华源](https://mirrors.tuna.tsinghua.edu.cn/)上面下载的docker\****

 

```
wget -O /etc/yum.repos.d/docker-ce.repo https://download.docker.com/linux/centos/docker-ce.repo
sed -i 's+download.docker.com+mirrors.tuna.tsinghua.edu.cn/docker-ce+' /etc/yum.repos.d/docker-ce.repo
yum install docker-ce -y
systemctl start docker && systemctl enable  docker

[root@docker01 ~]# docker version
Client: Docker Engine - Community
 Version:           19.03.3
 API version:       1.40
 Go version:        go1.12.10
 Git commit:        a872fc2f86
 Built:             Tue Oct  8 00:58:10 2019
 OS/Arch:           linux/amd64
 Experimental:      false
```

### docker加速配置

**[加速教程](https://www.runoob.com/docker/docker-mirror-acceleration.html)，当配置某一个加速器地址之后，若发现拉取不到镜像，请切换到另一个加速器地址。centos7安装docker之后没有daemon.json文件，需要自己动手创建一个**

```
mkdir -p /etc/docker
tee /etc/docker/daemon.json <<-'EOF'
{
"registry-mirrors": ["https://4pwh0wn5.mirror.aliyuncs.com"]
}
EOF
#更新配置
sudo systemctl daemon-reload
sudo systemctl restart docker
```

## 安装gitlab准备工作

### 获取gitlab镜像包

```
docker search gitlab  #查看镜像
docker pull gitlab/gitlab-ce  #拉取
```

### 创建准备gitlab工作目录

```
#创建 config 目录 
mkdir -p /home/gitlab/config 
#创建 logs 目录 
mkdir -p /home/gitlab/logs
#创建 dat 目录
mkdir -p /home/gitlab/data 
```

### 运行脚本启动gitlab

```
docker run -d \
    --hostname 10.0.0.11 \
    --network 
    --publish 7001:443  --publish 7002:80  --publish 7003:22 \
    --name gitlab \
    --restart always \
    --volume /home/gitlab/config:/etc/gitlab \
    --volume /home/gitlab/logs:/var/log/gitlab \
    --volume /home/gitlab/data:/var/opt/gitlab \
    gitlab/gitlab-ce
```

**参数说明：**

| 参数名称       | 参数说明                                                     |
| -------------- | ------------------------------------------------------------ |
| detach         | 指定容器运行于前台还是后台                                   |
| hostname       | 指定主机地址，如果有域名可以指向域名                         |
| publish        | 指定容器暴露的端口，左边的端口代表宿主机的端口，右边代表容器的端口 |
| name           | 容器名字                                                     |
| restart always | 跟随docker服务启动                                           |
| volume         | 数据卷，在docker中是最重要的一个知识点                       |

### 修改gitlab.rb配置文件

按照上面的方式，gitlab容器运行没有问题，但在gitlab上创建项目的时候，生成项目的URL访问地址是按容器的hostname来生成的，也就是容器的id。作为gitlab服务器，我们需要一个固定的URL访问，于是需要配置gitlab.rb（宿主机路径：/home/gitlab/config/gitlab.rb）配置三个重要的参数：

```
external_url 'http://10.0.0.11' 
gitlab_rails['gitlab_ssh_host'] = '' 
gitlab_rails['gitlab_shell_ssh_port'] = 703

#配置gitlab通过smtp发送邮件
gitlab_rails['gitlab_email_enabled'] = true
gitlab_rails['gitlab_email_from'] = '1354586675@qq.com'
gitlab_rails['gitlab_email_display_name'] = 'china'
gitlab_rails['smtp_enable'] = true
gitlab_rails['smtp_address'] = "smtp.qq.com"
gitlab_rails['smtp_port'] = 465
gitlab_rails['smtp_user_name'] = "1354586675@qq.com"
gitlab_rails['smtp_password'] = "xxxxxx"
gitlab_rails['smtp_domain'] = "qq.com"
gitlab_rails['smtp_authentication'] = "login"
gitlab_rails['smtp_enable_starttls_auto'] = true
gitlab_rails['smtp_tls'] = true
```

### 进入gitlab容器重启服务

```
docker exec -it gitlab /bin/bash  #进入gitlab容器服务
gitlab-ctl reconfigure  #重置gitlab客户端的命令
```

由于我们运行是使用数据卷参数进行运行的，宿主机的gitlab.rb文件修改，容器内的配置文件也会修改，但是容器的文件不会跟着生效，必须要进去容器里面进行命令执行，重置配置文件比较浪费时间，需要耐心等待，如果时间比较短说明成功率不高这里内存建议3G因为内存如果是1G这个命令执行不起来。

### 重启gitlab容器命令

```
docker restart gitlab #这里重启容器也需要耐心等待。
```

### 检查启动信息

```
[root@docker01 ~]# docker ps 
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS                    PORTS                                                               NAMES
67b087a72785        gitlab/gitlab-ce    "/assets/wrapper"   41 minutes ago      Up 34 minutes (healthy)   0.0.0.0:7003->22/tcp, 0.0.0.0:7002->80/tcp, 0.0.0.0:7001->443/tcp   gitlab


[root@docker01 ~]# netstat -lnt
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State      
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN     
tcp6       0      0 :::22                   :::*                    LISTEN     
tcp6       0      0 :::7001                 :::*                    LISTEN     
tcp6       0      0 :::7002                 :::*                    LISTEN     
tcp6       0      0 :::7003                 :::*                    LISTEN     
```

### gitlab常用命令

```
gitlab-ctl reconfigure     // 重新应用 gitlab 的配置 
gitlab-ctl restart         // 重启 gitlab 服务 
gitlab-ctl status         // 查看 gitlab 运行状态 
gitlab-ctl stop         // 停止 gitlab 服务 
gitlab-ctl tail         // 查看 gitlab 运行日志
```

### 登录gitlab界面

**这里直接访问：http://10.0.0.11:7002/，使用宿主ip地址访问映射端口**

**注：这里并没有像其他博主修改什么端口，在创建容器时也需要查看端口是否被占用，如果被占用会出现502的报错。**

![img](https://img2020.cnblogs.com/blog/1740081/202005/1740081-20200513224118910-704373461.png)

![img](https://img2020.cnblogs.com/blog/1740081/202005/1740081-20200513224133737-713338969.png)