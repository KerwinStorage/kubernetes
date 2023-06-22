## Portainer介绍

官网地址：https://www.portainer.io/

官方文档：https://www.portainer.io/documentation/

官方部署文档：https://www.portainer.io/documentation/quick-start/

介绍：Portainer给我们提供了友好的web界面，可以让我们在web界面轻松的管理docker容器和查看容器状态

## 安装docker

```
wget -O /etc/yum.repos.d/docker-ce.repo https://download.docker.com/linux/centos/docker-ce.repo
sed -i 's+download.docker.com+mirrors.tuna.tsinghua.edu.cn/docker-ce+' /etc/yum.repos.d/docker-ce.repo
yum install docker-ce -y

systemctl enable docker
systemctl start docker
```

## 拉取镜像

```
docker search portainer
docker pull portainer/portainer
```

## 启动镜像

```
docker run -d -p 9000:9000 -v /var/run/docker.sock:/var/run/docker.sock -v protainer_data:/data  --restart=always portainer/portainer
```

**参数说明：**

> -d：后台启动
>
> -p：主机9000端口映射容器9000端口
>
> --restart：跟随docker服务启动
>
> -v /var/run/docker.sock:/var/run/docker.sock ：把宿主机的Docker守护进程(Docker daemon)默认监听的Unix域套接字挂载到容器中；
>
> --name：容器名

## web访问

网页登录：httpd://ip地址:9000 首次登录需要注册用户，给用户admin设置密码

**注：我们每次启动时镜像`/var/run/docker.sock:/var/run/docker.sock`目录与本地目录要挂载上**

![img](https://img2020.cnblogs.com/blog/1740081/202004/1740081-20200417102227992-875919622.png)

选择连接谁？

![img](https://img2020.cnblogs.com/blog/1740081/202004/1740081-20200417102300910-1741674377.png)

查看容器启动状态

各个框架作用请看链接：https://www.portainer.io/overview/#dashboard

![img](https://img2020.cnblogs.com/blog/1740081/202004/1740081-20200417102327704-293952058.png)

**注：具体操作请多尝试点击web界面熟悉**