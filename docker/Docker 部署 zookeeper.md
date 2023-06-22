# 1.单节点安装

官方镜像：https://registry.hub.docker.com/_/zookeeper/tags

```bash
docker pull zookeeper:3.6.4
# 创建卷
docker volume create zookeeper ; docker volume ls
docker run -d \
-e TZ="Asia/Shanghai" \
-p 2181:2181 \
-v zookeeper:/data \
--name zookeeper \
--restart always zookeeper:3.6.4
docker run -it --rm --link zookeeper:zookeeper zookeeper:3.6.4 zkCli.sh -server zookeeper
```

# 2.集群安装

## 2.1.docker-compose命令安装

Github：https://github.com/docker/compose/tree/v2.17.3

```bash
curl -L "https://github.com/docker/compose/releases/download/v2.17.3/docker-compose-linux-x86_64" -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
docker-compose --version
```

## 2.2.配置docker-compose

```bash
cat > docker-compose.yml <<EOF
version: '2'
services:
    zoo1:
        image: zookeeper:3.6.4
        restart: always
        container_name: zoo1
        ports:
            - "2181:2181"
        environment:
            ZOO_MY_ID: 1
            ZOO_SERVERS: server.1=zoo1:2888:3888;2181 server.2=zoo2:2888:3888;2181 server.3=zoo3:2888:3888;2181

    zoo2:
        image: zookeeper:3.6.4
        restart: always
        container_name: zoo2
        ports:
            - "2182:2181"
        environment:
            ZOO_MY_ID: 2
            ZOO_SERVERS: server.1=zoo1:2888:3888;2181 server.2=zoo2:2888:3888;2181 server.3=zoo3:2888:3888;2181

    zoo3:
        image: zookeeper:3.6.4
        restart: always
        container_name: zoo3
        ports:
            - "2183:2181"
        environment:
            ZOO_MY_ID: 3
            ZOO_SERVERS: server.1=zoo1:2888:3888;2181 server.2=zoo2:2888:3888;2181 server.3=zoo3:2888:3888;2181

networks:
  default:
    driver: bridge
EOF
```

- `ZOO_MY_ID`：zk服务的ID，取值为1-255之间的整数。
- `ZOO_SERVERS`：表示zk集群的主机列表

**注：这里3.5之后，ZOO_SERVERS后面要加上`;2181`，客户端端口。**

**启动：**

```bash
docker-compose up -d
# 查看集群状态
docker-compose ps
NAME                IMAGE               COMMAND                  SERVICE             CREATED             STATUS              PORTS
zoo1                zookeeper:3.6.4     "/docker-entrypoint.…"   zoo1                6 minutes ago       Up 6 minutes        2888/tcp, 3888/tcp, 0.0.0.0:2181->2181/tcp, :::2181->2181/tcp, 8080/tcp
zoo2                zookeeper:3.6.4     "/docker-entrypoint.…"   zoo2                6 minutes ago       Up 6 minutes        2888/tcp, 3888/tcp, 8080/tcp, 0.0.0.0:2182->2181/tcp, :::2182->2181/tcp
zoo3                zookeeper:3.6.4     "/docker-entrypoint.…"   zoo3                6 minutes ago       Up 6 minutes        2888/tcp, 3888/tcp, 8080/tcp, 0.0.0.0:2183->2181/tcp, :::2183->2181/tcp

docker exec -it zoo1 /bin/bash
# 查看选举
zkServer.sh status

ps -ef | grep zookeeper
netstat -lntup | grep 2181

zkCli.sh -server zoo1:2181
zkCli.sh -server zoo2:2181
zkCli.sh -server zoo3:2181
```

其他：

```bash
# 停止docker-compose服务
docker-compose stop
# 启动docker-compose服务
docker-compose start
# 重启docker-compose服务
docker-compose restart
```

# 3.可视化工具

- https://github.com/vran-dev/PrettyZoo/blob/master/README_CN.md

![img](https://img2023.cnblogs.com/blog/1740081/202305/1740081-20230514132413916-1808331588.png)![img](https://img2023.cnblogs.com/blog/1740081/202305/1740081-20230514132443346-1960720293.png)

点击`connect`。