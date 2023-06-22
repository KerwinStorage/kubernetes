## 一、Argo CD部署

> 部署请看[官网](https://argo-cd.readthedocs.io/en/release-2.5/)，ArgoCD镜像还是比较容易下载。🥳🥳🥳

- [Argo CD - Declarative GitOps CD for Kubernetes (argo-cd.readthedocs.io)](https://argo-cd.readthedocs.io/en/release-2.5/)

```bash
# 1.启动argocd
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.5.4/manifests/install.yaml

# 2.查看pod
kubectl get pod -n argocd
NAME                                               READY   STATUS    RESTARTS   AGE
argocd-application-controller-0                    1/1     Running   0          10m
argocd-applicationset-controller-fb8d96cb5-tsxfb   1/1     Running   0          10m
argocd-dex-server-5b8cd5db88-qnfrt                 1/1     Running   0          10m
argocd-notifications-controller-7fcbc4756-68g9z    1/1     Running   0          10m
argocd-redis-6d7c4576-7krrr                        1/1     Running   0          10m
argocd-repo-server-55df454d5d-w8f5g                1/1     Running   0          10m
argocd-server-5fddc7cc69-wmsrw                     1/1     Running   0          10m

# 3.查看svc
kubectl get svc -n argocd
NAME                                      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
argocd-applicationset-controller          ClusterIP   10.254.219.31    <none>        7000/TCP,8080/TCP            22m
argocd-dex-server                         ClusterIP   10.254.1.199     <none>        5556/TCP,5557/TCP,5558/TCP   22m
argocd-metrics                            ClusterIP   10.254.151.172   <none>        8082/TCP                     22m
argocd-notifications-controller-metrics   ClusterIP   10.254.127.124   <none>        9001/TCP                     22m
argocd-redis                              ClusterIP   10.254.16.0      <none>        6379/TCP                     22m
argocd-repo-server                        ClusterIP   10.254.153.90    <none>        8081/TCP,8084/TCP            22m
argocd-server                             ClusterIP   10.254.91.215    <none>        80/TCP,443/TCP               22m
argocd-server-metrics                     ClusterIP   10.254.29.253    <none>        8083/TCP                     22m
```

## 二、CLI客户端部署

- https://argo-cd.readthedocs.io/en/stable/cli_installation

```bash
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/download/v2.5.4/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64
argocd version
```

**注：网络有点慢的话建议浏览器下载。**

## 三、创建ingress

### 构建TLS站点

**创建证书：**

> 这里只是演示如何去创建secret，后面并没有用到，也可以替换。🧑‍💻🧑‍💻🧑‍💻

```bash
mkdir -p /root/argocd/tls &&  cd /root/argocd/tls
# 1.准备证书
openssl genrsa -out tls.key 2048
openssl req -new -x509 -key tls.key -out tls.crt -subj /C=CN/ST=Beijing/L=Beijing/O=DevOps/CN=argocd.server.com

# 2.生成secret
kubectl create secret tls argocd-ingress-secret -n argocd --cert=tls.crt --key=tls.key

# 3.查看secret
kubectl get secret -n argocd argocd-ingress-secret
NAME                    TYPE                DATA   AGE
argocd-ingress-secret   kubernetes.io/tls   2      38s

# 4.查看证书描述
kubectl describe secret -n argocd argocd-ingress-secret
Name:         argocd-ingress-secret
Namespace:    argocd
Labels:       <none>
Annotations:  <none>

Type:  kubernetes.io/tls

Data
====
tls.crt:  1314 bytes
tls.key:  1675 bytes
```

**Ingress 配置：**

[入口配置 - Argo CD - 用于 Kubernetes 的声明式 GitOps CD (argo-cd.readthedocs.io)](https://argo-cd.readthedocs.io/en/release-2.5/operator-manual/ingress/)

```bash
mkdir -p /root/argocd/tls &&  cd /root/argocd/tls
cat >argocd-server-ingress.yaml <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server-ingress
  namespace: argocd
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    kubernetes.io/ingress.class: nginx
    kubernetes.io/tls-acme: "true"
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
    # If you encounter a redirect loop or are getting a 307 response code
    # then you need to force the nginx ingress to connect to the backend using HTTPS.
    #
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
spec:
  rules:
  - host: argocd.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: argocd-server
            port:
              name: https
  tls:
  - hosts:
    - argocd.example.com
    secretName: argocd-secret # do not change, this is provided by Argo CD
EOF
```

> 这里我们https使用默认的secret：`argocd-secret`.

**部署：**

```bash
# 1.部署
kubectl apply -f  argocd-server-ingress.yaml
ingress.networking.k8s.io/argocd-server-ingress created

# 2.查看
kubectl get ing -n argocd
NAME                    CLASS    HOSTS               ADDRESS   PORTS     AGE
argocd-server-ingress   <none>   argocd.server.com             80, 443   35s
kubectl describe ing -n argocd argocd-server-ingress
```

**ingress的命名空间必须与它反向代理的service所处的命名空间一致。**

本地Windows：`C:\Windows\System32\drivers\etc\hosts` 进行解析：

```bash
# 45地址为mastet节点
192.168.80.45 argocd.local
```

访问：[https://argocd.local:30443](https://argocd.local:30443/)

> 本地ingress访问https已经设置成从`30443`端口进入了.

![img](https://img2023.cnblogs.com/blog/1740081/202212/1740081-20221209225100822-1314389759.png)

### 登录argocd

```bash
# 默认密码获取
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
th878zLGK9IsDTlR

# 登录
user：admin
password：th878zLGK9IsDTlR
```

![img](https://img2023.cnblogs.com/blog/1740081/202212/1740081-20221209225148875-726321616.png)

结尾，当访问不到ingress配置的地址时，确认下 dns地址是否配置`echo "nameserver 8.8.8.8 >> /etc/resolv.conf"`，还有VPN是否关掉。👈