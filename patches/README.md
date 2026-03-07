# Patch 记录

本目录包含用于修复 torch_npu 与 PyTorch main 分支兼容性问题的 patch 文件。

## Patch 列表

| 编号 | 文件名 | 目标文件 | 问题描述 | 创建日期 | 状态 |
|------|--------|----------|----------|----------|------|
| 0001 | 0001-fix-pytorch-main-compatibility.patch | torch_npu/csrc/core/npu/CachingHostAllocator.cpp | PyTorch main 分支 CachingHostAllocatorImpl 接口变化，process_events() 方法签名不兼容 | 2026-03-07 | 待验证 |

## 详细说明

### 0001 - PyTorch main 分支兼容性修复

**问题现象：**
```
error: 'void at_npu::native::NPUExpandableHostAllocatorImpl::process_events()' marked 'override', but does not override
```

**问题原因：**
- PyTorch main 分支重构了 `CachingHostAllocatorImpl` 类
- `process_events()` 方法签名从 `void process_events()` 变为 `void process_events(BlockPool& pool)`
- torch_npu 使用自己的 `BlockPool` 结构体，无法适配新接口

**修复方案：**
- 移除 `process_events()` 方法的 `override` 关键字
- torch_npu 的 `NPUExpandableHostAllocatorImpl` 使用自己的 `process_events()` 实现

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
