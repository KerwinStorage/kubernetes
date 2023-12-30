# 1.gitlab部署

```bash
# 1.下载镜像
docker pull gitlab/gitlab-ee:14.2.1-ee.0

# 2.创建挂载目录
mkdir -p /home/gitlab/config /home/gitlab/logs /home/gitlab/data

# 3.启动
docker run -d  \
	--name gitlab \
	--hostname ip地址 \
	--publish 443:443 --publish 80:80 --publish 8022:22  \
	--restart always \
	-v /home/gitlab/config:/etc/gitlab  \
	-v /home/gitlab/logs:/var/log/gitlab \
	-v /home/gitlab/data:/var/opt/gitlab \
	gitlab/gitlab-ee:14.2.1-ee.0
# 获取默认    
docker exec -it gitlab cat /etc/gitlab/initial_root_password
```

- `-d`：后台运行，如果去掉会显示日志
- `--hostname`：指定运行的 hostname，可以是域名也可以是 IP。
- `--publish`：端口的映射，可以缩写成 `-p` 443 用于 HTTPS 协议访问，222 用户 SSH 协议访问，因为 22 端口已经被占用。
- `--name`：容器名字
- `--restart`：重启方式，自动重启

注：gitlab比较消耗资源，内存给到4G

# 2.gitlab-runner部署

runner的作用就是给gitlab-ci提供了一个跑程序的环境，优先配置runner选择docker方式。

## 容器部署

这里部署runner请选择跟gitlab通版本的runner

镜像列表：[这里](https://hub.docker.com/r/gitlab/gitlab-runner/tags?page=1&ordering=last_updated)

```bash
# 1.拉去gitlab-runner镜像: 注意需要与gitlab版本相同
docker pull gitlab/gitlab-runner:v14.2.0

# 2.运行gitlab runner镜像
docker run -d --name gitlab-runner --restart always \
       -v /srv/gitlab-runner/config:/etc/gitlab-runner \
       -v /var/run/docker.sock:/var/run/docker.sock \
       gitlab/gitlab-runner:v14.2.0

# 3.注册gitlab runner到gitlab，进入下面👇🏻地址看状态
http://ip地址/admin/runners

# 4.在上面👆🏻这个地址页面找到And this registration token:
# 服务器运行一下命令，需要替换url为gitlab地址、registration-token为复制的token：
docker exec gitlab-runner gitlab-runner register -n \
       --url http://ip地址 \
       --registration-token W3xW5kTP8JMs3mdTEw3h \
       --tag-list pinpoint \
       --executor shell \
       --description "pinpoint"

# 5.刷新gitlab页面观察到runer已经online:
```

- `--url`：gitlab地址
- `--registration-token`：gitlab的token
- `--tag-list`：runner的tag，在编写yaml时可以根据需求进行制定
- `--executor`：运行的环境
- `--description`：描述
- `--docker-image`：这个参数是runner用docker方式使用的

注：这里选择的是shell方式去配置的，那么环境就是选择了容器本地的环境，job本地仓库路径也是在容器内。

runner环境使用docker方式启动：

```bash
docker exec gitlab-runner gitlab-runner register -n \
       --url http://192.168.16.243 \
       --registration-token W3xW5kTP8JMs3mdTEw3h \
       --tag-list pinpoint \
       --executor docker \
       --docker-image centos:v1 \
       --description "pinpoint"
```

成功后会在`/srv/gitlab-runner/config`目录下生成一个`config.toml`配置文件，并且在gitlab的Admin Area -> Runners界面看到注册成功的runner，如果运行环境选择docker的话，我们可以将环境改成docker在`yaml`文件上方写入`image: docker:20.10.16`，具体介绍请看[介绍文档](https://docs.gitlab.com/ee/ci/docker/using_docker_build.html)。

注：这里token注册获取方式有两个地方，一个是admin全局的另外一个是在仓库本地CICD配置中可以找到，会给到一个地址和一个token。

## 3.本地部署

本地部署在配置CICD界面会给出部署，下面就是根据官方提供的安装的，但是版本不和gitlab同步为15版本。

```bash
# 1.下载
sudo curl -L --output /usr/local/bin/gitlab-runner https://gitlab-runner-downloads.s3.amazonaws.com/latest/binaries/gitlab-runner-linux-amd64
# 2.可执行权限
sudo chmod +x /usr/local/bin/gitlab-runner
# 3.创建gitlab-runner用户
sudo useradd --comment 'GitLab Runner' --create-home gitlab-runner -m -d /home/gitlab-runner  -s /bin/bash
# 4.安装服务
sudo gitlab-runner install --user=gitlab-runner --working-directory=/home/gitlab-runner
# 5.开启
sudo gitlab-runner start
# 6.token连接，这个是使用仓库里面的token，一路回车
sudo gitlab-runner register --url http://192.168.16.243 --tag-list pinpoint --executor shell --registration-token ocTBKJNWog42s86udNhs --description "pinpoint"
# 7.最后可以在仓库内显示结果,/etc/gitlab-runner/config.toml文件生成
```

gitlab-runner总结：runner才是cicd的执行环境，一个gitlab可以配置多个runner，每个runner只能服务于一个gitlab，一个项目可以由多个runner服务