# OpenClaw 架构总览

> 基于官方文档 (docs.openclaw.ai) 和公开资料整理。最后更新: 2026-05-20 (v2026.5.12)

## 零、项目基本信息

| 属性 | 值 |
|------|-----|
| 仓库 | https://github.com/openclaw/openclaw |
| 官方文档 | https://docs.openclaw.ai/ |
| 入口文件 | `openclaw.mjs` |
| 语言 | JavaScript/TypeScript (Node.js) |
| 运行要求 | Node 24 (推荐) / Node 22.14+ (最低) |
| 安装方式 | `openclaw onboard --install-daemon` |
| Gateway 端口 | 18789 |
| 当前版本 | **2026.5.12** (稳定) |
| GitHub Stars | 350k+ |
| 历史名称 | Moltbot → Clawdbot → OpenClaw |

## 一、整体架构

OpenClaw 是一个 **local-first** 的自托管 AI Agent 框架。它作为 daemon 进程运行在用户设备上，通过 Gateway 将消息通道连接到 AI Agent 运行时。

### 架构层次图

```
┌─────────────────────────────────────────────────────────────┐
│                    Channels (通道层)                          │
│  WhatsApp / Telegram / Slack / Discord / Signal / iMessage   │
│  Google Chat / Matrix / MS Teams / Zalo / Email / ...        │
│  (内置通道 + 通道插件, 50+ 平台)                              │
└───────────────────────────┬─────────────────────────────────┘
                            │ WebSocket / Webhook
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                   Gateway (网关/控制平面)                     │
│  WebSocket Server (port 18789)                               │
│  Channel Adapters / Session Mgmt / Lane Queue / Hooks        │
│  Cron (Chronix) / Webhook Router / Plugin System             │
└───────────────────────────┬─────────────────────────────────┘
                            │ 分发到 Agent Runtime
                            ▼
┌─────────────────────────────────────────────────────────────┐
│               Agent Runtime (Pi Agent Core)                   │
│  上下文组装 / 多模型调度 / Agentic Loop / Tool Execution      │
│  Compaction / Streaming / Session Transcript                  │
│  支持模型: GPT-5.x, Opus 4.6, Grok, Codex 等               │
└───────────────────────────┬─────────────────────────────────┘
                            │ 工具调用
                            ▼
┌─────────────────────────────────────────────────────────────┐
│              Execution Environment (执行层)                    │
│  Shell / Agent-Browser / 文件系统 / API / Canvas              │
│  Skills (ClawHub 市场) / MCP Servers / 定时任务              │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│              Memory & Knowledge (记忆层)                      │
│  SOUL.md / AGENTS.md / TOOLS.md / IDENTITY.md                │
│  SQLite Index / Append-only Transcripts / Activation-Decay   │
└─────────────────────────────────────────────────────────────┘
```

## 二、核心组件详解

### 2.1 Gateway（网关 / 控制平面）

Gateway 是 OpenClaw 的核心 — 一个长期运行的 **WebSocket 服务器**，充当所有消息通道、AI Agent 和平台应用的中央控制平面。

#### 核心职责

| 功能 | 说明 |
|------|------|
| Channel Adapters | 为每个消息平台提供适配器（内置 + 插件），统一消息格式 |
| Session 管理 | 每个对话维持独立会话，append-only transcript + mutable index |
| Lane Queue | 默认串行执行（同 session 内），防止竞态；并行为可选 |
| 路由 | 消息分配给正确的 Agent 实例 |
| Hooks | Gateway hooks + Plugin hooks，用于拦截/扩展 |
| Cron / Scheduling | Chronix 调度引擎，支持定时任务、心跳 |
| Webhook Router | 接收外部 webhook 并路由到对应 handler |
| Plugin System | 支持外部 channel 插件和 tool 插件 |

#### 技术细节

- 监听端口: **18789**
- 协议: WebSocket (通道通信) + HTTP (webhook / health check)
- 进程模型: 单进程长驻 daemon
- 会话存储: append-only transcript + mutable index (支持 compaction 和 pruning)

### 2.2 Agent Runtime（Pi Agent Core）

Agent Runtime 是核心推理引擎，在 OpenClaw 中称为 **Pi Agent Core**：

- **上下文组装**: 收集 system prompt、SOUL.md、AGENTS.md、对话历史、工具定义
- **多模型调度**: 支持多个 LLM 提供商，可配置主模型和 fallback
- **Codex 集成** (2026.5+): 默认使用 OpenAI Codex app-server runtime 管理 agent turns
- **流式输出**: streaming + partial replies
- **Runtime Fallbacks**: 模型调用失败时自动切换到备选提供商
- **Stalled-stream Recovery**: 流处理中断时自动恢复，不丢失数据

#### 支持的模型提供商

| 提供商 | 模型示例 | 认证方式 |
|--------|----------|----------|
| OpenAI | GPT-5.5, GPT-5.4, GPT-5.3 Codex | openai-codex OAuth / API Key |
| Anthropic | Opus 4.6, Claude | API Key |
| xAI | Grok (支持 /think 推理) | API Key |
| Google | Gemini | API Key |
| 其他 | 通过 provider surface 扩展 | 各提供商方式 |

