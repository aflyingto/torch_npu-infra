# torch_npu-infra

torch_npu 基础设施仓库，用于从最新 PyTorch 源码构建 torch_npu。

## 目录结构

```
torch_npu-infra/
├── .github/
│   └── workflows/
│       ├── build-torch-npu.yml    # 主构建 workflow
│       └── test-submodules.yml     # 子模块测试 workflow
├── doc/
│   └── pytorch-torch_npu-compatibility-analysis.md  # 兼容性分析报告
├── patches/
│   ├── README.md                  # Patch 记录文件
│   └── 0001-fix-pytorch-main-compatibility.patch   # PyTorch兼容性patch
├── skills/
│   └── torch_npu-Build/
│       └── SKILL.md                # torch_npu 构建技能文档
├── AGENTS.md                       # Agent 协作指南
└── README.md                       # 本文件
```

## 使用方法

### 自动构建

推送到 `main` 或 `master` 分支会自动触发 GitHub Actions 构建。

### 手动触发

在 GitHub Actions 页面可以手动触发构建，支持以下参数：
- `python_version`: Python 版本（默认 3.10）
- `pytorch_branch`: PyTorch 分支/标签（默认 main）

### 本地构建

参考 `skills/torch_npu-Build/SKILL.md` 获取完整的构建指南。

## 关键文件说明

| 文件 | 说明 |
|------|------|
| `.github/workflows/build-torch-npu.yml` | 主构建 workflow，包含完整的构建流程 |
| `skills/torch_npu-Build/SKILL.md` | 构建技能文档，包含问题解决方案和 workflow 模板 |
| `AGENTS.md` | Agent 协作指南，包含提交规范 |

## 构建流程

1. **构建 PyTorch** - 从源码编译 PyTorch（main 分支）
2. **构建 CANN Stub** - 生成 CANN 桩库
3. **构建 torch_npu** - 编译 torch_npu wheel 包

## 已知问题

使用 PyTorch main 分支可能因 API 变化导致构建失败，这是预期行为。详见 `skills/torch_npu-Build/SKILL.md`。

## 最新变更

- 2026-03-07: 修复 ccache 缓存保存失败问题，在缓存恢复前创建目录
- 2026-03-07: 优化 workflow 触发逻辑，忽略非关键文件变更（文档、LICENSE 等）
- 2026-03-07: 添加 patch 记录文件，记录所有 patch 的关键信息
- 2026-03-07: 添加 PyTorch main 分支兼容性 patch，修复 CachingHostAllocator API 变化问题
- 2026-03-07: 添加 PyTorch/torch_npu 兼容性问题分析报告 (Issue #1)
- 2026-03-07: 精简 SKILL 文档，移除完整 workflow 模板，保留构建步骤提炼
- 2026-03-07: 添加 torch_npu 构建 SKILL 文档
- 2026-03-07: 添加 ccache 远端缓存支持
- 2026-03-07: 修复子模块初始化问题
- 2026-03-06: 初始化项目，添加 GitHub Actions workflow
