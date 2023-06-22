# 1.NexusVolume

```bash
cat > nexus-volume.yaml <<EOF
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nexus-data-pvc
  namespace: kube-ops
spec:
  accessModes:
    - ReadWriteMany
  # 指定 storageClass 的名字，这里使用默认的 standard
  storageClassName: nfs-storage
  resources:
    requests:
      storage: 10Gi
EOF
kubectl apply -f nexus-volume.yaml
```

# 2.NexusDeployment

镜像地址：https://registry.hub.docker.com/r/sonatype/nexus3/tags

```bash
cat > nexus-deployment.yaml <<EOF
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nexus3
  namespace: kube-ops  
  labels:
    app: nexus3
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nexus3
  template:
    metadata:
      labels:
        app: nexus3
    spec:
      containers:
      - name: nexus3
        image: sonatype/nexus3:3.54.1
        imagePullPolicy: IfNotPresent
        ports:
          - containerPort: 8081
            name: web
            protocol: TCP
        livenessProbe:
          httpGet:
            path: /
            port: 8081
          initialDelaySeconds: 100
          periodSeconds: 30
          failureThreshold: 6
        readinessProbe:
          httpGet:
            path: /
            port: 8081
          initialDelaySeconds: 100
          periodSeconds: 30
          failureThreshold: 6
        resources:
          limits:
            cpu: 4000m
            memory: 2Gi
          requests:
            cpu: 500m
            memory: 512Mi
        volumeMounts:
        - name: nexus-data
          mountPath: /nexus-data
      volumes:
        - name: nexus-data
          persistentVolumeClaim:
            claimName: nexus-data-pvc
EOF
kubectl apply -f nexus-deployment.yaml
```

# 3.NexusService

```bash
cat > nexus-service.yaml <<EOF
---
apiVersion: v1
kind: Service
metadata:
  name: nexus3
  namespace: kube-ops
  labels:
    app: nexus3
spec:
  selector:
    app: nexus3
  type: ClusterIP
  ports:
    - name: web
      protocol: TCP
      port: 8081
      targetPort: 8081
EOF
kubectl apply -f nexus-service.yaml
```

# 4.ingress

```bash
cat > nexus3-ingress.yaml <<EOF
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nexus3
  namespace: kube-ops
  annotations:
    nginx.ingress.kubernetes.io/service-weight: ""
spec:
  ingressClassName: nginx
  rules:
    - host: nexus.ingress.com
      http:
        paths:
        - backend:
            service:
              name: nexus3
              port:
                number: 8081
          path: /
          pathType: Prefix
EOF
kubectl apply -f nexus3-ingress.yaml

# 不要忘记本地Windows进行域名解析
```

默认登录 nexus 的账号和密码如下：

- 用户名：admin
- 密码：默认的初始密码在服务器的`/nexus-data/admin.password`文件中

```bash
NEXUS_PASSWORD=`kubectl exec -it -n kube-ops nexus3-76fd8646-8qmb6  -- cat /nexus-data/admin.password`
echo $NEXUS_PASSWORD
```

![img](https://img2023.cnblogs.com/blog/1740081/202306/1740081-20230604144331065-259931092.png)