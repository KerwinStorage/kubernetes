# 							cert-manager使用

# 1.简介

- 官方文档：https://cert-manager.io
- Github：https://github.com/cert-manager/cert-manager
- letsencrypt：https://letsencrypt.org/zh-cn/docs/challenge-types

Cert-manager 是一个在 Kubernetes 上自动化管理和发放 TLS/SSL 证书的工具。它通过自动化证书的发行、续期和使用，来简化 Kubernetes 集群中的密钥和证书的管理工作。

## 1.1.核心概念

Cert-manager 包含几个核心概念：

- **Issuer 和 ClusterIssuer：** 这些资源表示证书颁发者。Issuer 在单个命名空间中操作，而 ClusterIssuer 在整个集群中操作。它们定义了如何向证书颁发机构（CA）请求证书。
- **Certificate：** 代表一个证书请求。这个资源包含请求证书所需的所有信息，比如域名、密钥算法、证书颁发者等。
- **Secrets：** Cert-manager 使用 Kubernetes 的 Secrets 存储生成的证书和私钥，以便可以安全地在集群内部共享和使用。
- **ACME：** 自动证书管理环境（ACME）是一种协议，它允许自动验证域名所有权并颁发证书。Cert-manager 支持 ACME，使得与 Let's Encrypt 这样的 CA 无缝集成成为可能。

## 1.2.功能

Cert-manager 的一些关键功能包括：

- **证书自动续期：** 它可以自动检测即将到期的证书并重新申请证书，确保服务的 TLS 加密不会因证书过期而中断。
- **多供应商支持：** 支持从多个证书颁发机构申请证书，包括但不限于 Let's Encrypt、HashiCorp Vault、Venafi 等。
- **多种验证方法：** 支持多种域验证方式，包括 HTTP-01、DNS-01 等，来满足不同的环境和要求。
- **Webhook 支持：** 允许通过 Webhook 扩展 cert-manager 来支持额外的验证方法和颁发者类型。

## 1.3.使用场景

Cert-manager 主要用于需要自动化证书管理的场景，比如：

- **HTTPS 服务：** 在 Kubernetes 集群中运行的任何服务，如果需要通过 HTTPS 提供服务，都需要有效的 TLS 证书。
- **内部服务通信：** 集群内部服务之间的通信，使用 TLS 可以确保数据在传输中的安全性。
- **微服务架构：** 在微服务架构中，各个服务可能需要独立的证书来保障服务间的加密通信

# 2.创建资源

## 2.1.yaml文件

```shell
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.2/cert-manager.yaml
```

## 2.2.helm部署

### 2.2.1添加 Helm 仓库

cert-manager 通常通过 Helm 包管理器安装。首先，你需要将 Jetstack 的 Helm 仓库添加到你的 Helm 仓库列表中：

```shell
helm repo add jetstack https://charts.jetstack.io
helm repo update
```

### 2.2.2安装 cert-manager

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
helm install cert-manager jetstack/cert-manager -n cert-manager --create-namespace --version v1.14.2 --set installCRDs=true

#如果想查看生成的清单，可以使用
helm template cert-manager jetstack/cert-manager -n cert-manager  > cert-manager.yaml
```

## 2.3.检查 cert-manager

安装完成后，检查 cert-manager 的 pods 是否在运行状态：

```shell
# helm部署
kubectl wait --for=condition=Ready pods --all -n cert-manager
pod/cert-manager-858f558868-2gzk2 condition met
pod/cert-manager-cainjector-748b48cf55-cnrlc condition met
pod/cert-manager-webhook-6b79759579-r94cc condition met
```



参考文章：

- https://todoit.tech/k8s/cert
- https://cert-manager.io/docs