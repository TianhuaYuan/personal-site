---

title: "pre-commit 代码规范：ruff + black + isort + trailing-whitespace 自动化校验"

tags:

  - pre-commit

  - 技术学习

created: "2026-07-21"

---

# pre-commit 代码规范：ruff + black + isort + trailing-whitespace 自动化校验

> **一句话**：pre-commit是一个Git钩子管理工具，可以在代码提交前自动运行代码检查、格式化等任务。结合ruff、black、isort等工具，可以自动化代码质量控制。

## 1. pre-commit 基础
### 1.1 什么是pre-commit？

```mermaid

graph LR

    A[代码提交] --> B[pre-commit钩子]

    B --> C[运行检查]

    C --> D{通过?}

    D -->|是| E[允许提交]

    D -->|否| F[阻止提交]

    style B fill:#e1f5fe

```

**pre-commit**：Git钩子管理工具，可以在代码提交前自动运行检查任务。

### 1.2 安装和配置

```bash

# 安装

pip install pre-commit

# 初始化

pre-commit install

```

### 1.3 配置文件

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

  - repo: https://github.com/pre-commit/pre-commit-hooks

    rev: v4.4.0

    hooks:

      - id: trailing-whitespace

      - id: end-of-file-fixer

      - id: check-yaml

      - id: check-added-large-files

```

## 2. 代码格式化工具
### 2.1 Black

```yaml

# Black配置

repos:

  - repo: https://github.com/psf/black

    rev: 23.3.0

    hooks:

      - id: black

        language_version: python3.11

        args: [--line-length=88]

```

**Black特点**：

- 自动格式化代码

- 统一代码风格

- 不可配置（只有少量选项）

### 2.2 isort

```yaml

# isort配置

repos:

  - repo: https://github.com/pycqa/isort

    rev: 5.12.0

    hooks:

      - id: isort

        args: [--profile=black]

```

**isort特点**：

- 自动排序import语句

- 按类型分组

- 与Black兼容

### 2.3 Ruff

```yaml

# Ruff配置

repos:

  - repo: https://github.com/charliermarsh/ruff

    rev: v0.0.261

    hooks:

      - id: ruff

        args: [--fix]

```

**Ruff特点**：

- 极快的Python linter

- 替代flake8、pylint等

- 支持自动修复

## 3. 配置示例
### 3.1 完整配置

```yaml

# .pre-commit-config.yaml

repos:

  # 代码格式化

  - repo: https://github.com/psf/black

    rev: 23.3.0

    hooks:

      - id: black

        language_version: python3.11

  - repo: https://github.com/pycqa/isort

    rev: 5.12.0

    hooks:

      - id: isort

        args: [--profile=black]

  # 代码检查

  - repo: https://github.com/charliermarsh/ruff

    rev: v0.0.261

    hooks:

      - id: ruff

        args: [--fix]

  # 通用检查

  - repo: https://github.com/pre-commit/pre-commit-hooks

    rev: v4.4.0

    hooks:

      - id: trailing-whitespace

      - id: end-of-file-fixer

      - id: check-yaml

      - id: check-added-large-files

      - id: check-merge-conflict

  # 类型检查

  - repo: https://github.com/pre-commit/mirrors-mypy

    rev: v1.3.0

    hooks:

      - id: mypy

        additional_dependencies: [types-requests]

```

### 3.2 项目特定配置

```yaml

# .pre-commit-config.yaml

repos:

  # Python代码质量

  - repo: https://github.com/psf/black

    rev: 23.3.0

    hooks:

      - id: black

        language_version: python3.11

  - repo: https://github.com/pycqa/isort

    rev: 5.12.0

    hooks:

      - id: isort

        args: [--profile=black]

  - repo: https://github.com/charliermarsh/ruff

    rev: v0.0.261

    hooks:

      - id: ruff

        args: [--fix]

  # 前端代码质量

  - repo: https://github.com/pre-commit/mirrors-eslint

    rev: v8.40.0

    hooks:

      - id: eslint

        files: \.(js|jsx|ts|tsx)$

        types: [file]

        additional_dependencies:

          - eslint@8.40.0

          - eslint-config-standard@17.0.0

  # 文档检查

  - repo: https://github.com/igorshubovych/markdownlint-cli

    rev: v0.33.0

    hooks:

      - id: markdownlint

        args: [--fix]

```

## 4. 使用技巧
### 4.1 手动运行

```bash

# 运行所有钩子

pre-commit run --all-files

# 运行特定钩子

pre-commit run black --all-files

pre-commit run isort --all-files

```

### 4.2 跳过钩子

```bash

# 跳过钩子提交

git commit -m "feat: 添加功能" --no-verify

# 临时跳过特定钩子

