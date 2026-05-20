# OpenClaw 学习笔记工程

> 用于系统学习 [OpenClaw](https://github.com/openclaw/openclaw) 架构、功能与更新的知识库。

## 项目简介

**OpenClaw** 是一个开源的、自托管的个人 AI 助手框架。它运行在你自己的设备上（而非云服务），通过连接你已有的消息平台（WhatsApp、Telegram、Slack、Discord、Signal、iMessage、Microsoft Teams 等 50+ 平台），将大语言模型转变为可以自主执行任务的 AI Agent。

### 核心特点

- **本地优先** — 作为 daemon 进程运行在你的机器上，数据不出本地
- **多平台接入** — 支持 50+ 消息平台作为输入/输出通道
- **自主执行** — 不仅回复消息，还能执行 shell 命令、文件管理、浏览器自动化、API 调用等
- **持久记忆** — 基于 Markdown 文件的记忆系统（SOUL.md / AGENTS.md），透明可审计
- **可扩展** — 通过 Skills/Tools 系统扩展能力
- **开源社区** — GitHub 350k+ Stars，活跃的社区贡献

### 技术栈

- **语言**: JavaScript/TypeScript (Node.js)
- **入口文件**: `openclaw.mjs`
- **记忆**: Markdown 文件 + SQLite 索引
- **架构模式**: Gateway → Agent Runner → Agentic Loop → Response Path

## 学习目录

```
.
├── README.md                  # 本文件
├── docs/
│   ├── architecture.md        # 架构总览
│   ├── changelog.md           # 版本更新追踪
│   └── references.md          # 参考资料链接
├── notes/
│   ├── 01-gateway.md          # Gateway 层笔记
│   ├── 02-agent-loop.md       # Agentic Loop 笔记
│   ├── 03-memory.md           # 记忆系统笔记
│   ├── 04-tools-skills.md     # Tools/Skills 系统笔记
│   ├── 05-channels.md         # 消息通道笔记
│   └── 06-security.md         # 安全机制笔记
└── experiments/               # 实验代码/配置
    └── .gitkeep
```

## 学习路线

1. **阶段一**: 理解整体架构（四层模型）
2. **阶段二**: 深入 Gateway 和 Agent Loop 核心流程
3. **阶段三**: 研究 Memory/SOUL 系统设计
4. **阶段四**: 了解 Tools/Skills 扩展机制
5. **阶段五**: 追踪最新版本更新与社区动态

## 相关链接

- 源码仓库: https://github.com/openclaw/openclaw
- 官方文档: https://docs.openclaw.ai/
- 中文版: https://github.com/jiulingyun/openclaw-cn
