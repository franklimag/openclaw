# AI Agent 团队协同机制设计

> 针对完整开发生命周期的多 Agent 协作方案
> 团队: main(Coordinator) / pm / architect / dev-front / dev-backend / reviewer
> 基础设施: OpenClaw + 飞书通道 + sessions_send (A2A)

---

## 一、项目生命周期阶段

```
需求调研 → 需求评审 → 架构设计 → 功能设计 → 设计评审
    → 开发计划 → 任务下发 → 分阶段开发 → 测试验收 → 联调联测
```

### 1.1 阶段-角色矩阵

| 阶段 | 主导角色 | 参与角色 | 产出物 |
|------|----------|----------|--------|
| 需求调研 | pm | main(转发用户输入) | requirements/REQ-xxx.md |
| 需求评审 | pm + architect | main(组织) | requirements/REQ-xxx.md (approved) |
| 架构设计 | architect | pm(需求澄清) | architecture/ARCH-xxx.md |
| 功能设计 | architect + pm | dev-*(可行性反馈) | design/DESIGN-xxx.md |
| 设计评审 | main(组织) | architect, pm, reviewer | design/DESIGN-xxx.md (approved) |
| 开发计划 | main | pm(优先级), architect(依赖) | plans/PLAN-xxx.md |
| 任务下发 | main | dev-front, dev-backend | tasks/TASK-xxx.md |
| 分阶段开发 | dev-front, dev-backend | architect(指导) | code + dev-logs/ |
| 测试验收 | reviewer | pm(验收标准) | reviews/REVIEW-xxx.md |
| 联调联测 | main(协调) | dev-front, dev-backend, reviewer | integration/INTEG-xxx.md |

---

## 二、工程共享文档与代码管理

### 2.1 共享工作区目录结构

```
/shared-workspace/
├── PROJECT.md                    ← 项目总览（全员可读，main 维护）
├── TEAM.md                       ← 团队角色和通信规则（全员可读）
│
├── requirements/                 ← 需求文档
│   ├── _index.md                 ← 需求列表总览
│   ├── REQ-001.md                ← 具体需求（含状态 frontmatter）
│   └── REQ-002.md
│
├── architecture/                 ← 架构设计
│   ├── _index.md
│   ├── ARCH-001-system-design.md
│   └── ARCH-002-api-design.md
│
├── design/                       ← 功能设计
│   ├── _index.md
│   ├── DESIGN-001-login.md
│   └── DESIGN-002-dashboard.md
│
├── plans/                        ← 开发计划
│   ├── _index.md
│   └── PLAN-001-sprint1.md
│
├── tasks/                        ← 任务卡片
│   ├── _index.md                 ← 任务看板（状态汇总）
│   ├── TASK-001.md
│   ├── TASK-002.md
│   └── ...
│
├── dev-logs/                     ← 开发日志
│   ├── dev-front/
│   │   └── 2026-05-28.md
│   └── dev-backend/
│       └── 2026-05-28.md
│
├── reviews/                      ← 审查记录
│   ├── REVIEW-001.md
│   └── REVIEW-002.md
│
├── integration/                  ← 联调记录
│   └── INTEG-001.md
│
├── decisions/                    ← 决策日志（不可变，只追加）
│   ├── DEC-001-tech-stack.md
│   └── DEC-002-api-pattern.md
│
└── status/                       ← 实时状态（各角色定期写入）
    ├── coordinator.md            ← main 维护的全局状态
    ├── pm.md
    ├── architect.md
    ├── dev-front.md
    ├── dev-backend.md
    └── reviewer.md
```

### 2.2 文档读写权限矩阵

| 目录 | main | pm | architect | dev-* | reviewer |
|------|------|-----|-----------|-------|----------|
| PROJECT.md | RW | R | R | R | R |
| requirements/ | R | **RW** | R | R | R |
| architecture/ | R | R | **RW** | R | R |
| design/ | R | R(参与) | **RW** | R | R |
| plans/ | **RW** | R(输入) | R(输入) | R | R |
| tasks/ | **RW**(创建/分配) | R | R | **RW**(更新状态) | R |
| dev-logs/ | R | R | R | **RW**(各自目录) | R |
| reviews/ | R | R | R | R | **RW** |
| integration/ | **RW** | R | R | RW(记录) | RW(验证) |
| decisions/ | **W**(追加) | W(追加) | W(追加) | R | R |
| status/ | **RW**(自己的) | **RW**(自己的) | **RW**(自己的) | **RW**(自己的) | **RW**(自己的) |

