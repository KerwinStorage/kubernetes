# 										MinIO 搭建

官网：https://min.io
中文官网：http://www.minio.org.cn，http://dl.minio.org.cn
GitHub：https://github.com/minio

对象存储服务OSS（Object Storage Service）是一种海量、安全、低成本、高可靠的云存储服务，**适合存放任意类型的文件**。容量和处理能力弹性扩展，多种存储类型供选择，全面优化存储成本。

# 1.安装MinIO Operator

在Kubernetes集群中安装MinIO Operator的最简单方法是使用Helm。首先，我们需要添加MinIO Operator的Helm存储库。可以使用以下命令

```shell
helm repo add minio https://operator.min.io
helm install minio-operator minio/minio-operator --namespace minio-operator --create-namespace
NAME: minio-operator
LAST DEPLOYED: Tue Feb 20 17:21:48 2024
NAMESPACE: minio-operator
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
1. Get the JWT for logging in to the console:
  kubectl get secret $(kubectl get serviceaccount console-sa --namespace minio-operator -o jsonpath="{.secrets[0].name}") --namespace minio-operator -o jsonpath="{.data.token}" | base64 --decode
2. Get the Operator Console URL by running these commands:
  kubectl --namespace minio-operator port-forward svc/console 9090:9090
  echo "Visit the Operator Console at http://127.0.0.1:9090"
```

# 2.创建MinIO实例

```shell
cat > minio.yaml <<EOF
apiVersion: minio.min.io/v2
kind: Tenant
metadata:
  name: my-minio
spec:
  # Number of MinIO instances.
  size: 4

  # MinIO instance version.
  version: "RELEASE.2022-03-30T23-11-56Z"

  # Access key and secret key to use for all MinIO instances.
  credentials:
    accessKey: "admin"
    secretKey: "minio"

  # Storage configuration for all MinIO instances.
  storage:
    # Storage class to use for MinIO instance volumes.
    storageClass: "minio-storage-class"

    # Storage size for each MinIO instance.
    size: 10Gi
EOF
```



