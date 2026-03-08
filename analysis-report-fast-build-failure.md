# Fast Build Workflow 失败分析报告

## 执行信息

- **Workflow ID**: #22813746350
- **Workflow 名称**: Fast Build torch_npu
- **执行时间**: 2026-03-08 04:20:40 - 04:30:38 (约 10 分钟)
- **执行状态**: 失败
- **失败步骤**: Build torch_npu

## 环境信息

### 系统环境
- **操作系统**: Ubuntu 24.04.3 LTS
- **Runner Image**: ubuntu-24.04 (20260302.42.1)
- **Python 版本**: 3.10.19
- **CMake 版本**: 3.28.3-1build7
- **编译器**: GCC (Ubuntu 24.04)

### 软件版本
- **PyTorch 版本**: 2.12.0.dev20260307+cpu (Nightly build, 2026-03-07)
- **PyTorch 安装方式**: pip install --pre torch --index-url https://download.pytorch.org/whl/nightly/cpu
- **torch_npu 仓库**: Ascend/pytorch (master branch)
- **torch_npu 分支**: master

## 错误详情

### 主要错误类型
**编译错误**: C++ 源代码编译失败

### 错误文件
`torch_npu/torch_npu/csrc/core/npu/CachingHostAllocator.cpp`

### 错误原因分析

#### 错误 1: BlockPool 结构体成员访问错误

**错误信息**:
```
error: 'using at::CachingHostAllocatorImpl<...>::BlockPool = struct at::HostBlockPool<...>' has no member named 'pool'
```

**位置**: CachingHostAllocator.cpp:1076

**代码**:
```cpp
BlockPool &pool = *to_map->pool;
```

**原因**: PyTorch 的 `HostBlockPool` 结构体没有名为 `pool` 的公共成员。

---

#### 错误 2: BlockPool 结构体成员访问错误

**错误信息**:
```
error: 'using at::CachingHostAllocatorImpl<...>::BlockPool = struct at::HostBlockPool<...>' has no member named 'unmapped'
```

**位置**: CachingHostAllocator.cpp:1077, 1087

**代码**:
```cpp
pool.unmapped.erase(to_map);  // Line 1077
pool.unmapped.insert(remaining);  // Line 1087
```

**原因**: PyTorch 的 `HostBlockPool` 结构体没有名为 `unmapped` 的公共成员。

---

#### 错误 3: BlockPool 结构体成员访问错误

**错误信息**:
```
error: 'using at::CachingHostAllocatorImpl<...>::BlockPool = struct at::HostBlockPool<...>' has no member named 'blocks'; did you mean 'blocks_'?
```

**位置**: CachingHostAllocator.cpp:1097, 1128

**代码**:
```cpp
pool.blocks.insert(to_map);  // Line 1097
pool->blocks.erase(candidate);  // Line 1128
```

**原因**: PyTorch 的 `HostBlockPool` 结构体将 `blocks` 成员重命名为 `blocks_`（私有成员）。

### 编译错误汇总

| 行号 | 错误类型 | 访问的成员 | 建议 |
|------|---------|-----------|------|
| 1076 | 成员不存在 | `pool` | 需要检查 HostBlockPool 的正确成员名称 |
| 1077 | 成员不存在 | `unmapped` | 需要检查 HostBlockPool 的正确成员名称 |
| 1087 | 成员不存在 | `unmapped` | 需要检查 HostBlockPool 的正确成员名称 |
| 1097 | 成员不存在 | `blocks` | 应该使用 `blocks_` (私有成员) |
| 1128 | 成员不存在 | `blocks` | 应该使用 `blocks_` (私有成员) |

## 根本原因

**API 不兼容问题**:

1. **PyTorch 版本过新**: 使用了 PyTorch 2.12.0.dev20260307+cpu (2026-03-07 的 nightly build)

2. **torch_npu 代码未同步**: torch_npu 的 `CachingHostAllocator.cpp` 代码基于较旧的 PyTorch API 编写

3. **HostBlockPool 结构体变更**: PyTorch 在新版本中修改了 `HostBlockPool` 结构体的内部实现：
   - 移除了 `pool` 成员
   - 移除了 `unmapped` 成员
   - 将 `blocks` 重命名为 `blocks_` 并设为私有

## 影响分析

### 构建失败的影响
- 无法生成 torch_npu wheel 包
- 无法进行后续的测试和验证
- CI/CD 流程中断

### 兼容性问题
- torch_npu 与最新的 PyTorch nightly 版本不兼容
- 需要同步更新 torch_npu 代码以适配新的 PyTorch API

## 建议解决方案

### 方案 1: 使用兼容的 PyTorch 版本 (推荐)

