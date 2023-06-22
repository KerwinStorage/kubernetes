前言：rancher主要可以管理和创建k8s集群并在rancher上面做操作，类似于k8s自带的控制面板能够监控集群但是功能有比面板多，详细专业的解释请看官网。🙈🙉🙊

官网：https://www.rancher.cn/、GitHub：https://github.com/rancher/rancher

规划：一台安装`rancher`的机器、一个`1.9.11` 的k8s集群。

ip地址：`192.168.80.47`

# 1.部署

```bash
docker run --privileged -d --name rancher --restart=unless-stopped -p 80:80 -p 443:443 -v /opt/rancher:/var/lib/rancher rancher/rancher:v2.5.11
docker ps | grep rancher
```

- --privileged：可以使我们启动的容器用 root 的方式启动（在 Rancher 2.5 版本以上需要加）
- --restart：重启策略，我们配置的是 unless-stopped，表示当容器退出时，便会重新启动容器（除非容器之前就处于停止）

访问地址：[https://192.168.80.47](https://192.168.80.47/)

![img](https://img2022.cnblogs.com/blog/1740081/202211/1740081-20221108174146058-1056338556.png)

# 2. rancher添加k8s集群

![img](https://img2022.cnblogs.com/blog/1740081/202211/1740081-20221108174129543-1429165046.png)

导入自己的集群：

![img](https://img2022.cnblogs.com/blog/1740081/202211/1740081-20221108174205765-1440241391.png)

![img](https://img2022.cnblogs.com/blog/1740081/202211/1740081-20221108174219614-127728895.png)

![img](https://img2022.cnblogs.com/blog/1740081/202211/1740081-20221108174336810-2084884343.png)

将上面的命令在能访问 `/root/.kube/config` 文件机器上执行。

**注：****一开始状态会是**`**pending**`**的状态，等好了就是**`**Active**`**状态。**

**![img](https://img2022.cnblogs.com/blog/1740081/202211/1740081-20221108174357025-599042816.png)**

集群内容：

![img](https://img2022.cnblogs.com/blog/1740081/202211/1740081-20221108174446747-258932806.png)

 

结尾：本篇文档只是大致介绍入门，具体操作可以在里面点吧点吧，这里推荐一个软件[lens](https://github.com/lensapp/lens)可以玩玩，功能类似。拜拜🙈🙊🙉

参考：https://blog.csdn.net/weixin_46902396/article/details/122433622