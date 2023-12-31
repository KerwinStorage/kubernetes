# 														kubernetes部署Gitlab

# 1. yaml文件

PostgreSQL：[Omnibus GitLab 附带的 PostgreSQL 版本 | 极狐GitLab](https://docs.gitlab.cn/jh/administration/package_information/postgresql_versions.html)

环境变量介绍：[sameersbn/docker-gitlab： Dockerized GitLab (github.com)](https://github.com/sameersbn/docker-gitlab/#available-configuration-parameters)

## 1.1.gitlab资源

镜像地址：https://hub.docker.com/r/sameersbn/gitlab/tags

```bash
cat > gitlab-deploy.yaml <<EOF
# Service
kind: Service
apiVersion: v1
metadata:
  name: gitlab
  labels:
    name: gitlab
spec:
  ports:
    - name: http
      protocol: TCP
      port: 80
    - name: ssh
      protocol: TCP
      port: 22
      targetPort: ssh
  selector:
    name: gitlab
---
# PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: gitlab-pv-claim
  labels:
    app: gitlab
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi
---
# Deployment
kind: Deployment
apiVersion: apps/v1
metadata:
  name: gitlab
  labels:
    name: gitlab
spec:
  replicas: 1
  selector:
    matchLabels:
      name: gitlab
  template:
    metadata:
      name: gitlab
      labels:
        name: gitlab
    spec:
      containers:
      - name: gitlab
        image: 'sameersbn/gitlab:16.7.0'
        ports:
        - name: ssh
          containerPort: 22
        - name: http
          containerPort: 80
        - name: https
          containerPort: 443
        env:
        - name: TZ
          value: Asia/Shanghai
        - name: GITLAB_TIMEZONE
          value: Beijing
        - name: GITLAB_SECRETS_DB_KEY_BASE
          value: long-and-random-alpha-numeric-string
        - name: GITLAB_SECRETS_SECRET_KEY_BASE
          value: long-and-random-alpha-numeric-string
        - name: GITLAB_SECRETS_OTP_KEY_BASE
          value: long-and-random-alpha-numeric-string
        - name: GITLAB_ROOT_PASSWORD
          value: admin@1234
        - name: GITLAB_ROOT_EMAIL 
          value: z0ukun@163.com     
        - name: GITLAB_HOST           
          value: 'gitlab.z0ukun.com'
        - name: GITLAB_PORT        
          value: '80'                   
        - name: GITLAB_SSH_PORT   
          value: '22'
        - name: GITLAB_NOTIFY_ON_BROKEN_BUILDS
          value: 'true'
        - name: GITLAB_NOTIFY_PUSHER
          value: 'false'
        - name: DB_TYPE             
          value: postgres
        - name: DB_HOST         
          value: gitlab-postgresql           
        - name: DB_PORT          
          value: '5432'
        - name: DB_USER        
          value: gitlab
        - name: DB_PASS         
          value: admin@1234
        - name: DB_NAME          
          value: gitlab_production
        - name: REDIS_HOST
          value: gitlab-redis
        - name: REDIS_PORT      
          value: '6379'
        livenessProbe:
          httpGet:
            path: /
            port: 80
            scheme: HTTP
          initialDelaySeconds: 300
          timeoutSeconds: 5
          periodSeconds: 10
          successThreshold: 1
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /
            port: 80
            scheme: HTTP
          initialDelaySeconds: 5
          timeoutSeconds: 30
          periodSeconds: 10
          successThreshold: 1
          failureThreshold: 3
        volumeMounts:
        - name: gitlab-persistent-storage
          mountPath: /home/git/data
        - name: localtime
          mountPath: /etc/localtime
      volumes:
      - name: gitlab-persistent-storage
        persistentVolumeClaim:
          claimName: gitlab-pv-claim
      - name: localtime
        hostPath:
          path: /etc/localtime
EOF
```

## 1.2.redis资源

镜像地址：https://hub.docker.com/_/redis/tags

```shell
cat > redis-deploy.yaml <<EOF
# Service
kind: Service
apiVersion: v1
metadata:
  name: gitlab-redis
  labels:
    name: gitlab-redis
spec:
  type: ClusterIP
  ports:
    - name: redis
      protocol: TCP
      port: 6379
      targetPort: redis
  selector:
    name: gitlab-redis
# PVC
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: gitlab-redis-pv-claim
  labels:
    app: gitlab
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
---
# Deployment
kind: Deployment
apiVersion: apps/v1
metadata:
  name: gitlab-redis
  labels:
    name: gitlab-redis
spec:
  replicas: 1
  selector:
    matchLabels:
      name: gitlab-redis
  template:
    metadata:
      name: gitlab-redis
      labels:
        name: gitlab-redis
    spec:
      containers:
      - name: gitlab-redis
        image: 'redis:6.2'
        ports:
        - name: redis
          containerPort: 6379
          protocol: TCP
        volumeMounts:
          - name: gitlab-redis-persistent-storage
            mountPath: /var/lib/redis
        livenessProbe:
          exec:
            command:
              - redis-cli
              - ping
          initialDelaySeconds: 5
          timeoutSeconds: 5
          periodSeconds: 10
          successThreshold: 1
          failureThreshold: 3
        readinessProbe:
          exec:
            command:
              - redis-cli
              - ping
          initialDelaySeconds: 5
          timeoutSeconds: 5
          periodSeconds: 10
          successThreshold: 1
          failureThreshold: 3
      # 持久化存储配置
      volumes:
      - name: gitlab-redis-persistent-storage
        persistentVolumeClaim:
          claimName: gitlab-redis-pv-claim
EOF
```

## 1.3.postgresql资源

镜像地址：https://hub.docker.com/r/sameersbn/postgresql/tags

```shell
cat > postgresql-deploy.yaml <<EOF
# Service
kind: Service
apiVersion: v1
metadata:
  name: gitlab-postgresql
  labels:
    name: gitlab-postgresql
spec:
  ports:
    - name: postgres
      protocol: TCP
      port: 5432
      targetPort: postgres
  selector:
    name: postgresql
  type: ClusterIP
---
# PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: gitlab-postgresql-pv-claim
  labels:
    app: gitlab-postgresql
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi
---
# Deployment
kind: Deployment
apiVersion: apps/v1
metadata:
  name: gitlab-postgresql
  labels:
    name: gitlab-postgresql
spec:
  replicas: 1
  selector:
    matchLabels:
      name: postgresql
  template:
    metadata:
      name: postgresql
      labels:
        name: postgresql
    spec:
      containers:
      - name: gitlab-postgresql
        image: sameersbn/postgresql:14-20230628
        ports:
        - name: postgres
          containerPort: 5432
        env:
        - name: DB_USER
          value: gitlab
        - name: DB_PASS
          value: admin@1234
        - name: DB_NAME
          value: gitlab_production
        - name: DB_EXTENSION
          value: 'pg_trgm,btree_gist'
        livenessProbe:
          exec:
            command: ["pg_isready","-h","localhost","-U","postgres"]
          initialDelaySeconds: 30
          timeoutSeconds: 5
          periodSeconds: 10
          successThreshold: 1
          failureThreshold: 3
        readinessProbe:
          exec:
            command: ["pg_isready","-h","localhost","-U","postgres"]
          initialDelaySeconds: 5
          timeoutSeconds: 1
          periodSeconds: 10
          successThreshold: 1
          failureThreshold: 3
        volumeMounts:
        - name: gitlab-postgresql-persistent-storage
          mountPath: /var/lib/postgresql
      # 持久化存储配置
      volumes:
      - name: gitlab-postgresql-persistent-storage
        persistentVolumeClaim:
          claimName: gitlab-postgresql-pv-claim
EOF
```

**注：这里使用默认的`storageclass`，自己之前部署的，本地使用nfs服务共享目录。**

```bash
kubectl get storageclasses.storage.k8s.io
NAME                    PROVISIONER                                   RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
nfs-storage (default)   k8s-sigs.io/nfs-subdir-external-provisioner   Delete          Immediate           false                  21d
```

# 2.部署

```bash
kubectl apply -f .
# 查看启动的资源
kubectl get pod,svc,pvc,deployments.apps | grep gitlab
pod/gitlab-56ff55dff8-66p8v                   1/1     Running   2 (31m ago)     43m
pod/gitlab-postgresql-85dd6d8c88-92ffr        1/1     Running   0               43m
pod/gitlab-redis-75574b7cd9-sq2r4             1/1     Running   0               43m
service/gitlab              ClusterIP   10.111.72.205   <none>        80/TCP,22/TCP   43m
service/gitlab-postgresql   ClusterIP   10.109.38.11    <none>        5432/TCP        43m
service/gitlab-redis        ClusterIP   10.104.28.18    <none>        6379/TCP        43m
persistentvolumeclaim/gitlab-postgresql-pv-claim   Bound    pvc-a2f7b41c-bae5-4ad1-b1a0-639c761b13c7   50Gi       RWO            nfs-storage    43m
persistentvolumeclaim/gitlab-pv-claim              Bound    pvc-ff3ecf68-d427-4151-8893-648ad9e7b79f   50Gi       RWO            nfs-storage    43m
persistentvolumeclaim/gitlab-redis-pv-claim        Bound    pvc-c423261c-73ce-4a4d-a0e0-13389df83f8f   5Gi        RWO            nfs-storage    43m
deployment.apps/gitlab                   1/1     1            1           43m
deployment.apps/gitlab-postgresql        1/1     1            1           43m
deployment.apps/gitlab-redis             1/1     1            1           43m
```

**注：这里gitlab启动需要的[启动标准,](https://docs.gitlab.com/ee/install/requirements.html#cpu)，一定要查看gitlab日志的启动情况判断是否正常启动。**

**参考:[kubernetes快速部署GitLab – 邹坤个人博客 (z0ukun.com)](https://blog.z0ukun.com/?p=3601)**