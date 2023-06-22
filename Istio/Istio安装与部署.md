## 1. 介绍

- k8s版本：1.22.10
- Istio：1.15.3
- [Release Istio 1.15.3 · istio/istio (github.com)](https://github.com/istio/istio/releases/tag/1.15.3)
- [Istio / 入门](https://istio.io/latest/zh/docs/setup/getting-started/#uninstall)

介绍内容：[Istio / Istio 服务网格](https://istio.io/latest/zh/about/service-mesh/) [什么是 Istio？为什么 Kubernetes 需要 Istio？ · Jimmy Song](https://jimmysong.io/blog/what-is-istio-and-why-does-kubernetes-need-it/)

## 2. 安装Istio

### 2.1 下载

[Release Istio 1.15.3 · istio/istio (github.com)](https://github.com/istio/istio/releases/tag/1.15.3)

```bash
wegt https://github.com/istio/istio/releases/download/1.15.3/istio-1.15.3-linux-amd64.tar.gz
tar zxvf istio-1.15.3-linux-amd64.tar.gz
cd istio-1.15.3
```

这些安装目录包含如下：

- `install/kubernetes`: 针对kubernetes的安装YAML文件.
- `samples/`: 应用案例
- `bin/`: istioctl客户端二进制文件。

### 2.2 设置环境变量

```bash
export PATH=$PWD/bin:$PATH
echo $PATH
# 加载生效
source /etc/profile
# 查看版本
istioctl version
no running Istio pods in "istio-system"
1.15.3
#检查Istio版本是否符合
istioctl x precheck
✔ No issues found when checking the cluster. Istio is safe to install or upgrade!
  To get started, check out https://istio.io/latest/docs/setup/getting-started/
```

或者执行如下：

```bash
ln -s  /root/istio-1.15.3/bin/istioctl /usr/local/bin/istioctl
```

### 2.3 使用demo配置安装Istio

1. 针对安装，在这里使用`demo`的配置文件。它被选择为具有一组用于测试的良好默认设置，但是还有用于生产或性能测试的其他配置文件。

```bash
istioctl install --set profile=demo -y
✔ Istio core installed
✔ Istiod installed
✔ Ingress gateways installed
✔ Egress gateways installed
✔ Installation complete                                                                                                                                                      Making this installation the default for injection and validation.

Thank you for installing Istio 1.15.  Please take a few minutes to tell us about your install/upgrade experience!  https://forms.gle/SWHFBmwJspusK1hv6
```

1. 给命名空间添加标签，指示 Istio 在部署应用的时候，自动注入 Envoy 边车代理：

```bash
kubectl label namespace default istio-injection=enabled
namespace/default labeled
kubectl get namespace -L istio-injection
# 取消自动注入命令如下
kubectl label namespace default istio-injection-
```

## 3. 部署Bookinfo示例

这个应用模仿在线书店的一个分类，显示一本书的信息。 页面上会显示一本书的描述，书籍的细节（ISBN、页数等），以及关于这本书的一些评论。

Bookinfo 应用分为四个单独的微服务：

- `productpage`. 这个微服务会调用 `details` 和 `reviews` 两个微服务，用来生成页面。
- `details`. 这个微服务中包含了书籍的信息。
- `reviews`. 这个微服务中包含了书籍相关的评论。它还会调用 `ratings` 微服务。
- `ratings`. 这个微服务中包含了由书籍评价组成的评级信息。

`reviews` 微服务有 3 个版本：

- v1 版本不会调用 `ratings` 服务。
- v2 版本会调用 `ratings` 服务，并使用 1 到 5 个黑色星形图标来显示评分信息。
- v3 版本会调用 `ratings` 服务，并使用 1 到 5 个红色星形图标来显示评分信息。

下图展示了这个应用的端到端架构。

![img](https://istio.io/v1.15/zh/docs/examples/bookinfo/noistio.svg)

1. 部署 [`Bookinfo` 示例应用](https://istio.io/latest/zh/docs/examples/bookinfo/)：

```bash
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
service/details created
serviceaccount/bookinfo-details created
deployment.apps/details-v1 created
service/ratings created
serviceaccount/bookinfo-ratings created
deployment.apps/ratings-v1 created
service/reviews created
serviceaccount/bookinfo-reviews created
deployment.apps/reviews-v1 created
deployment.apps/reviews-v2 created
deployment.apps/reviews-v3 created
service/productpage created
serviceaccount/bookinfo-productpage created
deployment.apps/productpage-v1 created
```

1. 应用很快会启动起来。当每个 Pod 准备就绪时，Istio 边车将伴随应用一起部署。

```bash
kubectl get services
NAME          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
details       ClusterIP   10.96.51.104     <none>        9080/TCP   30s
kubernetes    ClusterIP   10.96.0.1        <none>        443/TCP    2d14h
productpage   ClusterIP   10.100.101.131   <none>        9080/TCP   29s
ratings       ClusterIP   10.106.170.86    <none>        9080/TCP   30s
reviews       ClusterIP   10.100.0.211     <none>        9080/TCP   29s
```

和

```bash
kubectl get pods
NAME                                      READY   STATUS    RESTARTS        AGE
details-v1-6758dd9d8d-nzrcb               2/2     Running   0               2m5s
productpage-v1-797d845774-zh8m8           2/2     Running   0               2m3s
ratings-v1-f849dc6d-9lx6j                 2/2     Running   0               2m4s
reviews-v1-74fb8fdbd8-2npwm               2/2     Running   0               2m4s
reviews-v2-58d564d4db-45tjh               2/2     Running   0               2m4s
reviews-v3-55545c459b-m9b8c               2/2     Running   0               2m4s
```

1. 校验所有的工作是否正常

```bash
kubectl exec -it $(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}') -c ratings -- curl productpage:9080/productpage | grep -o "<title>.*</title>"
```

### 3.1 对外开放Bookinfo

### 为应用程序定义Ingress网关：

```bash
kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml
kubectl get gateway
```

确认网关创建完成：

```bash
kubectl get gateway -n=default
NAME               AGE
bookinfo-gateway   80s

kubectl get svc -n=istio-system
NAME                   TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                                                                      AGE
istio-egressgateway    ClusterIP      10.100.166.29    <none>        80/TCP,443/TCP                                                               18m
istio-ingressgateway   LoadBalancer   10.104.120.244   <pending>     15021:32112/TCP,80:30215/TCP,443:32547/TCP,31400:30735/TCP,15443:31409/TCP   18m
istiod                 ClusterIP      10.108.51.181    <none>        15010/TCP,15012/TCP,443/TCP,15014/TCP                                        18m
```

将istio-ingressgateway Service的Type修改为LoadBalancer：

```bash
kubectl edit svc istio-ingressgateway -n=istio-system
spec:
  type: LoadBalancer
  externalIPs:
  - 节点ip地址
  
kubectl get svc istio-ingressgateway -n istio-system
NAME                   TYPE           CLUSTER-IP       EXTERNAL-IP     PORT(S)                                                                      AGE
istio-ingressgateway   LoadBalancer   10.104.120.244   192.168.80.45   15021:32112/TCP,80:30215/TCP,443:32547/TCP,31400:30735/TCP,15443:31409/TCP   22h
```

浏览访问：http://192.168.80.45:30215/productpage

## 4. 安装Kiali和其他插件

```bash
kubectl apply -f samples/addons
serviceaccount/grafana created
configmap/grafana created
service/grafana created
deployment.apps/grafana created
configmap/istio-grafana-dashboards created
configmap/istio-services-grafana-dashboards created
deployment.apps/jaeger created
service/tracing created
service/zipkin created
service/jaeger-collector created
serviceaccount/kiali created
configmap/kiali created
clusterrole.rbac.authorization.k8s.io/kiali-viewer created
clusterrole.rbac.authorization.k8s.io/kiali created
clusterrolebinding.rbac.authorization.k8s.io/kiali created
role.rbac.authorization.k8s.io/kiali-controlplane created
rolebinding.rbac.authorization.k8s.io/kiali-controlplane created
service/kiali created
deployment.apps/kiali created
serviceaccount/prometheus created
configmap/prometheus created
clusterrole.rbac.authorization.k8s.io/prometheus created
clusterrolebinding.rbac.authorization.k8s.io/prometheus created
service/prometheus created
deployment.apps/prometheus created
```

确认所有Pod正常启动：

```bash
kubectl get pod -n istio-system
NAME                                    READY   STATUS    RESTARTS   AGE
grafana-6d69f655fb-w4fq5                1/1     Running   0          2m48s
istio-egressgateway-77cf54b878-rrl9v    1/1     Running   0          26m
istio-ingressgateway-7f5ddd54c8-nps8b   1/1     Running   0          26m
istiod-76db9fbfc-x5gfh                  1/1     Running   0          26m
jaeger-5858c698bf-nlt2c                 1/1     Running   0          2m47s
kiali-64c4f869fb-rvw6l                  1/1     Running   0          2m43s
prometheus-6956c8c6c5-2tpgv             2/2     Running   0          2m41s
kubectl rollout status deployment/kiali -n istio-system
```

将kiali Service的Type修改为LoadBalancer：

```bash
kubectl edit svc -n istio-system kiali
spec:
  type: LoadBalancer
  externalIPs:
  - 节点ip地址
kubectl get svc -n istio-system kiali
NAME    TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)                          AGE
kiali   LoadBalancer   10.110.103.63   192.168.80.45   20001:32161/TCP,9090:30392/TCP   159m
```

访问：[http://192.168.80.45:32161](http://192.168.80.45:32161/)

## 遇到的报错：

1. 连接不到istio的8080端口

![img](https://img2023.cnblogs.com/blog/1740081/202212/1740081-20221220152711123-84790479.png)

> ```bash
> unable to proxy Istiod pods. Make sure your Kubernetes API server has access to the Istio control plane through 8080 port
> ```

解决链接：[Istio / LocalhostListener](https://istio.io/latest/docs/reference/config/analysis/ist0143/) [无法获取 Istio 对象列表：无法代理 Istiod pod。确保您的 Kubernetes API 服务器可以通过 8080 端口访问 Istio 控制平面 ·问题 #4679 ·基亚利/基亚利 (github.com)](https://github.com/kiali/kiali/issues/4679) [识别、记录并可能删除 kubelet 依赖项 ·问题 #26093 ·Kubernetes/Kubernetes (github.com)](https://github.com/kubernetes/kubernetes/issues/26093)

```bash
# 用于容器内的端口转发
apt-get install socat
```

1. Grafana URL is not set in Kiali configuration

```bash
kubectl get svc -n istio-system
NAME                   TYPE           CLUSTER-IP       EXTERNAL-IP     PORT(S)                                                                      AGE
grafana                ClusterIP      10.104.155.33    <none>          3000/TCP                                                                     132m
istio-egressgateway    ClusterIP      10.100.166.29    <none>          80/TCP,443/TCP                                                               22h
istio-ingressgateway   LoadBalancer   10.104.120.244   192.168.80.45   15021:32112/TCP,80:30215/TCP,443:32547/TCP,31400:30735/TCP,15443:31409/TCP   22h
istiod                 ClusterIP      10.108.51.181    <none>          15010/TCP,15012/TCP,443/TCP,15014/TCP                                        22h
jaeger-collector       ClusterIP      10.96.127.129    <none>          14268/TCP,14250/TCP,9411/TCP                                                 132m
kiali                  LoadBalancer   10.110.103.63    192.168.80.45   20001:32161/TCP,9090:30392/TCP                                               132m
prometheus             ClusterIP      10.110.76.52     <none>          9090/TCP                                                                     132m
tracing                ClusterIP      10.101.47.150    <none>          80/TCP,16685/TCP                                                             132m
zipkin                 ClusterIP      10.109.151.121   <none>          9411/TCP                                                                     132m


kubectl get ConfigMap -n istio-system
NAME                                  DATA   AGE
grafana                               4      133m
istio                                 2      22h
istio-ca-root-cert                    1      22h
istio-gateway-deployment-leader       0      22h
istio-gateway-status-leader           0      22h
istio-grafana-dashboards              2      133m
istio-leader                          0      22h
istio-namespace-controller-election   0      22h
istio-services-grafana-dashboards     4      133m
istio-sidecar-injector                2      22h
kiali                                 1      133m
kube-root-ca.crt                      1      22h
prometheus                            5      133m

# kiali 报错Grafana URL is not set in Kiali ，需要配置kiali configmap：
vim ./samples/addons/kiali.yaml
external_services:
  grafana:
    url: "http://10.104.155.33:3000"
    
kubectl apply -f ./samples/addons/kiali.yaml

# 然后删除kiali pod重新启动一个
```

## 5. 卸载

删除 `Bookinfo` 示例应用和配置, 参阅[清理 `Bookinfo`](https://istio.io/latest/zh/docs/examples/bookinfo/#cleanup)。

Istio 卸载程序按照层次结构逐级的从 `istio-system` 命令空间中删除 RBAC 权限和所有资源。对于不存在的资源报错，可以安全的忽略掉，毕竟它们已经被分层地删除了。

```bash
kubectl delete -f samples/addons
istioctl uninstall -y --purge
```

命名空间 `istio-system` 默认情况下并不会被移除。 不需要的时候，使用下面命令移除它：

```bash
kubectl delete namespace istio-system
```

指示 Istio 自动注入 Envoy 边车代理的标签默认也不移除。 不需要的时候，使用下面命令移除它。

```bash
kubectl label namespace default istio-injection-
```

 

[Istio / 生态系统](https://istio.io/latest/zh/about/ecosystem/)