# TensorFlow Serving Docker 编译日志

## 任务目标
为 ARM 架构 (arm64v8) 编译 TensorFlow Serving Docker 镜像

---

## 遇到的问题及解决方案

### 问题 1: 基础镜像不匹配
**现象:**
- 原始 Dockerfile 使用 `ubuntu:22.04@sha256:xxx` (x86_64 架构)
- 需要为 ARM 平台编译

**决策:**
- 修改 FROM 指令为 `arm64v8/ubuntu` (ARM64 架构)

**操作:**
```dockerfile
# 修改前
FROM ubuntu:22.04@sha256:eb29ed27b0821dca09c2e28b39135e185fc1302036427d5f4d70a41ce8fd7659 as base_build

# 修改后
FROM arm64v8/ubuntu as base_build
```

**结果:** ✅ 成功使用 ARM 基础镜像

---

### 问题 2: 包不可用 - mlocate
**现象:**
```
E: Unable to locate package mlocate
```

**原因:**
- mlocate 在某些 Ubuntu 版本或 ARM 仓库中不可用

**决策:**
- 从安装包列表中移除 mlocate（这是一个可选的搜索工具，不是核心依赖）

**操作:**
```dockerfile
# 移除该行
mlocate \
```

**结果:** ✅ apt-get 成功完成

---

### 问题 3: Python 3.10 包不存在
**现象:**
```
E: Unable to locate package python3.10-dev
E: Unable to locate package python3.10-venv
```

**原因:**
- arm64v8/ubuntu 的最新版本是 Ubuntu 24.04 (Noble)
- Ubuntu 24.04 默认 Python 版本是 3.12，不再提供 3.10 包

**决策:**
- 改用系统默认 Python 版本 (python3 -> python3.12)
- 移除硬指定的 python3.10 安装

**操作:**
```dockerfile
# 修改前
RUN apt-get update && apt-get install -y \
    python3.10 python3.10-dev python3-pip python3.10-venv && \
    rm -rf /var/lib/apt/lists/* && \
    python3.10 -m pip install pip --upgrade && \
    update-alternatives --install /usr/bin/python3 python3 /usr/bin/python3.10 0

# 修改后
RUN apt-get update && apt-get install -y \
    python3-dev python3-pip python3-venv && \
    rm -rf /var/lib/apt/lists/*
```

**结果:** ✅ Python 包安装成功

---

### 问题 4: PEP 668 - 系统 Python 保护
**现象:**
```
error: externally-managed-environment
× This environment is externally managed
```

**原因:**
- Ubuntu 24.04 实现了 PEP 668，防止 pip 直接修改系统 Python
- 尝试升级 pip 失败

**决策:**
- 移除 pip 升级步骤（系统 pip 版本足够）
- 在 pip install 时使用 `--break-system-packages` 标志

**操作:**
```dockerfile
# 移除此命令
# RUN curl -fSsL -O https://bootstrap.pypa.io/get-pip.py && \
#     python3 get-pip.py && \
#     rm get-pip.py

# 在 pip install 时添加标志
RUN pip3 --no-cache-dir install --break-system-packages \
    future>=0.17.1 \
    grpcio \
    ...

RUN pip --no-cache-dir install --break-system-packages --upgrade \
    /tmp/pip/tensorflow_serving_api-*.whl && \
```

**结果:** ✅ Python 依赖安装成功

---

### 问题 5: 编译错误 - x86 指令集在 ARM 上不支持
**现象:**
```
gcc: error: unrecognized command-line option '-mavx'
gcc: error: unrecognized command-line option '-msse4.2'
```

**原因:**
- Bazel 使用了 x86 特定的编译标志 (`-mavx`, `-msse4.2`)
- ARM 架构的 gcc 不支持这些标志
- 编译配置被设置为 x86 优化，而在 ARM 架构上运行

**影响的编译目标:**
- curl (lib/hmac.c)
- tensorflow lite (array.cc, kernel_util.cc)
- tensorflow core (error_logging.cc, path_utils.cc, etc.)
- tensorflow_text (boise_offset_converter.cc)

