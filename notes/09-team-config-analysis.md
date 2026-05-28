# 真实团队配置分析报告

> 基于 `notes/08-session-mechanism.md` 中的源码级理解，对当前飞书 Agent 团队的实际配置进行分析。
> 分析日期: 2026-05-28
> **角色修正**: main (飞书应用 "assistant") 是团队协调者 (Coordinator)，pm 是产品经理负责需求管理。

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
| **Coordinator 能力未激活** | main Agent 作为协调者，但未配置 delegation/spawn 能力 | 有协调者角色但缺少自动派发机制 |
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
用户 → 飞书私聊 assistant(main) → main Agent 接收指令并协调
用户 → 飞书私聊 pm → pm Agent 管理需求、拆解任务
用户 → 可能手动切换到其他应用 → 各角色独立工作
```

**问题**：用户需要手动充当协调者角色，在各个飞书应用间切换传递信息。

### 3.2 理想工作流（基于源码能力）

```
用户 → 飞书私聊 assistant(main) → main Agent 接收指令
  main → sessions_send(agentId="pm", message="分析需求:...") → 让 PM 分析
  main → sessions_send(agentId="architect", message="设计方案:...") → 让架构师设计
  main → sessions_yield() → 等待回报
  pm/architect 完成 → A2A announce 回传给 main
  main → sessions_send(agentId="dev-front", ...) → 派发开发任务
  ...
  main → 向用户汇报最终结果
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
  用户 → 飞书私聊 assistant(main) → main 协调 → A2A 通信各 Agent
  用户 → 也可能直接私聊各角色应用 → 各 Agent 独立工作
  问题: main 的协调能力未被充分配置，A2A 通信后 session 混杂

目标:
  用户 → 主要与 main(assistant) 对话 → main 自动派发/追踪/汇总
  用户 → 也可直接与特定角色对话（专项讨论时）
  main 的 session 中区分用户对话和团队回报
```

### 9.2 实施步骤

**Step 1**: 在 main 的 SOUL.md/AGENTS.md 中定义协调者行为：

```markdown
# AGENTS.md (main/assistant workspace)

## Role
你是团队协调者 (Coordinator)。用户通过飞书与你对话，你负责理解意图、分配任务、追踪进度、汇总结果。

## Team Members
- pm: 产品经理，负责需求分析、优先级排序、验收标准定义
- architect: 技术架构师，负责方案设计和技术决策
- dev-front: 前端开发，负责 UI 实现
- dev-backend: 后端开发，负责 API 和服务实现
- reviewer: 代码审查，负责质量把关

## Workflow
1. 收到用户需求 → 判断需求清晰度
2. 需求不清晰 → sessions_send(agentId="pm", ...) 让 PM 分析拆解
3. 需要架构决策 → sessions_send(agentId="architect", ...)
4. 需要实现 → sessions_send(agentId="dev-front/dev-backend", ...)
5. 需要审查 → sessions_send(agentId="reviewer", ...)
6. 收到 [Inter-session message] 回报 → 汇总后向用户报告

## Communication Rules
- 与用户的对话：直接回复，语气专业友好
- 与团队的通信：使用 sessions_send，fire-and-forget 模式 (timeoutSeconds=0)
- 收到 [Inter-session message]：这是团队成员的回报，不要原样转发给用户，要整合摘要
- 用户询问进度时：使用 sessions_list(activeMinutes=30) 查看各 Agent 状态
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

   **补充**：main 作为 Coordinator 与其他 Agent（pm/architect/dev/reviewer）通信时使用 `sessions_send`。如果 main 自身内部需要做复杂的多步骤编排（如并行跟踪多个任务），可以在 main 内部使用 `sessions_spawn` 创建子 session 来隔离不同任务的上下文。

2. **Session 混杂**：即使启用 A2A 通信，PM 的 session 中仍会收到 `[Inter-session message]` 前缀的回报。需要在 PM 的 SOUL.md 中明确指导如何处理这类消息。

3. **并行派发**：PM 可以同时向多个 Agent 发送 `sessions_send(timeoutSeconds=0)`，但需要在 AGENTS.md 中指导 PM 如何跟踪和汇总多个异步回报。

---

## 十、团队角色定义与配置建议

### 10.1 角色定位修正

| Agent ID | 飞书应用 | 实际角色 | 职责定位 |
|----------|----------|----------|----------|
| `main` | assistant | **Coordinator (协调者)** | 接收用户指令，派发任务，追踪进度，汇总回报 |
| `pm` | pm | Product Manager (产品经理) | 需求分析，优先级管理，任务拆解，验收标准 |
| `architect` | architect | Architect (架构师) | 技术方案设计，架构评审，技术选型 |
| `dev-front` | dev-front | Frontend Developer | 前端代码实现 |
| `dev-backend` | dev-backend | Backend Developer | 后端代码实现 |
| `reviewer` | reviewer | Code Reviewer | 代码质量审查，反馈改进意见 |

### 10.2 main 与 pm 的职责边界

main (Coordinator) 和 pm (Product Manager) 存在功能重叠区域，需要明确边界：

