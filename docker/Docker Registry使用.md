# 1. 本地主机访问

DockerHub镜像地址：https://hub.docker.com/_/registry/tags

官网：https://docs.docker.com/registry

参考：https://docs.docker.com/registry/deploying

```bash
docker pull registry:2.8.1
docker volume create --name registry
docker run -d -v  registry:/var/lib/registry -p 5000:5000 --restart=always --name registry registry:2.8.1

# 推一个镜像上去
docker pull ubuntu:16.04
docker tag ubuntu:16.04 localhost:5000/ubuntu:16.04
docker push localhost:5000/ubuntu:16.04
curl localhost:5000/v2/_catalog

docker exec -it registry sh
# 镜像存放地址
/var/lib/registry/docker/registry/v2/repositories

# 删除本地的镜像在去拉pull下镜像
docker image remove ubuntu:16.04
docker image remove localhost:5000/ubuntu:16.04
docker pull localhost:5000/ubuntu:16.04
```

# 2. 其他主机访问

这里咱没有申请证书也是内部集群访问就直接写在docker配置文件内了。

```bash
# 修改配置文件
vim /etc/docker/daemon.json
{
"insecure-registries": ["192.168.80.45:5000"]
}
 
# 重启docker服务
systemctl daemon-reload
systemctl restart docker
```

**注：注意json格式，如果不对docker会重启失败，如果`/etc/docker/daemon.json`没有就创建一个。**

**其他主机尝试拉取镜像：**

```bash
docker pull 192.168.80.45:5000/ubuntu:16.04
16.04: Pulling from ubuntu
58690f9b18fc: Pull complete
b51569e7c507: Pull complete
da8ef40b9eca: Pull complete
fb15d46c38dc: Pull complete
Digest: sha256:a3785f78ab8547ae2710c89e627783cfa7ee7824d3468cae6835c9f4eae23ff7
Status: Downloaded newer image for 192.168.80.45:5000/ubuntu:16.04
192.168.80.45:5000/ubuntu:16.04
```

# 3. 配置registry webui

https://github.com/klausmeyer/docker-registry-browser

https://hub.docker.com/r/klausmeyer/docker-registry-browser

```bash
docker run \
--name registry-browser \
-p 8080:8080  \
--restart=always \
--link registry \
-e DOCKER_REGISTRY_URL=http://192.168.80.45:5000/v2 \
-d klausmeyer/docker-registry-browser:1.6.1
```

访问：ip地址:8080端口

![img](https://img2023.cnblogs.com/blog/1740081/202304/1740081-20230425145645966-236201699.png)

# 4. 清理registry容器

```bash
docker stop registry
docker rm registry
docker volume rm registry
docker stop registry-browser
docker rm registry-browser
```

 

关于主从架构的思考，两台主机都有容器镜像仓库启动，但是本地存放镜像目录是同步的：

1. 本地docker挂载目录共享（nfs挂载）

2. 使用nginx或者proxy进行负载均衡