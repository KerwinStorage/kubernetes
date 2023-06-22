Kube-prometheus：https://github.com/coreos/kube-prometheus

## 1.部署👍

```bash
# 下载安装文件
git clone -b release-0.11 --single-branch https://github.com/coreos/kube-prometheus.git
# 安装operator
cd manifests/setup && kubectl create -f .
# 安装prometheus
cd .. && kubectl create -f . 
```

报错处理：

```
unable to recognize "alertmanager-podDisruptionBudget.yaml": no matches for kind "PodDisruptionBudget" in version "policy/v1"
unable to recognize "prometheus-podDisruptionBudget.yaml": no matches for kind "PodDisruptionBudget" in version "policy/v1"
unable to recognize "prometheusAdapter-podDisruptionBudget.yaml": no matches for kind "PodDisruptionBudget" in version "policy/v1"
```

**注：本次部署集群的版本为v1.20.11，PodDisruptionBudget 看在1.20中还是v1beta1，修改为policy/v1beta1**

```bash
# 执行命令行：kubectl api-versions
kubectl api-versions |  grep policy
policy/v1beta1
```

镜像拉取不到处理：

```bash
[root@k8s-node01 ~]# kubectl get po -n monitoring
NAME                                   READY   STATUS             RESTARTS   AGE
kube-state-metrics-54bd6b479c-vrnd5    2/3     ImagePullBackOff   2          44m
prometheus-adapter-7dbf69cc-h4x6s      0/1     ImagePullBackOff   0          44m
prometheus-adapter-7dbf69cc-q6jzb      0/1     ImagePullBackOff   0          44m

# 拉取镜像失败列表
k8s.gcr.io/prometheus-adapter/prometheus-adapter:v0.9.1
k8s.gcr.io/kube-state-metrics/kube-state-metrics:v2.5.0

# 处理方法，获取镜像列表
docker search prometheus-adapter
docker search kube-state-metrics

# 拉取镜像
docker pull lbbi/prometheus-adapter:v0.9.1
docker pull bitnami/kube-state-metrics

# 替换成yaml文件内需要的镜像名称
docker tag lbbi/prometheus-adapter:v0.9.1 k8s.gcr.io/prometheus-adapter/prometheus-adapter:v0.9.1
docker tag bitnami/kube-state-metrics:latest  k8s.gcr.io/kube-state-metrics/kube-state-metrics:v2.5.0
```

## 2. 配置web界面🥱

```bash
# kubectl  get svc -n monitoring
alertmanager-main       ClusterIP   172.16.207.239   <none>        9093/TCP,8080/TCP            97m #告警规则
alertmanager-operated   ClusterIP   None             <none>        9093/TCP,9094/TCP,9094/UDP   97m
blackbox-exporter       ClusterIP   172.16.55.46     <none>        9115/TCP,19115/TCP           97m
grafana                 ClusterIP   172.16.42.30     <none>        3000/TCP                     97m #web界面展示用
kube-state-metrics      ClusterIP   None             <none>        8443/TCP,9443/TCP            97m
node-exporter           ClusterIP   None             <none>        9100/TCP                     97m
prometheus-adapter      ClusterIP   172.16.155.223   <none>        443/TCP                      97m
prometheus-k8s          ClusterIP   172.16.137.76    <none>        9090/TCP,8080/TCP            97m #监控服务界面
prometheus-operated     ClusterIP   None             <none>        9090/TCP                     97m
prometheus-operator     ClusterIP   None             <none>        8443/TCP                     97m
```

创建service使用nodePort去访问

```bash
# grafana
apiVersion: v1
kind: Service
metadata:
  name: grafana-nodeport       # service的名称，全局唯一
  namespace: monitoring
  labels:
    app.kubernetes.io/component: grafana
    app.kubernetes.io/name: grafana
    app.kubernetes.io/part-of: kube-prometheus
    app.kubernetes.io/version: 8.5.5
spec:
  type: NodePort            
  ports:
    - port: 3000
      targetPort: 3000
      nodePort: 30000      # 映射到物理机的端口号
  selector:
    app.kubernetes.io/component: grafana
    app.kubernetes.io/name: grafana
    app.kubernetes.io/part-of: kube-prometheus

# alertmanager-main
apiVersion: v1
kind: Service
metadata:
  name: alertmanager-main-nodeport       # service的名称，全局唯一
  namespace: monitoring
  labels:
    app.kubernetes.io/component: alert-router
    app.kubernetes.io/instance: main
    app.kubernetes.io/name: alertmanager
    app.kubernetes.io/part-of: kube-prometheus
    app.kubernetes.io/version: 0.24.0
spec:
  type: NodePort            
  ports:
    - name: web
      protocol: TCP
      port: 9093
      targetPort: web
      nodePort: 32093 # 映射到物理机的端口号
    - name: reloader-web
      protocol: TCP
      port: 8080
      targetPort: reloader-web
      nodePort: 32080 # 映射到物理机的端口号
  selector:
    app.kubernetes.io/component: alert-router
    app.kubernetes.io/instance: main
    app.kubernetes.io/name: alertmanager
    app.kubernetes.io/part-of: kube-prometheus
    
# prometheus-k8s
apiVersion: v1
kind: Service
metadata:
  name: prometheus-k8s-nodeport       # service的名称，全局唯一
  namespace: monitoring
  labels:
    app.kubernetes.io/component: prometheus
    app.kubernetes.io/instance: k8s
    app.kubernetes.io/name: prometheus
    app.kubernetes.io/part-of: kube-prometheus
    app.kubernetes.io/version: 2.36.1
spec:
  type: NodePort            
  ports:
    - name: web
      protocol: TCP
      port: 9090
      targetPort: web
      nodePort: 31090 # 映射到物理机的端口号
    - name: reloader-web
      protocol: TCP
      port: 8080
      targetPort: reloader-web
      nodePort: 31080 # 映射到物理机的端口号
  selector:
    app.kubernetes.io/component: prometheus
    app.kubernetes.io/instance: k8s
    app.kubernetes.io/name: prometheus
    app.kubernetes.io/part-of: kube-prometheus
# grafana
http://ip:30000 账户密码：admin:admin
# alertmanager-main
http://ip:32093
# prometheus-k8s
http://ip:31090
```

\# grafana页面：

![img](https://img2022.cnblogs.com/blog/1740081/202206/1740081-20220624232742546-1622387934.png)

\# alertmanager-main界面：

![img](https://img2022.cnblogs.com/blog/1740081/202206/1740081-20220624232848224-233657087.png)
\# prometheus-k8s界面：
![img](https://img2022.cnblogs.com/blog/1740081/202206/1740081-20220624232923125-867258100.png)