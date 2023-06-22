**前言：因为 [官方仓库](https://hub.docker.com/) 在国外的原因，我们不容易拉取镜像，可以我们可以使用国内的 [Daocloud](https://account.daocloud.io/) 镜像仓库和开通 [阿里云仓库](https://www.aliyun.com/product/kubernetes?utm_content=se_1005527063) 或者 [Harbor](https://index.tenxcloud.com/) 如果是企业内部也可以使用自己搭建的私有仓库**

## docker-registry使用

介绍：Registry用于保存docker镜像，包括镜像的层次结构和元数据

```
#第一步：pull官方镜像
docker pull registry

#第二步：启动容器
docker run  -d -p 5000:5000 -d -p 5000:5000 --restart=always  --name=registry -v /opt/myregistry:/var/lib/registry registry

#第三步：添加/etc/docker/daemon.json
cat /etc/docker/daemon.json
{
    "registry-mirrors": [
        "https://1nj0zren.mirror.aliyuncs.com",
        "https://docker.mirrors.ustc.edu.cn",
        "http://f1361db2.m.daocloud.io",
        "https://registry.docker-cn.com"
    ],
    "insecure-registries": ["10.0.0.12:5000"]
}

systemctl restart docker

#第四部：上传镜像
打标签： docker tag alpine:latest 10.0.0.12:5000/alpine:latest
上传：      docker image push 10.0.0.12:5000/alpine:latest
```

**登录：**http://10.0.0.12:5000/v2/_catalog

## 官方仓库使用

**在官方[仓库](https://hub.docker.com/)免费注册一个docker账号**

```
#登录和退出
docker login
docker logout

#查找镜像
docker  search alpine
#拉取
docker pull alpine

#推送镜像
docker tag alpine:latest username/alpine:latest
docker push username/alpine:latest
docker search username/alpine
```

## 搭建企业级镜像仓库

**GitHub项目地址：**https://github.com/goharbor/harbor

### Harbor介绍

VMware 在中国的团体开发的

Harbor，是一个英文单词，意思是港湾，Harbor真是一个用于存储Docker镜像的企业级Registry服务。Registry是Docker官方的一个私有仓库镜像，可以将本地的镜像打标签进行标记然后push到以Registry起的容器的私有仓库中。去也可以根据自己的需求，使用Dockerfile生成自己的镜像，并推到私有仓库中，这样可以大大提高拉取镜像的效率。

![img](https://img2020.cnblogs.com/blog/1740081/202004/1740081-20200416190126637-1790045293.png)

### Harbor核心组件解释

| **组件**                       | **说明**                                                     | **实现**                                               |
| ------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------ |
| Proxy                          | 用于转发用户的请求到registry/ui/token service的反向代理      | nginx：使用nginx官方镜像进行配置                       |
| Registry                       | 镜像的push/pull命令实施功能                                  | registry：使用registry官方镜像                         |
| Database                       | 保存项目/角色/复制策略等信息到数据库中                       | harbor-db：Mariadb的官方镜像用于保存harbor的数据库信息 |
| Core Service：UI/token/webhook | 用户进行镜像操作的界面实现，通过webhook的机制保证镜像状态的变化harbor能够即使了解以便进行日志更新等操作，而项目用户角色则通过token的进行镜像的push/pull等操作 | harbor-ui等                                            |
| job service                    | 镜像复制，可以在harbor实例之间进行镜像的复制或者同步等操作   | harbor-jobservice                                      |
| Log collector                  | 负责收集各个镜像的日志信息进行统一管理                       | harbor-log：缺省安装下日志的保存场所为/var/log/harbor  |

### harbor安装部署

**注：这里我们使用1.8.0版本安装，文档下载是1.10.2版本**

\1. 配置环境

```
#安装docker
yum install docker -y
systemctl start docker
systemctl enable docker
#安装docker-compose
yum install docker-compose -y

#下载harbor
wget https://github.com/goharbor/harbor/releases/download/v1.10.2/harbor-offline-installer-v1.10.2.tgz
#解压
tar xf harbor-offline-installer-v1.8.0.tgz
cd harbor/

#修改登录密码和主机名
vim harbor.yaml
hostname: 10.0.0.12
harbor_admin_password: 123456

./install.sh
```

**用户：`admin` 密码：`123456`**

![img](https://img2020.cnblogs.com/blog/1740081/202004/1740081-20200416190556154-1713716918.png)

### 上传镜像

```
#添加/etc/docker/daemon.json
cat /etc/docker/daemon.json
{
    "registry-mirrors": [
        "https://1nj0zren.mirror.aliyuncs.com",
        "https://docker.mirrors.ustc.edu.cn",
        "http://f1361db2.m.daocloud.io",
        "https://registry.docker-cn.com"
    ],
    "insecure-registries": ["10.0.0.11"]
}

systemctl restart docker

#上传镜像的格式，先打标签在上传
docker tag SOURCE_IMAGE[:TAG] 10.0.0.11/library/IMAGE[:TAG]
docker push 10.0.0.11/library/IMAGE[:TAG]

#登录仓库
docker login  10.0.0.11
<用户名和密码>
#打标签
docker tag alpine:latest 10.0.0.11/library/alpine:latest
#上传镜像
docker push  10.0.0.11/library/alpine:latest
```

![img](https://img2020.cnblogs.com/blog/1740081/202004/1740081-20200416190633487-1013195273.png)

### 拉取镜像

```
[root@docker01 harbor]# docker pull 10.0.0.11/library/alpine:latest
latest: Pulling from library/alpine
Digest: sha256:cb8a924afdf0229ef7515d9e5b3024e23b3eb03ddbba287f4a19c6ac90b8d221
Status: Image is up to date for 10.0.0.11/library/alpine:latest
10.0.0.11/library/alpine:latest
```

### harbor添加https证书

步骤一：上传https证书，这里我们可以去**[阿里云](https://www.aliyun.com/product/cas?spm=5176.10695662.1171680.1.442459c5VO0DPK)**申请一个免费的证书 前提必须有一个域名才能绑定

步骤二：主机HOSTS文件做域名劫持 如：`10.0.0.11 wjx.ink` linux hosts文件也要修改

步骤三：修改harbor配置文件

- 申请证书

![img](https://img2020.cnblogs.com/blog/1740081/202004/1740081-20200416210051211-1195855934.png)

- 修改配置文件

```
#修改harbor.yml http加上注释，https取消注释
vim harbor.yml
hostname: wjx.ink

#添加证书目录
https:
  port: 443
  certificate: /opt/3778145_www.wjx.ink.pem
  private_key: /opt/3778145_www.wjx.ink.key
  
  
#本机HOSTS劫持
10.0.0.11 wjx.ink

#主机host文件劫持
10.0.0.11 wjx.ink
#启动执行安装脚本
./install.sh
#访问wjx.ink
https://wjx.ink
```

![img](https://img2020.cnblogs.com/blog/1740081/202004/1740081-20200416210226780-1232518790.png)

## 将registry镜像迁移到harbor

原理图：registry与harbor直接连接迁移

![img](https://img2020.cnblogs.com/blog/1740081/202004/1740081-20200418063953843-2086800932.png)

### 构建

仓库管理：

![img](https://img2020.cnblogs.com/blog/1740081/202004/1740081-20200418064053012-1603145028.png)

同步管理：

![img](https://img2020.cnblogs.com/blog/1740081/202004/1740081-20200418064133419-1975993746.png)

点击同步：

![img](https://img2020.cnblogs.com/blog/1740081/202004/1740081-20200418064215608-1846130594.png)

 