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

# 解决：使用minikube Docker环境

eval $(minikube docker-env)

docker build -t myagent:latest .

```

### 2. 资源不足

```bash

# 解决：增加minikube资源

minikube start --memory=4096 --cpus=4

```

### 3. 网络问题

```bash

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

## 速记卡（面试闪卡）

**Q1：一句话讲清「minikube 本地部署 + 将 Agent 服务跑在 K8s 上」到底是什么？**

A：minikube 是在本机一键拉起单节点 K8s 集群的工具，用来把 Agent 服务容器化部署、跑上 K8s 做弹性扩缩。

**Q2：一、minikube 是什么（local single-node K8s） —— 怎么理解？**

A：minikube 像在笔记本里搭一个迷你 Kubernetes——一条 minikube start 就起好单节点集群，专供本地开发测试，不用买云服务器。brew/choco/curl 装好，start 完用 status 看状态。本质是「把 K8s 装进一台机器」的沙盒。

**Q3：二、Deployment + Service 跑服务（deploy & expose） —— 怎么理解？**

A：先 docker build 镜像；用 minikube docker-env 让镜像直接进集群内部，免推远程仓库。写 deployment.yaml 定副本数/资源/secret，service.yaml 用 LoadBalancer 暴露端口。kubectl apply 后 get pods 看状态，port-forward 或 minikube service 访问。

**Q4：三、HPA 自动扩缩与常见坑（autoscaling & pitfalls） —— 怎么理解？**

A：对 Agent 服务配 HPA：按 CPU/内存利用率在 2~10 副本间自动扩缩。常见坑：镜像拉不到→用 minikube docker-env 本地构建；资源不足→start 加 --memory=4096；Service 不通→查 Pod 标签和 Service selector 是否对上。

**Q5：四、一句话核心流程（start → apply → access） —— 怎么理解？**

A：记住主干就够：minikube start 起集群 → kubectl apply -f deployment.yaml/service.yaml 部署 → kubectl get pods/services 看状态 → minikube service <名> --url 拿地址访问。Secret 用 kubectl create secret generic 注入数据库/API Key，别写死在配置里。

**Q6：核心速记主线有哪些？**

- 概念：minikube = 本机单节点 K8s 沙盒

- 构建：docker build + minikube docker-env 进集群

- 部署：Deployment 定副本/资源，Service 暴露端口

- 伸缩：HPA 按 CPU/内存自动扩缩容

- 坑点：镜像拉取、资源不足、Service selector 标签要对

**口诀**

A：minikube 本地起，

镜像入 env 里；

部署 apply 皆已毕，

服务跑上 K8s 体。

## 相关链接

- 📋 目录：[[00-K8s]]

- 📚 学习清单：[[技术学习路线图#K8s 容器编排]]

- 🔗 [[01-Pod-Service-Deployment核心概念|核心概念]]

- 🔗 [[03-GPU调度原理|GPU调度原理]]

