# Tools / Skills 系统学习笔记

> OpenClaw 的能力扩展系统 — 让 Agent 具备执行真实世界操作的能力。

## 系统概述

OpenClaw 通过 Tools 和 Skills 实现能力扩展：
- **Tools**: Agent 可调用的具体功能（shell、文件、浏览器等）
- **Skills**: 教 Agent 如何使用工具的能力模块（包含说明文档）
- **Plugins**: 封装工具的可分发包（v2026.5.16+）

## Skills 架构

### 目录结构

Skills 使用 **AgentSkills** 兼容格式：

```
skills/
└── <author>/
    └── <skill-name>/
        └── SKILL.md          # YAML frontmatter + Markdown 指令
```

### SKILL.md 格式

```markdown
---
name: agent-browser
version: 1.2.0
author: matrixy
description: Headless browser automation for AI agents
requires:
  - chromium
environment:
  - linux
  - macos
---

# Agent Browser

## Usage
...使用说明...

## Commands
...命令列表...
```

### 加载机制

1. **Bundled Skills**: 随 OpenClaw 安装的默认 skills
2. **Local Overrides**: 用户本地自定义 skills
3. **过滤**: 按环境 (OS)、配置、二进制依赖存在性过滤
4. **优先级**: Local > Bundled

## 内置工具 (截至 2026.5)

| 工具 | 说明 | 备注 |
|------|------|------|
| Shell | 系统命令执行 | 核心能力 |
| File System | 文件读写、目录操作 | 核心能力 |
| Agent-Browser | 无头浏览器自动化 | v2026.5 起默认集成 |
| Canvas | 可视化画布操作 | |
| HTTP/API | 外部 API 调用 | |
| Chronix | 定时任务/Cron 调度 | |
| MCP Tools | Model Context Protocol 工具 | |
| Codex Tools | workspace/edit/exec | Codex runtime 提供 |

### Agent-Browser 详解

Agent-Browser 是 OpenClaw 的浏览器自动化工具：

- **类型**: 无头 Chromium 浏览器 CLI
- **特性**:
  - Accessibility-tree 快照 + stable refs
  - 确定性元素选择
  - JSON 输出用于可靠解析
  - 跨动态 SPA 页面的 rehydration
- **v2026.5.16+**: 支持浏览器对话框 (dialog) 可见性
  - 展示 pending/handled 模态对话框
  - `blockedByDialog` 状态
  - 按 dialog id 回答对话框
- **v2026.3.13+**: Chrome DevTools MCP 作为 first-class 驱动
  - 连接已有浏览器 session（含认证 tabs）

### Chrome DevTools MCP

```
OpenClaw ←→ Chrome DevTools MCP ←→ 你的浏览器 (已认证的 session)
```

- 只需一个配置变更即可连接
- 复用已有的登录状态
- v2026.3.13-beta.1 起支持

## ClawHub 市场

ClawHub 是 OpenClaw 的公共 Skills/Plugins 注册表：

| 属性 | 值 |
|------|-----|
| Skills 数量 | 1200+ |
| 安全扫描 | 每个 skill 上架前扫描 |
| 安全分级 | A-F 等级 |
| 安装方式 | 通过 CLI 安装 |

⚠️ **安全警告**: ClawHub 曾被投毒（恶意 packages），安装前注意检查评级。

## Plugin 系统 (v2026.5.16+)

### defineToolPlugin API

新的 typed tool plugin 开发方式：

```javascript
// 概念示例
import { defineToolPlugin } from '@openclaw/plugin-sdk';

export default defineToolPlugin({
  name: 'my-tool',
  description: 'My custom tool',
  // 生成的 metadata
  // 可选的 declarations
  // context factories
  execute: async (params, context) => {
    // 工具实现
  }
});
```

### CLI 工具链

```bash
# 初始化插件项目
openclaw plugins init my-plugin

# 构建
openclaw plugins build

# 验证
openclaw plugins validate

# 发布到 ClawHub
openclaw plugins publish
```

## MCP (Model Context Protocol) 支持

OpenClaw 兼容 MCP 协议：
- 可以接入外部 MCP servers 提供的工具
- Chrome DevTools MCP 是典型案例
- 扩展 Agent 的工具箱而不修改核心

## Tool Execution 流程

```
LLM 输出 tool_call
    │
    ▼
权限检查 (Exec Policy Engine)
    │ 是否允许？
    ├── 拒绝 → 返回错误给 LLM
    │
    ▼ (允许)
环境选择
    ├── 沙箱执行 (Container Boundary)
    └── 直接执行 (本地)
    │
    ▼
执行工具
    │
    ▼
收集结果 → 返回给 Agentic Loop → 下一轮推理
```

## 安全考量

### Exec Policy Engine
- 最主要的攻击面
- 控制哪些工具可以执行
- 控制执行参数的限制

### Container Boundary
- 可选的沙箱隔离
- 防止工具执行逃逸
- Claw Chain 漏洞曾突破此边界

### 权限分级
- 低风险：文件读取、信息查询
- 中风险：文件修改、API 调用
- 高风险：Shell 执行、系统操作
- 敏感操作：需要用户确认

## 问题与思考

- [x] Skill 的格式？→ SKILL.md with YAML frontmatter
- [x] 如何安装第三方 skill？→ ClawHub + CLI
- [x] 浏览器自动化方案？→ Agent-Browser (默认) + Chrome DevTools MCP
- [ ] defineToolPlugin 的完整 TypeScript 类型定义？
- [ ] Tool 调用的超时配置？
- [ ] 沙箱的具体技术实现 (Docker? seccomp? chroot?)？
- [ ] Skills 的加载性能影响？
- [ ] 自定义 skill 的调试方法？

## 学习记录

| 日期 | 内容 | 来源 |
|------|------|------|
| 2026-05-20 | 全面更新: Plugin 系统、Agent-Browser、ClawHub、安全 | 多源 |

---
