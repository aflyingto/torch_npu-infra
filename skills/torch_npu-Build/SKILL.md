# torch_npu 构建 SKILL

本文档描述如何从源码构建 torch_npu，包含完整的 GitHub Actions workflow 模板和关键问题解决方案。

## 一、构建流程概述

torch_npu 的构建分为以下主要步骤：

```
1. 环境准备 → 2. 构建PyTorch → 3. 构建CANN Stub → 4. 构建torch_npu
```

### 依赖关系

```
torch_npu 依赖 PyTorch (源码构建)
     ↓
CANN Stub 库 (提供 NPU 相关符号)
     ↓
最终生成 torch_npu wheel 包
```

## 二、环境要求

### 系统依赖

```yaml
- git, cmake, build-essential
- libopenblas-dev, liblapack-dev, libeigen3-dev
- libprotobuf-dev, protobuf-compiler
- ninja-build, ccache, patchelf
- libomp-dev, libnuma-dev
- libssl-dev, libffi-dev
```

### Python 依赖

```yaml
- pip, setuptools, wheel
- numpy, pyyaml, typing_extensions
- cmake, ninja, six
```

### 版本选择

| 组件 | 推荐版本 | 说明 |
|------|---------|------|
| Python | 3.10 | 推荐 3.8-3.11 |
| PyTorch | main / 稳定分支 | main 分支可能有 API 不兼容问题 |
| torch_npu | Ascend/pytorch main | 华为官方仓库 |

> **注意：** 使用 PyTorch main 分支构建可能因 API 变化而失败，这是预期行为。失败时说明 torch_npu 需要适配最新的 PyTorch API。

## 三、关键问题及解决方案

### 问题1: PyTorch API 不兼容

**现象：**
```
error: 'void at_npu::native::NPUExpandableHostAllocatorImpl::process_events()' marked 'override', but does not override
error: no member named 'blocks' in 'BlockPool'; did you mean 'blocks_'?
error: no member named 'unmapped' in 'BlockPool'
```

**原因：** PyTorch main 分支 API 频繁变化，torch_npu 代码未同步更新。

**分析：**
- PyTorch 的 `CachingHostAllocatorImpl` 类接口发生变化
- `BlockPool` 结构体成员 `blocks` 改名为 `blocks_`
- `unmapped` 成员被移除
- `AllocParams` 构造函数签名变化
- `process_events()` 方法签名变化

**处理方式：**
- **如果使用最新代码**：这是预期行为，说明 torch_npu 需要适配最新 PyTorch API，等待官方更新
- **如果需要成功构建**：使用 PyTorch 稳定版本（如 `v2.5.0` 或 `release/2.5`）

### 问题2: 子模块初始化失败

**现象：**
```
fatal: repository 'https://github.com/fmt.git' not found
fatal: repository 'https://github.com/googletest.git' not found
```

**原因：** 错误的 URL 替换逻辑导致子模块 URL 不正确。

**解决方案：**
```yaml
# 直接使用原始 .gitmodules，无需 URL 替换
- name: Initialize torch_npu submodules
  run: |
    cd torch_npu
    git submodule update --init --recursive
```

**验证：** 本地测试确认 `git submodule update --init --recursive` 可直接成功。

### 问题3: ccache 与汇编器冲突

**现象：**
```
/usr/bin/ccache: invalid option -- 'D'
```

**原因：** 设置 `CC=ccache gcc` 导致 ccache 被错误地用于汇编文件 (.S)。

**解决方案：**
```yaml
# 只设置 CMAKE 编译器启动器，不设置 CC/CXX
- name: Configure ccache
  run: |
    ccache --set-config=cache_dir=~/.ccache
    ccache --set-config=max_size=5G
    ccache --set-config=compression=true
    ccache -p
    ccache -z
    # ✅ 正确方式：只设置 CMAKE_COMPILER_LAUNCHER
    echo "CMAKE_C_COMPILER_LAUNCHER=ccache" >> $GITHUB_ENV
    echo "CMAKE_CXX_COMPILER_LAUNCHER=ccache" >> $GITHUB_ENV
    # ❌ 错误方式：不要设置 CC/CXX
    # echo "CC=ccache gcc" >> $GITHUB_ENV
    # echo "CXX=ccache g++" >> $GITHUB_ENV
```

### 问题4: ccache 缓存无法持久化

**解决方案：** 使用 cache/restore 和 cache/save 分离模式

```yaml
# 构建前恢复缓存
- name: Cache ccache
  uses: actions/cache/restore@v4
  with:
    path: ~/.ccache
    key: ${{ runner.os }}-ccache-${{ github.run_id }}
    restore-keys: |
      ${{ runner.os }}-ccache-

# ... 构建步骤 ...

# 构建后保存缓存（即使失败也保存）
- name: Save ccache (always)
  if: always()
  uses: actions/cache/save@v4
  with:
    path: ~/.ccache
    key: ${{ runner.os }}-ccache-${{ github.run_id }}
```

## 四、完整 Workflow 模板

