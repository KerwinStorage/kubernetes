# 							ubuntu 22.04部署docker

```shell
# 1.更新源
sudo apt update
# 2.安装必要的依赖项：
sudo apt install apt-transport-https ca-certificates curl software-properties-common
# 3.添加Docker官方GPG密钥：
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
# 4.设置Docker存储库（设置Docker存储库：）：
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
# 5.更新软件包索引并安装Docker CE（社区版）：
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io
# 6.将当前用户添加到docker组中，这样就不需要使用sudo来运行Docker命令了：
sudo usermod -aG docker $USER
docker version
# 7.开机自启动
systemctl enable docker.service
# 8.docker镜像加速
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
    "registry-mirrors": [
        "https://dockerproxy.com",
        "https://hub-mirror.c.163.com",
        "https://mirror.baidubce.com",
        "https://ccr.ccs.tencentyun.com"
    ]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```

