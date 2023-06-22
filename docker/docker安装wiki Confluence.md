## wiki Confluence的简单介绍

官方地址：https://www.atlassian.com/software/confluence

Confluence是一个专业的企业知识管理与协同软件，也可以用于构建企业wiki。使用简单，但它强大的编辑和站点管理特征能够帮助团队成员之间共享信息、文档协作、集体讨论，信息推送。

**空间（space）**

空间是Confluence系统中的一个区域，用于存储wiki页面，并可实现对空间中的所有文档进行统一的权限管理。

通常，我们可以针对每个项目单独创建一个空间，然后将与该项目相关的文档信息放置到该空间中，并只对项目成员开设访问/编辑权限。

除了项目空间，每个成员都有一个个人空间。平时成员可以将工作总结或笔记等文档放置到自己的空间中；对于对团队有帮助的文档，就可以将文档移动至团队项目空间中。

可以理解为SVN或Git的一个库

#### Dashboard

Dashboard是Confluence系统的主页，在Dashboard界面中包含了Confluence站点中的所有空间列表，以及最近更新内容的列表。

#### Dashboard

Dashboard是Confluence系统的主页，在Dashboard界面中包含了Confluence站点中的所有空间列表，以及最近更新内容的列表。

#### 页面（Page）

在Confluence系统中，页面是存储和共享信息的主要方式。页面可以互相链接、连接、组织和访问，并以树状结构进行组织，放置于空间之中。

页面遵循所见即所得的编辑方式，操作上简单易用。更强大的地方在于，页面支持大量的内容展现形式，除了富文本文档外，还包括图表、视频、附件（可预览）、流程图、公式等等；如果还不够，还可以通过海量的第三方插件进行扩展。

在页面中可以通过@其它成员，通知相关成员查看文档。文档保存成功后，被@的成员就会收到邮件，并可根据邮件中的链接访问到该文档，然后进行评论或者协同编辑。

#### 模板（template）

创建页面时除了采用空白文档，也可以选择模板。模板是在空白文档的基础上，根据特定需求添加了一些文档要素，可辅助用户更好更快地创建文档。

Confluence内置了大量的模板，可辅助用于项目工作的各个环节，包括产品需求、会议记录、决策记录、指导手册（How-to）、回顾记录、工作计划、任务报告等等。并且由于Confluence和JIRA是同一家公司的产品，在Confluence中可以和JIRA进行无缝衔接，实现对产品质量实现更好的展现。

如果对Confluence自带的模板不满意，还可以对模板进行调整，或者根据自己的需求创建其它类型的模板。

#### 权限（Permission）

在安全性方面，Confluence具有完善和精细的权限控制，可以很好地控制用户在Wiki中创建、编辑内容和添加注释。

权限控制分3个维度，分别是团队（Group），个人（Individual Users），匿名用户（Anonymous）。

使用团队级的权限控制时，需要在Confluence服务器中对公司员工进行分组，好处在于配置比较方便，只需要对整个团队进行统一的权限配置。

但在实际项目中，经常会存在同一个项目包含多个跨团队成员的情况，这个时候就不适合采用团队权限配置方式，只能采用逐个添加成员的方式，并对各个成员分别配置权限。

另外一种情况，就是对于未登录的用户，以及项目成员以外的用户，可以开设部分权限，例如只读（View）

摘自：https://www.jianshu.com/p/f79236289793

## docker部署

### centos部署docker

```
wget -O /etc/yum.repos.d/docker-ce.repo https://download.docker.com/linux/centos/docker-ce.repo
sed -i 's+download.docker.com+mirrors.tuna.tsinghua.edu.cn/docker-ce+' /etc/yum.repos.d/docker-ce.repo
#安装docker
yum install docker-ce -y
#启动docker
systemctl enable docker
systemctl start docker
#查看版本
docker version

#docker加速
vim /etc/docker/daemon.json
{
 "registry-mirrors": [
    "https://registry.docker-cn.com",
    "http://hub-mirror.c.163.com",
    "https://docker.mirrors.ustc.edu.cn"
     ]
}
```

### ubantu部署docker

安装docker：

```
1. 卸载旧版本
sudo apt-get remove docker docker-engine docker-ce docker.io

2. 更新包
sudo apt-get update

3. 安装包允许apt通过HTTPS使用仓库
sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    software-properties-common

4. 添加docker官方GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

5. 设置docker稳定版仓库
sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
   
6. 添加仓库后，更新apt源索引
sudo apt-get update

7. 安装最新版docker-ce （社区版）
sudo apt-get install docker-ce

8. 检查docker-ce是否安装正确
sudo docker  run hello-world
```

日常启动docker命令：

```
#重启docker
sudo systemctl restart docker
#关闭docker
sudo systemctl stop docker
#查看状态
sudo systemctl status docker
#开机自启
sudo systemctl enable docker
```

将普通用户加入docker组（之后就不用加`sudo`命令了。）：

```
sudo usermod  -a -G docker $USER
#exit重新登录一下
```

docker加速：

