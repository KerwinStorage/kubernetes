# 							Keycloak的使用

[Keycloak](https://www.keycloak.org/) 是一个由红帽（Red Hat）开发的开源身份和访问管理解决方案，用于保护应用程序和服务的身份验证和授权。它提供了强大的身份管理功能，包括用户认证、授权、单点登录（SSO）、多因素认证等，可以帮助开发人员轻松地为他们的应用程序添加安全性和访问控制。

以下是 Keycloak 的一些主要特性和功能：

1. **单点登录（SSO）**：用户只需通过一次登录即可访问多个关联的应用程序，无需重复登录。
2. **身份验证和授权**：支持多种身份验证方法，包括用户名密码、LDAP、OAuth、OpenID Connect 等，以及细粒度的授权策略。
3. **用户管理**：管理用户、组织机构和角色，支持用户自注册、密码重置等功能。
4. **安全性**：提供了一套完整的安全特性，包括加密、角色基础的访问控制、会话管理、日志审计等。
5. **多因素认证**：支持多种多因素认证方法，如短信验证码、一次性密码、硬件令牌等，提高安全性。
6. **可扩展性**：支持插件和扩展，可以根据需要定制和扩展功能。
7. **集成**：可以与各种平台和技术集成，如 Java、Node.js、Spring Boot、Kubernetes 等，为不同类型的应用程序提供身份和访问管理。
8. **开源和社区支持**：作为开源项目，拥有活跃的社区支持和贡献，持续地改进和更新。

# 1.部署 postgresql

添加 helm 仓库

```shell
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

查询版本

```shell
helm search repo  keycloak
NAME                    CHART VERSION   APP VERSION     DESCRIPTION
bitnami/keycloak        19.3.3          23.0.7          Keycloak is a high performance Java-based ident...
```

部署keycloak

```shell
# 创建 keycloak 命名空间
kubectl create ns keycloak
helm install my-keycloak bitnami/keycloak --version 19.3.3 -n keycloak \
    --set global.storageClass=openebs-lvmpv \
    --set auth.adminUser=admin \
    --set auth.adminPassword=admin

# 生成 values.yaml
helm show values bitnami/keycloak --version 19.3.3 > values.yaml

# 查看将要部署的资源清单
helm template my-keycloak bitnami/keycloak --version 19.3.3 > keycloak.yaml

# 卸载操作
helm delete -n keycloak my-keycloak
```

这里我们指定了自己的storageClass，这个可以根据自己环境去定义，关于相关配置可以查看这个文档：[artifacthub](https://artifacthub.io/packages/helm/bitnami/keycloak)。

登录界面：admin/admin

参考：

- https://todoit.tech/k8s/keycloak
- [Basic Keycloak deployment](https://www.keycloak.org/operator/basic-deployment)
- [artifacthub-keycloak](https://artifacthub.io/packages/helm/bitnami/keycloak)