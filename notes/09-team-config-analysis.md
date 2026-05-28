# 真实团队配置分析报告

> 基于 `notes/08-session-mechanism.md` 中的源码级理解，对当前飞书 Agent 团队的实际配置进行分析。
> 分析日期: 2026-05-28

---

## 一、当前架构总评

### 1.1 架构模式

当前采用的是**「一 Agent 一飞书应用」**模式：每个角色对应一个独立的飞书企业应用，通过 `accountId` 路由到独立的 Agent。

```
飞书应用 "pm"        → binding(accountId="pm")       → Agent pm
飞书应用 "architect"  → binding(accountId="architect") → Agent architect
飞书应用 "dev-front"  → binding(accountId="dev-front") → Agent dev-front
飞书应用 "dev-backend" → binding(accountId="dev-backend") → Agent dev-backend
飞书应用 "reviewer"   → binding(accountId="reviewer")  → Agent reviewer
飞书应用 "assistant"  → binding(accountId="assistant") → Agent main
```

### 1.2 架构优势

| 优势 | 说明 |
|------|------|
| **完全隔离** | 每个 Agent 有独立 workspace、session store、auth profile，互不干扰 |
| **独立人格** | 每个 Agent 有自己的 SOUL.md、AGENTS.md，人格完全独立 |
| **独立记忆** | 每个 Agent 有自己的 memory/ 目录和 embedding 索引 |
| **飞书私聊直达** | 用户向不同飞书应用发消息，直接对话对应角色 |
| **用户体验清晰** | 从飞书用户视角，每个机器人就是一个独立的角色 |

### 1.3 架构隐患

| 隐患 | 原因 | 影响 |
|------|------|------|
| **缺少 Coordinator** | 没有明确的协调者角色编排整体工作流 | 任务流转需要用户手动在不同应用间切换 |
| **A2A 全开放** | `allow` 列表包含全部 Agent，无方向限制 | 任何 Agent 可向任何 Agent 发消息，缺乏管控 |
| **dmScope 未配置** | 使用默认 `"main"`，所有 DM 共享一个 session | 同一用户与 pm 的所有对话都在一个 session 中 |
| **无 session 维护配置** | 未配置 `session.maintenance` | 依赖默认值（500 条/30 天），可能不够积极 |
| **无 delegation mode** | 未配置 `subagents.delegationMode` | Agent 不会主动委派任务给子 agent |
| **无 queueMode 配置** | 未设置消息队列模式 | 默认 followup，忙碌时响应延迟 |
| **单一模型** | 全部 Agent 使用同一模型 deepseek-v4-flash | 复杂推理场景可能能力不足 |

---

## 二、Session 行为分析

### 2.1 当前 Session Key 生成分析

根据源码 `src/routing/resolve-route.ts` 的逻辑，当前配置下的 session key：

| 场景 | Session Key | 原因 |
|------|------------|------|
| 用户 DM → pm 应用 | `agent:pm:main` | dmScope="main"(默认)，所有 DM 合并 |
| 用户 DM → architect 应用 | `agent:architect:main` | 同上 |
| pm 通过 sessions_send 给 architect | `agent:architect:main` | 发到 architect 的 main session |
| 群聊中 @pm 应用 | `agent:pm:feishu:group:<群id>` | 群聊有独立 session |
| 另一个用户 DM → pm 应用 | `agent:pm:main` | **问题：与第一个用户共享 session！** |

### 2.2 关键问题：dmScope="main" 的影响

当前配置未设置 `session.dmScope`，默认为 `"main"`。这意味着：

```
用户 A → 飞书私聊 pm 应用 → session: agent:pm:main
用户 B → 飞书私聊 pm 应用 → session: agent:pm:main  ← 同一个 session！
```

**如果有多个用户使用，所有用户的私聊对话会混在同一个 session 中。**

如果当前只有一个用户使用，这暂时不是问题。但一旦多人使用，必须改为 `"per-peer"` 或 `"per-channel-peer"`。

### 2.3 A2A 通信时的 Session 行为

当 pm Agent 使用 `sessions_send(agentId="architect", message="...")` 时：
1. Gateway 将 architect 的 `agent:architect:main` session 作为目标
2. 消息带 `[Inter-session message]` 前缀注入到 architect 的 main session
3. architect 处理后，通过 A2A announce 回传到 pm 的 session