#### 模型配置

```bash
# 认证模型提供商
openclaw models auth login --provider openai-codex

# 查看可用模型
openclaw models list

# 设置默认模型
openclaw models set openai/gpt-5.4
```

### 2.3 Agentic Loop（智能循环）

这是 OpenClaw 与普通聊天机器人的核心区别 — Observe-Plan-Act 循环：

```
┌──────────────────────────────────────────────────┐
│              Agentic Loop                        │
│                                                  │
│  ┌──────────┐    ┌───────────┐    ┌─────────┐  │
│  │ Observe  │───▶│   Plan    │───▶│   Act   │  │
│  │ (上下文)  │    │ (LLM推理) │    │ (工具)   │  │
│  └──────────┘    └───────────┘    └────┬────┘  │
│       ▲                                 │       │
│       │                                 ▼       │
│  ┌────┴──────────────────────────────────────┐  │
│  │              Result (结果收集)              │  │
│  └───────────────────────────────────────────┘  │
│                                                  │
│  循环直到: 任务完成 / 达到迭代限制 / 用户中断      │
└──────────────────────────────────────────────────┘
```

#### 循环控制

- **自主链式调用**: 无需每步用户确认
- **Compaction**: 长上下文自动压缩，保留关键信息
- **Retries**: 工具调用失败自动重试
- **Stalled-stream Recovery**: 网络中断或 API 限速时自动恢复
- **终止条件**: 任务完成 / 迭代上限 / 超时 / 用户中断

### 2.4 Memory 系统（记忆）

OpenClaw 的记忆系统是 **Markdown-first** 设计，透明可审计：

| 文件 | 用途 | 生命周期 |
|------|------|----------|
| `SOUL.md` | 人格定义、边界、语气 | 持久 |
| `AGENTS.md` | 操作指令 + 工作记忆 | 持久（动态更新） |
| `TOOLS.md` | 用户维护的工具使用说明 | 持久 |
| `IDENTITY.md` | 身份定义 | 持久 |
| `BOOTSTRAP.md` | 首次运行引导脚本 | 一次性（完成后删除） |

#### 记忆层次

1. **Session Transcript** — append-only 对话记录，支持 compaction & pruning
2. **Durable Memory** — 持久 Markdown 文件，跨 session 生存
3. **SQLite Index** — 加速语义检索
4. **Activation/Decay** — 记忆活跃度系统，频繁使用的记忆优先召回

### 2.5 Tools / Skills 系统

#### Skills 架构

Skills 是 OpenClaw 的能力扩展单元，使用 **AgentSkills** 兼容格式：

```
skills/
└── <author>/
    └── <skill-name>/
        └── SKILL.md          # YAML frontmatter + 使用说明
```

- **加载机制**: 内置 bundled skills + 本地 overrides，按环境/配置过滤
- **ClawHub 市场**: 公共 Skills 注册表 (1200+ 安全扫描的 Skills)
- **安全分级**: A-F 等级评分
- **MCP 兼容**: 支持 Model Context Protocol servers 作为工具源

#### 内置工具 (2026.5)

| 工具 | 说明 |
|------|------|
| Shell | 系统命令执行 |
| Agent-Browser | 无头浏览器自动化 (现为默认集成) |
| File System | 文件读写操作 |
| Canvas | 可视化画布 |
| API / HTTP | 外部 API 调用 |
| Chronix | 定时任务 / cron 调度 |
| MCP Tools | Model Context Protocol 工具 |

#### 插件系统 (2026.5.16+)

```bash
# 创建插件
openclaw plugins init my-plugin

# 构建和验证
openclaw plugins build
openclaw plugins validate

# 发布到 ClawHub
openclaw plugins publish
```

使用 `defineToolPlugin` API 创建 typed tool plugins。

### 2.6 Channels（消息通道）

OpenClaw 支持 **内置通道 + 通道插件**，覆盖 50+ 平台：

| 类别 | 平台 |
|------|------|
| 即时通讯 | WhatsApp, Telegram, Signal, iMessage (via BlueBubbles) |
| 协作工具 | Slack, Discord, Microsoft Teams, Google Chat |
| 开源协议 | Matrix |
| 区域平台 | Zalo, 飞书 (中文版) |
| 邮箱/日历 | Email (SMTP/IMAP), Calendar |
| 任务管理 | 各类 task management |
| CRM | 客户关系管理平台 |
| Web UI | 内置 Web 界面 |
| 语音 | Android 实时语音 (2026.5.18+, 基于 gateway relay) |

#### 通道分类

- **内置通道 (Built-in)**: 核心支持，随 OpenClaw 一起安装
- **通道插件 (Channel Plugins)**: 社区贡献，通过 ClawHub 安装

## 三、数据流（完整消息生命周期）