**优点**:
- 快速解决构建问题
- 稳定性高
- 风险低

**缺点**:
- 无法使用最新的 PyTorch 特性
- 可能存在功能限制

**实施步骤**:
1. 检查 torch_npu 支持的 PyTorch 版本范围
2. 修改 workflow 使用稳定版本的 PyTorch（如 2.4.x 或 2.5.x）
3. 更新 workflow 配置

**示例修改**:
```yaml
- name: Install PyTorch Stable
  run: |
    pip install torch==2.4.0 --index-url https://download.pytorch.org/whl/cpu
```

---

### 方案 2: 修复 torch_npu 代码以适配新 PyTorch API

**优点**:
- 可以使用最新的 PyTorch 特性
- 长期解决方案

**缺点**:
- 需要深入了解 PyTorch 内部 API
- 开发和测试成本高
- 可能引入新的 bug

**实施步骤**:
1. 分析 PyTorch 新版本中 `HostBlockPool` 的实现
2. 修改 `CachingHostAllocator.cpp` 以使用新的 API
3. 编写测试验证修改
4. 提交 PR 到 torch_npu 仓库

---

### 方案 3: 等待 torch_npu 官方更新

**优点**:
- 无需自行维护代码
- 官方支持更可靠

**缺点**:
- 时间不可控
- 可能需要等待较长时间

**实施步骤**:
1. 关注 torch_npu 仓库的更新
2. 等待官方发布支持新 PyTorch 版本的更新
3. 更新到新版本的 torch_npu

---

### 方案 4: 使用特定的 PyTorch commit

**优点**:
- 可以精确控制 PyTorch 版本
- 便于调试和复现

**缺点**:
- 需要手动管理 PyTorch 构建
- 增加构建复杂度

**实施步骤**:
1. 找到与 torch_npu 兼容的 PyTorch commit
2. 从源码构建 PyTorch
3. 使用构建的 PyTorch 构建 torch_npu

## 推荐行动方案

### 短期方案 (立即可行)
采用 **方案 1**，修改 workflow 使用 PyTorch 2.4.0 或 2.5.0 稳定版本：

```yaml
- name: Install PyTorch Stable
  run: |
    pip install torch==2.4.0 --index-url https://download.pytorch.org/whl/cpu
```

### 中期方案 (1-2 周)
采用 **方案 2**，修复 torch_npu 代码以适配 PyTorch 2.12+：

1. 创建 patch 文件修复 `CachingHostAllocator.cpp`
2. 在 workflow 中应用 patch
3. 测试验证

### 长期方案 (持续)
采用 **方案 3**，与 torch_npu 社区合作：

1. 向 torch_npu 仓库提交 issue
2. 参与 torch_npu 社区讨论
3. 贡献代码修复兼容性问题

## 附录

### 完整错误日志片段

```
/home/runner/work/torch_npu-infra/torch_npu-infra/torch_npu/torch_npu/csrc/core/npu/CachingHostAllocator.cpp:1076:14: error: 'using at::CachingHostAllocatorImpl<...>::BlockPool = struct at::HostBlockPool<...>' has no member named 'pool'
  1076 |         BlockPool &pool = *to_map->pool;
       |                           ^~~~~~~~~~~~~

/home/runner/work/torch_npu-infra/torch_npu-infra/torch_npu/torch_npu/csrc/core/npu/CachingHostAllocator.cpp:1077:14: error: 'using at::CachingHostAllocatorImpl<...>::BlockPool = struct at::HostBlockPool<...>' has no member named 'unmapped'
  1077 |         pool.unmapped.erase(to_map);
       |              ^~~~~~~~

/home/runner/work/torch_npu-infra/torch_npu-infra/torch_npu/torch_npu/csrc/core/npu/CachingHostAllocator.cpp:1097:14: error: 'using at::CachingHostAllocatorImpl<...>::BlockPool = struct at::HostBlockPool<...>' has no member named 'blocks'; did you mean 'blocks_'?
  1097 |         pool.blocks.insert(to_map);
       |              ^~~~~~
       |              blocks_
```

### 相关文件

- Workflow 文件: `.github/workflows/fast-build.yml`
- 错误源文件: `torch_npu/torch_npu/csrc/core/npu/CachingHostAllocator.cpp`
- PyTorch 源码: `https://github.com/pytorch/pytorch`
- torch_npu 源码: `https://github.com/Ascend/pytorch`

### 参考资源

- PyTorch GitHub: https://github.com/pytorch/pytorch
- torch_npu GitHub: https://github.com/Ascend/pytorch
- PyTorch C++ API 文档: https://pytorch.org/cppdocs/

---

**报告生成时间**: 2026-03-08
**分析工具**: GitHub Actions Logs
**报告版本**: 1.0