**这就是你之前提到的 session 混杂问题**：pm 的 `agent:pm:main` session 中会同时包含用户对话和 architect 的回报。

---

## 三、团队协作流程分析

### 3.1 当前工作流（推测）

```
用户 → 飞书私聊 pm → pm Agent 分析需求
用户 → 手动切换到 architect 飞书应用 → architect 设计方案
用户 → 手动切换到 dev-front 飞书应用 → dev-front 实现前端
用户 → 手动切换到 reviewer 飞书应用 → reviewer 审查代码
```

**问题**：用户需要手动充当协调者角色，在各个飞书应用间切换传递信息。

### 3.2 理想工作流（基于源码能力）

```
用户 → 飞书私聊 pm → pm Agent 分析需求
  pm → sessions_spawn(taskName="arch_review") → 派发给 architect
  pm → sessions_yield() → 等待结果
  architect 完成 → A2A announce 回传给 pm
  pm → sessions_spawn(taskName="dev_impl") → 派发给 dev-front
  ...
  pm → 向用户汇报最终结果
```

### 3.3 当前配置缺失的关键能力

| 能力 | 当前状态 | 需要的配置 |
|------|----------|-----------|
| PM 自动委派 | 未配置 | `agents.defaults.subagents.delegationMode: "prefer"` |
| Spawn 深度 | 未配置 | `agents.defaults.subagents.maxSpawnDepth: 2` |
| Subagent 工具 | 未明确开放 | `tools.subagents.tools.alsoAllow: ["sessions_send"]` |
| Session 隔离 | dmScope=main | `session.dmScope: "per-channel-peer"` |
| 队列模式 | 默认 followup | PM 配置 `queueMode: "steer"` |
| 维护策略 | 默认值 | `session.maintenance: { pruneAfter: "7d", maxEntries: 200 }` |

---

## 四、安全性分析

### 4.1 凭据暴露风险

配置中包含 6 个飞书应用的 `appId` + `appSecret` 明文。

**建议**：
- 使用环境变量: `appSecret: "${FEISHU_PM_SECRET}"`
- 或使用 OpenClaw 的 secrets 管理功能

### 4.2 A2A 通信权限过宽

当前 `tools.agentToAgent.allow` 列表是全量开放：

```json
["main", "pm", "architect", "dev-front", "dev-backend", "reviewer"]
```

这意味着 `dev-front` 可以向 `reviewer` 发消息，`reviewer` 也可以向 `dev-backend` 下指令。在团队中这可能不是期望的行为。

**建议**：改为方向性控制：

```json5
{
  tools: {
    agentToAgent: {
      enabled: true,
      allow: [
        { from: "pm", to: "*" },           // PM 可以给所有人派发
        { from: "architect", to: "pm" },    // architect 只向 PM 汇报
        { from: "dev-front", to: "pm" },    // dev 只向 PM 汇报
        { from: "dev-backend", to: "pm" },
        { from: "reviewer", to: "pm" },
        { from: "reviewer", to: "dev-front" },  // reviewer 可向 dev 反馈
        { from: "reviewer", to: "dev-backend" }
      ]
    }
  }
}
```

> 注：需要确认 OpenClaw 当前版本是否支持 from/to 方向性配置。源码中 `createAgentToAgentPolicy` 的 `isAllowed(from, to)` 方法表明支持方向性控制。

### 4.3 工具权限未限制

未配置 `tools.subagents.tools` 的 allow/deny，子 agent 会继承父 agent 的全部工具权限。对于 dev agent 这可能没问题，但 reviewer 不应该有写文件的能力。

---

## 五、模型配置分析

### 5.1 当前模型配置

```json5
{
  model: {
    primary: "deepseek/deepseek-v4-flash",
    fallbacks: ["deepseek/deepseek-v4-flash"]
  }
}
```

**问题**：
1. **Fallback 无意义**：fallback 和 primary 是同一个模型，无法提供真正的容灾
2. **单一模型**：所有角色使用相同模型，无法根据角色需求差异化
3. **Flash 模型局限**：flash 模型适合快速响应，但架构设计和代码审查可能需要更强推理能力

### 5.2 建议的模型分层

| Agent | 推荐模型 | 原因 |
|-------|----------|------|
| pm | deepseek-v4-flash | 快速响应用户，任务分解不需要极深推理 |
| architect | deepseek-v4 / claude-opus | 架构设计需要深度推理 |
| dev-front / dev-backend | deepseek-v4-flash | 代码生成 flash 足够 |
| reviewer | deepseek-v4 | 代码审查需要更强理解力 |

