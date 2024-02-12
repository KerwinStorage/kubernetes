# 						kubeclipper的使用

# 1.部署 AIO

官网：https://kubeclipper.io

Github地址：https://github.com/kubeclipper/kubeclipper/blob/master/README_zh.md

项目CNCF地址：https://www.cncf.io/projects/kubeclipper

## 1.1.下载并安装 kcctl

下载指定版本：https://github.com/kubeclipper/kubeclipper/releases

```shell
# 默认安装最新的发行版
curl -sfL https://oss.kubeclipper.io/get-kubeclipper.sh | bash -
# 安装指定版本
curl -sfL https://oss.kubeclipper.io/get-kubeclipper.sh | KC_VERSION=v1.3.1 bash -
# 如果您在中国， 您可以在安装时使用 cn  环境变量, 此时 KubeClipper 会使用 registry.aliyuncs.com/google_containers 代替 k8s.gcr.io
curl -sfL https://oss.kubeclipper.io/get-kubeclipper.sh | KC_REGION=cn bash -
```

## 1.2.开始安装

```shell
# 使用私钥，我使用的这个，因为节点已经免密了。
kcctl deploy --user root --pk-file /root/.ssh/id_rsa
2024-02-12T18:56:28+08:00       INFO    Using auto detected IPv4 address on interface ens33: 192.168.80.45/24
[2024-02-12T18:56:28+08:00][INFO] node-ip-detect inherits from ip-detect: first-found
[2024-02-12T18:56:28+08:00][INFO] run in aio mode.
[2024-02-12T18:56:28+08:00][INFO] ============>kc-etcd PRECHECK ...
[2024-02-12T18:56:28+08:00][INFO] ============>kc-etcd PRECHECK OK!
[2024-02-12T18:56:28+08:00][INFO] ============>kc-server PRECHECK ...
[2024-02-12T18:56:29+08:00][INFO] ============>kc-server PRECHECK OK!
[2024-02-12T18:56:29+08:00][INFO] ============>kc-agent PRECHECK ...
[2024-02-12T18:56:29+08:00][INFO] ============>kc-agent PRECHECK OK!
[2024-02-12T18:56:29+08:00][INFO] ============>TIME-LAG PRECHECK ...
[2024-02-12T18:56:29+08:00][INFO] BaseLine Time: 2024-02-12T18:56:29+08:00
[2024-02-12T18:56:29+08:00][INFO] [192.168.80.45] -0.577350486 seconds
[2024-02-12T18:56:29+08:00][INFO] all nodes time lag less then 5 seconds
[2024-02-12T18:56:29+08:00][INFO] ============>TIME-LAG PRECHECK OK!
[2024-02-12T18:56:29+08:00][INFO] ============>NTP PRECHECK ...
[2024-02-12T18:56:30+08:00][INFO] ============>NTP PRECHECK OK!
[2024-02-12T18:56:30+08:00][INFO] ============>sudo PRECHECK ...
[2024-02-12T18:56:30+08:00][INFO] ============>sudo PRECHECK OK!
[2024-02-12T18:56:30+08:00][INFO] ============>ipDetect PRECHECK ...
[2024-02-12T18:56:30+08:00][INFO] ============>ipDetect PRECHECK OK!
[2024-02-12T18:56:33+08:00][INFO] [192.168.80.45]transfer total size is: 1.64KB ;speed is 1KB
[2024-02-12T18:56:35+08:00][INFO] [192.168.80.45]transfer total size is: 1.09KB ;speed is 1KB
[2024-02-12T18:56:37+08:00][INFO] [192.168.80.45]transfer total size is: 1.64KB ;speed is 1KB
[2024-02-12T18:56:38+08:00][INFO] [192.168.80.45]transfer total size is: 1.21KB ;speed is 1KB
[2024-02-12T18:56:40+08:00][INFO] [192.168.80.45]transfer total size is: 1.64KB ;speed is 1KB
[2024-02-12T18:56:42+08:00][INFO] [192.168.80.45]transfer total size is: 1.22KB ;speed is 1KB
[2024-02-12T18:56:43+08:00][INFO] [192.168.80.45]transfer total size is: 1.64KB ;speed is 1KB
[2024-02-12T18:56:45+08:00][INFO] [192.168.80.45]transfer total size is: 1.22KB ;speed is 1KB
[2024-02-12T18:56:47+08:00][INFO] [192.168.80.45]transfer total size is: 1.64KB ;speed is 1KB
[2024-02-12T18:56:48+08:00][INFO] [192.168.80.45]transfer total size is: 1.23KB ;speed is 1KB
[2024-02-12T18:56:50+08:00][INFO] [192.168.80.45]transfer total size is: 1.64KB ;speed is 1KB
[2024-02-12T18:56:52+08:00][INFO] [192.168.80.45]transfer total size is: 1.25KB ;speed is 1KB
[2024-02-12T18:56:54+08:00][INFO] [192.168.80.45]transfer total size is: 1.64KB ;speed is 1KB
[2024-02-12T18:56:55+08:00][INFO] [192.168.80.45]transfer total size is: 1.25KB ;speed is 1KB
[2024-02-12T18:56:57+08:00][INFO] [192.168.80.45]transfer total size is: 1.64KB ;speed is 1KB
[2024-02-12T18:56:58+08:00][INFO] [192.168.80.45]transfer total size is: 1.09KB ;speed is 1KB
[2024-02-12T18:57:00+08:00][INFO] [192.168.80.45]transfer total size is: 1.64KB ;speed is 1KB
[2024-02-12T18:57:02+08:00][INFO] [192.168.80.45]transfer total size is: 1.25KB ;speed is 1KB
[2024-02-12T18:57:04+08:00][INFO] [192.168.80.45]transfer total size is: 1.64KB ;speed is 1KB
[2024-02-12T18:57:05+08:00][INFO] [192.168.80.45]transfer total size is: 1.25KB ;speed is 1KB
[2024-02-12T18:57:07+08:00][INFO] [192.168.80.45]transfer total size is: 1.09KB ;speed is 1KB
[2024-02-12T18:57:09+08:00][INFO] [192.168.80.45]transfer total size is: 1.64KB ;speed is 1KB
[2024-02-12T18:57:11+08:00][INFO] [192.168.80.45]transfer total size is: 1.24KB ;speed is 1KB
[2024-02-12T18:57:11+08:00][INFO] ------ Send packages ------
--2024-02-12 18:57:11--  https://oss.kubeclipper.io/release/v1.4.0/kc-amd64.tar.gz
Resolving oss.kubeclipper.io (oss.kubeclipper.io)... 161.117.118.73
Connecting to oss.kubeclipper.io (oss.kubeclipper.io)|161.117.118.73|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 965806270 (921M) [application/x-gzip]
Saving to: ‘kc-amd64.tar.gz’

kc-amd64.tar.gz                             100%[========================================================================================>] 921.06M  5.00MB/s    in 2m 53s

2024-02-12 19:00:06 (5.32 MB/s) - ‘kc-amd64.tar.gz’ saved [965806270/965806270]

192.168.80.45: done!
[2024-02-12T19:00:24+08:00][INFO] ------ Install kc-etcd ------
[2024-02-12T19:00:31+08:00][INFO] ------ Install kc-server ------
[2024-02-12T19:00:44+08:00][INFO] ------ Install kc-agent ------
[2024-02-12T19:00:47+08:00][INFO] ------ Install kc-console ------
[2024-02-12T19:00:49+08:00][INFO] ------ Delete intermediate files ------
[2024-02-12T19:00:50+08:00][INFO] ------ Dump configs ------
[2024-02-12T19:00:50+08:00][INFO] ------ Upload configs ------
192.168.80.45: done!

 _   __      _          _____ _ _
| | / /     | |        /  __ \ (_)
| |/ / _   _| |__   ___| /  \/ |_ _ __  _ __   ___ _ __
|    \| | | | '_ \ / _ \ |   | | | '_ \| '_ \ / _ \ '__|
| |\  \ |_| | |_) |  __/ \__/\ | | |_) | |_) |  __/ |
\_| \_/\__,_|_.__/ \___|\____/_|_| .__/| .__/ \___|_|
                                 | |   | |
                                 |_|   |_|
        repository: github.com/kubeclipper


# 使用密码
kcctl deploy --user root --passwd password
```

