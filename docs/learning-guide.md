# OpenClaw 学习计划指南

> 系统化的学习路径，从理解架构到动手部署，按依赖关系编排。

## 使用说明

本指南提供两条学习路线：
- **路线 A: 架构理解** — 侧重理解 OpenClaw 的设计思想和组件关系
- **路线 B: 实践部署** — 侧重动手搭建和配置，最终跑通飞书多 Agent 团队

两条路线的前三个阶段相同，之后分叉。建议**先完成路线 A 再进入路线 B**。

---

## 笔记依赖关系图

```
docs/architecture.md (总览，入口)
        │
        ├── notes/01-gateway.md (Gateway 控制平面)
        │       │
        │       └── notes/05-channels.md (通道，依赖 Gateway)
        │
        ├── notes/02-agent-loop.md (Agentic Loop，核心推理)
        │       │
        │       └── notes/04-tools-skills.md (Tools/Skills，被 Loop 调用)
        │
        ├── notes/03-memory.md (Memory，被所有组件使用)
        │
        └── notes/06-security.md (安全，横切所有组件)

notes/07-team-setup.md (实践：多 Agent 团队 + 飞书)
        └── 依赖以上所有笔记
```

## 推荐阅读顺序

| 顺序 | 文件 | 原因 |
|------|------|------|
| 1 | `docs/architecture.md` | 全局视角，建立心智模型 |
| 2 | `notes/01-gateway.md` | 系统入口，理解消息如何进入 |
| 3 | `notes/02-agent-loop.md` | 核心引擎，理解推理如何发生 |
| 4 | `notes/03-memory.md` | 状态层，理解 Agent 如何"记住" |
| 5 | `notes/04-tools-skills.md` | 能力层，理解 Agent 如何"行动" |
| 6 | `notes/05-channels.md` | 接入层，理解平台适配 |
| 7 | `notes/06-security.md` | 横切面，理解风险和防护 |
| 8 | `notes/07-team-setup.md` | 综合实践 |
| 参考 | `docs/changelog.md` | 随时查阅版本变化 |
| 参考 | `docs/references.md` | 深入某话题时查阅原始资料 |

---

## 路线 A: 架构理解（预计 3-5 天）

### 阶段 1: 建立全局视角（Day 1）

**目标**: 理解 OpenClaw 是什么、解决什么问题、整体分几层。

| 任务 | 阅读材料 | 完成标准 |
|------|----------|----------|
| 了解 OpenClaw 定位 | `README.md` | 能用一句话描述 OpenClaw |
| 理解四层架构 | `docs/architecture.md` 第一~二节 | 能画出四层架构图 |
| 理解数据流 | `docs/architecture.md` 第三节 | 能描述一条消息的完整生命周期 |

**验证**: 能向他人解释 "一条 Telegram 消息从发送到收到回复经历了哪些步骤"。

### 阶段 2: 理解核心引擎（Day 2）

**目标**: 深入 Gateway 和 Agentic Loop，理解"控制平面"和"推理循环"。

| 任务 | 阅读材料 | 完成标准 |
|------|----------|----------|
| Gateway 职责 | `notes/01-gateway.md` | 理解 8 个核心职责 |
| Lane Queue 串行模型 | `notes/01-gateway.md` 第 3 节 | 理解为什么默认串行 |
| Observe-Plan-Act 循环 | `notes/02-agent-loop.md` | 能解释 ReAct 模式在 OpenClaw 中的实现 |
| Runtime Fallbacks | `notes/02-agent-loop.md` | 理解模型失败时的恢复机制 |

**验证**: 能解释 "如果 GPT-5 调用失败，OpenClaw 如何保证任务不中断"。

### 阶段 3: 理解状态与能力（Day 3）

**目标**: 理解 Agent 如何"记住"和如何"行动"。

| 任务 | 阅读材料 | 完成标准 |
|------|----------|----------|
| Memory 文件体系 | `notes/03-memory.md` 前半部分 | 知道 SOUL.md/AGENTS.md 的作用 |
| Embedding 与 memorySearch | `notes/03-memory.md` Embedding 章节 | 理解是否需要配置 embedding |
| Skills/Plugins 机制 | `notes/04-tools-skills.md` | 理解 SKILL.md 格式和 ClawHub |
| SKILL.md 安全约束 | `notes/04-tools-skills.md` 安全考量 | 知道不能在 SKILL.md 中硬编码密钥 |

