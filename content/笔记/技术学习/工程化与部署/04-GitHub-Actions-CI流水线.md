---
title: "GitHub Actions CI 流水线：pre-commit + pytest + 前端构建"
tags:
  - github-actions
  - 技术学习
created: "2026-07-21"
---

# GitHub Actions CI 流水线：pre-commit + pytest + 前端构建

> **一句话**：GitHub Actions是GitHub提供的CI/CD服务，允许在代码提交、PR合并等事件触发时自动执行构建、测试、部署等任务。CI流水线可以自动化代码质量检查、测试、构建等流程。

## 1. GitHub Actions 基础

### 1.1 什么是GitHub Actions？

```mermaid
graph LR
    A[代码推送] --> B[触发工作流]
    B --> C[执行任务]
    C --> D[构建]
    C --> E[测试]
    C --> F[部署]
    
    style B fill:#e1f5fe
```

**GitHub Actions**：GitHub提供的CI/CD平台，允许自动化构建、测试和部署流程。

### 1.2 基础配置

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.11'
    
    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        pip install -r requirements.txt
    
    - name: Run tests
      run: |
        pytest
```

## 2. 完整CI流水线

### 2.1 多阶段流水线

```yaml
# .github/workflows/ci.yml
name: CI Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  # 阶段1：代码质量检查
  lint:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.11'
    
    - name: Install linting tools
      run: |
        pip install ruff black isort mypy
    
    - name: Run ruff
      run: ruff check .
    
    - name: Run black
      run: black --check .
    
    - name: Run isort
      run: isort --check-only .
    
    - name: Run mypy
      run: mypy .

  # 阶段2：单元测试
  test:
    runs-on: ubuntu-latest
    needs: lint
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.11'
    
    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        pip install -r requirements.txt
        pip install pytest pytest-cov
    
    - name: Run tests with coverage
      run: |
        pytest --cov=app --cov-report=xml
    
    - name: Upload coverage to Codecov
      uses: codecov/codecov-action@v3
      with:
        file: ./coverage.xml
        flags: unittests

  # 阶段3：构建
  build:
    runs-on: ubuntu-latest
    needs: test
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v2
    
    - name: Build Docker image
      uses: docker/build-push-action@v4
      with:
        context: .
        push: false
        tags: myapp:latest
        cache-from: type=gha
        cache-to: type=gha,mode=max
```

### 2.2 前端构建

```yaml
# .github/workflows/frontend-ci.yml
name: Frontend CI

on:
  push:
    branches: [ main, develop ]
    paths:
      - 'frontend/**'
  pull_request:
    branches: [ main ]
    paths:
      - 'frontend/**'

jobs:
  frontend:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '18'
        cache: 'npm'
        cache-dependency-path: frontend/package-lock.json
    
    - name: Install dependencies
      working-directory: ./frontend
      run: npm ci
    
    - name: Run linting
      working-directory: ./frontend
      run: npm run lint
    
    - name: Run tests
      working-directory: ./frontend
      run: npm run test
    
    - name: Build
      working-directory: ./frontend
      run: npm run build
    
    - name: Upload build artifacts
      uses: actions/upload-artifact@v3
      with:
        name: frontend-build
        path: frontend/dist/
```

## 3. Pre-commit 集成

### 3.1 Pre-commit配置

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/psf/black
    rev: 23.3.0
    hooks:
      - id: black
        language_version: python3.11
  
  - repo: https://github.com/pycqa/isort
    rev: 5.12.0
    hooks:
      - id: isort
  
  - repo: https://github.com/charliermarsh/ruff
    rev: v0.0.261
    hooks:
      - id: ruff
        args: [--fix]
  
  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.3.0
    hooks:
      - id: mypy
        additional_dependencies: [types-requests]
```

### 3.2 在CI中使用pre-commit

```yaml
# .github/workflows/pre-commit.yml
name: Pre-commit

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  pre-commit:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.11'
    
    - name: Install pre-commit
      run: pip install pre-commit
    
    - name: Run pre-commit
      run: pre-commit run --all-files
```

## 4. 测试覆盖率

### 4.1 配置pytest-cov

```python
# pytest.ini
[pytest]
addopts = --cov=app --cov-report=html --cov-report=xml
testpaths = tests
```

### 4.2 在CI中生成覆盖率报告