### 5.3 每个 Agent 可单独配置模型

```json5
{
  agents: {
    list: [
      {
        id: "architect",
        workspace: "...",
        model: {
          primary: "deepseek/deepseek-v4",
          fallbacks: ["deepseek/deepseek-v4-flash"]
        }
      }
    ]
  }
}
```

---

## 六、Workspace 结构分析

### 6.1 各 Agent Workspace 职责对照

| Agent | SOUL.md 应包含 | AGENTS.md 应包含 |
|-------|---------------|-----------------|
| pm | 需求分析师人格，沟通风格 | 需求分析流程，任务拆解模板，团队成员清单 |
| architect | 架构师人格，严谨思维 | 架构评审流程，设计文档规范，技术选型原则 |
| dev-front | 前端开发者人格 | 前端技术栈，代码规范，组件设计约定 |
| dev-backend | 后端开发者人格 | 后端技术栈，API 设计规范，数据库约定 |
| reviewer | 审慎的审查者人格 | 代码审查清单，质量标准，反馈格式 |

### 6.2 缺少共享上下文

当前各 Agent workspace 完全独立。但团队协作需要共享：
- 项目需求文档
- 架构设计结论
- 代码仓库信息
- 团队约定

**解决方案**：

方案 A: 每个 workspace 中放置相同的项目上下文文件（维护成本高）

方案 B: 使用 symlink 指向共享目录：
```
workspace-pm/project-context.md → /shared/project-context.md
workspace-architect/project-context.md → /shared/project-context.md
```

方案 C: 利用 `cwd` 向上遍历机制：
```
/shared-workspace/AGENTS.md          ← 项目级共享指令
/shared-workspace/workspace-pm/      ← pm 特有文件
/shared-workspace/workspace-architect/ ← architect 特有文件
```

但这需要修改 workspace 路径结构。当前平铺的 `~/.openclaw/workspace-*` 布局无法利用目录遍历机制共享上下文。

---

## 七、改进建议（按优先级排序）

### P0 - 立即修复

| 项目 | 当前 | 建议 | 原因 |
|------|------|------|------|
| dmScope | 未配置(=main) | `"per-channel-peer"` | 多用户时 session 会混杂 |
| 凭据明文 | appSecret 明文 | 环境变量或 secrets | 安全风险 |
| Fallback 模型 | 与 primary 相同 | 配置不同的备用模型 | 无容灾效果 |

### P1 - 实现协调者模式

| 项目 | 配置 |
|------|------|
| PM 作为 coordinator | `pm` 的 SOUL.md 中明确协调者角色 |
| 开启 spawn 能力 | `agents.defaults.subagents.maxSpawnDepth: 2` |
| PM delegation mode | `agents.defaults.subagents.delegationMode: "prefer"` |
| sessions_send 权限 | `tools.subagents.tools.alsoAllow: ["sessions_send"]` |

### P2 - 优化运行时行为

| 项目 | 配置 |
|------|------|
| Session 维护 | `session.maintenance: { pruneAfter: "7d", maxEntries: 200 }` |
| Queue mode | PM 配置 `queueMode: "steer"` |
| 模型分层 | architect/reviewer 使用更强模型 |
| A2A 方向控制 | 限制通信方向（如果源码支持） |

### P3 - 增强团队协作

| 项目 | 说明 |
|------|------|
| 共享项目上下文 | 通过 symlink 或目录重构实现 |
| 任务状态文件 | 定义共享的任务结果存放目录 |
| Heartbeat 定时汇报 | PM 配置心跳定期汇总任务进度 |
| 群组路由 | 增加飞书群组 binding，支持 @机器人 路由 |

---

## 八、推荐的完整配置升级

