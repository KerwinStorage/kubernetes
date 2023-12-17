# 							Komodor的使用

官网：[地址](https://komodor.com/)

# 1.helm部署

```shell
kubectl create ns komodor

helm repo add komodorio https://helm-charts.komodor.io
helm repo update
helm install komodor-agent komodorio/komodor-agent --set apiKey=71231bce-4443-4d26-9361-9ff340d020f2 --set clusterName=default  --timeout=90s -n komodor

start https://app.komodor.com/main/services
```

查看资源：

```shell
kubectl get all  -n  komodor
NAME                                 READY   STATUS    RESTARTS   AGE
pod/komodor-agent-844f459bcc-9jmfn   4/4     Running   0          7m27s
pod/komodor-agent-daemon-5d4x4       2/2     Running   0          7m27s
pod/komodor-agent-daemon-sffr8       2/2     Running   0          7m27s

NAME                             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/network-mapper-default   ClusterIP   10.107.172.42   <none>        9090/TCP   7m28s

NAME                                  DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset.apps/komodor-agent-daemon   2         2         2       2            2           <none>          7m27s

NAME                            READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/komodor-agent   1/1     1            1           7m27s

NAME                                       DESIRED   CURRENT   READY   AGE
replicaset.apps/komodor-agent-844f459bcc   1         1         1       7m27s

NAME                                                                            AGE
componentresourceconstraint.apps.kubeblocks.io/kb-resource-constraint-general   14d
```