**第一次尝试 (b7h85xayl):**
- 添加 `--copt=-march=armv8-a --copt=-mtune=generic`
- 结果: ❌ 编译在 1386 秒失败，仍然报告 x86 指令集错误

**原因分析:**
- `--copt` 添加的编译选项还不足以覆盖所有的 x86 标志
- 需要更激进的方法来禁用 x86 优化

**第二次尝试 (biyicsryq - 进行中):**
- 添加更多 Bazel 编译选项来禁用 x86 特定优化
- 清除旧的编译缓存 `/root/.cache/bazel`

**结果:** ⏳ 编译进行中（T+530秒）...


---

## 编译时间线

| 时间 | 事件 |
|------|------|
| T+0 | 初始编译开始，修改基础镜像为 arm64v8/ubuntu |
| T+1 | 遇到 mlocate 包不可用，移除该包 |
| T+2 | 遇到 python3.10 包不存在，改用系统默认 Python 版本 |
| T+3 | 遇到 PEP 668 限制，添加 --break-system-packages 标志 |
| T+4 | 编译进行到 ~1750/13549 action，遇到 x86 指令集错误 |
| T+5 | 添加 ARM Bazel 编译选项，清除缓存重新编译 |

---

## 关键变更汇总

### Dockerfile 修改清单
- [x] 基础镜像: `ubuntu:22.04` → `arm64v8/ubuntu`
- [x] 移除不可用包: `mlocate`
- [x] Python: `python3.10` → `python3`（系统默认）
- [x] 移除 pip 升级步骤（避免 PEP 668 冲突）
- [x] 添加 `--break-system-packages` 到 pip install
- [x] Bazel 编译选项: 添加 ARM 架构标志

### 环境相关
- 清除 Bazel 缓存: `/root/.cache/bazel`
- Docker 基础镜像: `arm64v8/ubuntu:latest` (Ubuntu 24.04 Noble)
- Python 版本: 3.12 (系统默认)

---

## 编译历史记录

### 编译运行 #2 (2026-04-09)
**开始时间:** 2026-04-09 10:00 UTC  
**失败时间:** T+760秒 (约12分40秒)

**构建命令:**
```bash
docker build \
  -f tensorflow_serving/tools/docker/Dockerfile.devel \
  -t tensorflow-serving:devel-arm64 \
  --build-arg TF_SERVING_VERSION_GIT_BRANCH=master \
  --build-arg TF_SERVING_VERSION_GIT_COMMIT=HEAD \
  .
```

**状态:** ❌ **编译失败**

**失败原因：**
```
Error in fail: error running 'git -c protocol.file.allow=always submodule update --init --recursive --checkout --force' while working with @org_boost:
Timed out
```

**根本原因分析：**
- Boost 库 git 克隆超时
- boost 子模块递归更新耗时超过 10 分钟
- git 操作超过默认超时时间

**进展时间线：**
- [x] 加载 Dockerfile 配置（0.1s）
- [x] 拉取 arm64v8/ubuntu 基础镜像（2.9s）
- [x] 安装依赖包（137s）
- [x] 下载 Bazel 依赖（~200s）
- ❌ Bazel 分析阶段 - **boost 子模块更新超时**（T+760s）

**编译输出行数:** 7,034

---

## 问题 6: Boost 库下载超时
**现象:**
```
Error running 'git submodule update --init --recursive' while working with @org_boost: Timed out
```

**原因:**
- boost 库的递归子模块克隆非常耗时（>600 秒）
- git 默认超时时间不足
- ARM 架构上网络速度较慢可能加剧此问题

**可能的解决方案:**

### 方案 1: 增加 git 超时时间（推荐）
在 Dockerfile 中设置 git 超时环境变量：
```dockerfile
ENV GIT_SSH_COMMAND="ssh -o ConnectTimeout=300 -o StrictHostKeyChecking=no"
RUN git config --global http.timeout 600
RUN git config --global core.compression 0
```

### 方案 2: 禁用递归子模块更新
修改 WORKSPACE 文件中的 boost 配置：
```python
git_repository(
    name = "org_boost",
    # ... 其他配置
    recursive_init_submodules = False,  # 禁用递归
)
```

