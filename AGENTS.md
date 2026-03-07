# torch_npu-infra AGENTS.md

## Project Overview

This repository contains infrastructure for building torch_npu with the latest PyTorch source code.

## Key Components

- `.github/workflows/build-torch-npu.yml`: GitHub Actions workflow for building torch_npu
- Build process uses PyTorch main branch source code instead of pre-built wheels
- `skills/torch_npu-Build/SKILL.md`: Complete build skill documentation

## Build Process

1. Build PyTorch from source (main branch)
2. Build CANN stub libraries for torch_npu
3. Build torch_npu against the compiled PyTorch

## Important Notes

- PyTorch is built from source to ensure compatibility
- CANN stub libraries are built to provide necessary symbols
- The build uses GitHub Actions free runners (ubuntu-latest)

## Environment Variables

- `PYTHON_VERSION`: Python version to use (default: 3.10)
- `PYTORCH_BRANCH`: PyTorch branch to build from (default: main)
- `MAX_JOBS`: Maximum parallel compile jobs
- `USE_CUDA=0`: Disable CUDA support
- `_GLIBCXX_USE_CXX11_ABI=1`: Use CXX11 ABI

## Dependencies

- torch_npu requires PyTorch to be built from source
- CANN libraries are stubbed for build purposes
- op-plugin submodule is required

## Commit Guidelines

**重要：每次提交代码必须同步更新 README.md**

当进行以下操作时，必须更新 README.md：

1. 添加新的 workflow 或修改现有 workflow
2. 添加新的 SKILL 文档
3. 修改项目结构或关键配置
4. 修复重要问题或添加新功能

README.md 应包含：
- 项目简介和目的
- 目录结构说明
- 使用方法
- 关键文件说明
- 最新变更记录