```json5
{
  session: {
    dmScope: "per-channel-peer",
    maintenance: {
      mode: "enforce",
      pruneAfter: "7d",
      maxEntries: 200,
      maxDiskBytes: "200MB"
    }
  },
  agents: {
    defaults: {
      workspace: "/home/camel/.openclaw/workspace",
      model: {
        primary: "deepseek/deepseek-v4-flash",
        fallbacks: ["deepseek/deepseek-v4"]
      },
      memorySearch: {
        provider: "ollama",
        model: "ollama/nomic-embed-text:latest"
      },
      subagents: {
        maxSpawnDepth: 2,
        delegationMode: "prefer"
      }
    },
    list: [
      { id: "main" },
      {
        id: "pm",
        workspace: "/home/camel/.openclaw/workspace-pm",
        agentDir: "/home/camel/.openclaw/agents/pm/agent"
        // PM 使用默认 flash 模型即可
      },
      {
        id: "architect",
        workspace: "/home/camel/.openclaw/workspace-architect",
        agentDir: "/home/camel/.openclaw/agents/architect/agent",
        model: { primary: "deepseek/deepseek-v4" }  // 架构师需要深度推理
      },
      {
        id: "dev-front",
        workspace: "/home/camel/.openclaw/workspace-dev-front",
        agentDir: "/home/camel/.openclaw/agents/dev-front/agent"
      },
      {
        id: "dev-backend",
        workspace: "/home/camel/.openclaw/workspace-dev-backend",
        agentDir: "/home/camel/.openclaw/agents/dev-backend/agent"
      },
      {
        id: "reviewer",
        workspace: "/home/camel/.openclaw/workspace-reviewer",
        agentDir: "/home/camel/.openclaw/agents/reviewer/agent",
        model: { primary: "deepseek/deepseek-v4" }  // 审查需要强理解力
      }
    ]
  },
  tools: {
    agentToAgent: {
      enabled: true,
      allow: ["main", "pm", "architect", "dev-front", "dev-backend", "reviewer"]
    },
    subagents: {
      tools: {
        alsoAllow: ["sessions_send"]
      }
    }
  }
}
```

---

## 九、关于 Coordinator 模式的实施路径

### 9.1 当前状态 → 目标状态

```
当前:
  用户 → 手动切换各飞书应用 → 各 Agent 独立工作 → 用户手动汇总

目标:
  用户 → 只与 PM 应用对话 → PM 自动派发给团队 → PM 汇总回报用户
```

### 9.2 实施步骤

**Step 1**: 在 PM 的 SOUL.md/AGENTS.md 中定义协调者行为：

```markdown
# AGENTS.md (pm workspace)

## Team Members
- architect: 技术架构师，负责方案设计和技术决策
- dev-front: 前端开发，负责 UI 实现
- dev-backend: 后端开发，负责 API 和服务实现
- reviewer: 代码审查，负责质量把关

## Workflow
1. 收到用户需求 → 分析拆解为子任务
2. 需要架构决策 → sessions_send(agentId="architect", ...)
3. 需要前端实现 → sessions_send(agentId="dev-front", ...)
4. 需要后端实现 → sessions_send(agentId="dev-backend", ...)
5. 需要代码审查 → sessions_send(agentId="reviewer", ...)
6. 收到回报 → 汇总后向用户报告

## Communication Rules
- 与用户的对话：直接回复，使用结构化格式
- 与团队的通信：使用 sessions_send，fire-and-forget 模式
- 收到 [Inter-session message]：这是团队成员的回报，整合后再回复用户
```

**Step 2**: 启用 `agentToAgent` + `subagents` 配置（如上节推荐配置）

**Step 3**: 测试 PM → architect 的端到端通信：
```
用户对 PM 说: "设计一个用户认证方案"
PM 应自动: sessions_send(agentId="architect", message="请设计用户认证方案...")
architect 回复后 PM 收到: [Inter-session message] ...
PM 向用户汇总: "架构师的方案如下..."
```

### 9.3 注意事项

1. **sessions_send vs sessions_spawn**：当前配置下各 Agent 是独立的顶级 Agent（非子 agent），应使用 `sessions_send` 而非 `sessions_spawn`。`sessions_spawn` 适用于在同一 Agent 内派生子 session。

2. **Session 混杂**：即使启用 A2A 通信，PM 的 session 中仍会收到 `[Inter-session message]` 前缀的回报。需要在 PM 的 SOUL.md 中明确指导如何处理这类消息。

3. **并行派发**：PM 可以同时向多个 Agent 发送 `sessions_send(timeoutSeconds=0)`，但需要在 AGENTS.md 中指导 PM 如何跟踪和汇总多个异步回报。

---

## 学习记录

| 日期 | 内容 | 来源 |
|------|------|------|
| 2026-05-28 | 分析真实团队配置，对照源码级 session 机制给出改进建议 | 实际配置 + 源码分析 |

---

*基于 notes/08-session-mechanism.md 的源码分析*
