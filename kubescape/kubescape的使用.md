# 								kubescape的使用

# 1.使用

前言：Kubescape 是一个开源的 Kubernetes 安全平台。它包括风险分析、安全合规性和错误配置扫描。它面向 DevSecOps 从业者或平台工程师，提供易于使用的 CLI 界面、灵活的输出格式和自动扫描功能。它为 Kubernetes 用户和管理员节省了宝贵的时间、精力和资源。（摘自官网）

相关链接：[官网](https://kubescape.io)、[github地址](https://github.com/kubescape/kubescape)、[CNCF地址](https://www.cncf.io/blog/2021/09/03/kubescape-the-first-open-source-tool-for-running-nsa-and-cisa-kubernetes-hardening-tests)、[官网](https://www.armosec.io/kubescape/)

## 1.1.安装

```shell
curl -s https://raw.githubusercontent.com/kubescape/kubescape/master/install.sh | /bin/bash
```

注：这里需要科学访问，否则容易执行失败或者速度过慢。

## 1.1执行扫描

```shell
kubescape scan --enable-host-scan   --format html --output results.html  --verbose
Flag --enable-host-scan has been deprecated, To activate the host scanner capability, proceed with the installation of the kubescape operator chart found here: https://github.com/kubescape/helm-charts/tree/main/charts/kubescape-operator. The flag will be removed at 1.Dec.2023
 ℹ️   Installing host scanner
 ✅  Initialized scanner
 ✅  Loaded policies
 ✅  Loaded exceptions
 ✅  Loaded account configurations
 ✅  Accessed Kubernetes objects
 ℹ️   Requesting Host scanner data
◐ ℹ️   Host scanner version : v1.0.61
 ✅  Requested Host scanner data
Control: C-0036 100% |████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| (45/45, 16 it/s)
 ✅  Done scanning. Cluster: kubernetes-admin-kubernetes
 ✅  Done aggregating results



Kubescape security posture overview for cluster: kubernetes-admin-kubernetes

In this overview, Kubescape shows you a summary of your cluster security posture, including the number of users who can perform administrative actions. For each result greater than 0, you should evaluate its need, and then define an exception to allow it. This baseline can be used to detect drift in future.

Control plane
┌────┬─────────────────────────────────────┬────────────────────────────────────┐
│    │ Control name                        │ Docs                               │
├────┼─────────────────────────────────────┼────────────────────────────────────┤
│ ✅ │ API server insecure port is enabled │ https://hub.armosec.io/docs/c-0005 │
│ ❌ │ Anonymous access enabled            │ https://hub.armosec.io/docs/c-0262 │
│ ⚠️  │ Audit logs enabled                  │ https://hub.armosec.io/docs/c-0067 │
│ ⚠️  │ RBAC enabled                        │ https://hub.armosec.io/docs/c-0088 │
│ ⚠️  │ Secret/etcd encryption enabled      │ https://hub.armosec.io/docs/c-0066 │
└────┴─────────────────────────────────────┴────────────────────────────────────┘
* failed to get cloud provider, cluster: kubernetes-admin-kubernetes

Access control
┌─────────────────────────────────────────────────┬───────────┬────────────────────────────────────┐
│ Control name                                    │ Resources │ View details                       │
├─────────────────────────────────────────────────┼───────────┼────────────────────────────────────┤
│ Cluster-admin binding                           │     3     │ $ kubescape scan control C-0035 -v │
│ Data Destruction                                │     5     │ $ kubescape scan control C-0007 -v │
│ Exec into container                             │     4     │ $ kubescape scan control C-0002 -v │
│ List Kubernetes secrets                         │     9     │ $ kubescape scan control C-0015 -v │
│ Minimize access to create pods                  │     3     │ $ kubescape scan control C-0188 -v │
│ Minimize wildcard use in Roles and ClusterRoles │     3     │ $ kubescape scan control C-0187 -v │
│ Portforwarding privileges                       │     3     │ $ kubescape scan control C-0063 -v │
│ Validate admission controller (mutating)        │     0     │ $ kubescape scan control C-0039 -v │
│ Validate admission controller (validating)      │     1     │ $ kubescape scan control C-0036 -v │
└─────────────────────────────────────────────────┴───────────┴────────────────────────────────────┘

Secrets
┌─────────────────────────────────────────────────┬───────────┬────────────────────────────────────┐
│ Control name                                    │ Resources │ View details                       │
├─────────────────────────────────────────────────┼───────────┼────────────────────────────────────┤
│ Applications credentials in configuration files │     5     │ $ kubescape scan control C-0012 -v │
└─────────────────────────────────────────────────┴───────────┴────────────────────────────────────┘

Network
┌────────────────────────┬───────────┬────────────────────────────────────┐
│ Control name           │ Resources │ View details                       │
├────────────────────────┼───────────┼────────────────────────────────────┤
│ Missing network policy │    13     │ $ kubescape scan control C-0260 -v │
└────────────────────────┴───────────┴────────────────────────────────────┘

Workload
┌─────────────────────────┬───────────┬────────────────────────────────────┐
│ Control name            │ Resources │ View details                       │
├─────────────────────────┼───────────┼────────────────────────────────────┤
│ Host PID/IPC privileges │     0     │ $ kubescape scan control C-0038 -v │
│ HostNetwork access      │     2     │ $ kubescape scan control C-0041 -v │
│ HostPath mount          │     2     │ $ kubescape scan control C-0048 -v │
│ Non-root containers     │     7     │ $ kubescape scan control C-0013 -v │
│ Privileged container    │     2     │ $ kubescape scan control C-0057 -v │
└─────────────────────────┴───────────┴────────────────────────────────────┘


Highest-stake workloads
───────────────────────

High-stakes workloads are defined as those which Kubescape estimates would have the highest impact if they were to be exploited.

1. namespace: ingress-nginx, name: ingress-nginx-controller, kind: DaemonSet
   '$ kubescape scan workload DaemonSet/ingress-nginx-controller --namespace ingress-nginx'
2. namespace: kube-system, name: calico-node, kind: DaemonSet
   '$ kubescape scan workload DaemonSet/calico-node --namespace kube-system'
3. namespace: kube-system, name: openebs-lvm-node, kind: DaemonSet
   '$ kubescape scan workload DaemonSet/openebs-lvm-node --namespace kube-system'


Compliance Score
────────────────

The compliance score is calculated by multiplying control failures by the number of failures against supported compliance frameworks. Remediate controls, or configure your cluster baseline with exceptions, to improve this score.

* MITRE: 77.81%
* NSA: 64.96%

View a full compliance report by running '$ kubescape scan framework nsa' or '$ kubescape scan framework mitre'


What now?
─────────

* Run one of the suggested commands to learn more about a failed control failure
* Scan a workload with '$ kubescape scan workload' to see vulnerability information
* Install Kubescape in your cluster for continuous monitoring and a full vulnerability report: https://kubescape.io/docs/install-operator/

 ✅  Scan results saved. filename: results.html

#执行扫描时会在每个节点出现以下容器.
kubectl get pod -n kubescape  -owide
NAME                 READY   STATUS    RESTARTS   AGE   IP              NODE           NOMINATED NODE   READINESS GATES
host-scanner-2fqxz   1/1     Running   0          64s   172.16.58.206   k8s-node02     <none>           <none>
host-scanner-4cz4b   1/1     Running   0          64s   172.16.85.213   k8s-node01     <none>           <none>
host-scanner-kq4ln   1/1     Running   0          64s   172.16.32.132   k8s-master01   <none>           <none>
```

# 2.helm安装

官网helm：[地址](https://kubescape.io/docs/install-operator)，[地址](https://hub.armosec.io/docs/configuration-of-image-vulnerabilities#add-authentication-keys-to-kubescape-helm-installation)、[github](https://github.com/kubescape/helm-charts/blob/master/charts/kubescape-cloud-operator/README.md)、[helm官网部署方式](https://artifacthub.io/packages/helm/kubescape/kubescape-operator)

参数：[github](https://github.com/kubescape/helm-charts/blob/main/charts/kubescape-operator/README.md#chart-support)

```shell
helm repo add kubescape https://kubescape.github.io/helm-charts/
helm repo update
helm upgrade --install kubescape kubescape/kubescape-operator -n kubescape --create-namespace --set clusterName=`kubectl config current-context` 
Release "kubescape" does not exist. Installing it now.
NAME: kubescape
LAST DEPLOYED: Sun Dec 17 13:10:44 2023
NAMESPACE: kubescape
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Thank you for installing kubescape-operator version 1.16.5.
View your image vulnerabilities scan summaries:
> kubectl get vulnerabilitymanifestsummaries -A

Detailed reports are also available:
> kubectl get vulnerabilitymanifests -A
```

问题描述：这里遇到了查看`kubescape`命名空间下pod失败的日志，本次k8s集群版本为：`v1.26.5`.

```shell
kubectl get pod -n  kubescape
E1217 14:00:13.997846  413148 memcache.go:287] couldn't get resource list for spdx.softwarecomposition.kubescape.io/v1beta1: the server is currently unable to handle the request
E1217 14:00:14.008416  413148 memcache.go:121] couldn't get resource list for spdx.softwarecomposition.kubescape.io/v1beta1: the server is currently unable to handle the request
E1217 14:00:14.023351  413148 memcache.go:121] couldn't get resource list for spdx.softwarecomposition.kubescape.io/v1beta1: the server is currently unable to handle the request
E1217 14:00:14.039064  413148 memcache.go:121] couldn't get resource list for spdx.softwarecomposition.kubescape.io/v1beta1: the server is currently unable to handle the request
NAME                         READY   STATUS    RESTARTS   AGE
kubescape-66f67d9ccb-jv49l   1/1     Running   0          20m
kubevuln-76f4c4c5d8-cgzs7    1/1     Running   0          20m
node-agent-7glr4             1/1     Running   0          20m
node-agent-bhn2h             1/1     Running   0          20m
operator-dfd48955c-hwpdx     1/1     Running   0          20m
storage-94f878f9b-kg26j      1/1     Running   0          20m
```

