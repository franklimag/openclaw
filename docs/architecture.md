# OpenClaw 架构总览

> 基于公开资料整理，持续更新中。

## 一、整体架构（四层模型）

OpenClaw 采用清晰的分层架构，每层职责明确、松耦合：

```
┌─────────────────────────────────────────────────────────┐
│                   Input Channels (输入层)                 │
│  WhatsApp / Telegram / Slack / Discord / Signal / ...    │
└───────────────────────────┬─────────────────────────────┘
                            │ 消息进入
                            ▼
┌─────────────────────────────────────────────────────────┐
│                    Gateway (网关层)                       │
│  路由 / 会话管理 / 队列 / Channel Adapters               │
└───────────────────────────┬─────────────────────────────┘
                            │ 分发到 Agent
                            ▼
┌─────────────────────────────────────────────────────────┐
│                Agent Runner (推理层)                      │
│  上下文组装 / 模型选择 / Agentic Loop / Tool Execution   │
└───────────────────────────┬─────────────────────────────┘
                            │ 执行结果
                            ▼
┌─────────────────────────────────────────────────────────┐
│              Execution Environment (执行层)               │
│  Shell / 浏览器 / 文件系统 / API / Canvas                │
└─────────────────────────────────────────────────────────┘
```

## 二、核心组件详解

### 2.1 Gateway（网关）

Gateway 是 OpenClaw 的控制平面，负责：

- **Channel Adapters**: 为每个消息平台提供适配器，统一消息格式
- **Session 管理**: 每个对话维持独立会话，隔离状态
- **Lane Queue（队列）**: 默认每个 session 串行执行，防止共享状态的竞态条件；并行执行为低风险任务可选
- **路由**: 将消息分配给正确的 Agent 实例
- **Hook 点**: 提供 Gateway hooks 和 Plugin hooks 用于拦截/扩展

### 2.2 Agent Runner（Agent 运行时）

Agent Runner 是核心推理引擎：

- **上下文组装**: 收集 system prompt、记忆文件、对话历史、工具定义
- **模型调用**: 支持多种 LLM 后端
- **Prompt 组装**: 系统提示词 + SOUL.md + AGENTS.md + 工具列表
- **流式输出**: 支持 streaming 和 partial replies

### 2.3 Agentic Loop（智能循环）

这是 OpenClaw 与普通聊天机器人的核心区别：

```
┌──────────────────────────────────────┐
│          Agentic Loop                │
│                                      │
│  ┌─────────┐    ┌──────────────┐    │
│  │ Observe │───▶│    Plan      │    │
│  └─────────┘    └──────┬───────┘    │
│       ▲                 │            │
│       │                 ▼            │
│  ┌────┴────┐    ┌──────────────┐    │
│  │ Result  │◀───│     Act      │    │
│  └─────────┘    └──────────────┘    │
│                                      │
│  循环直到任务完成或达到限制            │
└──────────────────────────────────────┘
```

- **Observe**: 观察当前状态和上下文
- **Plan**: 推理下一步行动
- **Act**: 执行工具调用
- **Result**: 收集执行结果，反馈给下一轮循环
- 自主链式调用工具，无需每步用户提示
- 支持 compaction（压缩长上下文）和 retries

### 2.4 Memory 系统（记忆）

OpenClaw 的记忆系统是 **Markdown-first** 设计：

| 文件 | 用途 |
|------|------|
| `SOUL.md` | 人格、边界、语气定义 |
| `AGENTS.md` | 操作指令 + 工作记忆 |
| `TOOLS.md` | 用户维护的工具说明 |
| `BOOTSTRAP.md` | 首次运行引导（完成后删除） |
| `IDENTITY.md` | 身份定义 |

特点：
- 透明可审计（纯文本，非不透明的二进制）
- SQLite 索引加速检索
- 滚动记忆（对话 transcript）+ 持久记忆（Markdown 笔记）

### 2.5 Tools / Skills 系统

- **内置工具**: Shell 执行、文件操作、浏览器控制、Canvas 绘图
- **Skills**: 可扩展的能力模块，按需加载
- **权限控制**: 工具调用有权限管理，可选沙箱隔离
- **MCP 支持**: 兼容 Model Context Protocol

### 2.6 Channels（消息通道）

支持 50+ 消息平台，主要包括：
- 即时通讯: WhatsApp, Telegram, Signal, iMessage
- 协作工具: Slack, Discord, Microsoft Teams
- 邮箱/日历: Email, Calendar
- 任务管理: 各类 task management 平台
- CRM 系统集成

## 三、数据流（完整消息生命周期）

```
用户在 Telegram 发送消息
    │
    ▼
Channel Adapter 接收 → 标准化消息格式
    │
    ▼
Gateway 路由 → 找到/创建 Session
    │
    ▼
放入 Lane Queue → 等待前序任务完成（串行）
    │
    ▼
Agent Runner 激活 → 组装 Prompt（SOUL + Memory + History + Tools）
    │
    ▼
调用 LLM → 获得响应（可能包含 tool_call）
    │
    ▼
进入 Agentic Loop:
    ├── 执行 Tool Call（如 shell 命令）
    ├── 收集结果
    ├── 再次调用 LLM
    └── 循环直到完成
    │
    ▼
生成最终回复 → 通过 Gateway 回传
    │
    ▼
Channel Adapter 格式化 → 发送到 Telegram
```

## 四、关键设计原则

1. **Local-first**: 数据和计算都在本地，隐私优先
2. **Daemon 模式**: 持久运行，非请求-响应模式
3. **Markdown-first Memory**: 记忆透明、可编辑、可版本控制
4. **Observe-Plan-Act Loop**: 序列化的推理循环，避免并发问题
5. **轻量安全框架**: 相比商业产品更少限制，便于实验
6. **通道无关**: 核心逻辑与消息平台解耦

## 五、与其他项目对比

| 维度 | OpenClaw | 传统 Chatbot | 商业 AI 助手 |
|------|----------|-------------|-------------|
| 部署 | 自托管 | 云端 | 云端 |
| 记忆 | 持久 Markdown | 上下文窗口 | 有限历史 |
| 执行 | 自主 Agent Loop | 单轮问答 | 受限工具 |
| 隐私 | 完全本地 | 数据上传 | 供应商控制 |
| 扩展 | 开源 Skills | 固定功能 | API 限制 |
| 平台 | 50+ 通道 | 单一入口 | 单一入口 |

## 六、待深入研究

- [ ] Gateway 内部的 hook 机制具体实现
- [ ] Agentic Loop 的 compaction 策略
- [ ] Memory 的 activation/decay 系统
- [ ] 多 Agent 协作模式
- [ ] 安全沙箱的隔离级别
- [ ] Heartbeat daemon 和定时任务机制

---

*参考来源见 [references.md](./references.md)*
