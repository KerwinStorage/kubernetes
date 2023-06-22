问题描述：

> **进入pod内发现只能ping通内部node和pod地址，baidu.com解析不到**

1. CoreDNS 的ConfigMap重定向到文件内

```bash
kubectl get cm -n kube-system coredns -o yaml > CoreDNS_ConfigMap.yaml

vim CoreDNS_ConfigMap.yaml
apiVersion: v1
data:
  Corefile: |
    .:53 {
        log
        errors
        health
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
          pods insecure
          fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        # forward . /etc/resolv.conf
        forward . 8.8.8.8 #加入本地用的dns解析服务器
        cache 30
        loop
        reload
        loadbalance
    }
kind: ConfigMap
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","data":{"Corefile":".:53 {\n    log\n    errors\n    health\n    ready\n    kubernetes cluster.local in-addr.arpa ip6.arpa {\n      pods insecure\n      fallthrough in-addr.arpa ip6.arpa\n    }\n    prometheus :9153\n    forward . 8.8.8.8\n    cache 30\n    loop\n    reload\n    loadbalance\n}\n"},"kind":"ConfigMap","metadata":{"annotations":{},"name":"coredns","namespace":"kube-system"}}
  creationTimestamp: "2022-12-18T10:11:57Z"
  name: coredns
  namespace: kube-system
  resourceVersion: "663977"
  uid: 97b64ce7-3850-4d34-974f-ae53f47c3a08

# 删除之前的旧pod
# 生效
kubectl replace -f CoreDNS_ConfigMap.yaml
```

- errors: 输出错误信息到控制台。
- health：CoreDNS 进行监控检测，检测地址为 http://localhost:8080/health 如果状态为不健康则让 Pod 进行重启。
- ready: 全部插件已经加载完成时，将通过 endpoints 在 8081 端口返回 HTTP 状态 200。
- kubernetes：CoreDNS 将根据 Kubernetes 服务和 pod 的 IP 回复 DNS 查询。
- prometheus：是否开启 CoreDNS Metrics 信息接口，如果配置则开启，接口地址为 http://localhost:9153/metrics
- forward：任何不在Kubernetes 集群内的域名查询将被转发到预定义的解析器 (/etc/resolv.conf)或者8.8.8.8dns服务器。
- cache：启用缓存，30 秒 TTL。
- loop：检测简单的转发循环，如果找到循环则停止 CoreDNS 进程。
- reload：监听 CoreDNS 配置，如果配置发生变化则重新加载配置。
- loadbalance：DNS 负载均衡器，默认 round_robin。

 

1. 创建pod测试在pod内是否可以ping通baidu.com

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: busybox
  namespace: default
spec:
  containers:
  - name: busybox
    image: docker.io/library/busybox:1.28
    command:
      - sleep
      - "3600"
    imagePullPolicy: IfNotPresent
  restartPolicy: Always
EOF

kubectl  get pod

kubectl exec -ti busybox -- sh
/ # ping baidu.com
PING baidu.com (39.156.66.10): 56 data bytes
64 bytes from 39.156.66.10: seq=0 ttl=127 time=41.059 ms
64 bytes from 39.156.66.10: seq=1 ttl=127 time=71.731 ms
64 bytes from 39.156.66.10: seq=2 ttl=127 time=123.897 ms
64 bytes from 39.156.66.10: seq=3 ttl=127 time=142.284 ms
64 bytes from 39.156.66.10: seq=4 ttl=127 time=43.085 ms
```

这样做的方法是：域名解析不用pod里的dns服务了，强制转发到外边，用外边的dns服务来做解析，从而避免pod里dns服务解析不了的问题。

参考：[k8s中pod内dns无法解析的问题 | Pod (lmlphp.com)](https://www.lmlphp.com/user/60738/article/item/1516437/)