```
用户在 Telegram 发送消息
    │
    ▼
Channel Adapter 接收 (Webhook/WebSocket)
    │ 协议转换 → 标准化内部消息格式
    ▼
Gateway (port 18789) 路由
    │ 找到/创建 Session
    │ 放入 Lane Queue (串行)
    ▼
Agent Runtime (Pi Agent Core) 激活
    │ 组装上下文: SOUL + Memory + History + Tools + Skills
    │ 选择模型 (主模型 + fallback 链)
    ▼
调用 LLM (e.g., GPT-5.4 via Codex runtime)
    │ 获得响应 (可能包含 tool_call)
    ▼
Agentic Loop:
    ├── 执行 Tool Call (shell / browser / file / API)
    ├── 收集结果 → 反馈给 LLM
    ├── LLM 决策: 继续 / 完成
    ├── [如流中断] Stalled-stream Recovery
    ├── [如失败] Runtime Fallback → 切换模型
    └── 循环直到完成
    │
    ▼
生成最终回复
    │ 更新 Session Transcript
    │ 更新 Memory (如需要)
    ▼
通过 Gateway → Channel Adapter 回传
    │ 格式化为平台特定格式
    ▼
用户在 Telegram 收到回复
```

## 四、关键设计原则

1. **Local-first**: 数据和计算都在本地，隐私优先
2. **Daemon 模式**: 持久运行的 Node.js 进程，非请求-响应模式
3. **Gateway 即控制平面**: 单一 WebSocket 服务器统管所有通道和 Agent
4. **Markdown-first Memory**: 记忆透明、可编辑、可版本控制
5. **Observe-Plan-Act Loop**: 序列化推理循环
6. **多模型 + Fallback**: 不绑定单一模型提供商
7. **通道无关**: 核心逻辑与消息平台完全解耦
8. **插件优先**: 通道、工具、Skills 均可通过插件扩展
9. **安全分层**: Exec Policy Engine + Container Boundary + 权限系统

## 五、CLI 命令概览

```bash
# 安装与初始化
openclaw onboard --install-daemon    # 首次安装
openclaw doctor --fix                # 诊断修复
openclaw status --all                # 全面状态检查

# 模型管理
openclaw models auth login           # 认证提供商
openclaw models list                 # 列出可用模型
openclaw models set <provider/model> # 设置默认模型

# Agent 管理
openclaw agents list                 # 列出 agents
openclaw sessions list               # 列出 sessions

# 插件管理
openclaw plugins init                # 创建插件
openclaw plugins build               # 构建插件
openclaw plugins validate            # 验证插件
openclaw plugins publish             # 发布到 ClawHub
```

## 六、与其他项目对比

| 维度 | OpenClaw | n8n | AutoGPT | 商业 AI 助手 |
|------|----------|-----|---------|-------------|
| 部署 | 自托管 (local daemon) | 自托管/云 | 云端 | 云端 |
| 核心模式 | Agent Loop | 工作流 | Agent Loop | 受限 Agent |
| 记忆 | 持久 Markdown | 无 | 向量DB | 有限历史 |
| 通道 | 50+ 消息平台 | Webhook/API | CLI/Web | 单一入口 |
| 模型 | 多模型 + Fallback | 可配置 | 固定 | 绑定供应商 |
| 扩展 | Skills + Plugins | Nodes | Plugins | API |
| 隐私 | 完全本地 | 可自托管 | 云端 | 供应商控制 |
| 社区 | 350k+ stars | 活跃 | 活跃 | 封闭 |

## 七、安全架构

### 攻击面 (根据学术研究 arxiv:2603.27517)

| 攻击面 | 风险等级 | 说明 |
|--------|----------|------|
| Exec Policy Engine | 最高 | 最主要的攻击入口 |
| Gateway WebSocket | 高 | 最宽的集成面 |
| Container Boundary | 高 | 沙箱逃逸风险 |
| Memory & Knowledge | 中 | 持久状态篡改 |
| LLM Provider | 中 | Prompt injection |

### 已知安全事件 (2026)

- **ClawJacked**: 高严重度漏洞，任何网站可静默接管 Agent（已修复，24h 内）
- **Claw Chain**: 四个链式漏洞（沙箱逃逸 + 凭证窃取 + 输入验证绕过 + 持久控制）
- **ClawHub 恶意包**: 公共 Skills 注册表被投毒
- **Prompt Injection**: 通过注入 BOOTSTRAP.md 实施隐蔽操纵

### 安全增强方向 (2026.5+)

- 更少 core 中的 magic，更清晰的插件边界
- 更好的扫描机制
- 更严格的发布卫生
- 改进的安全态势

## 八、待深入研究

- [ ] Codex app-server runtime 的具体工作机制
- [ ] Chronix 调度引擎的 cron 表达式格式
- [ ] defineToolPlugin API 的完整接口定义
- [ ] Gateway WebSocket 协议细节
- [ ] 多 Agent 协作模式 (Multi-Agent)
- [ ] ACP runtime options 与后端配置映射
- [ ] Token usage dashboard 的实现

---

*参考来源见 [references.md](./references.md)*
*最后更新: 2026-05-20*