### 方案 3: 使用 Bazel 下载缓存
```bash
# 使用 Bazel 的仓库缓存
docker build \
  --build-arg http_proxy=... \
  --build-arg https_proxy=... \
  ...
```

### 方案 4: 增加 Docker 构建超时
```bash
docker build \
  --progress=plain \
  --timeout=7200 \
  ...
```

**建议下一步操作:**
1. 在 Dockerfile 中添加 git 配置（方案 1）
2. 重新编译测试

---

### 编译运行 #3 (2026-04-09) - 修复 Git 超时
**开始时间:** 2026-04-09 (T+760 秒后)
**失败时间:** T+753秒（约12.5分钟）

**修改内容:**
```dockerfile
# Configure git for slow/large operations (boost library download)
RUN git config --global http.timeout 600 && \
    git config --global core.compression 0 && \
    git config --global http.postBuffer 524288000
```

**状态:** ❌ **编译再次失败**

**失败信息：**
```
Error in fail: error running 'git -c protocol.file.allow=always submodule update --init --recursive --checkout --force' while working with @org_boost:
Timed out
```

**失败分析:**
- boost 子模块递归更新耗时：~600 秒
- 实际超时时间：753 秒，但 git 超时配置为 600 秒
- **问题：600 秒超时仍然不足！**

**编译输出行数:** 5,181

**失败原因分析:**
boost 库的递归子模块更新非常耗时：
- 主要耗时操作：`git submodule update --init --recursive --checkout --force`
- 预计耗时：>600 秒
- 当前配置：600 秒（不足）

---

## 问题 7: 增加 Git 超时时间还是不够

**症状:**
- 编译 #2：在 T+760 秒失败，git timeout 默认
- 编译 #3：在 T+753 秒失败，git timeout 600 秒配置

**根本原因:**
- boost 库的递归子模块更新需要 >600 秒
- 需要进一步增加 git 超时时间到 1200 秒（20 分钟）
- 或者使用 DNS 缓存/本地镜像来加速

**可能的解决方案:**

### 方案 1: 增加 git 超时到 1200 秒（推荐，快速修复）
```dockerfile
RUN git config --global http.timeout 1200 && \
    git config --global core.compression 0 && \
    git config --global http.postBuffer 524288000 && \
    git config --global core.packedRefsTimeout 1200
```

### 方案 2: 跳过递归子模块初始化（激进方案）
修改 WORKSPACE 文件中 org_boost 配置，禁用递归子模块更新。但这可能导致编译时缺少依赖。

### 方案 3: 预先缓存 boost 库
在 Docker 镜像构建前下载并缓存 boost 库，避免重复下载。

### 方案 4: 使用网络优化
- 增加 DNS 超时
- 使用本地 git 镜像
- 禁用 IPv6

**建议下一步操作:**
1. **优先采用方案 1** - 增加 git 超时到 1200 秒
2. 如果仍然超时，考虑方案 3（预缓存）或方案 4（网络优化）

---

### 编译运行 #4 (2026-04-09) - 增加 Git 超时到 1200 秒
**开始时间:** 2026-04-09 (T+753 秒后)
**失败时间:** T+769秒（约12.8分钟）

**状态:** ❌ **编译失败 - boost 超时问题依然存在**

**失败信息：**
```
Error in fail: error running 'git -c protocol.file.allow=always submodule update --init --recursive --checkout --force' while working with @org_boost:
Timed out
```

**失败分析:**
- boost 子模块递归更新耗时：602 秒
- git http.timeout 配置：1200 秒
- **但编译仍然超时！** - 说明问题不是 git timeout，而是 **Bazel 内部的超时机制**

**根本原因:**
- git 超时配置对 Bazel 的git_repository 规则没有完全生效
- Bazel 可能有独立的超时配置或更严格的限制
- boost 库的递归子模块更新本质上太耗时

**编译历史汇总：**
| 编译 | Git 超时 | 失败时间 | 原因 |
|------|---------|--------|------|
| #2 | 默认 | T+760s | git 超时 ❌ |
| #3 | 600s | T+753s | git 超时 ❌ |
| #4 | 1200s | T+769s | Bazel 内部超时 ❌ |