> 注：OpenClaw 中 Agent 读写文件通过 shell/file 工具实现。权限控制通过各 Agent 的 AGENTS.md 行为约定实现（软约束），而非文件系统级别（硬约束）。

### 2.3 文档 Frontmatter 规范

每个文档使用 YAML frontmatter 记录元信息：

```yaml
---
id: REQ-001
title: 用户登录模块
status: approved          # draft → in-review → approved → implemented → verified
owner: pm                 # 主要负责人
reviewers: [architect, main]
created: 2026-05-28
updated: 2026-05-29
version: 2               # 每次重大修改 +1
parent: null             # 父文档引用
depends: []              # 依赖的其他文档
tags: [auth, user, mvp]
---
```

### 2.4 版本迭代规则

| 规则 | 说明 |
|------|------|
| **只追加不删除** | 历史版本不删除，用 status 标记废弃 |
| **版本号递增** | frontmatter.version 在重大修改时 +1 |
| **变更记录** | 文档末尾保留 ## Changelog 章节 |
| **评审才能升级状态** | draft→approved 需要指定 reviewers 确认 |
| **decisions/ 不可变** | 决策记录一旦写入不修改，只能追加新决策覆盖 |

### 2.5 代码管理

```
/project-repo/                   ← Git 仓库（开发 Agent 可读写）
├── src/
│   ├── frontend/                ← dev-front 负责
│   └── backend/                 ← dev-backend 负责
├── tests/
├── docs/                        ← 生成的 API 文档
└── .openclaw-tasks/             ← 任务状态快照（可选）
```

代码操作规则：
- dev-front 只写 `src/frontend/` 和对应测试
- dev-backend 只写 `src/backend/` 和对应测试
- reviewer 只读代码，写审查意见到 `reviews/`
- architect 可读全部代码，写架构决策到 `decisions/`
- main/pm 不直接操作代码

---

## 三、消息与上下文管理

### 3.1 通信拓扑

```
                    ┌──────────┐
          ┌────────│   用户   │────────┐
          │        └──────────┘        │
          │ (飞书私聊)          (飞书私聊，专项讨论)
          ▼                            ▼
    ┌───────────┐                ┌──────────┐
    │   main    │◄──────────────►│    pm    │
    │Coordinator│   sessions_send│          │
    └─────┬─────┘                └──────────┘
          │
          │ sessions_send (派发/收集)
          │
    ┌─────┼──────────────────────────────┐
    │     │                              │
    ▼     ▼              ▼               ▼
┌────────┐ ┌──────────┐ ┌──────────┐ ┌────────┐
│architect│ │dev-front │ │dev-backend│ │reviewer│
└────┬───┘ └────┬─────┘ └────┬─────┘ └───┬────┘
     │          │             │            │
     └──────────┴─────────────┴────────────┘
              (architect→dev: 技术指导)
              (reviewer→dev: 审查反馈)
```

### 3.2 通信方式分类