**验证**: 能解释 "OpenClaw 的记忆和 ChatGPT 的记忆有什么本质区别"。

### 阶段 4: 理解生态与安全（Day 4-5）

**目标**: 了解平台接入、安全风险、版本演进。

| 任务 | 阅读材料 | 完成标准 |
|------|----------|----------|
| 通道分类与语音 | `notes/05-channels.md` | 了解 50+ 平台支持和语音能力 |
| 攻击面分析 | `notes/06-security.md` | 知道三大攻击面和主要安全事件 |
| 版本演进 | `docs/changelog.md` | 了解从 v2026.5.12 到 5.22 的关键变化 |
| 性能优化 | `notes/01-gateway.md` 性能优化节 | 理解 Gateway 4,100x 提速的原理 |

**验证**: 能列出 "部署 OpenClaw 时必须注意的 3 个安全风险"。

---

## 路线 B: 实践部署（预计 5-7 天，在路线 A 之后）

### 阶段 5: 单 Agent + 飞书（Day 1-2）

**目标**: 跑通一个最简单的 OpenClaw 实例 + 飞书连接。

| 任务 | 参考 | 完成标准 |
|------|------|----------|
| 安装 OpenClaw | `notes/07-team-setup.md` 第七节 Step 1 | Gateway 正常启动 |
| 配置模型 | `docs/architecture.md` CLI 命令节 | `openclaw models list` 有输出 |
| 创建飞书应用 | `notes/07-team-setup.md` 第五节 5.3 | 获得 App ID + Secret |
| 连接飞书 | `notes/07-team-setup.md` 第五节 5.4 | 飞书消息能收到回复 |

**验证**: 在飞书中向机器人发消息，收到 AI 回复。

### 阶段 6: 配置记忆与 Embedding（Day 3）

**目标**: 让 Agent 具备持久记忆和语义检索能力。

| 任务 | 参考 | 完成标准 |
|------|------|----------|
| 编写 SOUL.md | `notes/03-memory.md` 文件体系节 | Agent 有自定义人格 |
| 安装 Ollama embedding | `notes/03-memory.md` Embedding 章节 | memorySearch 可用 |
| 测试记忆检索 | — | 存入信息后能语义搜索到 |

**验证**: 告诉 Agent "我的生日是 3 月 15 日"，隔天问"我的生日是什么时候"能正确回答。

### 阶段 7: 多 Agent 团队搭建（Day 4-5）

**目标**: 搭建多角色 Agent 团队，配置路由和协作。

| 任务 | 参考 | 完成标准 |
|------|------|----------|
| 创建多 Agent 目录 | `notes/07-team-setup.md` Step 2-3 | 每个角色有独立 agentDir |
| 编写各角色 SOUL.md | `notes/07-team-setup.md` 第二节 | PM/Arch/Dev 各有人格 |
| 配置飞书路由 | `notes/07-team-setup.md` 第五节 5.5 | @不同角色触达不同 Agent |
| 创建共享文档目录 | `notes/07-team-setup.md` 第四节 | shared-workspace 就绪 |

**验证**: 在飞书中 @PM 和 @开发 分别收到不同角色的回复。

### 阶段 8: 协作流程验证（Day 6-7）

**目标**: 验证完整的任务流转流程。

| 任务 | 参考 | 完成标准 |
|------|------|----------|
| 配置 Chronix 定时任务 | `notes/07-team-setup.md` 第六节 6.2 | 每日简报自动推送 |
| 测试需求流转 | `notes/07-team-setup.md` 第三节 3.2 | 新需求 → PM 拆解 → 分发 |
| 安全加固 | `notes/06-security.md` 最佳实践 | Tailscale / env vars |
| Meeting Notes 配置 | `notes/04-tools-skills.md` | 会议记录自动生成 |

**验证**: 发送"新需求: 开发用户登录"，观察 PM → Arch → Dev 的自动流转。

---

## 难度与时间预估

