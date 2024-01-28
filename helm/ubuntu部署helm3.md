# 							Ubuntu安装helm3实例

为了在Ubuntu系统上部署Helm，你需要按照以下步骤操作：

1. **先决条件**: 确保你有一个Kubernetes集群，并且你的本地机器已经安装了`kubectl`工具，并且配置好了与你的Kubernetes集群通信。

2. **下载Helm**: 你可以从Helm的官方发布页面下载最新版本的Helm包。或者，你可以使用脚本自动安装它。

   使用Helm官方脚本安装：
   ```bash
   curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
   ```

   ubuntu下载：

   ```shell
   curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
   sudo apt-get install apt-transport-https --yes
   echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
   sudo apt-get update
   sudo apt-get install helm
   ```

3. **验证Helm安装**: 通过运行以下命令来检查Helm版本，以确认它是否已正确安装。

   ```bash
   helm version
   ```

4. **添加仓库（可选）**: Helm有许多可用的chart仓库，你可以根据需要添加它们。默认情况下，你可能想要添加官方的stable仓库。
   ```bash
   helm repo add stable https://charts.helm.sh/stable
   helm repo update
   ```

5. **安装Charts**: 一旦Helm安装完成，你就可以开始安装charts了。例如，你可以安装一个名为`my-release`的nginx chart实例，如下所示：

   ```bash
   helm install my-release stable/nginx
   ```

6. **查看Releases**: 要查看你部署的Helm releases列表，使用以下命令：
   ```bash
   helm list
   ```

7. **命令永久补全**:

   ```shell
   echo 'source <(helm completion bash)' >>~/.bashrc
   source ~/.bashrc
   ```

现在已经在Ubuntu上成功安装并配置了Helm，并且知道如何添加仓库和安装charts。