---

## 问题 8: Bazel 内部超时机制

**症状:**
- 简单的 git 超时配置不足以解决问题
- boost 库递归子模块更新需要 >600 秒
- 即使设置 git timeout 为 1200 秒仍然超时

**根本原因:**
- Bazel 的 git_repository 规则有内部超时机制
- git 命令本身的超时与 Bazel 框架级的超时不同
- boost 库太大，递归子模块更新本质上低效

**可能的解决方案:**

### 方案 1: 清除 Bazel 缓存并禁用递归子模块更新（推荐）
```bash
# 在 WORKSPACE 中修改 org_boost 配置
git_repository(
    name = "org_boost",
    # ... 其他配置
    # 禁用递归子模块初始化
    recursive_init_submodules = False,
)

# 清除 Bazel 缓存
rm -rf ~/.cache/bazel/
```

### 方案 2: 增加 Bazel 内部超时（复杂）
在 Bazel 配置或 BUILD 文件中设置全局超时（较复杂，需要修改多个配置文件）。

### 方案 3: 预先缓存 boost 库（根治方案）
在 Dockerfile 中提前下载并缓存 boost 库，避免在构建时重新下载：
```dockerfile
RUN git clone --depth 1 https://github.com/boostorg/boost.git /cache/boost && \
    cd /cache/boost && \
    git submodule update --init --recursive --depth 1
```

### 方案 4: 修改 WORKSPACE 使用本地或镜像 boost
使用本地 boost 库或使用预构建的 boost 库包。

**建议下一步操作:**
1. **优先采用方案 1** - 禁用递归子模块更新
   - 如果 boost 库在没有子模块的情况下可用，这是最快的解决方案
   - 需要修改 TensorFlow Serving 的 WORKSPACE 文件

2. 如果方案 1 不可行，采用 **方案 3** - 预缓存
   - 根治 boost 下载超时问题
   - 在 Dockerfile 中提前下载并缓存整个 boost 库

---

### 编译运行 #5 (2026-04-09) - 清除缓存和 Bazel 超时缩放 ⭐ **失败！**
**开始时间:** 2026-04-09 (T+769 秒后)
**失败时间:** T+753.5秒（约12分33秒）

**修改内容:**
```dockerfile
# Clear Bazel cache to avoid stale state and increase Bazel's fetch timeout
RUN rm -rf /root/.cache/bazel && \
    bazel clean --expunge 2>/dev/null || true

RUN bazel build --color=yes --curses=yes \
    ${TF_SERVING_BAZEL_OPTIONS} \
    --verbose_failures \
    --output_filter=DONT_MATCH_ANYTHING \
    --http_timeout_scaling=5.0 \
    --spawn_strategy=standalone \
    ${TF_SERVING_BUILD_OPTIONS} \
    tensorflow_serving/model_servers:tensorflow_model_server && \
    ...
```

**状态:** ❌ **编译失败 - boost 超时仍未解决**

**失败错误:**
```
Error in fail: error running 'git -c protocol.file.allow=always submodule update --init --recursive --checkout --force' while working with @org_boost:
Timed out
```

**失败分析:**
- boost 递归子模块更新耗时：约 602 秒
- 总运行时间：753.5 秒
- `--http_timeout_scaling=5.0` 未能解决问题
- **根本原因：Bazel 内部 git 命令超时，不是 HTTP 超时**

**编译失败汇总 (所有 5 次编译):**
| 编译 | 超时设置 | 失败时间 | boost 更新耗时 | 原因 |
|------|--------|---------|----------------|------|
| #1 | 无 | ~610s | N/A | x86 指令集错误 |
| #2 | 无 | T+760s | ~600s | git 超时 ❌ |
| #3 | 600s | T+753s | ~600s | git 超时 ❌ |
| #4 | 1200s | T+769s | ~602s | git 超时 ❌ |
| #5 | http_timeout_scaling=5.0 | T+753.5s | ~602s | Bazel git 超时 ❌ |

