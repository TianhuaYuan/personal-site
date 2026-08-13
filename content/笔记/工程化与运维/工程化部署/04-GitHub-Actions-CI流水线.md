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

# 解决：在工作流中请求必要权限

permissions:

  contents: read

  packages: write

```

### 2. 密钥管理

```yaml

# 解决：使用GitHub Secrets

env:

  DATABASE_URL: ${{ secrets.DATABASE_URL }}

  API_KEY: ${{ secrets.API_KEY }}

```

### 3. 环境变量

```yaml

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

## 速记卡（面试闪卡）

**Q1：一句话讲清「GitHub Actions CI 流水线：pre-commit + pytest + 前端构建」到底是什么？**

A：GitHub Actions CI 流水线：代码一推上 GitHub，就自动跑代码检查、测试和前端构建，省去手动操作。

**Q2：工作流三阶段怎么理解 —— 怎么理解？**

A：像工厂三条流水线：先 lint 质检（ruff/black）挑毛病，再 test 单元测试（pytest）看对不对，最后 build 打包镜像（Docker）。每个 job 用 needs 串起来，上一关过才进下一关。

**Q3：pre-commit 守门口怎么理解 —— 怎么理解？**

A：像进门安检：你每次 git commit，pre-commit 先拿 black/isort/ruff/mypy 扫一遍代码，不合格根本不让提交。它在本地和 CI 双保险，把脏代码挡在仓库门外。

**Q4：缓存与密钥怎么理解 —— 怎么理解？**

A：像出门前收拾行李：用 actions/cache 把 pip 依赖和 Docker 层缓存起来，下次秒开；密钥（Secrets）像保险箱，DATABASE_URL、API_KEY 只存 GitHub 后台，绝不写进代码。

**Q5：多 job 环境隔离怎么理解 —— 怎么理解？**

A：像不同车间各干各的：每个 job 跑在全新 ubuntu 容器里，env 变量不跨 job 共享，要传值得用 outputs 或环境变量文件。测试还要起个 postgres 服务容器当"临时同事"一起干活。

**Q6：核心速记主线有哪些？**

- 三阶段流水线：lint 质检 → test 单测 → build 打包（needs 串联）

- pre-commit 守门：commit 前自动跑格式化与静态检查

- 缓存省时间：pip / Docker 层用 cache 复用，密钥走 Secrets

- 环境隔离：每 job 独立容器，env 不共享，跨 job 用 outputs

**口诀**

A：推码自动跑，三关流水线；

lint 测构串，pre-commit 守门。

缓存省时间，密钥进保险；

环境各隔离，CI 不翻车。

## 相关链接

- 📋 目录：[[00-工程化与部署]]

- 📚 学习清单：[[技术学习路线图#工程化与部署]]

- 🔗 [[01-Docker多阶段构建|Docker多阶段构建]]

- 🔗 [[05-GitHub-Actions-CD流水线|GitHub Actions CD]]