SKIP=ruff git commit -m "feat: 添加功能"

```

### 4.3 更新钩子

```bash

# 更新所有钩子

pre-commit autoupdate

# 更新特定钩子

pre-commit autoupdate --repo https://github.com/psf/black

```

## 5. 实际案例
### 5.1 FastAPI项目配置

```yaml

# .pre-commit-config.yaml

repos:

  # Python代码质量

  - repo: https://github.com/psf/black

    rev: 23.3.0

    hooks:

      - id: black

        language_version: python3.11

  - repo: https://github.com/pycqa/isort

    rev: 5.12.0

    hooks:

      - id: isort

        args: [--profile=black]

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

  # 通用检查

  - repo: https://github.com/pre-commit/pre-commit-hooks

    rev: v4.4.0

    hooks:

      - id: trailing-whitespace

      - id: end-of-file-fixer

      - id: check-yaml

      - id: check-added-large-files

```

### 5.2 多语言项目配置

```yaml

# .pre-commit-config.yaml

repos:

  # Python

  - repo: https://github.com/psf/black

    rev: 23.3.0

    hooks:

      - id: black

  # JavaScript/TypeScript

  - repo: https://github.com/pre-commit/mirrors-eslint

    rev: v8.40.0

    hooks:

      - id: eslint

        files: \.(js|jsx|ts|tsx)$

  # Markdown

  - repo: https://github.com/igorshubovych/markdownlint-cli

    rev: v0.33.0

    hooks:

      - id: markdownlint

```

## 6. 常见坑点
### 1. 钩子执行缓慢

```yaml

# 解决：只检查修改的文件

repos:

  - repo: https://github.com/psf/black

    rev: 23.3.0

    hooks:

      - id: black

        files: \.py$

```

### 2. 钩子冲突

```yaml

# 解决：调整钩子执行顺序

repos:

  - repo: https://github.com/pycqa/isort

    rev: 5.12.0

    hooks:

      - id: isort

  - repo: https://github.com/psf/black

    rev: 23.3.0

    hooks:

      - id: black

```

### 3. 配置错误

```yaml

# 解决：使用pre-commit验证配置

pre-commit validate-config .pre-commit-config.yaml

```

## 核心要点

```bash

# 安装

pip install pre-commit

pre-commit install

# 运行

pre-commit run --all-files

# 更新

pre-commit autoupdate

# 配置文件

.pre-commit-config.yaml

```

## 速记卡（面试闪卡）

**Q1：一句话讲清「pre-commit 代码规范：ruff + black + isort + trailing-whitespace 自动化校验」到底是什么？**

A：pre-commit 是 Git 提交前自动跑检查/格式化的钩子管理器，串起 ruff、black、isort 等工具，把代码质量卡在提交前。

**Q2：1. pre-commit 基础 —— 怎么理解？**

A：像单位门口的安检机：你一提交代码，钩子就拦下自动体检，合格才放行进库、不合格当场退回。它本质是 Git hook 管理器，pip install + pre-commit install 一条命令接管提交闸门。

**Q3：2. 代码格式化工具 —— 怎么理解？**

A：black 是无可辩驳的格式化器（只少量选项，统一风格）；isort 按类型给 import 排序并与 black 兼容；ruff 是极快 linter，替代 flake8/pylint 还支持 --fix 自动修。三者分工：格式化、排序、挑刺。

**Q4：3/4. 配置与技巧 —— 怎么理解？**

A：在 .pre-commit-config.yaml 里列 repos 挂 hooks 即可，按需加 mypy 做类型检查、markdownlint 查文档。手动 pre-commit run --all-files 全量跑；SKIP=ruff 临时跳过；autoupdate 升版本。--no-verify 能强跳（别滥用）。

**Q5：5/6. 实际案例与坑点 —— 怎么理解？**

A：多语言项目可同时挂 Python(black/ruff)、前端(eslint)、文档(markdownlint)。坑：钩子慢就 files 限定后缀；多钩子改同一文件要调顺序；配置写错用 pre-commit validate-config 校验。

**Q6：核心速记主线有哪些？**

- pre-commit 是 Git 提交前自动跑检查/格式化的钩子管理器

- black 统一风格、isort 排 import、ruff 快查兼自动修

- 配置文件 .pre-commit-config.yaml 列 repos 挂 hooks

- 慢就限 files、冲突调顺序、配置用 validate-config 校验

**口诀**

A：pre-commit 守门

提交前先体检

black ruff isort

代码质量稳如山

## 相关链接

- 📋 目录：[[00-工程化与部署]]

- 📚 学习清单：[[技术学习路线图#工程化与部署]]

- 🔗 [[08-pytest异步测试基础设施|pytest异步测试]]

- 🔗 unittest.mock

