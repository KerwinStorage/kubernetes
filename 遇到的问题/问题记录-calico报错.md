报错内容：

![img](https://img2022.cnblogs.com/blog/1740081/202211/1740081-20221110223849213-566378535.png)

Unhealthy  80s (x5315 over 2d4h)  kubelet  (combined from similar events): Readiness probe failed: 2022-11-10 13:23:27.611 [INFO][155224] confd/health.go 180: Number of node(s) with BGP peering established = 0

问题描述：原因是节点网卡比较多，calico选择了错误的网卡，所以我们指定一下网卡即可。

```bash
kubectl set env daemonset/calico-node -n kube-system IP_AUTODETECTION_METHOD=interface=ens33
```

结果：

![img](https://img2022.cnblogs.com/blog/1740081/202211/1740081-20221110224031767-1131479804.png)

 