**关键发现：**
- 每次编译都在 boost 递归子模块更新时失败
- boost 更新耗时始终在 600-602 秒之间
- 问题不是 HTTP 超时，而是 Bazel 内部 git 命令的超时机制
- `--http_timeout_scaling=5.0` 不对症下药

---

---

---

---

---

### 编译运行 #8 (2026-04-09) - 极限 HTTP 超时缩放 (50.0倍) ❌ **失败**
**开始时间:** 2026-04-09 (T+768 秒后)
**失败时间:** T+787秒（约13分钟）

**状态:** ❌ **编译失败 - 仍然超时！**

**失败分析:**
- 超时时间：T+787 秒（比之前的 T+753-768s 更晚）
- boost 更新耗时：约 600 秒（与之前相同）
- http_timeout_scaling=50.0 仍然不足
- 结论：**问题不是简单的超时数字，而是 Bazel 的 git_repository 对 boost 的处理方式本身**

**所有编译失败汇总:**
| 编译 | 关键改进 | 超时参数 | 失败时间 | boost耗时 |
|------|--------|--------|---------|---------|
| #5 | http_timeout_scaling | 5.0x | T+753s | 602s |
| #7 | recursive_init_submodules=False | 5.0x | T+768s | 602s |
| #8 | recursive_init_submodules=False + 50.0x | 50.0x | T+787s | 600s |

**关键发现:**
- boost 子模块更新耗时始终在 **600-602 秒**
- 增加 HTTP 超时倍数不能根本解决问题
- **根本原因：Bazel 的 git_repository 规则不适合处理大型 git 仓库的递归初始化**
- 即使不递归初始化，boost 的第一层子模块初始化仍然需要 600 秒

---

## 推荐解决方案：完全绕过 Bazel 的 git_repository

**问题的本质:**
- Bazel 的 `new_git_repository` 规则依赖于 git 子模块初始化
- boost 仓库的子模块初始化本质上很慢（即使非递归）
- 无法通过调整超时参数来根本解决

**方案 A: 使用预构建的 Boost 二进制包（最简单）** ⭐ **推荐**
- 使用系统 apt 安装的 boost-all-dev：`apt-get install -y libboost-all-dev`
- 或从 https://packages.ubuntu.com 下载预编译的 ARM64 boost 包
- 修改 BUILD 文件以使用系统 boost 而不是从源码编译
- 预期效果：完全避免 git 下载和编译，节省 10+ 分钟编译时间

**方案 B: 在 Docker 镜像中预先完全缓存 boost 源码**
```dockerfile
# 在 base_build 阶段预先下载并缓存 boost（只需一次）
RUN mkdir -p /boost-src && \
    cd /boost-src && \
    git clone --depth 1 --single-branch https://github.com/boostorg/boost.git . && \
    # 修改 WORKSPACE 以指向本地 /boost-src
```

**方案 C: 使用 http_archive 而不是 git_repository**
```python
# 在 workspace.bzl 中替换：
http_archive(
    name = "org_boost",
    url = "https://github.com/boostorg/boost/archive/refs/tags/boost-1.75.0.tar.gz",
    # 无需子模块初始化，更快更稳定
)
```

**方案 D: 使用官方的 TensorFlow Boost 规则**
- TensorFlow 可能已经有内置的 boost 依赖管理
- 检查是否可以重用 TensorFlow 的 boost 配置

**立即建议:**
采用 **方案 A**（系统 boost 包）是最快的解决方案：
1. 在 Dockerfile 中添加：`apt-get install -y libboost-all-dev`
2. 修改 third_party/boost/BUILD 文件以使用系统 boost
3. 预期编译时间从 40+ 分钟降低到 10-15 分钟









**症状:**
- 所有 5 次编译都在 boost 递归子模块更新时失败
- boost 更新耗时 600-602 秒
- git 超时配置、http_timeout_scaling 都无效
- 问题在 Bazel 内部的 git 命令执行机制

**根本原因:**
- Bazel 的 git_repository 规则执行 `git submodule update --init --recursive` 时有内置超时
- 该超时无法通过简单的编译标志完全禁用
- boost 库的递归子模块本质上就需要 600+ 秒

