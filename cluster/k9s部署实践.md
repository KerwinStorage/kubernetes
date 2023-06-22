GitHub：https://github.com/derailed/k9s 官网：https://k9scli.io/ 各系统安装：https://snapcraft.io/k9s（环境有坑，需要进一步排查）

兼容列表：https://github.com/derailed/k9s/tree/v0.27.4#k8s-compatibility-matrix

# 1. 安装

```bash
# 1.脚本安装
curl -sS https://webinstall.dev/k9s | bash

# 2.命令安装
wget https://github.com/derailed/k9s/releases/download/v0.26.7/k9s_Linux_x86_64.tar.gz
tar xf k9s_Linux_x86_64.tar.gz  k9s
mv k9s /usr/local/bin/
k9s info
```

# 2. 二进制安装问题处理

问题一：k9s: unable to locate k8s cluster configuration

确认本地配置文件是否存在

```bash
# 1.先确认 kubectl config文件配置正常
kubectl get nodes
# 2.添加环境变量
sudo vim /etc/profile
export KUBECONFIG=$HOME/.kube/config
# 3.生效
source /etc/profile
```

问题二：Boom!! mkdir /root/.k9s: permission denied.

```bash
mkdir /root/.k9s
```

 