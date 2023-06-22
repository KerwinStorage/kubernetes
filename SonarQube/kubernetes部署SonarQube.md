# 1.PostgreSQL

参考博客：https://hanggi.me/post/kubernetes/k8s-postgresql

## 1.1.配置PostgreSQL的ConfigMap

```bash
cat > postgres-configmap.yaml <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-config
  namespace: kube-ops
  labels:
    app: postgres
data:
  POSTGRES_DB: sonarDB
  POSTGRES_USER: postgresadmin
  POSTGRES_PASSWORD: admin12345
EOF
kubectl apply -f postgres-configmap.yaml
```

## 1.2.持久化卷Persistent StorageVolume

```bash
cat > postgres-volume.yaml <<EOF
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: postgres-pv-claim
  namespace: kube-ops
  labels:
    app: postgres
spec:
  storageClassName: nfs-storage
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
EOF
kubectl apply -f postgres-volume.yaml
```

## 1.3.PostgreSQLDeployment

```bash
cat > postgres-deployment.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres-deployment
  namespace: kube-ops
spec:
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: postgres
  replicas: 1
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:11.7
          imagePullPolicy: "IfNotPresent"
          ports:
            - containerPort: 5432
          envFrom:
            - configMapRef:
                name: postgres-config
          volumeMounts:
            - mountPath: /var/lib/postgresql/data
              name: postgredb
      volumes:
        - name: postgredb
          persistentVolumeClaim:
            claimName: postgres-pv-claim
EOF
kubectl apply -f postgres-deployment.yaml
```

## 1.4.PostgreSQLService

```bash
cat > postgres-service.yaml <<EOF
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
  namespace: kube-ops
  labels:
    app: postgres
spec:
  type: ClusterIP
  ports:
  - port: 5432
    targetPort: 5432
    protocol: TCP
  selector:
   app: postgres
EOF
kubectl apply -f postgres-service.yaml
```

# 2.部署SonarQube

## 2.1.持久化卷Persistent StorageVolume

```bash
cat > sonarqube-volume.yaml <<EOF
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: sonarqube-data
  namespace: kube-ops
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: nfs-storage
  resources:
    requests:
      storage: 10Gi
EOF
kubectl apply -f sonarqube-volume.yaml
```

## 2.2.SonarQubeDeployment

```bash
cat > sonarqube-deployment.yaml  <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sonarqube
  namespace: kube-ops
  labels:
    app: sonarqube
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sonarqube
  template:
    metadata:
      labels:
        app: sonarqube
    spec:
      initContainers:
      - name: init-sysctl
        image: busybox
        imagePullPolicy: IfNotPresent
        command: ["sysctl", "-w", "vm.max_map_count=262144"]
        securityContext:
          privileged: true
      containers:
      - name: sonarqube
        image: sonarqube:lts
        ports:
        - containerPort: 9000
        env:
        - name: SONARQUBE_JDBC_USERNAME
          value: "postgresadmin"
        - name: SONARQUBE_JDBC_PASSWORD
          value: "admin12345"        
        - name: SONARQUBE_JDBC_URL
          value: "jdbc:postgresql://postgres-service:5432/sonarDB"
        livenessProbe:
          httpGet:
            path: /sessions/new
            port: 9000
          initialDelaySeconds: 60
          periodSeconds: 30
        readinessProbe:
          httpGet:
            path: /sessions/new
            port: 9000
          initialDelaySeconds: 60
          periodSeconds: 30
          failureThreshold: 6
        resources:
          limits:
            cpu: 2000m
            memory: 2048Mi
          requests:
            cpu: 1000m
            memory: 1024Mi
        volumeMounts:
        - mountPath: /opt/sonarqube/conf
          name: data
          subPath: conf
        - mountPath: /opt/sonarqube/data
          name: data
          subPath: data
        - mountPath: /opt/sonarqube/extensions
          name: data
          subPath: extensions
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: sonarqube-data
EOF
kubectl apply -f sonarqube-deployment.yaml
```

## 2.3.SonarQubeService

```bash
cat > sonarqube-service.yaml <<EOF
apiVersion: v1
kind: Service
metadata:
  name: sonarqube
  namespace: kube-ops
  labels:
    app: sonarqube
spec:
  type: ClusterIP
  ports:
    - name: sonarqube
      port: 9000
      targetPort: 9000
      protocol: TCP
  selector:
    app: sonarqube
EOF
kubectl apply -f sonarqube-service.yaml
```

## 2.4.ingress

```bash
cat > sonarqube-ingress.yaml <<EOF
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: sonarqube
  namespace: kube-ops
  annotations:
    nginx.ingress.kubernetes.io/service-weight: ""
spec:
  ingressClassName: nginx
  rules:
    - host: sonarqube.ingress.com
      http:
        paths:
        - backend:
            service:
              name: sonarqube
              port:
                number: 9000
          path: /
          pathType: Prefix
EOF
kubectl apply -f sonarqube-ingress.yaml

# 不要忘记本地Windows进行域名解析
```

登录：[http://sonarqube.ingress.com](http://sonarqube.ingress.com/)

账户密码：`admin/admin`

![img](https://img2023.cnblogs.com/blog/1740081/202306/1740081-20230604125300363-968366019.png)