```
vim /etc/docker/daemon.json
{
 "registry-mirrors": [
    "https://registry.docker-cn.com",
    "http://hub-mirror.c.163.com",
    "https://docker.mirrors.ustc.edu.cn"
     ]
}


#重启docker
systemctl restart docker
```

## 安装数据库PostgresSQL

安装PostgresSQL所使用的镜像在：https://hub.docker.com/_/postgres/

 

### 安装PostgresSQL

\1. 起postgres数据库的镜像

　　这里我们也可以持久化到指定的目录

```
#创建持久化卷
docker volume create confluencedb
#创建postgres容器
docker run --name  postgres -p 5432:5432 -e POSTGRES_PASSWORD=dreame2019 -v confluencedb:/var/lib/postgresql/data -d postgres
```

**注：docker容器的持久化卷目录是`/var/lib/docker/volumes`下，只有root才可以看到下面的目录。**

>  
>
> 注意：
>
> \1. -p 5432:5432 选项是可选的，因为在后面启动Confluence容器的时候，postgres这个容器会以别名db连接到confluence容器，也就是说对confluence这个容器来说，可以通过db:5432的网络地址访问到postgres服务，不需要主机上开放5432端口。
>
> \2. -e POSTGRES_PASSWORD=dream2019是我们需要设置的密码

 

\2. 进入docker容器并创建confluence数据库

```
docker exec -it postgres /bin/bash #进入docker容器
# psql -U postgres 
psql (12.3 (Debian 12.3-1.pgdg100+1))
Type "help" for help
postgres=# create database confluence with owner postgres;
CREATE DATABASE
```

## 安装wiki Confluence

