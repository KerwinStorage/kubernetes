# 						kube-prometheus的使用

# 1.兼容性

Github：https://github.com/prometheus-operator/kube-prometheus

| kube-prometheus stack                                        | Kubernetes 1.22 | Kubernetes 1.23 | Kubernetes 1.24 | Kubernetes 1.25 | Kubernetes 1.26 | Kubernetes 1.27 | Kubernetes 1.28 |
| ------------------------------------------------------------ | --------------- | --------------- | --------------- | --------------- | --------------- | --------------- | --------------- |
| [`release-0.10`](https://github.com/prometheus-operator/kube-prometheus/tree/release-0.10) | ✔               | ✔               | ✗               | ✗               | x               | x               | x               |
| [`release-0.11`](https://github.com/prometheus-operator/kube-prometheus/tree/release-0.11) | ✗               | ✔               | ✔               | ✗               | x               | x               | x               |
| [`release-0.12`](https://github.com/prometheus-operator/kube-prometheus/tree/release-0.12) | ✗               | ✗               | ✔               | ✔               | x               | x               | x               |
| [`release-0.13`](https://github.com/prometheus-operator/kube-prometheus/tree/release-0.13) | ✗               | ✗               | ✗               | x               | ✔               | ✔               | ✔               |
| [`main`](https://github.com/prometheus-operator/kube-prometheus/tree/main) | ✗               | ✗               | ✗               | x               | x               | ✔               | ✔               |

# 2.部署

## 2.1.拉取仓库

```
git clone https://github.com/prometheus-operator/kube-prometheus.git
cd kube-prometheus && git checkout v0.13.0
```

## 2.2.创建命名空间和CRD

```
#首先创建需要的命名空间和 CRDs，等待它们可用后再创建其余资源
kubectl create -f manifests/setup
```

## 2.3.创建容器资源

```
kubectl create -f manifests/

# 查看pod启动情况
kubectl get pods -n monitoring
NAME                                   READY   STATUS    RESTARTS        AGE
alertmanager-main-0                    2/2     Running   2 (8m20s ago)   43m
alertmanager-main-1                    2/2     Running   2 (9m50s ago)   15m
alertmanager-main-2                    2/2     Running   2 (8m20s ago)   43m
blackbox-exporter-59dddb7bb6-jgk8m     3/3     Running   3 (8m19s ago)   44m
grafana-79f47474f7-w4qpb               1/1     Running   1 (8m19s ago)   44m
kube-state-metrics-56859545d5-8pl9s    3/3     Running   0               59s
node-exporter-2wkx2                    2/2     Running   2 (9m50s ago)   44m
node-exporter-m9zfn                    2/2     Running   2 (10m ago)     44m
node-exporter-rczfn                    2/2     Running   2 (8m19s ago)   44m
prometheus-adapter-758445cd66-69744    1/1     Running   0               85s
prometheus-adapter-758445cd66-pt52f    1/1     Running   0               86s
prometheus-k8s-0                       2/2     Running   2 (8m19s ago)   43m
prometheus-k8s-1                       2/2     Running   2 (9m50s ago)   15m
prometheus-operator-57cf88fbcb-jqs4z   2/2     Running   2 (8m19s ago)   27m

# 查看svc
kubectl get svc -n monitoring
NAME                    TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
alertmanager-main       ClusterIP   10.100.13.120    <none>        9093/TCP,8080/TCP            45m
alertmanager-operated   ClusterIP   None             <none>        9093/TCP,9094/TCP,9094/UDP   44m
blackbox-exporter       ClusterIP   10.105.76.51     <none>        9115/TCP,19115/TCP           45m
grafana                 ClusterIP   10.96.156.214    <none>        3000/TCP                     45m
kube-state-metrics      ClusterIP   None             <none>        8443/TCP,9443/TCP            45m
node-exporter           ClusterIP   None             <none>        9100/TCP                     45m
prometheus-adapter      ClusterIP   10.111.156.171   <none>        443/TCP                      45m
prometheus-k8s          ClusterIP   10.111.115.151   <none>        9090/TCP,8080/TCP            45m
prometheus-operated     ClusterIP   None             <none>        9090/TCP                     44m
prometheus-operator     ClusterIP   None             <none>        8443/TCP                     45m
```

### 2.3.1.拉起镜像问题

问题描述：镜像在国外导致访问不到

- registry.k8s.io/kube-state-metrics/kube-state-metrics:v2.9.2
- registry.k8s.io/prometheus-adapter/prometheus-adapter:v0.11.1

问题处理：使用`containerd`配置`clash`拉取代码，修改tag将镜像推送到自己的[阿里镜像仓库](https://cr.console.aliyun.com/cn-hangzhou/instance/dashboard)内。

- registry.cn-hangzhou.aliyuncs.com/image-storage/kube-state-metrics:v2.9.2
- registry.cn-hangzhou.aliyuncs.com/image-storage/prometheus-adapter:v0.11.1

参考：https://mp.weixin.qq.com/s/jlHJCBnub3a33UeFLBgeAA