# 			使用cert-manager自动签发证书

# 1.ingress准备

整体部署参考文档：https://cert-manager.io/docs/tutorials/acme/nginx-ingress

## 1.1.helm部署

前言：本地部署版本不是高可用版本，为单节点，目的是用于演示。

```shell
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

根据对应kubernetes支持的[support](https://github.com/kubernetes/ingress-nginx/#supported-versions-table)，本次集群版本：1.26，选定的chart版本为：4.9.1。

```shell
helm search repo ingress-nginx -l
NAME                            CHART VERSION   APP VERSION     DESCRIPTION
ingress-nginx/ingress-nginx     4.9.1           1.9.6           Ingress controller for Kubernetes using NGINX a...
ingress-nginx/ingress-nginx     4.9.0           1.9.5           Ingress controller for Kubernetes using NGINX a...
ingress-nginx/ingress-nginx     4.8.3           1.9.4           Ingress controller for Kubernetes using NGINX a...
```

生成 values.yaml：

```shell
helm show values ingress-nginx/ingress-nginx --version 4.9.1 > ingress-nginx-values.yaml
```

查看清单：

```shell
helm template ingress-nginx ingress-nginx/ingress-nginx -n ingress-nginx --create-namespace -f ./ingress-nginx-values.yaml --version 4.9.1 > ingress-nginx.yaml
```

部署ingress：

```shell
helm install ingress-nginx ingress-nginx/ingress-nginx -n ingress-nginx --create-namespace -f ./ingress-nginx-values.yaml --version 4.9.1
```

查看svc：

```shell
kubectl get svc -n ingress-nginx
NAME                                 TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.98.31.255     <pending>     80:30838/TCP,443:32535/TCP   5m15s
ingress-nginx-controller-admission   ClusterIP      10.100.236.138   <none>        443/TCP                      5m15s
```

## 1.2.测试

```shell
mkdir  -p ~/cert-manager && cd ~/cert-manager
# ingress/nginx/whoami.yaml
cat > whoami.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: whoami
  namespace: sandbox
  labels:
    app: containous
    name: whoami
spec:
  replicas: 2
  selector:
    matchLabels:
      app: containous
      task: whoami
  template:
    metadata:
      labels:
        app: containous
        task: whoami
    spec:
      containers:
        - name: containouswhoami
          image: containous/whoami
          resources:
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: whoami
  namespace: sandbox
spec:
  ports:
    - name: http
      port: 80
  selector:
    app: containous
    task: whoami
  type: ClusterIP

---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: whoami-ingress
  namespace: sandbox
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
    - host: www.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: whoami
                port:
                  number: 80
EOF
```

创建`whoami`资源：

```shell
kubectl apply -f whoami.yaml

# 查看资源创建
kubectl get  pod -l app=containous
NAME                     READY   STATUS    RESTARTS   AGE
whoami-bc9597656-2lh82   1/1     Running   0          12m
whoami-bc9597656-86ss4   1/1     Running   0          12m

# 查看Ingress
kubectl get ingress
NAME             CLASS    HOSTS                ADDRESS   PORTS   AGE
whoami-ingress   <none>   whoami.todoit.tech             80      7m59s