| 通信类型 | 实现方式 | 适用场景 |
|----------|----------|----------|
| **指令派发** | `sessions_send(timeoutSeconds=0)` | main → 各角色，下发任务 |
| **结果回报** | A2A announce (自动) | 各角色 → main，任务完成 |
| **同步协商** | `sessions_send(timeoutSeconds=30)` | 需要立即得到回答 |
| **文档交换** | 写入共享文档 + 通知 | 大块结构化内容传递 |
| **状态广播** | 写入 status/*.md | 各角色定期更新自身状态 |
| **紧急中断** | `queueMode=steer` + sessions_send | 需要中断正在执行的任务 |

### 3.3 消息协议设计

#### 指令消息格式（main → 各角色）

```markdown
## 任务指令

**任务ID**: TASK-001
**类型**: [需求分析 | 架构设计 | 开发实现 | 代码审查 | 联调测试]
**优先级**: [P0-紧急 | P1-高 | P2-中 | P3-低]
**截止要求**: 尽快 / 具体时间描述
**上下文**: 参见 /shared-workspace/requirements/REQ-001.md
**交付物**: 将结果写入 /shared-workspace/architecture/ARCH-001.md
**验收标准**:
- 条件 1
- 条件 2
```

#### 回报消息格式（各角色 → main）

```markdown
## 任务回报

**任务ID**: TASK-001
**状态**: [已完成 | 进行中(xx%) | 受阻 | 需要协助]
**产出物**: /shared-workspace/architecture/ARCH-001.md
**摘要**: 一句话描述关键结论
**问题/风险**: (如有)
**后续建议**: (如有)
```

### 3.4 上下文传递策略

#### 问题：sessions_send 消息长度有限

sessions_send 的消息体适合传递简短指令，不适合传递大段文档内容。

#### 解决方案：「指针式」上下文传递

```
方式 1 (推荐): 消息中引用文档路径
  sessions_send(message="请评审 /shared-workspace/design/DESIGN-001.md")
  → 接收方通过 read 工具自行读取文档

方式 2: 消息中包含关键摘要 + 文档路径
  sessions_send(message="需求摘要: xxx。详见 /shared-workspace/requirements/REQ-001.md")

方式 3: 对于小量上下文，直接包含在消息中
  sessions_send(message="API 接口定义如下:\n POST /api/login {...}")
```

#### 各角色的上下文加载策略

| 角色 | 启动时读取 | 任务时读取 | 写入 |
|------|-----------|-----------|------|
| main | PROJECT.md, tasks/_index.md, status/*.md | 各文档(按需) | plans/, tasks/, status/coordinator.md |
| pm | PROJECT.md, requirements/_index.md | REQ-*.md | requirements/, status/pm.md |
| architect | PROJECT.md, architecture/_index.md | ARCH-*.md, REQ-*.md | architecture/, design/, decisions/, status/architect.md |
| dev-* | 对应 TASK-*.md, DESIGN-*.md | 代码文件 | dev-logs/, tasks/(更新状态), status/ |
| reviewer | TASK-*.md(待审查), 代码文件 | 设计文档(对照) | reviews/, status/reviewer.md |

### 3.5 Session 隔离设计

```
main 的 session 结构:
  agent:main:feishu:direct:<userId>     ← 用户对话（per-channel-peer）
  agent:main:subagent:task-req-001      ← 内部 spawn: REQ-001 相关协调
  agent:main:subagent:task-dev-sprint1  ← 内部 spawn: Sprint1 开发协调

pm 的 session 结构:
  agent:pm:main                         ← 接收 main 的 sessions_send
  agent:pm:feishu:direct:<userId>       ← 用户直接讨论需求时

architect 的 session 结构:
  agent:architect:main                  ← 接收 main 的 sessions_send
  agent:architect:feishu:direct:<userId> ← 用户直接讨论架构时
```

**main 使用 sessions_spawn 隔离不同任务的协调上下文**：

```
用户说: "开发用户登录模块"

main 的处理:
  1. 在用户 session 中回复: "收到，开始协调团队处理"
  2. sessions_spawn(taskName="project_login") → 创建子 session
  3. 在子 session 中:
     - sessions_send(agentId="pm", message="分析登录模块需求...")
     - sessions_send(agentId="architect", message="设计登录架构...")
     - 收到回报后继续协调
  4. 子 session 完成后，在用户 session 中汇总回报
```

---

## 四、状态监控与 Coordinator 高可用

### 4.1 状态文件规范

每个角色维护自己的 `status/<agentId>.md`：

```yaml
---
agent: dev-front
updated: 2026-05-28T14:30:00+08:00
overall_status: busy    # idle | busy | blocked | waiting
current_task: TASK-003
---

## 当前任务
- **TASK-003**: 实现登录页面 UI — 进度 60%，预计今日完成

## 待处理队列
- TASK-005: Dashboard 布局（等待 TASK-003 完成）

## 阻塞项
(无)

## 最近完成
- TASK-001: 项目脚手架搭建 ✅ (2026-05-27)
```

### 4.2 Coordinator 状态看板

main 维护 `status/coordinator.md`：

```yaml
---
project: 用户管理系统
phase: development      # requirements | design | development | testing | integration
updated: 2026-05-28T14:35:00+08:00
---

## 项目进度总览

| 模块 | 状态 | 负责人 | 进度 | 阻塞 |
|------|------|--------|------|------|
| 用户登录 | 开发中 | dev-front + dev-backend | 60% | 无 |
| 权限管理 | 设计中 | architect | 30% | 等待需求确认 |
| Dashboard | 待开始 | dev-front | 0% | 依赖登录模块 |

## 风险项
- 权限管理需求不够清晰，已派 pm 跟进

## 今日动态
- 14:00 dev-front 完成登录页面框架
- 13:30 architect 提交权限模型设计初稿
- 10:00 pm 完成用户画像需求调研
```

### 4.3 Heartbeat 监控机制

利用 OpenClaw 的 Chronix (cron) 调度，让 main 定期检查团队状态：

```json5
{
  cron: [
    {
      schedule: "*/30 * * * *",  // 每 30 分钟
      agent: "main",
      action: "读取 /shared-workspace/status/ 下所有文件，检查是否有 blocked 状态或超时任务，如有则主动跟进"
    },
    {
      schedule: "0 9 * * 1-5",   // 工作日早9点
      agent: "main",
      action: "生成每日计划，检查昨日未完成任务，发送到飞书指挥群"
    },
    {
      schedule: "0 18 * * 1-5",  // 工作日晚6点
      agent: "main",
      action: "汇总今日进展，更新 status/coordinator.md，发送日报到飞书"
    }
  ]
}
```

### 4.4 主动监控策略

main 的 HEARTBEAT.md 内容：

```markdown
# Heartbeat Tasks