```yaml
name: Build torch_npu with PyTorch

on:
  push:
    branches: [main, master]
  pull_request:
    branches: [main, master]
  workflow_dispatch:
    inputs:
      python_version:
        description: 'Python version'
        required: false
        default: '3.10'
      pytorch_branch:
        description: 'PyTorch branch/tag'
        required: false
        default: 'main'  # 使用最新代码，可能因 API 变化而失败

env:
  PYTHON_VERSION: ${{ github.event.inputs.python_version || '3.10' }}
  PYTORCH_BRANCH: ${{ github.event.inputs.pytorch_branch || 'main' }}

jobs:
  build:
    runs-on: ubuntu-latest
    timeout-minutes: 360

    steps:
      # 1. 安装系统依赖
      - name: Install system dependencies
        run: |
          sudo apt-get update
          sudo apt-get install -y \
            git cmake build-essential \
            libopenblas-dev liblapack-dev libeigen3-dev \
            libprotobuf-dev protobuf-compiler \
            curl wget ninja-build ccache patchelf \
            libomp-dev libnuma-dev file binutils \
            libssl-dev libffi-dev

      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: ${{ env.PYTHON_VERSION }}

      - name: Install Python build dependencies
        run: |
          pip install --upgrade pip setuptools wheel
          pip install numpy pyyaml typing_extensions cmake ninja six

      # 2. 配置 ccache（关键：使用 cache/restore + cache/save 模式）
      - name: Restore ccache
        uses: actions/cache/restore@v4
        with:
          path: ~/.ccache
          key: ${{ runner.os }}-ccache-${{ github.run_id }}
          restore-keys: |
            ${{ runner.os }}-ccache-

      - name: Configure ccache
        run: |
          ccache --set-config=cache_dir=~/.ccache
          ccache --set-config=max_size=5G
          ccache --set-config=compression=true
          ccache -p
          ccache -z
          # 只设置 CMAKE 编译器启动器
          echo "CMAKE_C_COMPILER_LAUNCHER=ccache" >> $GITHUB_ENV
          echo "CMAKE_CXX_COMPILER_LAUNCHER=ccache" >> $GITHUB_ENV

      # 3. 检出 PyTorch 源码
      - name: Checkout PyTorch
        uses: actions/checkout@v4
        with:
          repository: pytorch/pytorch
          ref: ${{ env.PYTORCH_BRANCH }}
          path: pytorch
          submodules: recursive

      # 4. 检出 torch_npu 源码
      - name: Checkout torch_npu
        uses: actions/checkout@v4
        with:
          repository: Ascend/pytorch
          path: torch_npu
          submodules: false

      - name: Initialize torch_npu submodules
        run: |
          cd torch_npu
          git submodule update --init --recursive

      # 5. 构建 PyTorch
      - name: Build PyTorch from source
        run: |
          cd pytorch
          export CMAKE_PREFIX_PATH="${{ env.Python3_ROOT_DIR }}"
          export USE_CUDA=0
          export USE_DISTRIBUTED=1
          export BUILD_TEST=0
          export MAX_JOBS=$(nproc)
          export CMAKE_BUILD_TYPE=Release
          export USE_CCACHE=1
          python setup.py build develop

      - name: Show ccache statistics
        run: ccache -s

      - name: Verify PyTorch installation
        run: |
          python -c "import torch; print(f'PyTorch version: {torch.__version__}')"

      # 6. 构建 CANN Stub 库
      - name: Build CANN stub libraries
        run: |
          cd torch_npu/third_party/acl/libs
          bash build_stub.sh

      # 7. 构建 torch_npu
      - name: Build torch_npu
        run: |
          cd torch_npu
          export CMAKE_PREFIX_PATH="${{ env.Python3_ROOT_DIR }}"
          export MAX_JOBS=$(nproc)
          export CMAKE_BUILD_TYPE=Release
          export _GLIBCXX_USE_CXX11_ABI=1
          export DISABLE_INSTALL_TORCHAIR=TRUE
          export DISABLE_RPC_FRAMEWORK=TRUE
          python setup.py build bdist_wheel

      - name: Show ccache statistics
        run: ccache -s

      # 8. 上传构建产物
      - name: Upload wheel artifact
        uses: actions/upload-artifact@v4
        with:
          name: torch_npu-wheel
          path: torch_npu/dist/*.whl
          retention-days: 30

      # 9. 保存 ccache 缓存（即使失败也保存）
      - name: Save ccache (always)
        if: always()
        uses: actions/cache/save@v4
        with:
          path: ~/.ccache
          key: ${{ runner.os }}-ccache-${{ github.run_id }}
```

## 五、快速检查清单

构建前请确认：

- [ ] 子模块使用 `git submodule update --init --recursive` 直接初始化
- [ ] ccache 只设置 `CMAKE_*_COMPILER_LAUNCHER`，不设置 `CC/CXX`
- [ ] ccache 使用 restore/save 分离模式持久化
- [ ] 系统依赖完整安装
- [ ] Python 依赖完整安装

> **注意：** 使用 PyTorch main 分支时，API 不兼容导致的构建失败是预期行为。

## 六、常见错误速查表

| 错误信息 | 原因 | 解决方案 |
|---------|------|---------|
| `invalid option -- 'D'` | ccache 用于汇编文件 | 只设置 CMAKE_COMPILER_LAUNCHER |
| `no member named 'blocks'` | PyTorch API 变化 | 使用稳定版 PyTorch |
| `repository not found` | 子模块 URL 错误 | 直接使用原始 .gitmodules |
| `CMake Error` | 依赖缺失 | 检查系统依赖安装 |
| `ccache miss` | 缓存未命中 | 检查 cache key 配置 |

## 七、本地测试建议

在提交 workflow 修改前，建议本地测试：

```bash
# 1. 克隆 torch_npu
git clone --depth 1 https://github.com/Ascend/pytorch torch_npu

# 2. 测试子模块初始化
cd torch_npu
git submodule update --init --recursive

# 3. 验证关键子模块
ls -la third_party/fmt/
ls -la third_party/op-plugin/
ls -la third_party/googletest/
```

## 八、参考链接

- [PyTorch 官方仓库](https://github.com/pytorch/pytorch)
- [torch_npu 官方仓库](https://github.com/Ascend/pytorch)
- [GitHub Actions Cache 文档](https://github.com/actions/cache)
- [ccache 官方文档](https://ccache.dev/)