| 阶段 | 难度 | 预计时间 | 前置要求 |
|------|------|----------|----------|
| 1 全局视角 | ⭐ | 2-3 小时 | 无 |
| 2 核心引擎 | ⭐⭐ | 3-4 小时 | 阶段 1 |
| 3 状态与能力 | ⭐⭐ | 3-4 小时 | 阶段 1 |
| 4 生态与安全 | ⭐⭐ | 2-3 小时 | 阶段 1-3 |
| 5 单 Agent 部署 | ⭐⭐⭐ | 4-8 小时 | 阶段 1-4 + Node.js 环境 |
| 6 记忆配置 | ⭐⭐⭐ | 2-4 小时 | 阶段 5 |
| 7 多 Agent 团队 | ⭐⭐⭐⭐ | 6-10 小时 | 阶段 5-6 |
| 8 协作验证 | ⭐⭐⭐⭐ | 4-6 小时 | 阶段 7 |

**总计**: 架构理解约 12-15 小时 | 实践部署约 16-28 小时

---

## 学习过程中的关键决策点

在学习过程中你会遇到以下需要做决定的节点：

| 决策点 | 时机 | 选项 | 建议 |
|--------|------|------|------|
| LLM 提供商选择 | 阶段 5 | OpenAI / Anthropic / xAI / 本地 | 先用 OpenAI Codex（官方默认） |
| Embedding 方案 | 阶段 6 | 零配置 / Ollama / QMD | 个人用 Ollama，团队用 QMD |
| 飞书路由方案 | 阶段 7 | 多机器人 / 单机器人@路由 | 初学用单机器人，生产用多机器人 |
| 安全级别 | 阶段 8 | 最小限制 / 中等 / 严格 | 学习时最小限制，部署时严格 |

---

## 进度追踪模板

复制以下模板到你的笔记中追踪学习进度：

```markdown
## 我的学习进度

### 路线 A: 架构理解
- [ ] 阶段 1: 全局视角 (开始日期: ___)
  - [ ] 读完 README.md
  - [ ] 读完 architecture.md
  - [ ] 能画出四层架构图
- [ ] 阶段 2: 核心引擎 (开始日期: ___)
  - [ ] 读完 01-gateway.md
  - [ ] 读完 02-agent-loop.md
  - [ ] 理解 ReAct 循环
- [ ] 阶段 3: 状态与能力 (开始日期: ___)
  - [ ] 读完 03-memory.md
  - [ ] 读完 04-tools-skills.md
  - [ ] 理解 Embedding 选择
- [ ] 阶段 4: 生态与安全 (开始日期: ___)
  - [ ] 读完 05-channels.md
  - [ ] 读完 06-security.md
  - [ ] 了解版本演进

### 路线 B: 实践部署
- [ ] 阶段 5: 单 Agent + 飞书
- [ ] 阶段 6: 记忆与 Embedding
- [ ] 阶段 7: 多 Agent 团队
- [ ] 阶段 8: 协作验证
```

---

## 补充资源

### 每个阶段的推荐外部阅读

| 阶段 | 推荐外部资源 |
|------|-------------|
| 1 | [Architecture & Alternatives (turingpost)](https://www.turingpost.com/p/openclaw) |
| 2 | [Agent Control Loops (berkustun)](https://www.berkustun.com/notes/agent-control-loops/) |
| 3 | [Advanced Memory Management (lumadock)](https://lumadock.com/tutorials/openclaw-advanced-memory-management) |
| 4 | [Skills Explained (macaron.im)](https://macaron.im/blog/openclaw-skills-explained) |
| 5 | [Setup Guide (wayin.ai)](https://wayin.ai/blog/openclaw-setup-guide/) |
| 7 | [Multi-Agent Setup (lumadock)](https://lumadock.com/tutorials/openclaw-multi-agent-setup) |

### 官方文档入口

- 源码: https://github.com/openclaw/openclaw
- 文档: https://docs.openclaw.ai/
- 架构: https://docs.openclaw.ai/concepts/architecture
- Gateway: https://docs.openclaw.ai/concepts/gateway
- Memory: https://docs.openclaw.ai/concepts/memory
- Routing: https://docs.openclaw.ai/concepts/routing

---

*最后更新: 2026-05-26*