| 职责 | main (Coordinator) | pm (Product Manager) |
|------|-------------------|---------------------|
| 与用户直接沟通 | ✅ 主要入口 | ✅ 需求讨论时直接对话 |
| 理解用户意图 | ✅ 初步判断 | ✅ 深入需求分析 |
| 任务拆解 | ❌ 不做细节拆解 | ✅ 核心职责 |
| 派发到开发团队 | ✅ 分配给对应角色 | ❌ 通过 main 协调 |
| 追踪任务进度 | ✅ 全局视图 | ✅ 需求维度追踪 |
| 与架构师/开发沟通 | ✅ 转发和协调 | ✅ 需求澄清时直接沟通 |
| 汇总结果给用户 | ✅ 核心职责 | ❌ 向 main 汇报 |

**关键原则**：
- 用户日常交互的主入口是 **main**
- 需要深入需求讨论时，用户可直接与 **pm** 对话
- pm 完成需求分析后，结果通过 A2A 回报给 main
- main 负责后续的技术派发（不需要再经过 pm）

### 10.3 两种协作模式

#### 模式 A: main 全权协调（推荐起步）

```
用户 ↔ main (唯一入口)
         ↓ sessions_send
    pm / architect / dev-* / reviewer
         ↓ A2A announce
       main (汇总回报用户)
```

优势：用户只需与一个应用交互，体验最简单
适合：日常任务派发、进度查询、结果汇总

#### 模式 B: 用户可直接与角色对话

```
用户 ↔ main (协调)
用户 ↔ pm (深入需求讨论)
用户 ↔ architect (技术方案讨论)
用户 ↔ reviewer (审查结果讨论)
```

优势：专项深入讨论时效率更高
适合：需求变更澄清、架构方案选型讨论、审查意见交流

**建议**：两种模式并存。main 处理大部分协调工作，但保留用户直接与专项角色对话的能力。

### 10.4 Coordinator (main) 的完整配置建议

#### SOUL.md

```markdown
# Soul - Coordinator

## Identity
你是开发团队的协调者。用户通过你与整个 AI 开发团队交互。

## Personality
- 高效、条理清晰
- 善于分解复杂任务、分配到合适角色
- 主动汇报进度，不让用户等待
- 不亲自执行具体技术工作

## Boundaries
- 不写代码（交给 dev-front / dev-backend）
- 不做架构决策（交给 architect）
- 不做详细需求分析（交给 pm）
- 不做代码审查（交给 reviewer）
- 聚焦：协调、分配、追踪、汇总

## Communication Style
- 对用户：简洁明了，使用结构化格式
- 对团队：清晰指令，明确交付标准
- 收到回报：提炼关键信息，不原文转发
```

#### openclaw.json 中 main 相关配置

```json5
{
  agents: {
    defaults: {
      subagents: {
        maxSpawnDepth: 2,
        delegationMode: "prefer"
      }
    }
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

### 10.5 A2A 通信方向建议

```
main → pm           ✅ (main 请求需求分析)
main → architect    ✅ (main 请求方案设计)
main → dev-front    ✅ (main 派发开发任务)
main → dev-backend  ✅ (main 派发开发任务)
main → reviewer     ✅ (main 请求代码审查)

pm → main           ✅ (pm 回报需求分析结果)
architect → main    ✅ (architect 回报方案)
dev-front → main    ✅ (dev 回报开发完成)
dev-backend → main  ✅ (dev 回报开发完成)
reviewer → main     ✅ (reviewer 回报审查结果)

pm → architect      ✅ (需求澄清时直接沟通)
architect → dev-*   ✅ (技术指导)
reviewer → dev-*    ✅ (审查反馈)

dev-* → dev-*       ❌ (开发之间不直接通信，通过 main 协调)
dev-* → pm          ❌ (需求问题通过 main 转发)
```

### 10.6 关于 sessions_send vs sessions_spawn 的选择

在当前架构中：

| 场景 | 推荐工具 | 原因 |
|------|----------|------|
| main → pm/architect/dev/reviewer | `sessions_send` | 跨 Agent 通信，各 Agent 是独立顶级实例 |
| main 内部复杂任务编排 | `sessions_spawn` | 同一 Agent 内派生子 session，隔离不同任务上下文 |
| pm 内部多步需求分析 | `sessions_spawn` (在 pm 内部) | pm 自己内部的任务分解 |

**核心区别**：
- `sessions_send`: 给**另一个 Agent** 发消息（跨 agent，各自有独立 workspace）
- `sessions_spawn`: 在**自己内部**创建子 session（同一 agent，共享 workspace）

当前配置的各角色是独立 Agent，所以 main 协调时应使用 `sessions_send`。

如果未来 main 需要同时跟踪多个并行任务且避免 session 混杂，可以在 main 内部使用 `sessions_spawn` 为每个任务创建独立子 session，在子 session 中执行 `sessions_send` 与其他 Agent 通信。

---

## 学习记录

| 日期 | 内容 | 来源 |
|------|------|------|
| 2026-05-28 | 分析真实团队配置，对照源码级 session 机制给出改进建议 | 实际配置 + 源码分析 |
| 2026-05-28 | 修正角色定位(main=Coordinator, pm=PM)，给出协调者配置和团队协作建议 | 用户反馈 + 配置分析 |

---

*基于 notes/08-session-mechanism.md 的源码分析*
