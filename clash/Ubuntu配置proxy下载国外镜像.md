# 1.clash安装教程

**注意：clash是收费的，每个月十几块钱，网络还算稳定，主要用于查询报错问题，下载国外镜像。**

官方：https://ikuuu.eu/user/tutorial?os=linux&client=clash

GitHub：https://github.com/Dreamacro/clash/releases

参考博客：https://opclash.com/fenxiang/302.html

```bash
cd && mkdir clash && cd clash
wget https://github.com/Dreamacro/clash/releases/download/v1.16.0/clash-linux-amd64-v1.16.0.gz
gzip -d  clash-linux-amd64-v1.16.0.gz && chmod +x clash-linux-amd64-v1.16.0 && mv clash-linux-amd64-v1.16.0 /usr/local/bin/clash && clash -v
# 启动 Clash
clash
# 进入目录
cd $HOME/.config/clash/
# 导入订阅
wget -O config.yaml "https://api.sub-200.club/link/7CkZiLMMBfJjZJkL?clash=3"
```

## 1.1.systemd 配置文件

```bash
cat > /etc/systemd/system/clash.service << EOF
[Unit]
Description=Clash - A rule-based tunnel in Go
Documentation=https://github.com/Dreamacro/clash/wiki
[Service]
OOMScoreAdjust=-1000
ExecStart=/usr/local/bin/clash -f /root/.config/clash/config.yaml
Restart=on-failure
RestartSec=5
[Install]
WantedBy=multi-user.target
EOF

# 配置开机自启
systemctl enable clash

# 启动 clash 服务
systemctl start clash

# 配置环境变量
echo -e "export http_proxy=http://127.0.0.1:7890\nexport https_proxy=http://127.0.0.1:7890" >> ~/.bashrc
source ~/.bashrc
```

proxy加入地址：

![img](https://img2023.cnblogs.com/blog/1740081/202306/1740081-20230604212919392-234470252.png)

然后登录YouTube去测试能不能访问。

![img](https://img2023.cnblogs.com/blog/1740081/202306/1740081-20230604213056495-1888012441.png)

# 2.docker配置proxy

参考博客：https://www.jianshu.com/p/6e725d2266b4

```bash
vim /lib/systemd/system/docker.service
[Service]
Environment="HTTP_PROXY=http://127.0.0.1:7890/"
Environment="HTTPS_PROXY=http://127.0.0.1:7890/"
Environment="NO_PROXY=localhost,127.0.0.1,.example.com"
```

重启docker：

```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
```

## 2.1.验证下载速度

```bash
docker pull ghcr.io/fleeksoft/hbase/hbase-base:2.4.13.2
e7c96db7181b: Pull complete
f910a506b6cb: Pull complete
c2274a1a0e27: Pull complete
50eed7c68dcc: Pull complete
816234735ed9: Pull complete
594aea77f7a3: Pull complete
0a5c3050370f: Pull complete
842a5ff5a343: Pull complete
0bedaef96187: Downloading [====================================>              ]  210.7MB/285.3MB
a2e002669b73: Download complete
a13b4b86d6bb: Download complete
610c3fe3b298: Download complete
3acbae54d803: Downloading [==========>                                        ]  10.68MB/53.32MB
8c1d7795df2d: Download complete
```

可以看到下载国外镜像比没加proxy快了很多。🙈🙉🙊🐵

# 3.containerd配置proxy

## 3.1.配置文件

```bash
mkdir -p /etc/systemd/system/containerd.service.d
cat > /etc/systemd/system/containerd.service.d/http-proxy.conf <<EOF
[Service]
Environment="HTTP_PROXY=http://127.0.0.1:7890"
Environment="HTTPS_PROXY=http://127.0.0.1:7890"
Environment="NO_PROXY=localhost,127.0.0.1,.example.com"
EOF
sudo systemctl daemon-reload
systemctl restart containerd.service

echo -e "export http_proxy=http://127.0.0.1:7890\nexport https_proxy=http://127.0.0.1:7890" >> ~/.bashrc
source ~/.bashrc
```

注：其他主机更换成能fq的主机的ip地址，然后去`telnet 目标地址 7890`。