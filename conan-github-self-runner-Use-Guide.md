# fcpp 项目本地 Conan Server + GitHub Self-Hosted Runner 使用指南

**项目**: fcpp (Modern C/C++ Library Build System Template)
**版本**: 1.0.0
**日期**: 2026-01-16
**适用环境**: 本地 Conan Server (端口 9300) + GitHub Self-Hosted Runner

---

## 目录

1. [环境准备](#1-环境准备)
2. [项目配置](#2-项目配置)
3. [触发 CI 测试](#3-触发-ci-测试)
4. [CI 工作流程](#4-ci-工作流程)
5. [验证结果](#5-验证结果)
6. [故障排除](#6-故障排除)
7. [进阶使用](#7-进阶使用)

---

## 1. 环境准备

### 1.1 前置条件

确保以下服务已正常运行：

#### ✅ Conan Server

```bash
# 检查 Conan Server 状态
bash /home/lgq/conan/scripts/status_server.sh

# 应该看到：
# 状态: 运行中
# PID: XXXXX
# 端口: 9300 [监听中]
```

**预期输出示例**：
```
=== Conan Server 状态 ===

状态: 运行中
PID: 12345

  PID  PPID CMD                         %MEM %CPU     ELAPSED
 12345     1 conan_server --config ...  0.5  0.1    00:10:23

端口监听:
  Netid  State    Recv-Q   Send-Q     Local Address:Port    Peer Address:Port
  tcp    LISTEN   0        128          0.0.0.0:9300         0.0.0.0:*

最后10行日志:
  2026-01-16 10:30:45,123 - INFO - GET /v1/ping
  ...
```

#### ✅ GitHub Self-Hosted Runner

```bash
# 检查 Runner 状态
bash /home/lgq/conan_github_self_runner/scripts/check_all.sh

# 或直接检查服务
sudo systemctl status actions.runner.*
```

**预期输出示例**：
```
========================================
2. GitHub Runner
========================================
状态: 运行中
服务: actions.runner.用户名-仓库名.ai-server-runner.service
PID: 54321
内存: 150 MB
```

#### ✅ 验证网络连通性

```bash
# 测试 Conan Server
curl http://localhost:9300/v1/ping
# 预期: {"timestamp": "2026-01-16T10:30:45.123Z"}

# 测试 GitHub 连通性
ping -c 3 github.com
```

### 1.2 确认 Runner 标签

访问 GitHub 仓库设置页面，查看 Runner 的标签：
```
https://github.com/你的用户名/fcppFork/settings/actions/runners
```

常见的标签包括：
- `self-hosted`
- `linux`
- `conan`
- `build`
- 或自定义标签（如 `ai-server-runner`）

**重要**：记下你的 Runner 标签，后面需要修改 CI 配置文件使用。

---

## 2. 项目配置

### 2.1 克隆项目

```bash
# 确认项目路径
cd /home/lgq/conan_dev_github_self_runner/fcppFork

# 检查项目结构
ls -la
```

**预期文件结构**：
```
fcppFork/
├── .github/
│   └── workflows/
│       ├── ci-build-test.yml              # 原有 CI 配置
│       ├── ci-local-conan-server-test.yml # 🆕 本地测试专用配置
│       └── ...
├── conanfile.py
├── conandata.yml
├── metadata.json
├── CMakeLists.txt
├── include/
├── src/
└── test_package/
```

### 2.2 配置 Runner 标签

编辑 CI 配置文件，使用正确的 Runner 标签：

```bash
nano .github/workflows/ci-local-conan-server-test.yml
```

找到第 20 行左右的 `runs-on` 配置：

```yaml
# 选项 1: 使用标签组合（推荐）
runs-on: [self-hosted, linux, conan]

# 选项 2: 使用特定 Runner 名称
runs-on: ai-server-runner

# 选项 3: 仅使用 self-hosted（如果有多个 self-hosted runner 可能会冲突）
runs-on: self-hosted
```

**根据你的 Runner 配置选择合适的选项。**

### 2.3 配置 Conan Server 访问

在同一配置文件中，检查环境变量（第 13-15 行）：

```yaml
env:
  CONAN_HOME: ~/.conan2
  CONAN_SERVER_URL: http://localhost:9300
  CONAN_SERVER_USER: demo
```

如果你的 Conan Server 使用不同的：
- **端口**：修改 `CONAN_SERVER_URL`
- **用户名**：修改 `CONAN_SERVER_USER`（同时需要修改第 76 行的登录命令）

**示例**：如果用户名是 `admin`：
```yaml
env:
  CONAN_SERVER_USER: admin
```

并在第 76 行：
```bash
echo "${{ env.CONAN_SERVER_USER }}" | conan remote login local-server --password "${{ env.CONAN_SERVER_USER }}"
```

---

## 3. 触发 CI 测试

### 3.1 触发机制

CI 通过 **commit message** 中的特定 emoji 触发：

```
:building_construction: 本地conan—sever与github-self-runner测试
```

**完整提交命令**：

```bash
cd /home/lgq/conan_dev_github_self_runner/fcppFork

# 确保在 main 分支
git checkout main

# 查看当前状态
git status

# 添加所有更改（如果有）
git add .

# 提交并触发 CI
git commit -m ":building_construction: 本地conan—sever与github-self-runner测试"

# 推送到 GitHub
git push origin main
```

### 3.2 触发条件说明

CI 配置文件中的触发条件（第 17-19 行）：

```yaml
if: >-
  github.event_name == 'push' &&
  contains(join(github.event.commits.*.message, '\n'), ':building_construction:')
```

这意味着：
- ✅ **push 事件**触发（不是 pull_request）
- ✅ commit message **必须包含** `:building_construction:` emoji
- ✅ commit 到 **main 分支**

**其他常见的触发 emoji**：
- `:building_construction:` - 本地 Conan Server 测试
- `:beer:` - 完整自动化测试（full-test-automation.yml）
- 其他 emoji 不会触发本地测试 CI

---

## 4. CI 工作流程

### 4.1 工作流程概览

```
┌─────────────────────────────────────────────────────────────┐
│  1. Git Push with :building_construction: emoji             │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│  2. GitHub Actions 接收事件                                  │
│     - 检查 commit message                                   │
│     - 匹配触发条件                                          │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│  3. 分配到 Self-Hosted Runner                               │
│     - 使用标签: [self-hosted, linux, conan]                │
│     - 在本地机器执行                                         │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│  4. 环境准备                                                 │
│     - 设置 Python 3.12                                       │
│     - 安装 Conan                                            │
│     - 配置本地 Conan Server                                  │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│  5. 构建和测试 (Debug & Release)                            │
│     - 安装系统依赖                                           │
│     - 使用 Conan create 构建包                              │
│     - 上传到本地 Conan Server                               │
│     - 验证上传                                               │
│     - 下载测试                                               │
│     - 运行 test_package                                     │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│  6. 清理和报告                                               │
│     - 清理本地缓存                                          │
│     - 生成构建摘要                                          │
│     - 完成 CI 任务                                          │
└─────────────────────────────────────────────────────────────┘
```

### 4.2 详细步骤说明

#### 步骤 1: 代码检出
```yaml
- name: Checkout code
  uses: actions/checkout@v4
```
- 从 GitHub 拉取代码到 Runner 工作目录
- 工作目录：`/opt/actions-runner/_work/fcppFork/fcppFork`

#### 步骤 2: 环境设置
```yaml
- name: Set up Python
  uses: actions/setup-python@v5
  with:
    python-version: '3.12'

- name: Install Conan
  run: pip install conan
```
- 安装 Python 3.12
- 安装最新版 Conan 2.x

#### 步骤 3: 配置 Conan
```yaml
- name: Configure Conan for local server
  run: |
    conan profile detect --force
    conan remote add local-server http://localhost:9300 --insert 0
    echo "demo" | conan remote login local-server --password demo
    conan remote list
```
- 检测编译器配置
- 添加本地服务器到 remotes
- 登录本地服务器（用户名: demo, 密码: demo）
- 验证远程仓库列表

#### 步骤 4: 构建包
```yaml
- name: Build with Conan
  run: |
    conan create . \
      -pr:b=default \
      -pr:h=default \
      -s build_type=Debug \
      --build=missing
```
- 构建主机配置（`-pr:h=default`）
- 构建构建配置（`-pr:b=default`）
- 设置构建类型（Debug 或 Release）
- 从源构建缺失的依赖（`--build=missing`）

#### 步骤 5: 上传到本地服务器
```yaml
- name: Upload to local Conan Server
  run: |
    conan upload "fcpp/1.0.0@" --remote=local-server --confirm --all
```
- 上传所有包文件（二进制、头文件、库文件）
- `--all` 包含所有包变体

#### 步骤 6: 验证和测试
```yaml
- name: Verify upload
  run: conan search "fcpp/1.0.0" --remote=local-server

- name: Test package download
  run: conan download "fcpp/1.0.0" --remote=local-server

- name: Run test_package
  run: |
    cd test_package
    conan create . -pr:h=default -s build_type=Debug --build=missing
```
- 搜索已上传的包
- 测试下载功能
- 运行 test_package 验证包可用性

#### 步骤 7: 清理
```yaml
- name: Clean up local packages
  if: always()
  run: |
    conan remove "fcpp/*" --confirm || true
    conan remove "test_*" --confirm || true
```
- 清理本地缓存
- 释放磁盘空间
- `if: always()` 确保即使构建失败也会清理

---

## 5. 验证结果

### 5.1 查看 GitHub Actions 状态

#### Web 界面查看

1. **访问 Actions 页面**：
   ```
   https://github.com/你的用户名/fcppFork/actions
   ```

2. **查看工作流运行**：
   - 应该看到 "Local Conan Server Test" 工作流
   - 状态：✅ 成功、⚠️ 黄色（进行中）、❌ 失败

3. **点击查看详情**：
   - 展开 Debug 和 Release 两个任务
   - 查看每个步骤的日志

#### 命令行查看（使用 gh CLI）

```bash
# 安装 gh CLI（如果未安装）
sudo apt install gh

# 登录 GitHub
gh auth login

# 查看最近的工作流运行
gh run list --workflow=ci-local-conan-server-test.yml

# 查看特定运行的日志
gh run view <run-id>
gh run view <run-id> --log
```

### 5.2 验证 Conan Server 中的包

#### 方法 1: Web 界面

如果 Conan Server 配置了 Web UI（如 Artifactory），访问：
```
http://localhost:9300/ui/search
```

搜索 `fcpp/1.0.0`，应该能看到上传的包。

#### 方法 2: 命令行

```bash
# 搜索本地服务器上的包
~/.local/bin/conan search "fcpp/*" --remote=local-server

# 预期输出：
# Package ID: ...
# fcpp/1.0.0

# 查看包详细信息
~/.local/bin/conan get "fcpp/1.0.0@" --remote=local-server
```

#### 方法 3: 检查数据目录

```bash
# 检查 Conan Server 数据目录
ls -la /home/lgq/conan/data/

# 应该看到 fcpp 目录
ls -la /home/lgq/conan/data/fcpp/
```

**预期输出**：
```
/home/lgq/conan/data/
└── fcpp/
    └── 1.0.0/
        ├── _/_/
        │   ├── package/
        │   │   ├── include/    # 头文件
        │   │   ├── lib/        # 库文件
        │   │   └── ...
        │   └── package_metadata.json
        └── ...
```

### 5.3 测试下载和使用包

创建一个测试项目验证包可用：

```bash
# 创建测试目录
mkdir -p /tmp/test_fcpp
cd /tmp/test_fcpp

# 创建测试用的 conanfile.py
cat > conanfile.py << 'EOF'
from conan import ConanFile

class TestFcpp(ConanFile):
    settings = "os", "compiler", "build_type", "arch"
    requires = "fcpp/1.0.0"

    def build(self):
        self.output.info("Using fcpp library")

    def test(self):
        self.output.info("Test passed!")
EOF

# 创建使用示例
cat > main.cpp << 'EOF'
#include "cpptest.hpp"
#include <iostream>

int main() {
    std::vector<int> vec = {1, 2, 3, 4, 5};
    int sum = vector_sum(vec);
    std::cout << "Sum: " << sum << std::endl;
    return 0;
}
EOF

# 配置 Conan
~/.local/bin/conan remote add local http://localhost:9300/ --insert 0
~/.local/bin/conan remote login local demo

# 创建并测试
~/.local/bin/conan create . --build=missing
```

如果成功，说明包已正确上传到本地服务器并可以正常使用。

---

## 6. 故障排除

### 6.1 CI 任务未触发

**症状**：推送代码后，GitHub Actions 没有运行

**检查清单**：

1. **确认 commit message 包含 emoji**：
   ```bash
   git log --oneline -1
   # 应该看到: :building_construction: 本地conan—sever与github-self-runner测试
   ```

2. **确认推送到 main 分支**：
   ```bash
   git branch
   # 应该显示: * main
   ```

3. **确认 GitHub 仓库中的 CI 配置存在**：
   - 访问：`https://github.com/你的用户名/fcppFork/tree/main/.github/workflows`
   - 应该看到 `ci-local-conan-server-test.yml`

4. **查看 Actions 页面**：
   - 访问：`https://github.com/你的用户名/fcppFork/actions`
   - 检查是否有其他工作流运行

**解决方案**：

- 重新提交正确的 commit message
- 确保文件已推送到 GitHub
- 检查仓库设置中是否启用了 Actions

### 6.2 Runner 无法分配任务

**症状**：Actions 显示 "Waiting for a runner"

**检查清单**：

1. **检查 Runner 在线状态**：
   ```bash
   sudo systemctl status actions.runner.*
   # 应该显示: active (running)
   ```

2. **检查 Runner 标签**：
   ```bash
   cat /opt/actions-runner/.runner | grep -A 5 "labels"
   ```

3. **检查 CI 配置中的 `runs-on`**：
   - 确保标签匹配
   - 例如：如果 Runner 有 `[self-hosted, linux, conan]`
   - CI 配置应该是：`runs-on: [self-hosted, linux, conan]`

**解决方案**：

- 修改 CI 配置文件使用正确的标签
- 或在 Runner 配置中添加相应标签

### 6.3 Conan Server 连接失败

**症状**：CI 日志显示 "Cannot connect to local server"

**检查清单**：

1. **在 Runner 上测试连接**：
   ```bash
   curl http://localhost:9300/v1/ping
   ```

2. **检查 Conan Server 状态**：
   ```bash
   bash /home/lgq/conan/scripts/status_server.sh
   ```

3. **检查端口监听**：
   ```bash
   ss -tlnp | grep 9300
   ```

**解决方案**：

- 启动 Conan Server：`bash /home/lgq/conan/scripts/start_server.sh`
- 检查防火墙设置
- 确认 CI 配置中的 `CONAN_SERVER_URL` 正确

### 6.4 登录失败

**症状**：CI 日志显示 "Authentication failed"

**检查清单**：

1. **确认用户名和密码**：
   ```bash
   cat /home/lgq/conan/server/server.conf | grep -A 10 "\[users\]"
   ```

2. **手动测试登录**：
   ```bash
   ~/.local/bin/conan remote add local http://localhost:9300/
   ~/.local/bin/conan remote login local demo
   # 输入密码: demo
   ```

**解决方案**：

- 修改 CI 配置中的 `CONAN_SERVER_USER` 和密码
- 或在 Conan Server 中创建新用户

### 6.5 构建失败

**症状**：`conan create` 步骤失败

**常见原因和解决方案**：

#### 原因 1: 依赖缺失

```bash
# 错误信息: ERROR: Missing binary
# 解决方案: 添加 --build=missing
conan create . --build=missing
```

#### 原因 2: 编译器配置错误

```bash
# 错误信息: Invalid setting
# 解决方案: 检查 profile
conan profile show default
```

#### 原因 3: 系统依赖缺失

```bash
# 错误信息: libnsl not found
# 解决方案: 安装依赖
sudo apt install -y libnsl-dev build-essential
```

#### 原因 4: 磁盘空间不足

```bash
# 检查磁盘空间
df -h

# 清理 Conan 缓存
~/.local/bin/conan remove "*" --confirm
```

### 6.6 查看详细日志

#### GitHub Actions 日志

1. 访问 Actions 页面
2. 点击失败的工作流
3. 展开失败的步骤
4. 点击 "View raw logs" 查看完整日志

#### Runner 本地日志

```bash
# GitHub Runner 日志
tail -f /opt/actions-runner/_diag/Runner_*.log

# Conan Server 日志
tail -f /home/lgq/conan/server/server.log

# Systemd 日志
sudo journalctl -u actions.runner.* -f
```

---

## 7. 进阶使用

### 7.1 自定义触发条件

修改 CI 配置文件中的触发条件：

```yaml
# 当前: 只在 push 且包含特定 emoji 时触发
if: >-
  github.event_name == 'push' &&
  contains(join(github.event.commits.*.message, '\n'), ':building_construction:')

# 选项 1: 移除触发条件（每次 push 都触发）
if: github.event_name == 'push'

# 选项 2: 添加更多触发 emoji
if: >-
  github.event_name == 'push' &&
  (
    contains(join(github.event.commits.*.message, '\n'), ':building_construction:') ||
    contains(join(github.event.commits.*.message, '\n'), ':test:')
  )

# 选项 3: 在 PR 时也触发
if: >-
  (github.event_name == 'push' || github.event_name == 'pull_request') &&
  contains(join(github.event.commits.*.message, '\n'), ':building_construction:')
```

### 7.2 添加更多构建配置

修改 matrix 矩阵，添加更多构建变体：

```yaml
strategy:
  matrix:
    build_type: [Debug, Release]
    compiler: [gcc, clang]
    cppstd: [17, 20]
  exclude:
    # 排除某些组合
    - compiler: clang
      cppstd: 20
```

### 7.3 启用代码覆盖率

修改 `metadata.json`：

```json
{
  "activate_code_coverage": true,
  ...
}
```

CI 会自动生成代码覆盖率报告并上传为 artifact。

### 7.4 配置缓存

加速构建，添加缓存步骤：

```yaml
- name: Cache Conan packages
  uses: actions/cache@v4
  with:
    path: ${{ env.CONAN_HOME }}
    key: conan-${{ runner.os }}-${{ hashFiles('conanfile.py') }}
    restore-keys: |
      conan-${{ runner.os }}-
```

### 7.5 添加通知

#### Email 通知

```yaml
- name: Send email notification
  if: failure()
  uses: dawidd6/action-send-mail@v3
  with:
    server_address: smtp.example.com
    server_port: 465
    username: ${{ secrets.EMAIL_USERNAME }}
    password: ${{ secrets.EMAIL_PASSWORD }}
    subject: "CI Failed for fcpp"
    body: "Build failed. Check logs."
    to: admin@example.com
    from: ci@example.com
```

#### Slack 通知

```yaml
- name: Slack notification
  if: failure()
  uses: slackapi/slack-github-action@v1
  with:
    payload: |
      {
        "text": "CI Failed for fcpp"
      }
  env:
    SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

### 7.6 自动创建 Release

构建成功后自动创建 GitHub Release：

```yaml
- name: Create Release
  if: success() && matrix.build_type == 'Release'
  uses: softprops/action-gh-release@v1
  with:
    tag_name: v${{ steps.read_meta.outputs.pkg_version }}
    name: Release v${{ steps.read_meta.outputs.pkg_version }}
    draft: false
    prerelease: false
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### 7.7 多环境部署

添加到不同的 Conan Server（开发、测试、生产）：

```yaml
- name: Upload to production server
  if: matrix.build_type == 'Release' && github.ref == 'refs/heads/main'
  run: |
    conan remote add prod http://prod-server:9300/
    conan upload "fcpp/*" --remote=prod --confirm
```

---

## 附录

### A. 快速命令参考

#### 本地服务管理

```bash
# Conan Server
bash /home/lgq/conan/scripts/start_server.sh       # 启动
bash /home/lgq/conan/scripts/stop_server.sh        # 停止
bash /home/lgq/conan/scripts/restart_server.sh     # 重启
bash /home/lgq/conan/scripts/status_server.sh      # 状态

# GitHub Runner
cd /opt/actions-runner
sudo ./svc.sh start                                # 启动
sudo ./svc.sh stop                                 # 停止
sudo ./svc.sh restart                              # 重启
sudo ./svc.sh status                               # 状态

# 系统检查
bash /home/lgq/conan_github_self_runner/scripts/check_all.sh
```

#### Git 操作

```bash
# 触发 CI
git commit -m ":building_construction: 本地conan—sever与github-self-runner测试"
git push origin main

# 查看最近的 commit
git log --oneline -5

# 查看特定 commit 的触发信息
git show <commit-hash>
```

#### Conan 操作

```bash
# 配置远程仓库
conan remote add local http://localhost:9300/ --insert 0
conan remote login local demo

# 搜索包
conan search "*" --remote=local
conan search "fcpp/*" --remote=local

# 下载包
conan download "fcpp/1.0.0" --remote=local

# 清理缓存
conan remove "fcpp/*" --confirm
conan remove "*" --confirm
```

### B. CI 配置文件完整示例

```yaml
name: Local Conan Server Test

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

env:
  CONAN_HOME: ~/.conan2
  CONAN_SERVER_URL: http://localhost:9300
  CONAN_SERVER_USER: demo

jobs:
  local-conan-test:
    if: >-
      github.event_name == 'push' &&
      contains(join(github.event.commits.*.message, '\n'), ':building_construction:')

    name: Local Conan Server Test (${{ matrix.build_type }})
    runs-on: [self-hosted, linux, conan]

    strategy:
      matrix:
        build_type: [Debug, Release]

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'

      - name: Install Conan
        run: pip install conan

      - name: Get package name
        id: read_meta
        run: |
          pkg_name=$(jq -r '.name | tostring' metadata.json)
          pkg_version=$(jq -r '.version | tostring' metadata.json)
          echo "pkg_name=$pkg_name" >> $GITHUB_OUTPUT
          echo "pkg_version=$pkg_version" >> $GITHUB_OUTPUT

      - name: Configure Conan for local server
        run: |
          echo "CONAN_HOME=${{ env.CONAN_HOME }}" >> $GITHUB_ENV
          conan profile detect --force
          conan remote add local-server ${{ env.CONAN_SERVER_URL }} --insert 0
          echo "${{ env.CONAN_SERVER_USER }}" | conan remote login local-server --password "${{ env.CONAN_SERVER_USER }}"
          conan remote list
          curl -s ${{ env.CONAN_SERVER_URL }}/v1/ping || echo "Warning: Cannot ping local server"

      - name: System dependencies
        run: |
          sudo apt install -y libnsl-dev build-essential

      - name: Build with Conan
        run: |
          conan create . \
            -pr:b=default \
            -pr:h=default \
            -s build_type=${{ matrix.build_type }} \
            --build=missing

      - name: Upload to local Conan Server
        run: |
          conan upload \
            "${{ steps.read_meta.outputs.pkg_name }}/${{ steps.read_meta.outputs.pkg_version }}@" \
            --remote=local-server \
            --confirm \
            --all

      - name: Verify upload
        run: |
          conan search "${{ steps.read_meta.outputs.pkg_name }}/${{ steps.read_meta.outputs.pkg_version }}" --remote=local-server || echo "Package search failed"

      - name: Test package download
        run: |
          conan download \
            "${{ steps.read_meta.outputs.pkg_name }}/${{ steps.read_meta.outputs.pkg_version }}" \
            --remote=local-server

      - name: Run test_package
        run: |
          cd test_package
          conan create . \
            -pr:h=default \
            -s build_type=${{ matrix.build_type }} \
            --build=missing

      - name: Clean up test_package builds
        if: always()
        run: |
          python ./test_package/conanfile.py || true

      - name: Clean up local packages
        if: always()
        run: |
          conan remove "${{ steps.read_meta.outputs.pkg_name }}/*" --confirm || true
          conan remove "test_*" --confirm || true

      - name: Display build summary
        if: always()
        run: |
          echo "========================================="
          echo "Build Summary"
          echo "========================================="
          echo "Package: ${{ steps.read_meta.outputs.pkg_name }}"
          echo "Version: ${{ steps.read_meta.outputs.pkg_version }}"
          echo "Build Type: ${{ matrix.build_type }}"
          echo "Conan Server: ${{ env.CONAN_SERVER_URL }}"
          echo ""
          echo "Local Conan Cache:"
          conan list "*" || true
          echo ""
          echo "Remote Servers:"
          conan remote list
```

### C. 常见问题 FAQ

#### Q1: 为什么 CI 没有触发？

**A**: 检查以下几点：
1. commit message 是否包含 `:building_construction:`
2. 是否推送到 main 分支
3. 是否是 push 事件（不是 PR）

#### Q2: 如何跳过 CI？

**A**: 使用 `[skip ci]` 或 `[ci skip]` 在 commit message 中：
```bash
git commit -m ":building_construction: 本地测试 [skip ci]"
```

#### Q3: 如何只运行特定构建类型？

**A**: 修改 matrix 排除不需要的组合：
```yaml
strategy:
  matrix:
    build_type: [Debug]  # 只运行 Debug
```

#### Q4: 如何同时上传到多个服务器？

**A**: 添加多个 upload 步骤：
```yaml
- name: Upload to local server
  run: conan upload "fcpp/*" --remote=local-server --confirm

- name: Upload to ConanCenter
  if: matrix.build_type == 'Release'
  run: conan upload "fcpp/*" --remote=conancenter --confirm
```

#### Q5: 如何调试 CI 问题？

**A**:
1. 启用 debug 模式：
   ```yaml
   env:
     ACTIONS_RUNNER_DEBUG: true
     ACTIONS_STEP_DEBUG: true
   ```
2. 添加调试步骤：
   ```yaml
   - name: Debug environment
     run: |
       echo "User: $(whoami)"
       echo "Home: $HOME"
       echo "Working directory: $(pwd)"
       env | sort
   ```

### D. 相关文档链接

- [Conan 官方文档](https://docs.conan.io/)
- [GitHub Actions 文档](https://docs.github.com/en/actions)
- [GitHub Self-Hosted Runner 文档](https://docs.github.com/en/actions/hosting-your-own-runners)
- [fcpp 项目参考文档](/home/lgq/conan_github_self_runner/conan_github_self_runner_guide_all_in_one.md)

---

## 总结

本文档提供了 fcpp 项目在本地 Conan Server + GitHub Self-Hosted Runner 环境中的完整使用指南。

**关键要点**：

✅ **环境检查**：确保 Conan Server 和 GitHub Runner 都在运行
✅ **配置正确**：CI 配置中的 `runs-on` 标签必须匹配 Runner
✅ **触发机制**：使用 `:building_construction:` emoji 触发 CI
✅ **验证上传**：检查 Conan Server 中的包是否成功上传
✅ **故障排除**：参考第 6 节的常见问题解决方案

**下一步**：

1. 完成第一次 CI 测试
2. 验证包已上传到本地服务器
3. 在其他项目中使用本地 Conan Server 的包
4. 根据需要自定义 CI 配置

**版本历史**：

- v1.0.0 (2026-01-16): 初始版本，完整的本地环境 CI 使用指南

---

**文档维护**: lgq
**最后更新**: 2026-01-16
**文档状态**: Ready for Use
