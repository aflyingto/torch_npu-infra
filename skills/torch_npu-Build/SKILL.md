# torch_npu 构建 SKILL

本文档描述如何从源码构建 torch_npu，包含构建步骤提炼和关键问题解决方案。

## 一、构建流程

```
环境准备 → 构建PyTorch → 构建CANN Stub → 构建torch_npu
```

## 二、构建步骤

### 步骤1: 环境准备

**系统依赖：**
```
git, cmake, build-essential, libopenblas-dev, liblapack-dev, 
libeigen3-dev, libprotobuf-dev, protobuf-compiler, ninja-build, 
ccache, patchelf, libomp-dev, libnuma-dev, libssl-dev, libffi-dev
```

**Python依赖：**
```
pip, setuptools, wheel, numpy, pyyaml, typing_extensions, cmake, ninja, six
```

**版本：** Python 3.10，PyTorch main 分支

### 步骤2: 检出代码

```bash
# PyTorch (含子模块)
git clone --branch main --recursive https://github.com/pytorch/pytorch

# torch_npu (先不拉取子模块)
git clone https://github.com/Ascend/pytorch torch_npu
cd torch_npu && git submodule update --init --recursive
```

### 步骤3: 配置 ccache

```bash
ccache --set-config=cache_dir=~/.ccache
ccache --set-config=max_size=5G
ccache --set-config=compression=true
ccache -z
# 只设置 CMAKE 编译器启动器，不设置 CC/CXX
export CMAKE_C_COMPILER_LAUNCHER=ccache
export CMAKE_CXX_COMPILER_LAUNCHER=ccache
```

### 步骤4: 构建 PyTorch

```bash
cd pytorch
export CMAKE_PREFIX_PATH="${Python3_ROOT_DIR}"
export USE_CUDA=0
export USE_DISTRIBUTED=1
export BUILD_TEST=0
export MAX_JOBS=$(nproc)
export CMAKE_BUILD_TYPE=Release
export USE_CCACHE=1
python setup.py build develop
```

### 步骤5: 构建 CANN Stub

```bash
cd torch_npu/third_party/acl/libs
bash build_stub.sh
```

### 步骤6: 构建 torch_npu

```bash
cd torch_npu
export CMAKE_PREFIX_PATH="${Python3_ROOT_DIR}"
export MAX_JOBS=$(nproc)
export CMAKE_BUILD_TYPE=Release
export _GLIBCXX_USE_CXX11_ABI=1
export DISABLE_INSTALL_TORCHAIR=TRUE
export DISABLE_RPC_FRAMEWORK=TRUE
python setup.py build bdist_wheel
```

### 步骤7: ccache 缓存持久化

使用 `actions/cache/restore@v4` 构建前恢复，`actions/cache/save@v4` 构建后保存（`if: always()`）。

## 三、关键问题

### 问题1: PyTorch API 不兼容

**现象：**
```
error: no member named 'blocks' in 'BlockPool'; did you mean 'blocks_'?
error: cannot convert 'at_npu::native::BlockPool' to 'at::HostBlockPool<...>&'
error: 'void process_events()' marked 'override', but does not override
```

**原因：** PyTorch main 分支将 `BlockPool` 定义为类型别名，与 torch_npu 自定义的 `BlockPool` 结构体冲突

**解决方案：** 使用 patch 重命名 torch_npu 的 `BlockPool` 为 `NPUBlockPool`

```bash
# 在 torch_npu 目录应用 patch
cd torch_npu
git apply /path/to/patches/0001-fix-pytorch-main-compatibility.patch
```

**Patch 修改内容：**
1. `struct BlockPool` → `struct NPUBlockPool`
2. 移除 `process_events()` 的 `override` 关键字
3. 更新所有 `BlockPool` 引用为 `NPUBlockPool`

**Patch 制作方法：**

1. 克隆 torch_npu 源码
2. 修改需要修复的文件
3. 生成 patch: `git diff > 0001-fix-xxx.patch`
4. 将 patch 放入项目的 `patches/` 目录
5. 在构建流程中应用 patch

### 问题2: 子模块初始化失败

**解决：** 直接 `git submodule update --init --recursive`，无需 URL 替换

### 问题3: ccache 与汇编器冲突

**现象：** `/usr/bin/ccache: invalid option -- 'D'`

**解决：** 只设置 `CMAKE_*_COMPILER_LAUNCHER`，不设置 `CC/CXX`

### 问题4: ccache 缓存不持久

**解决：** 使用 `cache/restore` + `cache/save` 分离模式

## 四、错误速查

| 错误 | 原因 | 解决 |
|-----|------|-----|
| `invalid option -- 'D'` | ccache 用于汇编 | 只设 CMAKE_COMPILER_LAUNCHER |
| `no member named 'blocks'` | PyTorch API 变化 | 应用兼容性 patch |
| `process_events() override` | PyTorch API 变化 | 应用兼容性 patch |
| `repository not found` | 子模块 URL 错误 | 直接用原始 .gitmodules |

## 五、参考链接

- [PyTorch](https://github.com/pytorch/pytorch)
- [torch_npu](https://github.com/Ascend/pytorch)
- [GitHub Actions Cache](https://github.com/actions/cache)
