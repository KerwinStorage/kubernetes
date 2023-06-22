GitHub：https://github.com/hidetatz/kubecolor

```bash
wget https://github.com/hidetatz/kubecolor/releases/download/v0.0.25/kubecolor_0.0.25_Linux_x86_64.tar.gz
tar xf kubecolor_0.0.25_Linux_x86_64.tar.gz -C /usr/local/bin/ kubecolor
# 改成别的命令tab就顺畅多了。
mv /usr/local/bin/kubecolor /usr/local/bin/colorkube
alias kubectl="colorkube"
source ~/.bashrc
```

其他：

```bash
#kubectl配置自动tab
apt install -y bash-completion
source /usr/share/bash-completion/bash_completion
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc
```