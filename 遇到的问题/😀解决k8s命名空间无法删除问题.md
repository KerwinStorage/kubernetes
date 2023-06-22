## 一、执行内容

```bash
kubectl get ns
rook-ceph    Terminating   18h
# 开启另外一个窗口窗口
kubectl proxy --port=8081

kubectl get namespace rook-ceph -o json |jq '.spec = {"finalizers":[]}' >temp01.json
curl -k -H "Content-Type: application/json" -X PUT --data-binary @temp01.json 127.0.0.1:8081/api/v1/namespaces/rook-ceph/finalize #rook-ceph是我们的命名空间
```

文件内容参考模板（将内容改成下面这个样子）：

```bash
{
    "apiVersion": "v1",
    "kind": "Namespace",
    "metadata": {
        "annotations": {
            "kubectl.kubernetes.io/last-applied-configuration": "{\"apiVersion\":\"v1\",\"kind\":\"Namespace\",\"metadata\":{\"annotations\":{},\"name\":\"monitoring\"}}\n"
        },
        "creationTimestamp": "2020-05-26T06:29:13Z",
        "deletionTimestamp": "2020-05-26T07:16:09Z",
        "name": "monitoring",
        "resourceVersion": "6710357",
        "selfLink": "/api/v1/namespaces/monitoring",
        "uid": "db09b70a-6198-443b-8ad7-5287b2483a08"
    },
    "spec": {
    },
    "status": {
        "phase": "Terminating"
    }
}
```

## 二、使用命令

```bash
# 1.清空 finalizers 字段的值
kubectl patch namespace yourname -p '{"metadata":{"finalizers":[]}}' --type='merge' -n yourname
# 2.执行删除命令或者查看
kubectl delete namespace yourname --grace-period=0 --force
```

 