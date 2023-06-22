# 背景概述

- 需要再其他机器连接上k8s集群进行访问

# 具体操作

kubectl下载地址：https://kubernetes.io/zh-cn/docs/tasks/tools/install-kubectl-linux/

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
ln -s /usr/local/bin/kubectl /usr/bin/kubectl

# 默认配置文件访问地址,将json配置文件写在里面
mkdir $HOME/.kube
vim $HOME/.kube/config

# 验证
kubectl get node
NAME                       STATUS   ROLES    AGE   VERSION
cn-beijing.192.168.22.18   Ready    <none>   21h   v1.22.10-aliyun.1
cn-beijing.192.168.22.19   Ready    <none>   21h   v1.22.10-aliyun.1
cn-beijing.192.168.22.20   Ready    <none>   21h   v1.22.10-aliyun.1
```

注意⚠️：**请检查json文件的格式**

其他：自定义config文件位置： `export KUBECONFIG=文件位置` ，取消环境变量：`unset KUBECONFIG`。 只是补充大可不必哈哈哈😂。