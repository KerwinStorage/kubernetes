# 																			Docker部署DNS服务

前言：这里主要介绍如何在docker中启动自己dns服务，能解析自己的域名还能访问公网地址。

# 1.部署

## 1.1.检查项

解释：https://www.jinbuguo.com/systemd/systemd-resolved.service.html

```shell
# 1.查看53端口是否被占用
netstat -lntup | grep 53
# 2.
vim /etc/systemd/resolved.conf
LLMNR=0
systemctl stop systemd-resolved.service
systemctl disable systemd-resolved.service
```

## 1.2.创建 Corefile（配置文件）

先创建一个名为 Corefile 的文件，这是 CoreDNS 的配置文件。例如，要定义一个名为 example.com 的 zone，同时指定上游 DNS 为 Google 的 DNS 服务器（8.8.8.8 和 8.8.4.4）并且监听 Docker 容器的所有接口上的 53 端口，配置文件可能看起来像这样：

```shell
cat > Corefile <<EOF
zone "example.com" {
    type master
    file "/etc/coredns/db.example.com";
};
EOF
```

## 1.3.创建 DNS 区域文件

你需要创建一个区域文件来定义 example.com 的 DNS 记录。例如，创建一个名为 db.example.com 的文件：

```shell
cat > db.example.com <<"EOF"
$TTL 3600
@   IN  SOA sns.dns.icann.org. noc.dns.icann.org. (
        2017042745 ; Serial
        7200       ; Refresh
        3600       ; Retry
        1209600    ; Expire
        3600 )     ; Negative Cache TTL
;
@   IN  NS  ns1.example.com.

; A records
ns1     IN  A   192.168.80.10
EOF
```

## 1.4.运行 CoreDNS Docker 容器

使用以下命令启动 CoreDNS 的 Docker 容器，同时挂载 Corefile 和区域文件：

```shell
docker run -d --name coredns \
  -v $(pwd)/Corefile:/etc/coredns/Corefile \
  -v $(pwd)/db.example.com:/etc/coredns/db.example.com \
  --publish=53:53/udp --publish=53:53/tcp \
  coredns/coredns
```

这条命令告诉 Docker 启动一个名为 coredns 的容器，并将当前目录下的 Corefile 和 db.example.com 文件挂载到容器的 /etc/coredns 目录下。同时，它将容器的 53 端口（TCP 和 UDP）映射到宿主机的相同端口上。

## 1.5.测试 DNS 解析

使用 dig 或者 nslookup 来测试你的 DNS 设置：

```shell
dig @localhost ns1.example.com
```

或者

```shell
nslookup ns1.example.com 192.168.80.10
```





```shell
docker run -d \
  --name dns-container \
  -v /path/to/bind-config:/etc/bind \
  --publish=53:53/udp --publish=53:53/tcp \
  --restart=always \
  ubuntu/bind9
```