### 1.2.1报错处理

```shell
[2024-02-12T18:33:13+08:00][FATAL] Internal server error due to reason configmaps.core.kubeclipper.io "deploy-config" already exists
goroutine 1 [running]:
github.com/kubeclipper/kubeclipper/pkg/cli/logger.stacks(0x0)
        /home/runner/work/kubeclipper/kubeclipper/pkg/cli/logger/logger.go:163 +0x89
github.com/kubeclipper/kubeclipper/pkg/cli/logger.(*loggingT).output(0x262f9e0?, 0xc000810270?, 0x3)
        /home/runner/work/kubeclipper/kubeclipper/pkg/cli/logger/logger.go:130 +0x75
github.com/kubeclipper/kubeclipper/pkg/cli/logger.(*loggingT).println(0x1eb79a0?, 0x26518a0?, {0xc000961b18, 0x1, 0x1})
        /home/runner/work/kubeclipper/kubeclipper/pkg/cli/logger/logger.go:150 +0x85
github.com/kubeclipper/kubeclipper/pkg/cli/logger.Fatal(...)
        /home/runner/work/kubeclipper/kubeclipper/pkg/cli/logger/logger.go:320
github.com/kubeclipper/kubeclipper/pkg/cli/deploy.uploadDeployConfig(0xc000556b70?, 0xc000556ba0?)
        /home/runner/work/kubeclipper/kubeclipper/pkg/cli/deploy/deploy.go:969 +0x225
github.com/kubeclipper/kubeclipper/pkg/cli/deploy.(*DeployOptions).uploadConfig(0xc0004ac990)
        /home/runner/work/kubeclipper/kubeclipper/pkg/cli/deploy/deploy.go:931 +0x74d
github.com/kubeclipper/kubeclipper/pkg/cli/deploy.(*DeployOptions).RunDeploy(0xc0004ac990?)
        /home/runner/work/kubeclipper/kubeclipper/pkg/cli/deploy/deploy.go:458 +0x1c9
github.com/kubeclipper/kubeclipper/pkg/cli/deploy.NewCmdDeploy.func1(0xc000434000?, {0xc0000d0f40?, 0x4?, 0x4?})
        /home/runner/work/kubeclipper/kubeclipper/pkg/cli/deploy/deploy.go:160 +0x5d
github.com/spf13/cobra.(*Command).execute(0xc000434000, {0xc0000d0f00, 0x4, 0x4})
        /home/runner/go/pkg/mod/github.com/spf13/cobra@v1.2.1/command.go:860 +0x663
github.com/spf13/cobra.(*Command).ExecuteC(0xc000111400)
        /home/runner/go/pkg/mod/github.com/spf13/cobra@v1.2.1/command.go:974 +0x3bd
github.com/spf13/cobra.(*Command).Execute(...)
        /home/runner/go/pkg/mod/github.com/spf13/cobra@v1.2.1/command.go:902
main.main()
        /home/runner/work/kubeclipper/kubeclipper/cmd/kcctl/main.go:30 +0x4a
        
# 清理重新来一遍
kcctl clean -A --config ~/.kc/config
```

## 1.3.登录界面

web地址：http://192.168.80.45，账户密码：`admin / Thinkbig1`。

![img](https://img2024.cnblogs.com/blog/1740081/202402/1740081-20240212191114021-1285500230.png)

## 1.4.纳管集群

![img](https://img2024.cnblogs.com/blog/1740081/202402/1740081-20240212191513311-214653375.png)

纳管中：

![img](https://img2024.cnblogs.com/blog/1740081/202402/1740081-20240212191612301-440288900.png)

纳管完成：

![img](https://img2024.cnblogs.com/blog/1740081/202402/1740081-20240212191709357-1859406030.png)

结尾：因为节点是一个单master所以没有部署[高可用版本](https://kubeclipper.io/docs/deployment-docs/ha-deploy/)。控制台有很多功能可以使用，这里就不在一一赘述。完

参考文档：

- [www.lixueduan.com](https://www.lixueduan.com/posts/cloudnative/02-kubeclipper-join-cncf/#1%E4%BD%A0%E5%90%AC%E6%88%91%E8%A7%A3%E9%87%8A)
- https://kubeclipper.io/docs/getting-started/aio-env