```yaml
# .github/workflows/test-coverage.yml
name: Test Coverage

on:
  push:
    branches: [ main ]

jobs:
  coverage:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.11'
    
    - name: Install dependencies
      run: |
        pip install -r requirements.txt
        pip install pytest pytest-cov
    
    - name: Run tests with coverage
      run: |
        pytest --cov=app --cov-report=xml --cov-report=html
    
    - name: Upload coverage to Codecov
      uses: codecov/codecov-action@v3
      with:
        file: ./coverage.xml
        flags: unittests
        name: codecov-umbrella
    
    - name: Upload HTML coverage report
      uses: actions/upload-artifact@v3
      with:
        name: coverage-report
        path: htmlcov/
```

## 5. 缓存优化

### 5.1 Python依赖缓存

```yaml
# .github/workflows/ci.yml
jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.11'
    
    - name: Cache pip
      uses: actions/cache@v3
      with:
        path: ~/.cache/pip
        key: ${{ runner.os }}-pip-${{ hashFiles('requirements.txt') }}
        restore-keys: |
          ${{ runner.os }}-pip-
    
    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        pip install -r requirements.txt
```

### 5.2 Docker层缓存

```yaml
# .github/workflows/docker-build.yml
jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v2
    
    - name: Build and push
      uses: docker/build-push-action@v4
      with:
        context: .
        push: true
        tags: myapp:latest
        cache-from: type=gha
        cache-to: type=gha,mode=max
```

## 6. 实际案例

### 6.1 FastAPI项目CI

```yaml
# .github/workflows/fastapi-ci.yml
name: FastAPI CI

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.11'
    - name: Install linting tools
      run: pip install ruff black isort mypy
    - name: Run linting
      run: |
        ruff check .
        black --check .
        isort --check-only .
        mypy .

  test:
    runs-on: ubuntu-latest
    needs: lint
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: test_db
        ports:
          - 5432:5432
        options: --health-cmd pg_isready --health-interval 10s --health-timeout 5s --health-retries 5
    
    steps:
    - uses: actions/checkout@v3
    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.11'
    - name: Install dependencies
      run: |
        pip install -r requirements.txt
        pip install pytest pytest-cov httpx
    - name: Run tests
      env:
        DATABASE_URL: postgresql://test:test@localhost:5432/test_db
      run: pytest --cov=app

  build:
    runs-on: ubuntu-latest
    needs: test
    steps:
    - uses: actions/checkout@v3
    - name: Build Docker image
      run: docker build -t myapp:latest .
```

## 7. 常见坑点

### 1. 权限问题
```yaml
# 问题：GitHub Actions权限不足
# 解决：在工作流中请求必要权限
permissions:
  contents: read
  packages: write
```

### 2. 密钥管理
```yaml
# 问题：密钥泄露风险
# 解决：使用GitHub Secrets
env:
  DATABASE_URL: ${{ secrets.DATABASE_URL }}
  API_KEY: ${{ secrets.API_KEY }}
```

### 3. 环境变量
```yaml
# 问题：环境变量在不同job中不共享
# 解决：使用环境变量文件或outputs
jobs:
  job1:
    runs-on: ubuntu-latest
    outputs:
      output1: ${{ steps.step1.outputs.value }}
    steps:
    - id: step1
      run: echo "value=hello" >> $GITHUB_OUTPUT
  
  job2:
    runs-on: ubuntu-latest
    needs: job1
    steps:
    - run: echo "${{ needs.job1.outputs.output1 }}"
```

## 核心要点

```yaml
# 基础工作流
name: CI
on: [push, pull_request]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - name: Run command
      run: echo "Hello"

# 缓存依赖
- uses: actions/cache@v3
  with:
    path: ~/.cache/pip
    key: ${{ runner.os }}-pip-${{ hashFiles('requirements.txt') }}

# 使用密钥
env:
  API_KEY: ${{ secrets.API_KEY }}
```

## 相关链接

- 📋 目录：[[00-工程化与部署]]
- 📚 学习清单：[[技术学习清单#工程化与部署]]
- 🔗 [[01-Docker多阶段构建|Docker多阶段构建]]
- 🔗 [[05-GitHub-Actions-CD流水线|GitHub Actions CD]]
- 项目实践：[[笔记/技术学习/项目开发笔记/ai-resume笔记/01_技术研读/01_架构概览|ai-resume: 架构概览]]
- 项目实践：[[笔记/技术学习/项目开发笔记/cr-agent笔记/01-技术研读/08-GitHub集成|cr-agent: GitHub集成]]
