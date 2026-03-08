# Patch 记录

本目录包含用于修复 torch_npu 与 PyTorch main 分支兼容性问题的 patch 文件。

## Patch 列表

| 编号 | 文件名 | 目标文件 | 问题描述 | 创建日期 | 状态 |
|------|--------|----------|----------|----------|------|
| 0001 | 0001-fix-pytorch-main-compatibility.patch | torch_npu/csrc/core/npu/CachingHostAllocator.cpp | PyTorch main 分支 CachingHostAllocatorImpl 接口变化，BlockPool 类型冲突 | 2026-03-07 | 待验证 |
| 0002 | 0002-fix-pytorch-2.12-hostblockpool-compatibility.patch | torch_npu/csrc/core/npu/CachingHostAllocator.cpp | PyTorch 2.12+ HostBlockPool 结构体成员变化 (pool/unmapped/blocks) | 2026-03-08 | 待验证 |
| 0003 | 0003-fix-ge-error-codes-missing-cstdint.patch | third_party/acl/inc/ge/ge_error_codes.h | ge_error_codes.h 使用 uint32_t 但缺少 stdint.h 头文件 | 2026-03-08 | 待验证 |

## 详细说明

### 0001 - PyTorch main 分支兼容性修复

**问题现象：**
```
error: no member named 'blocks' in 'BlockPool'; did you mean 'blocks_'?
error: cannot convert 'at_npu::native::BlockPool' to 'at::HostBlockPool<...>&'
error: 'void process_events()' marked 'override', but does not override
```

**问题原因：**
- PyTorch main 分支将 `BlockPool` 定义为类型别名 `HostBlockPool<S_, E_, B_>`
- torch_npu 定义了自己的 `BlockPool` 结构体
- 两者名称冲突导致类型不匹配
- `process_events()` 方法签名变化

**修复方案：**
1. 将 torch_npu 的 `BlockPool` 重命名为 `NPUBlockPool`
2. 移除 `process_events()` 的 `override` 关键字
3. 更新所有 `BlockPool` 引用为 `NPUBlockPool`

**修改位置：**
- 第 49-54 行：`struct BlockPool` → `struct NPUBlockPool`
- 第 60 行：`BlockPool *pool` → `NPUBlockPool *pool`（成员变量）
- 第 75 行：构造函数参数类型
- 第 407 行：`map` 函数参数类型
- 第 545-559 行：`AllocParams` 结构体成员
- 第 687 行：成员变量 `blocks_pool` 类型
- 第 837 行：`try_merge_blocks` 参数类型
- 第 888 行：`process_events` 移除 `override` 关键字
- 第 1015 行：`find_expandable_block` 参数类型
- 第 1064 行：`map_block` 参数类型
- 第 1076 行：局部变量类型
- 第 1101 行：`try_allocate_expandable_block` 参数类型

**patch 文件格式：**
- 使用标准 unified diff 格式
- 每个 hunk 前必须有空行分隔
- 每个 hunk header 格式：`@@ -起始行,行数 +起始行,行数 @@ 上下文`
- 文件末尾包含签名 `-- 2.43.0`

**相关链接：**
- Issue: #1
- 分析报告: doc/pytorch-torch_npu-compatibility-analysis.md

## Patch 使用方法

```bash
# 在 torch_npu 目录应用所有 patch
cd torch_npu
for patch in /path/to/patches/*.patch; do
  git apply --check "$patch" && git apply "$patch"
done
```

## Patch 制作方法

1. 克隆 torch_npu 源码
2. 创建修改分支：`git checkout -b fix-xxx`
3. 修改需要修复的文件
4. 生成 patch：`git diff > 000x-fix-xxx.patch`
5. 更新本记录文件
6. 将 patch 放入 `patches/` 目录

## 注意事项

- Patch 文件命名格式：`000x-brief-description.patch`
- 每个 patch 必须在本文件中记录
- Patch 应该尽量小而专注，一个 patch 解决一个问题
- 应用 patch 前务必使用 `git apply --check` 验证

### 0002 - PyTorch 2.12+ HostBlockPool 兼容性修复

**问题现象：**
```
error: 'using at::CachingHostAllocatorImpl<...>::BlockPool = struct at::HostBlockPool<...>' has no member named 'pool'
error: 'using at::CachingHostAllocatorImpl<...>::BlockPool = struct at::HostBlockPool<...>' has no member named 'unmapped'
error: 'using at::CachingHostAllocatorImpl<...>::BlockPool = struct at::HostBlockPool<...>' has no member named 'blocks'; did you mean 'blocks_'?
```

**问题原因：**
- PyTorch 2.12+ 完全重构了 `HostBlockPool` 结构体
- 移除了 `pool` 成员
- 移除了 `unmapped` 成员
- 将 `blocks` 重命名为 `blocks_` 并设为私有成员
- 改用 `ska::flat_hash_set` 替代 `std::set`
- torch_npu 代码直接访问了这些不存在的成员

**修复方案：**
1. 将 torch_npu 的 `BlockPool` 重命名为 `NPUBlockPool` 以避免与 PyTorch 的类型别名冲突
2. 保持 torch_npu 自己的 `NPUBlockPool` 实现（包含 `blocks` 和 `unmapped` 成员）
3. 更新所有 `BlockPool` 引用为 `NPUBlockPool`
4. torch_npu 使用自己的 `NPUBlockPool` 实现，不依赖 PyTorch 的 `HostBlockPool`

**修改位置：**
- 第 49-54 行：`struct BlockPool` → `struct NPUBlockPool`
- 第 60 行：`BlockPool *pool` → `NPUBlockPool *pool`（成员变量）
- 第 75 行：构造函数参数类型
- 第 407 行：`map` 函数参数类型
- 第 545-559 行：`AllocParams` 结构体成员
- 第 687 行：成员变量 `blocks_pool` 类型
- 第 837 行：`try_merge_blocks` 参数类型
- 第 1015 行：`find_expandable_block` 参数类型
- 第 1064 行：`map_block` 参数类型
- 第 1076 行：局部变量类型
- 第 1101 行：`try_allocate_expandable_block` 参数类型

**相关链接：**
- Issue: #2
- 分析报告: analysis-report-fast-build-failure.md

### 0003 - ge_error_codes.h 缺少 stdint.h 头文件

**问题现象：**
```
error: 'uint32_t' does not name a type
static const uint32_t ACL_ERROR_GE_PARAM_INVALID = 145000;
note: 'uint32_t' is defined in header '<cstdint>'; did you forget to '#include <cstdint>'?
```

**问题原因：**
- `ge_error_codes.h` 使用了 `uint32_t` 类型
- 但只包含了 `<stddef.h>`，没有包含 `<stdint.h>` 或 `<cstdint>`
- 新版 GCC 编译器对此更严格

**修复方案：**
在 `#include <stddef.h>` 前添加 `#include <stdint.h>`

**修改位置：**
- 第 18 行：添加 `#include <stdint.h>`
