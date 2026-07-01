# Chromium net 模块学习

本仓库用于学习 Chromium 浏览器源码中的 `net` 模块，重点研究 socket 网络编程的实现。

## 学习目标

- 理解 Chromium `net` 模块的整体架构与分层设计
- 深入掌握 `net/socket` 下 socket 的封装、生命周期与异步 IO 模型
- 对照源码学习现代 C++ 网络编程的最佳实践（错误处理、跨平台抽象、缓冲管理）

## 目录说明

- `net/`：从 Chromium 源码中提取的 net 模块（完整目录，便于 AI 分析跨文件引用）
- `StudyPlan.md`：分阶段的学习路线与进度跟踪

## 环境与工具

- 使用 TRAE Work Code 模式 + 云端运行环境
- 仓库已托管至 GitHub，支持公司/家多设备同步查看学习记录

## 备注

源码版权归 Chromium 项目所有（BSD-3-Clause），本仓库仅用于个人学习研究。