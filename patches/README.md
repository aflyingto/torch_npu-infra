# Patch 记录

本目录包含用于修复 torch_npu 与 PyTorch main 分支兼容性问题的 patch 文件。

## Patch 列表

| 编号 | 文件名 | 目标文件 | 问题描述 | 创建日期 | 状态 |
|------|--------|----------|----------|----------|------|
| 0001 | 0001-fix-pytorch-main-compatibility.patch | torch_npu/csrc/core/npu/CachingHostAllocator.cpp | PyTorch main 分支 CachingHostAllocatorImpl 接口变化，BlockPool 类型冲突 | 2026-03-07 | 待验证 |

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
- 第 60 行：`BlockPool *pool` → `NPUBlockPool *pool`
- 第 75 行：构造函数参数
- 第 407 行：`map` 函数参数
- 第 545-559 行：`AllocParams` 结构体
- 第 687 行：成员变量
- 第 837 行：`try_merge_blocks` 参数
- 第 888 行：`process_events` 移除 override
- 第 1015 行：`find_expandable_block` 参数
- 第 1064 行：`map_block` 参数
- 第 1076 行：局部变量
- 第 1101 行：`try_allocate_expandable_block` 参数

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