**推荐解决方案:**

### 方案 A: 在 Dockerfile 中预先缓存 boost 库（最推荐）⭐

在编译前提前下载并缓存 boost 库，避免在 Bazel 分析阶段重新下载：

```dockerfile
# Pre-cache boost library to avoid git submodule timeout during Bazel analysis
RUN mkdir -p /boost-cache && \
    cd /boost-cache && \
    git clone --recursive --depth 1 https://github.com/boostorg/boost.git . 2>&1 | tail -10 && \
    echo "✅ boost library cached successfully" || echo "⚠️ boost cache failed (will retry in Bazel)"

# Then in the Bazel build command, use a local boost path if available
# This requires modifying WORKSPACE to use the pre-cached boost
```

**优点:**
- 将耗时的 boost 下载从 Bazel 分析阶段移出
- 可以在 Dockerfile 中单独处理和缓存
- 更好的 Docker 层缓存利用

**缺点:**
- 需要修改 TensorFlow Serving 的 WORKSPACE 文件
- boost 库体积大（下载 ~500MB+）

### 方案 B: 禁用 org_boost 的递归子模块初始化

修改 `tensorflow_serving/workspace.bzl` 中的 boost 配置，禁用递归初始化：

```python
# 在 workspace.bzl 中找到 org_boost 的 git_repository 定义
git_repository(
    name = "org_boost",
    # ... 其他配置
    recursive_init_submodules = False,  # 添加这一行
)
```

**优点:**
- 不需要预缓存，只需一行改动
- 快速修复

**缺点:**
- boost 库可能缺少某些子模块导致编译失败
- 需要验证 boost 在不包含子模块的情况下是否仍能正常工作

### 方案 C: 使用 Bazel 的 --nofetch 并在之前预缓存

```bash
# 第一步：预缓存所有外部依赖（可能需要 30-60 分钟）
bazel fetch //tensorflow_serving/model_servers:tensorflow_model_server

# 第二步：使用 --nofetch 进行实际编译（跳过下载）
bazel build --nofetch ...
```

**优点:**
- 完整的依赖缓存
- 编译时可以完全跳过下载

**缺点:**
- 增加 Dockerfile 步骤和整体编译时间
- 需要在 Docker 构建时运行两次 bazel 命令

### 方案 D: 增加全局 Bazel 超时配置文件

创建 `.bazelrc` 文件，配置更激进的超时参数：

```bash
# 在 Dockerfile 中添加
RUN echo "fetch --repository_cache=/root/.cache/bazel-repo-cache" > /root/.bazelrc && \
    echo "build:ci --@local_tls//:openssl_fips=false" >> /root/.bazelrc && \
    mkdir -p /root/.cache/bazel-repo-cache
```

**优点:**
- 在全局配置中管理超时
- 可以持久化到镜像中

**缺点:**
- 需要对 Bazel 配置有深入理解
- 效果可能有限

**强烈推荐：同时采用方案 A + B**

1. **先采用方案 B**（禁用递归初始化） - 快速验证是否有效
2. 如果 #6 编译成功，说明 boost 不需要完整的子模块
3. 如果 #6 编译失败（缺少子模块），改用方案 A（预缓存）



## 下一步

- [ ] 编译完成并验证镜像
- [ ] 测试编译的 TensorFlow Serving 二进制文件
- [ ] 验证 ARM 架构兼容性

---

## 参考信息

**编译命令:**
```bash
docker build -f tensorflow_serving/tools/docker/Dockerfile.devel -t tensorflow-serving:devel .
```

**编译参数:**
- `--copt=-march=armv8-a`: 指定 ARMv8 架构
- `--copt=-mtune=generic`: 使用通用 ARM CPU 优化（避免特定 CPU）
- `--break-system-packages`: 允许 pip 在系统 Python 中安装包（Ubuntu 24.04）

**相关文件:**
- `/workspace/serving/tensorflow_serving/tools/docker/Dockerfile.devel`
- `/root/.cache/bazel/` (编译缓存)