# 测试连通性
curl -x 10.98.31.255:80 http://www.example.com
Hostname: whoami-bc9597656-86ss4
IP: 127.0.0.1
IP: 10.244.85.240
RemoteAddr: 10.244.58.243:42416
GET / HTTP/1.1
Host: whoami.todoit.tech
User-Agent: curl/7.81.0
Accept: */*
Proxy-Connection: Keep-Alive
X-Forwarded-For: 10.244.32.128
X-Forwarded-Host: whoami.todoit.tech
X-Forwarded-Port: 80
X-Forwarded-Proto: http
X-Forwarded-Scheme: http
X-Real-Ip: 10.244.32.128
X-Request-Id: 6ee4fd71f49157f2e8ce33e89fa4ba75
X-Scheme: http
```

# 2.cert-manager简介

- 官方文档：https://cert-manager.io，中文文档：https://k8s-docs.github.io/cert-manager-docs/installation
- Github：https://github.com/cert-manager/cert-manager
- letsencrypt：https://letsencrypt.org/zh-cn/docs/challenge-types
- cloudflare：https://www.cloudflare.com/zh-cn

Cert-manager 是一个在 Kubernetes 上自动化管理和发放 TLS/SSL 证书的工具。它通过自动化证书的发行、续期和使用，来简化 Kubernetes 集群中的密钥和证书的管理工作。

## 2.1.核心概念

Cert-manager 包含几个核心概念：

- **Issuer 和 ClusterIssuer：** 这些资源表示证书颁发者。Issuer 在单个命名空间中操作，而 ClusterIssuer 在整个集群中操作。它们定义了如何向证书颁发机构（CA）请求证书。
- **Certificate：** 代表一个证书请求。这个资源包含请求证书所需的所有信息，比如域名、密钥算法、证书颁发者等。
- **Secrets：** Cert-manager 使用 Kubernetes 的 Secrets 存储生成的证书和私钥，以便可以安全地在集群内部共享和使用。
- **ACME：** 自动证书管理环境（ACME）是一种协议，它允许自动验证域名所有权并颁发证书。Cert-manager 支持 ACME，使得与 Let's Encrypt 这样的 CA 无缝集成成为可能。

## 2.2.功能

Cert-manager 的一些关键功能包括：

- **证书自动续期：** 它可以自动检测即将到期的证书并重新申请证书，确保服务的 TLS 加密不会因证书过期而中断。
- **多供应商支持：** 支持从多个证书颁发机构申请证书，包括但不限于 Let's Encrypt、HashiCorp Vault、Venafi 等。
- **多种验证方法：** 支持多种域验证方式，包括 HTTP-01、DNS-01 等，来满足不同的环境和要求。
- **Webhook 支持：** 允许通过 Webhook 扩展 cert-manager 来支持额外的验证方法和颁发者类型。

## 2.3.使用场景

Cert-manager 主要用于需要自动化证书管理的场景，比如：

- **HTTPS 服务：** 在 Kubernetes 集群中运行的任何服务，如果需要通过 HTTPS 提供服务，都需要有效的 TLS 证书。
- **内部服务通信：** 集群内部服务之间的通信，使用 TLS 可以确保数据在传输中的安全性。
- **微服务架构：** 在微服务架构中，各个服务可能需要独立的证书来保障服务间的加密通信

# 3.创建资源

支持`kubernetes`对应版本：https://cert-manager.io/docs/releases/#installing-with-regular-manifests

## 3.1.yaml文件

参考官方：https://cert-manager.io/docs/installation/kubectl

```shell
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.2/cert-manager.yaml
```

## 3.2.helm部署

### 3.2.1添加 Helm 仓库

参考官方：https://cert-manager.io/docs/installation/helm

cert-manager 通常通过 Helm 包管理器安装。首先，你需要将 Jetstack 的 Helm 仓库添加到你的 Helm 仓库列表中：

```shell
helm repo add jetstack https://charts.jetstack.io
helm repo update
```

### 3.2.2安装 cert-manager

现在可以使用 Helm 安装 cert-manager。这通常包括创建一个命名空间（例如 cert-manager），然后安装 cert-manager Helm 图表。

查看版本：

```shell
helm search repo cert-manager
NAME                                    CHART VERSION   APP VERSION     DESCRIPTION
jetstack/cert-manager                   v1.14.2         v1.14.2         A Helm chart for cert-manager
jetstack/cert-manager-approver-policy   v0.12.1         v0.12.1         approver-policy is a CertificateRequest approve...
jetstack/cert-manager-csi-driver        v0.7.1          v0.7.1          cert-manager csi-driver enables issuing secretl...
jetstack/cert-manager-csi-driver-spiffe v0.4.1          v0.4.1          csi-driver-spiffe is a Kubernetes CSI plugin wh...
jetstack/cert-manager-google-cas-issuer v0.8.0          v0.8.0          A Helm chart for jetstack/google-cas-issuer
jetstack/cert-manager-istio-csr         v0.8.1          v0.8.1          istio-csr enables the use of cert-manager for i...
jetstack/cert-manager-trust             v0.2.1          v0.2.0          DEPRECATED: The old name for trust-manager. Use...
jetstack/trust-manager                  v0.8.0          v0.8.0          trust-manager is the easiest way to manage TLS ...
```

部署cert-manager：

```shell
helm install cert-manager jetstack/cert-manager -n cert-manager --create-namespace --version v1.14.2 --set installCRDs=true --set prometheus.enabled=false

#如果想查看生成的清单，可以使用
helm template cert-manager jetstack/cert-manager -n cert-manager  > cert-manager.yaml
```

## 3.3.检查 cert-manager

安装完成后，检查 cert-manager 的 pods 是否在运行状态：

```shell
# helm部署
kubectl wait --for=condition=Ready pods --all -n cert-manager
pod/cert-manager-858f558868-2gzk2 condition met
pod/cert-manager-cainjector-748b48cf55-cnrlc condition met
pod/cert-manager-webhook-6b79759579-r94cc condition met
```

## 3.4.安装命令cmctl

### 3.4.1安装go

go官网：https://golang.google.cn/dl

```shell
sudo apt update
wget https://go.dev/dl/go1.20.1.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf go1.20.1.linux-amd64.tar.gz
ls /usr/local/go
echo "export PATH=$PATH:/usr/local/go/bin" >> ~/.bash_profile
source ~/.bash_profile
go version
```

### 3.4.2cmctl安装

安装参考：[链接](https://k8s-docs.github.io/cert-manager-docs/reference/cmctl)

```shell
curl -fsSL -o cmctl.tar.gz https://github.com/cert-manager/cert-manager/releases/latest/download/cmctl-linux-amd64.tar.gz
tar xzf cmctl.tar.gz cmctl
sudo mv cmctl /usr/local/bin
```

# 4.验证cert-manager签发证书

cert-manager 的 `Issuer` 和 `ClusterIssuer` 都是用来定义证书颁发的实体的资源对象。

- `Issuer` 是命名空间级别的资源，用于在命名空间内颁发证书。例如，当您需要使用自签名证书来保护您的服务，或者使用 Let's Encrypt 等公共证书颁发机构来颁发证书时，可以使用 `Issuer`。
- `ClusterIssuer` 是集群级别的资源，用于在整个集群内颁发证书，可跨命名空间使用。例如，当您需要使用公司的内部 CA 来颁发证书时，可以使用 `ClusterIssuer`。

参考官方：https://cert-manager.io/docs/configuration

## 4.1.cert-manager自签名证书

### 4.1.1创建自签名证书Issuer

```shell
cat > cert-resource.yaml <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: cert-manager-test
---
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: test-selfsigned
  namespace: cert-manager-test
spec:
  selfSigned: {}
EOF
```

创建资源：

```shell
kubectl apply -f cert-resource.yaml

# 查看创建Issuer
kubectl get  -n cert-manager-test Issuer
NAME              READY   AGE
test-selfsigned   True    12s
```

### 4.1.2生成证书

- 手动生成

```shell
cat > certificate-example-com.yaml <<EOF
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: selfsigned-cert
  namespace: cert-manager-test
spec:
  dnsNames:
    - example.com
  secretName: selfsigned-cert-tls
  issuerRef:
    name: test-selfsigned
EOF
```

创建资源：

```SH
kubectl apply -f certificate-example-com.yaml

#  查看创建Certificate
kubectl get  -n cert-manager-test Certificate
NAME              READY   SECRET                AGE
selfsigned-cert   True    selfsigned-cert-tls   23s
```

- 自动生成

```shell
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-myapp
  namespace: cert-manager-test
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/issuer: test-selfsigned # 这里直接指定issuer或者ClusterIssuer就会自动创建secret
    kubernetes.io/tls-acme: "true"
spec:
  ....
```

如果使用`ClusterIssuer`，ingress文件内配置应是`cert-manager.io/cluster-issuer: <name>`。

### 4.1.3创建nginx

镜像仓库：https://hub.docker.com/_/nginx/tags

```shell
cat > nginx.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deploy
  namespace: cert-manager-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
      release: canary
  template:
    metadata:
      labels:
        app: myapp
        release: canary
    spec:
      containers:
      - name: myapp
        image: nginx:stable
        ports:
        - name: http
          containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: cert-manager-test
spec:
  selector:
    app: myapp
    release: canary
  ports:
  - name: http
    port: 80
    targetPort: 80 # pod port
    
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-myapp
  namespace: cert-manager-test
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/issuer: test-selfsigned #这里选择自动创建
    kubernetes.io/tls-acme: "true"
spec:
  rules:
  - host: myapp.self.tech
    http:
      paths:
      - backend:
          service:
            name: myapp
            port:
              number: 80
        path: /
        pathType: Prefix
  tls:
  - hosts:
    - myapp.self.tech
    secretName: myapp # 后期生成的secret名称
EOF
```

创建资源：

```shell
kubectl apply -f nginx.yaml

# 查看创建ingress
kubectl get ingress -n  cert-manager-test
NAME            CLASS    HOSTS             ADDRESS        PORTS     AGE
ingress-myapp   <none>   myapp.self.tech   10.98.31.255   80, 443   36s

# 查看secret
kubectl get secrets  -n cert-manager-test
NAME                  TYPE                DATA   AGE
myapp                 kubernetes.io/tls   3      93s
selfsigned-cert-tls   kubernetes.io/tls   3      24m
```

### 4.1.4测试

- 页面测试：

```shell
kubectl get svc -n ingress-nginx
NAME                                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx-controller             NodePort    10.98.31.255     <none>        80:30838/TCP,443:32535/TCP   2d3h
ingress-nginx-controller-admission   ClusterIP   10.100.236.138   <none>        443/TCP                      2d3h
```

注：这里将svc类型改为了`NodePort`，是为了能直接演示。`kubectl edit svc -n ingress-nginx  ingress-nginx-controller`修改`type: NodePort`。

Windows解析：

```shell
<k8s节点地址>  myapp.self.tech
```

![](https://img2024.cnblogs.com/blog/1740081/202402/1740081-20240212153051900-1971707230.png)

- 本地测试

```shell
curl -kivL -H 'Host: myapp.self.tech' 'https://10.98.31.255' #ip地址为ingress的svc，这是访问nginx的入口
*   Trying 10.98.31.255:443...
* Connected to 10.98.31.255 (10.98.31.255) port 443 (#0)
* ALPN, offering h2
* ALPN, offering http/1.1
* TLSv1.0 (OUT), TLS header, Certificate Status (22):
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
* TLSv1.2 (IN), TLS header, Certificate Status (22):
* TLSv1.3 (IN), TLS handshake, Server hello (2):
* TLSv1.2 (IN), TLS header, Finished (20):
* TLSv1.2 (IN), TLS header, Supplemental data (23):
* TLSv1.3 (IN), TLS handshake, Encrypted Extensions (8):
* TLSv1.2 (IN), TLS header, Supplemental data (23):
* TLSv1.3 (IN), TLS handshake, Certificate (11):
* TLSv1.2 (IN), TLS header, Supplemental data (23):
* TLSv1.3 (IN), TLS handshake, CERT verify (15):
* TLSv1.2 (IN), TLS header, Supplemental data (23):
* TLSv1.3 (IN), TLS handshake, Finished (20):
* TLSv1.2 (OUT), TLS header, Finished (20):
* TLSv1.3 (OUT), TLS change cipher, Change cipher spec (1):
* TLSv1.2 (OUT), TLS header, Supplemental data (23):
* TLSv1.3 (OUT), TLS handshake, Finished (20):
* SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384
* ALPN, server accepted to use h2
* Server certificate:
*  subject: O=Acme Co; CN=Kubernetes Ingress Controller Fake Certificate
*  start date: Feb 11 10:12:58 2024 GMT
*  expire date: Feb 10 10:12:58 2025 GMT
*  issuer: O=Acme Co; CN=Kubernetes Ingress Controller Fake Certificate
*  SSL certificate verify result: self-signed certificate (18), continuing anyway.
* Using HTTP2, server supports multiplexing
* Connection state changed (HTTP/2 confirmed)
* Copying HTTP/2 data in stream buffer to connection buffer after upgrade: len=0
* TLSv1.2 (OUT), TLS header, Supplemental data (23):
* TLSv1.2 (OUT), TLS header, Supplemental data (23):
* TLSv1.2 (OUT), TLS header, Supplemental data (23):
* Using Stream ID: 1 (easy handle 0x5562f4f50eb0)
* TLSv1.2 (OUT), TLS header, Supplemental data (23):
> GET / HTTP/2
> Host: myapp.self.tech
> user-agent: curl/7.81.0
> accept: */*
>
* TLSv1.2 (IN), TLS header, Supplemental data (23):
* TLSv1.3 (IN), TLS handshake, Newsession Ticket (4):
* TLSv1.2 (IN), TLS header, Supplemental data (23):
* TLSv1.3 (IN), TLS handshake, Newsession Ticket (4):
* old SSL session ID is stale, removing
* TLSv1.2 (IN), TLS header, Supplemental data (23):
* Connection state changed (MAX_CONCURRENT_STREAMS == 128)!
* TLSv1.2 (OUT), TLS header, Supplemental data (23):
* TLSv1.2 (IN), TLS header, Supplemental data (23):
* TLSv1.2 (IN), TLS header, Supplemental data (23):
< HTTP/2 200
HTTP/2 200
< date: Mon, 12 Feb 2024 07:23:58 GMT
date: Mon, 12 Feb 2024 07:23:58 GMT
< content-type: text/html
content-type: text/html
< content-length: 615
content-length: 615
< last-modified: Tue, 11 Apr 2023 01:45:34 GMT
last-modified: Tue, 11 Apr 2023 01:45:34 GMT
< etag: "6434bbbe-267"
etag: "6434bbbe-267"
< accept-ranges: bytes
accept-ranges: bytes
< strict-transport-security: max-age=31536000; includeSubDomains
strict-transport-security: max-age=31536000; includeSubDomains

<
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
* TLSv1.2 (IN), TLS header, Supplemental data (23):
* Connection #0 to host 10.98.31.255 left intact
```

参考文章：

- https://todoit.tech/k8s/cert
- https://cert-manager.io/docs
- https://www.jianshu.com/p/e37e364bf177
- https://stackoverflow.com/questions/74956329/kubernetes-k3s-and-api-versions-not-included-what-to-do