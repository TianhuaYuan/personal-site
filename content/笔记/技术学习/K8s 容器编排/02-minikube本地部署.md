---
title: "minikube 本地部署 + 将 Agent 服务跑在 K8s 上"
tags:
  - k8s
  - 技术学习
  - 学习笔记
created: "2026-07-21"
---

# minikube 本地部署 + 将 Agent 服务跑在 K8s 上

> **一句话**：minikube是本地K8s集群工具，可以在本地快速搭建K8s环境。将Agent服务部署到K8s上，可以实现容器化部署、弹性扩缩和高可用。

## 1. minikube 基础

### 1.1 什么是minikube？

```mermaid
graph LR
    A[minikube] --> B[本地K8s集群]
    A --> C[单节点]
    A --> D[开发测试]
    A --> E[快速启动]
    
    style A fill:#e1f5fe
```

**minikube**：本地K8s集群工具，可以在本地快速搭建K8s环境。

### 1.2 安装和启动

```bash
# 安装minikube
# macOS
brew install minikube

# Windows
choco install minikube

# Linux
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# 启动集群
minikube start

# 查看状态
minikube status
```

## 2. 部署Agent服务

### 2.1 Docker镜像构建

```dockerfile
# Dockerfile
FROM python:3.11-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install --user -r requirements.txt

COPY . .
EXPOSE 8000

CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]
```

### 2.2 构建和推送镜像

```bash
# 构建镜像
docker build -t myagent:latest .

# 使用minikube Docker环境
eval $(minikube docker-env)

# 重新构建镜像（在minikube环境中）
docker build -t myagent:latest .
```

### 2.3 Kubernetes配置

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myagent-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myagent
  template:
    metadata:
      labels:
        app: myagent
    spec:
      containers:
      - name: myagent
        image: myagent:latest
        ports:
        - containerPort: 8000
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: agent-secrets
              key: database-url

---
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: myagent-service
spec:
  selector:
    app: myagent
  ports:
  - port: 80
    targetPort: 8000
  type: LoadBalancer
```

## 3. 部署到minikube

### 3.1 创建Secret

```bash
# 创建Secret
kubectl create secret generic agent-secrets \
  --from-literal=database-url='postgresql://user:pass@localhost/db'
```

### 3.2 部署应用

```bash
# 部署Deployment
kubectl apply -f deployment.yaml

# 部署Service
kubectl apply -f service.yaml

# 查看Pod状态
kubectl get pods

# 查看Service
kubectl get services
```

### 3.3 访问应用

```bash
# 获取Service URL
minikube service myagent-service --url

# 或者使用port-forward
kubectl port-forward service/myagent-service 8080:80

# 访问应用
curl http://localhost:8080
```

## 4. 实际案例

### 4.1 AI Agent部署

```yaml
# agent-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ai-agent
spec:
  replicas: 3
  selector:
    matchLabels:
      app: ai-agent
  template:
    metadata:
      labels:
        app: ai-agent
    spec:
      containers:
      - name: agent
        image: ai-agent:latest
        ports:
        - containerPort: 8000
        env:
        - name: OPENAI_API_KEY
          valueFrom:
            secretKeyRef:
              name: agent-secrets
              key: openai-api-key
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: agent-secrets
              key: database-url
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"

---
# agent-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: ai-agent-service
spec:
  selector:
    app: ai-agent
  ports:
  - port: 80
    targetPort: 8000
  type: LoadBalancer
```

### 4.2 配置HPA自动扩缩

```yaml
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: ai-agent-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: ai-agent
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

## 5. 常用命令

### 5.1 集群管理

```bash
# 启动集群
minikube start

# 停止集群
minikube stop

# 删除集群
minikube delete

# 查看集群状态
minikube status
```

### 5.2 应用管理

```bash
# 查看Pod
kubectl get pods

# 查看Pod详情
kubectl describe pod <pod-name>

# 查看日志
kubectl logs <pod-name>

# 进入Pod
kubectl exec -it <pod-name> -- /bin/bash
```

### 5.3 服务管理

```bash
# 查看Service
kubectl get services

# 获取Service URL
minikube service <service-name> --url

# 端口转发
kubectl port-forward service/<service-name> 8080:80
```

## 6. 常见坑点

### 1. 镜像拉取失败
```bash
# 问题：镜像拉取失败
# 解决：使用minikube Docker环境
eval $(minikube docker-env)
docker build -t myagent:latest .
```

### 2. 资源不足
```bash
# 问题：Pod因资源不足无法启动
# 解决：增加minikube资源
minikube start --memory=4096 --cpus=4
```

### 3. 网络问题
```bash
# 问题：Service无法访问
# 解决：检查Pod标签和Service selector
kubectl get pods --show-labels
kubectl describe service <service-name>
```

## 核心要点

```bash
# 安装启动
minikube start

# 部署应用
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml

# 查看状态
kubectl get pods
kubectl get services

# 访问应用
minikube service <service-name> --url
```

## 相关链接

- 📋 目录：[[00-K8s]]
- 📚 学习清单：[[技术学习清单#K8s 容器编排]]
- 🔗 [[01-Pod-Service-Deployment核心概念|核心概念]]
- 🔗 [[03-GPU调度原理|GPU调度原理]]
- 项目实践：[[笔记/技术学习/项目开发笔记/ai-resume笔记/01_技术研读/01_架构概览|ai-resume: 架构概览]]