下文中使用的镜像 [https://hub.docker.com/r/cptactionhank/atlassian-confluence/ ](https://hub.docker.com/r/cptactionhank/atlassian-confluence/)

也可以使用 https://github.com/jgrodziski/docker-confluence/blob/master/Dockerfile 这个镜像他把PostgreSQL和 Confluence包含在一个image里面，参考：http://blogs.atlassian.com/2013/11/docker-all-the-things-at-atlassian-automation-and-wiring/

### 安装wiki Confluence

```
docker run -d --name confluence -p 8090:8090 --link postgres:db --user root:root cptactionhank/atlassian-confluence:latest
#以上命令将在主机上开放8090端口，如果想使用80端口访问wiki请使用一下命令安装
docker run -d --name confluence -p 80:8090 --link postgresdb:db --user root:root cptactionhank/atlassian-confluence:latest
```

###  检查confluence是否启动

```
dreame@ubuntu:~$ docker ps -a
CONTAINER ID        IMAGE                                       COMMAND                  CREATED             STATUS              PORTS                              NAMES
c4c5e488ae04        cptactionhank/atlassian-confluence:latest   "/docker-entrypoint.…"   15 minutes ago      Up 15 minutes       0.0.0.0:8090->8090/tcp, 8091/tcp   confluence
de377de805b1        postgres                                    "docker-entrypoint.s…"   17 minutes ago      Up 17 minutes       0.0.0.0:5432->5432/tcp             postgresdb
254feace6105        portainer/portainer                         "/portainer"             23 hours ago        Up 2 hours          0.0.0.0:9000->9000/tcp             admiring_wozniak
```

### 破解confluence

在网页上面输入ip地址(宿主机):8090，这里我输入：http://192.168.124.27:8090/

\1. 选择语言为中文

![img](https://img2020.cnblogs.com/blog/1740081/202006/1740081-20200601000237970-1994482131.png)

选择产品安装：

![img](https://img2020.cnblogs.com/blog/1740081/202006/1740081-20200601000300952-1292762645.png)

记录ID号（后期会用到）：

![img](https://img2020.cnblogs.com/blog/1740081/202006/1740081-20200601000353258-187170062.png)

\2. 进入docker confluence容器，查找decoder.jar文件

```
docker exec -it confluence /bin/bash # 进入docker容器 confluence
su - 
c4c5e488ae04:~# find /  -name "*decoder*"
/opt/atlassian/confluence/confluence/WEB-INF/lib/atlassian-extras-decoder-v2-3.4.1.jar  #<--这个文件就是我们想要的
/opt/atlassian/confluence/confluence/WEB-INF/lib/atlassian-extras-decoder-api-3.4.1.jar
```

\3. 将decoder.jar文件从容器中复制出来，其中 “confluence:” 是Wiki confluence容器名称，atlassian-extras-decoder-v2-3.4.1.jar 是安装版本wiki的decode文件

```
$ docker cp confluence:/opt/atlassian/confluence/confluence/WEB-INF/lib/atlassian-extras-decoder-v2-3.4.1.jar .
```

**注：这里安装 lrzsz命令将文件导入到桌面。**

```
#centos安装lrzsz命令
yum install lrzsz -y
#ubantu安装lrzsz命令
sudo apt-get install lrzsz -y
```

将文件导出到桌面

 ![img](https://img2020.cnblogs.com/blog/1740081/202006/1740081-20200601000459159-1871428282.png)

\4. 破解文件

a）下载 atlassian-extras-decoder-v2-3.3.0.jar 文件到windows上

b）将文件名改为 “atlassian-extras-2.4.jar” 破解工具只识别这个文件名

c）下载破解文件 

d）解压缩此文件夹，dos命令行进入此文件夹，目录需根据你的实际情况修改 D:\360MoveData\Users\Administrator\Desktop\新建文件夹\confluence5.1-crack\iNViSiBLE/confluence_keygen.jar （我们之后要用到的破解文件）

e）执行 java -jar confluence_keygen.jar 运行破解文件

f）填入 name ，server id 处输入步骤1中得到的id，点击 “gen” 生成key

> 根据上面的需要我们已经将文件下载到了桌面，接下来就是进行修改文件名称，这里就不演示。然后需要运行破解文件，这里就需要电脑有java环境，否则运行不了文件这里需要在[官网](https://www.oracle.com/java/technologies/javase-downloads.html)下载安装java环境（安装步骤不在描述）

**执行命令打开破解文件：**

**win+R键打开运行输入`java -jar 破解文件文件路径`。（这里咱们的破解文件就出来了）或者直接使用打开方式直接打开文件**

***\*注：如果之前运行过请把atlassian-extras-2.4.bak文件删除掉，否则path失败.\****

 

![img](https://img2020.cnblogs.com/blog/1740081/202006/1740081-20200601000631531-1343879848.png)

填入name，server id处输入网页id，点击"gen"生产key

![img](https://img2020.cnblogs.com/blog/1740081/202006/1740081-20200601000648490-1084413097.png)

g）点击 patch，选择刚才改名为 “atlassian-extras-2.4.jar” 的jar包，显示 “jar success fully patched” 则破解成功（这时bak文件就会生成）

![img](https://img2020.cnblogs.com/blog/1740081/202006/1740081-20200601000707719-340244886.png)

 

h）将 “atlassian-extras-2.4.jar” 文件名改回原来的 “atlassian-extras-decoder-v2-3.4.1.jar”（根据历史原因，jar包的版本号会改变，根据自己导出的文件名在改回去）

i）复制key中的内容备用

j）将 “atlassian-extras-decoder-v2-3.3.0.jar” 文件上传回服务器

\5. 将破解后的文件复制回confluence容器

```
docker cp atlassian-extras-decoder-v2-3.4.1.jar  confluence:/opt/atlassian/confluence/confluence/WEB-INF/lib/atlassian-extras-decoder-v2-3.4.1.jar
#重启docker
$ docker restart confluence 
confluence
#访问页面
```

登录：http://192.168.124.27:8090/ 输入刚才的key：

![img](https://img2020.cnblogs.com/blog/1740081/202006/1740081-20200601000757622-1595651306.png)

\6. 点击"我自己的数据库"，下一步：

![img](https://img2020.cnblogs.com/blog/1740081/202006/1740081-20200601000824542-209632361.png)

\7. 数据数据库连接信息，用户名密码是之前创建数据库中的用户名和密码

![img](https://img2020.cnblogs.com/blog/1740081/202006/1740081-20200601000902548-118261659.png)

**注：如果在设置数据库的时候突然中断会出现大大的ERROR的界面，接下来重启confluence容器就可以，然后在重新连接数据库。**

**注：建立数据库时不要中断，否则需要重新创建新的数据库或者覆盖掉数据库之前设置的信息。也容易初夏错误页面就很麻烦**

![img](https://img2020.cnblogs.com/blog/1740081/202006/1740081-20200601000925112-1935017756.png)

 

![img](https://img2020.cnblogs.com/blog/1740081/202006/1740081-20200601000942522-1630635337.png)

![img](https://img2020.cnblogs.com/blog/1740081/202006/1740081-20200601000956626-1830478057.png)

![img](https://img2020.cnblogs.com/blog/1740081/202006/1740081-20200601001013175-1580552193.png)

![img](https://img2020.cnblogs.com/blog/1740081/202006/1740081-20200601001028925-1779230752.png)

现在我们搭建完成！

 ![img](https://img2020.cnblogs.com/blog/1740081/202006/1740081-20200601001050812-1747730407.png)

 

## 解决访问时长gc的问题

这里java的内存默认是1G，但是confluence已经使用了一半以上，我们需要扩大内存访问网页才会快一些。

路径：一般设置--->系统信息

![img](https://img2020.cnblogs.com/blog/1740081/202006/1740081-20200601230333343-752184825.png)

修改配置文件：

```
#进入容器修改最大内存
docker exec -it confluence  /bin/bash
vim /opt/atlassian/confluence/bin/setenv.sh
CATALINA_OPTS="-Xms1024m -Xmx2048m -XX:+UseG1GC ${CATALINA_OPTS} #这里最小给1G，最大给2G

#重启docker
docker restart confluence
```

重新登录网页查看内存：

![img](https://img2020.cnblogs.com/blog/1740081/202006/1740081-20200601230430425-798847929.png)

这里可以看到内存已经进行了修改，但是我们虚拟机给的4G内存已经剩下不到200M的空间，但是网页跳转确实快了不少。