## 每次心跳检查项
1. 读取 status/ 目录下所有角色状态文件
2. 检查 tasks/_index.md 中超过预期时间的任务
3. 如果发现:
   - 任何角色 overall_status=blocked → 立即 sessions_send 询问详情
   - 任务超时 → sessions_send 催促 + 记录到风险项
   - 角色状态文件超过 2 小时未更新 → sessions_send 询问进展
4. 更新 status/coordinator.md

## 阶段推进检查
- 当一个阶段的所有交付物 status=approved 时，自动推进到下一阶段
- 推进时通知用户并开始下一阶段的任务派发
```

### 4.5 异常处理机制

| 异常场景 | 检测方式 | 处理动作 |
|----------|----------|----------|
| 角色长时间无响应 | status 文件 >2h 未更新 | sessions_send 催促 + 通知用户 |
| 任务受阻 | status 中 blocked 标记 | 分析阻塞原因，协调解决或上报用户 |
| 设计冲突 | architect 和 pm 意见不一致 | 组织评审：同时 send 给双方，汇总后决策 |
| 开发返工 | reviewer 打回代码 | 将审查意见转发给对应 dev + 更新任务状态 |
| 依赖阻塞 | 任务 depends 中的前置未完成 | 调整优先级或重新规划 |
| 模型调用失败 | Agent 返回错误 | 记录错误，延迟重试或切换 fallback 模型 |

### 4.6 阶段门禁（Phase Gate）

每个阶段必须满足门禁条件才能推进：

```
需求调研 → 需求评审:
  门禁: requirements/REQ-*.md 全部 status=draft 且内容完整
  审批: main 组织，pm + architect 参与

需求评审 → 架构设计:
  门禁: requirements/REQ-*.md 全部 status=approved
  触发: main 自动 sessions_send(agentId="architect", ...)

架构设计 → 功能设计:
  门禁: architecture/ARCH-*.md status=approved
  触发: main 派发功能设计任务

设计评审 → 开发计划:
  门禁: design/DESIGN-*.md 全部 status=approved
  触发: main 生成开发计划 PLAN-*.md

开发计划 → 任务下发:
  门禁: plans/PLAN-*.md status=approved
  触发: main 拆解为 tasks/TASK-*.md 并分配

任务下发 → 分阶段开发:
  门禁: tasks/TASK-*.md 已分配 assignee
  触发: main sessions_send 通知各 dev 开始

开发 → 测试验收:
  门禁: tasks/TASK-*.md status=done (开发自测通过)
  触发: main sessions_send(agentId="reviewer", ...)

测试验收 → 联调联测:
  门禁: reviews/REVIEW-*.md 全部 passed
  触发: main 组织联调
```

---

## 五、Coordinator (main) 的编排状态机

### 5.1 项目级状态机

```
                        ┌──────────────────┐
                        │   REQUIREMENTS   │
                        │   (需求阶段)      │
                        └────────┬─────────┘
                                 │ 所有 REQ approved
                                 ▼
                        ┌──────────────────┐
                        │     DESIGN       │
                        │   (设计阶段)      │
                        └────────┬─────────┘
                                 │ 所有 DESIGN approved
                                 ▼
                        ┌──────────────────┐
                        │   DEVELOPMENT    │
                        │   (开发阶段)      │
                        └────────┬─────────┘
                                 │ 所有 TASK done + reviewed
                                 ▼
                        ┌──────────────────┐
                        │   INTEGRATION    │
                        │   (联调阶段)      │
                        └────────┬─────────┘
                                 │ 联调通过
                                 ▼
                        ┌──────────────────┐
                        │    COMPLETE      │
                        │   (项目完成)      │
                        └──────────────────┘
