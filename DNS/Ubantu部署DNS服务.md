# 													Ubantu部署DNS服务

## 1.1.安装Bind9

在终端中执行以下命令安装Bind9：

```shell
sudo apt update
sudo apt install bind9
```

## 1.2.配置Bind9

- **修改`named.conf.options`:**

```shell
sudo nano /etc/bind/named.conf.options
```

在文件中，确保以下配置适用于你的环境。根据需要，你可能需要更改`forwarders`和`allow-recursion`。

```shell
options {
    directory "/var/cache/bind";
    recursion yes;
    allow-recursion { any; };
    forwarders {
        8.8.8.8;
        8.8.4.4;
    };
    dnssec-validation auto;
    auth-nxdomain no;    # conform to RFC1035
    listen-on-v6 { any; };
};
```

- **创建自定义区域文件：**

创建一个新的区域文件，例如 `example.com.zone`：

```shell
mkdir -p /etc/bind/zones
sudo nano /etc/bind/zones/example.com.zone
```

在文件中添加以下内容，根据你的需求进行修改：

```shell
$TTL    604800
@       IN      SOA     ns1.example.com. admin.example.com. (
                        2022011301      ; Serial
                        604800         ; Refresh
                        86400          ; Retry
                        2419200        ; Expire
                        604800 )       ; Negative Cache TTL

; Name servers
@       IN      NS      ns1.example.com.

; A records
ns1     IN      A       192.168.80.10
```

注：这里的`192.168.80.10`是我本机的地址

- **修改`named.conf.local` 文件：**

```shell
sudo nano /etc/bind/named.conf.local
```

在文件中添加以下内容，指定新的区域文件：

```shell
zone "example.com" {
    type master;
    file "/etc/bind/zones/example.com.zone";
};
```

- **重启Bind9服务：**

```shell
sudo systemctl restart bind9
sudo systemctl enable named.service
```

## 1.3.测试DNS服务

使用 `nslookup` 或 `dig` 工具测试你的DNS服务。例如：

```shell
root@kb-virtual-machine:~# nslookup ns1.example.com localhost
Server:         localhost
Address:        127.0.0.1#53

Name:   ns1.example.com
Address: 192.168.80.10
```

测试外网跳转解析：

```shell
root@kb-virtual-machine:~# nslookup baidu.com localhost
Server:         localhost
Address:        127.0.0.1#53

Non-authoritative answer:
Name:   baidu.com
Address: 39.156.66.10
Name:   baidu.com
Address: 110.242.68.66
```

