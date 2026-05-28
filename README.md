# OpenClaw 学习笔记工程

> 用于系统学习 [OpenClaw](https://github.com/openclaw/openclaw) 架构、功能与更新的知识库。

## 项目简介

**OpenClaw** (原名 Clawdbot/Moltbot) 是一个开源的、自托管的个人 AI Agent 框架。它作为 daemon 进程运行在你自己的设备上，通过 Gateway 将 50+ 消息平台连接到 AI Agent 运行时，让大语言模型能够自主执行真实世界的任务。

### 核心数据

| 属性 | 值 |
|------|-----|
| 当前版本 | **2026.5.12** (稳定) |
| GitHub Stars | 350k+ |
| 用户数 | 3.2M+ |
| 创始人 | Peter Steinberger (现任 OpenAI) |
| 语言 | JavaScript/TypeScript (Node.js) |
| 运行要求 | Node 24 (推荐) / Node 22.14+ |
| 源码 | https://github.com/openclaw/openclaw |
| 文档 | https://docs.openclaw.ai/ |

### 核心特点

- **本地优先** — 作为 daemon 进程运行，数据不出本地
- **多平台接入** — 50+ 消息平台 (内置 + 通道插件)
- **自主执行** — Observe-Plan-Act 循环，自主完成复杂任务
- **多模型支持** — GPT-5.x / Opus 4.6 / Grok / Gemini + Runtime Fallbacks
- **持久记忆** — Markdown-first 记忆系统 (SOUL.md / AGENTS.md)
- **可扩展** — Skills + Plugins + MCP，ClawHub 市场 1200+ skills
- **开源社区** — GitHub 350k+ Stars，极活跃的社区

### 架构模式

```
Channels (50+) → Gateway (WebSocket:18789) → Agent Runtime (Pi Core) → Agentic Loop → Tools/Skills
                                                                                         ↕
                                                                            Memory (Markdown + SQLite)
```

## 学习目录

```
.
├── README.md                  # 本文件
├── docs/
│   ├── architecture.md        # 架构总览 (四层模型 + 组件详解)
│   ├── changelog.md           # 版本更新追踪 (2025 Q4 ~ 2026.5.18)
│   └── references.md          # 参考资料汇总 (70+ 链接)
├── notes/
│   ├── 01-gateway.md          # Gateway 控制平面 (WebSocket/Session/Queue)
│   ├── 02-agent-loop.md       # Agentic Loop (Observe-Plan-Act/Codex)
│   ├── 03-memory.md           # 记忆系统 (SOUL/AGENTS/Activation-Decay)
│   ├── 04-tools-skills.md     # Tools/Skills/Plugins (ClawHub/MCP/Browser)
│   ├── 05-channels.md         # 消息通道 (50+ 平台/BlueBubbles/Voice)
│   ├── 06-security.md         # 安全机制 (攻击面/事件/加固)
│   ├── 07-team-setup.md       # 从零搭建 AI 开发团队 (飞书 Channel)
│   ├── 08-session-mechanism.md # Session 机制深度分析 (源码级，含通信隔离方案)
│   ├── 09-team-config-analysis.md  # 真实团队配置分析报告
│   └── 10-team-collaboration-design.md  # 团队协同机制设计 (文档/通信/监控)
├── experiments/               # 实验代码/配置
└── .kiro/steering/            # Kiro 工作约定
```

## 学习路线

1. **阶段一** ✅ 理解整体架构（四层模型）
2. **阶段二**: 深入 Gateway WebSocket 和 Pi Agent Core 流程
3. **阶段三**: 研究 Memory/SOUL 系统和 Activation/Decay
4. **阶段四**: 了解 Skills/Plugins 开发和 ClawHub 生态
5. **阶段五**: 跟踪安全事件和版本更新
6. **阶段六**: 深入 Session 机制与多 Agent 通信设计（源码级分析）

## 相关链接

- 源码仓库: https://github.com/openclaw/openclaw
- 官方文档: https://docs.openclaw.ai/
- 官方博客: https://openclaw.ai/blog
- 中文版: https://github.com/jiulingyun/openclaw-cn
- Skills 市场: https://github.com/openclaw/skills
- 版本追踪: https://patchbot.io/ai/openclaw

---

*最后更新: 2026-05-20 | 基于 v2026.5.12*