```

### 5.2 main 的编排逻辑（写入 AGENTS.md）

```markdown
## Phase Management

根据 PROJECT.md 中的 phase 字段决定当前行为:

### phase=requirements
- 主动询问用户需求细节
- 将用户输入转发给 pm 分析
- 等待 pm 完成需求文档
- 组织需求评审（send 给 architect 请求反馈）
- 所有 REQ approved → 推进到 design

### phase=design
- 派发架构设计给 architect
- 派发功能设计给 architect (基于 approved 需求)
- 组织设计评审
- 所有 DESIGN approved → 推进到 development

### phase=development
- 生成开发计划，拆解任务
- 分配任务给 dev-front / dev-backend
- 定期检查进度
- 开发完成 → 派发审查给 reviewer
- 所有审查通过 → 推进到 integration

### phase=integration
- 协调前后端联调
- 组织联测
- 全部通过 → 标记完成
```

---

## 六、实施路径

### Phase 1: 基础通信（1-2天）

- [ ] 配置 `session.dmScope: "per-channel-peer"`
- [ ] 配置 `agents.defaults.subagents.maxSpawnDepth: 2`
- [ ] 在 main 的 AGENTS.md 中写入团队成员清单和基本协调规则
- [ ] 测试 main → pm 的 sessions_send 通信
- [ ] 测试 pm → main 的 A2A announce 回传

### Phase 2: 文档协作（2-3天）

- [ ] 创建 /shared-workspace/ 目录结构
- [ ] 为每个角色的 workspace 创建 symlink 到共享区
- [ ] 定义文档 frontmatter 规范
- [ ] 在各角色 AGENTS.md 中写入文档读写约定
- [ ] 测试端到端文档创建和审查流程

### Phase 3: 状态监控（2-3天）

- [ ] 定义 status/ 文件格式规范
- [ ] 配置 main 的 HEARTBEAT.md
- [ ] 配置 Chronix cron 定时任务
- [ ] 在各角色 AGENTS.md 中要求定期更新状态
- [ ] 测试异常检测和主动跟进

### Phase 4: 全流程验证（3-5天）

- [ ] 用一个小需求走完完整生命周期
- [ ] 验证阶段门禁自动推进
- [ ] 调整消息格式和协调策略
- [ ] 优化模型配置（architect/reviewer 用更强模型）

---

## 七、配置汇总

### openclaw.json 关键配置

```json5
{
  session: {
    dmScope: "per-channel-peer",
    maintenance: {
      pruneAfter: "14d",
      maxEntries: 300,
      maxDiskBytes: "500MB"
    }
  },
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
  },
  cron: [
    {
      schedule: "*/30 * * * *",
      agent: "main",
      action: "检查团队状态，读取 /shared-workspace/status/ 并处理异常"
    },
    {
      schedule: "0 9 * * 1-5",
      agent: "main",
      action: "生成每日计划并通知团队"
    }
  ]
}
```

### 各角色 AGENTS.md 必须包含的通信约定

```markdown
## Communication Protocol (所有角色通用)

### 收到 [Inter-session message] 时
- 这是来自其他 Agent 的消息，按消息中的指令执行
- 完成后回复执行结果（A2A announce 自动回传给发送方）

### 状态更新义务
- 每次任务状态变化后，更新 /shared-workspace/status/<myId>.md
- overall_status 字段必须准确反映当前状态

### 文档写入规范
- 创建文档时必须包含 frontmatter
- 修改文档时更新 frontmatter.updated 和 version
- 不修改不属于自己职责范围的文档
```

---

## 学习记录

| 日期 | 内容 | 来源 |
|------|------|------|
| 2026-05-28 | 设计完整开发生命周期的 AI Agent 团队协同机制 | 源码分析 + 配置分析 + 项目管理实践 |

---

*基于 notes/08-session-mechanism.md 源码分析 + notes/09-team-config-analysis.md 配置分析*
