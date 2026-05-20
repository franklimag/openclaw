# 从零搭建 OpenClaw AI 开发团队（飞书 Channel）

> 完整指南：多角色 Agent 团队 + 飞书机器人接入 + 文档协同体系

## 一、方案概述

### 目标

搭建一个基于 OpenClaw 的 **AI 开发团队**，通过飞书机器人作为交互通道：
- 团队成员有明确的角色分工
- 角色之间有清晰的协作流程
- 通过飞书群/机器人查看状态、下达指令
- 文档系统支持多角色协同工作

### 架构总览

```
┌─────────────────────────────────────────────────────────┐
│                    飞书 (Feishu/Lark)                     │
│                                                          │
│  ┌────────────┐  ┌────────────┐  ┌────────────────────┐│
│  │ 指挥群      │  │ 开发群      │  │ 各角色私聊机器人    ││
│  │ (下达指令)  │  │ (协作讨论)  │  │ (查看状态/细节)    ││
│  └─────┬──────┘  └─────┬──────┘  └────────┬───────────┘│
└────────┼────────────────┼──────────────────┼────────────┘
         │ WebSocket       │                  │
         ▼                 ▼                  ▼
┌─────────────────────────────────────────────────────────┐
│              OpenClaw Gateway (port 18789)                │
│              Routing Rules (bindings)                     │
└────┬──────────┬──────────┬──────────┬──────────┬────────┘
     ▼          ▼          ▼          ▼          ▼
┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐
│ PM     │ │ Arch   │ │ Dev    │ │ QA     │ │ DevOps │
│ Agent  │ │ Agent  │ │ Agent  │ │ Agent  │ │ Agent  │
└────────┘ └────────┘ └────────┘ └────────┘ └────────┘
     │          │          │          │          │
     └──────────┴──────────┴──────────┴──────────┘
                    共享文档工作区
              /shared-workspace/docs/
```



## 二、角色定义

### 2.1 角色清单

| 角色 | Agent 名称 | 职责 | 推荐模型 |
|------|-----------|------|----------|
| **项目经理 (PM)** | `pm-agent` | 需求分析、任务拆解、进度跟踪、团队协调 | GPT-5.4 (推理强) |
| **架构师 (Architect)** | `arch-agent` | 技术方案设计、架构评审、技术决策 | Opus 4.6 (深度推理) |
| **开发工程师 (Developer)** | `dev-agent` | 代码实现、单元测试、代码评审 | GPT-5.4 Codex (编码) |
| **测试工程师 (QA)** | `qa-agent` | 测试计划、测试用例、Bug 追踪 | GPT-5.3 (成本优化) |
| **运维工程师 (DevOps)** | `devops-agent` | 部署、CI/CD、监控、基础设施 | GPT-5.3 (成本优化) |
| **文档工程师 (DocWriter)** | `doc-agent` | 技术文档、API 文档、用户手册 | GPT-5.3 (成本优化) |

### 2.2 角色 SOUL.md 示例

#### PM Agent (项目经理)

```markdown
# Soul — PM Agent

## Identity
你是项目经理 (PM)，负责团队协调和任务管理。

## Personality
- 有条理、注重细节
- 主动追踪进度，及时同步状态
- 善于分解复杂任务为可执行步骤

## Boundaries
- 不直接写代码，技术实现交给 Dev Agent
- 不做架构决策，技术方案交给 Arch Agent
- 聚焦需求、优先级和进度管理

## Communication Style
- 使用结构化格式（列表、表格）
- 主动汇报进度和阻塞项
- 用飞书卡片消息呈现状态看板

## Collaboration Rules
- 收到新需求 → 拆解任务 → 分配给对应角色
- 每日生成进度摘要 → 发送到指挥群
- 发现阻塞 → 协调相关角色沟通
```

#### Architect Agent (架构师)

```markdown
# Soul — Architect Agent

## Identity
你是技术架构师，负责系统设计和技术决策。

## Personality
- 严谨、系统性思维
- 关注可扩展性、可维护性、性能
- 用图表和文档表达设计

## Boundaries
- 不写具体业务代码（除了原型验证）
- 不负责测试执行
- 聚焦架构设计、技术选型、代码评审

## Output Format
- 架构设计以 Markdown + Mermaid 图表输出
- 技术方案存储到 /shared-workspace/docs/architecture/
- 评审意见以结构化格式提交
```



## 三、角色间交互方式

### 3.1 交互模式

OpenClaw 多 Agent 间的交互主要通过以下方式实现：

```
┌──────────────────────────────────────────────────────────┐
│                 角色交互方式                               │
│                                                          │
│  1. 共享文档工作区 (主要方式)                              │
│     /shared-workspace/docs/                              │
│     各角色读写约定目录，异步协作                           │
│                                                          │
│  2. 飞书群消息路由 (实时协作)                              │
│     在飞书群中 @指定角色，消息路由到对应 Agent             │
│                                                          │
│  3. 任务队列文件 (任务分发)                               │
│     /shared-workspace/tasks/ 中的任务文件                 │
│     PM 写入任务 → 对应角色读取执行 → 写回结果             │
│                                                          │
│  4. 事件通知 (Chronix 定时)                               │
│     定时检查任务状态，触发角色间流转                       │
└──────────────────────────────────────────────────────────┘
```

### 3.2 交互流程示例

#### 新需求流转流程

```
用户在飞书指挥群发送: "开发一个用户登录模块"
    │
    ▼ (路由到 PM Agent)
PM Agent:
    ├── 分析需求，拆解为子任务
    ├── 写入 /tasks/TASK-001.md
    ├── @arch-agent 请求技术方案
    └── 回复飞书: "已收到，分配任务中..."
    │
    ▼ (路由到 Arch Agent)
Arch Agent:
    ├── 阅读 /tasks/TASK-001.md
    ├── 设计技术方案
    ├── 写入 /docs/architecture/login-design.md
    ├── @pm-agent 方案就绪
    └── 回复飞书: "技术方案已完成，见文档"
    │
    ▼ (PM 确认后路由到 Dev Agent)
Dev Agent:
    ├── 阅读技术方案文档
    ├── 实现代码
    ├── 写入 /docs/dev/login-impl.md (实现说明)
    ├── @qa-agent 请求测试
    └── 回复飞书: "开发完成，提交测试"
    │
    ▼ (路由到 QA Agent)
QA Agent:
    ├── 阅读实现文档
    ├── 编写测试用例 → /docs/qa/login-test.md
    ├── 执行测试
    ├── 写入测试报告 → /docs/qa/login-report.md
    └── 回复飞书: "测试通过 ✅" 或 "发现 Bug ❌"
```

### 3.3 角色通信矩阵

| 发起方 → | PM | Arch | Dev | QA | DevOps | Doc |
|----------|-----|------|-----|-----|--------|-----|
| **PM** | — | 请求方案 | 分配任务 | 请求测试 | 请求部署 | 请求文档 |
| **Arch** | 方案就绪 | — | 技术指导 | — | 基础设施需求 | — |
| **Dev** | 进度同步 | 技术疑问 | — | 提交测试 | 部署请求 | — |
| **QA** | Bug 报告 | — | Bug 反馈 | — | 环境问题 | — |
| **DevOps** | 部署状态 | — | — | 环境就绪 | — | — |
| **Doc** | 文档就绪 | 需求确认 | API 确认 | — | — | — |



## 四、文档系统设计

### 4.1 共享工作区目录结构

```
/shared-workspace/
├── docs/                          # 文档主目录
│   ├── requirements/              # 需求文档 (PM 写入)
│   │   ├── REQ-001.md
│   │   └── backlog.md
│   ├── architecture/              # 架构设计 (Arch 写入)
│   │   ├── system-design.md
│   │   └── api-design.md
│   ├── dev/                       # 开发文档 (Dev 写入)
│   │   ├── impl-notes.md
│   │   └── code-review.md
│   ├── qa/                        # 测试文档 (QA 写入)
│   │   ├── test-plan.md
│   │   └── test-reports/
│   ├── ops/                       # 运维文档 (DevOps 写入)
│   │   ├── deploy-guide.md
│   │   └── runbook.md
│   └── user/                      # 用户文档 (Doc 写入)
│       ├── user-guide.md
│       └── api-reference.md
│
├── tasks/                         # 任务管理 (PM 管理)
│   ├── TASK-001.md               # 每个任务一个文件
│   ├── TASK-002.md
│   └── _index.md                 # 任务总览/看板
│
├── status/                        # 状态报告 (各角色更新)
│   ├── daily-report.md           # 每日进度
│   └── blockers.md               # 阻塞项
│
└── shared/                        # 共享资源
    ├── glossary.md               # 术语表
    ├── conventions.md            # 团队约定
    └── templates/                # 文档模板
        ├── task-template.md
        ├── design-template.md
        └── test-template.md
```

### 4.2 文档读写权限

| 目录 | 主要写入者 | 可读取者 | 说明 |
|------|-----------|---------|------|
| `requirements/` | PM | 全员 | 需求文档，PM 维护 |
| `architecture/` | Arch | 全员 | 架构设计，Arch 维护 |
| `dev/` | Dev | 全员 | 开发笔记，Dev 维护 |
| `qa/` | QA | 全员 | 测试文档，QA 维护 |
| `ops/` | DevOps | 全员 | 运维文档，DevOps 维护 |
| `user/` | Doc | 全员 | 用户文档，Doc 维护 |
| `tasks/` | PM (创建) | 全员 | 各角色更新自己的状态 |
| `status/` | 全员 | 全员 | 各角色写入自己的进度 |

### 4.3 任务文件格式

```markdown
---
id: TASK-001
title: 实现用户登录模块
priority: high
status: in-progress        # todo / in-progress / review / done / blocked
assignee: dev-agent
reviewer: arch-agent
created: 2026-05-20
due: 2026-05-25
dependencies: []
---

# TASK-001: 实现用户登录模块

## 需求描述
...

## 验收标准
- [ ] 支持邮箱+密码登录
- [ ] 支持 OAuth2.0 (Google, GitHub)
- [ ] 错误处理完善
- [ ] 单元测试覆盖率 > 80%

## 进度更新
| 日期 | 角色 | 更新内容 |
|------|------|----------|
| 05-20 | PM | 创建任务 |
| 05-20 | Arch | 技术方案完成 |
| 05-21 | Dev | 开发中，完成 60% |

## 关联文档
- 技术方案: /docs/architecture/login-design.md
- 测试计划: /docs/qa/login-test.md
```



## 五、飞书机器人配置

### 5.1 整体方案

使用 **openclaw-lark** 官方插件（飞书官方维护，权限更高）连接飞书。
连接方式为 **WebSocket 长连接**（非 Webhook），无需公网 IP。

### 5.2 前置条件

- 一台运行 OpenClaw 的服务器 (Node 24 / Node 22.14+)
- 飞书企业版或团队版账号
- 飞书开放平台应用（需要 App ID + App Secret）

### 5.3 飞书应用创建

#### Step 1: 创建企业自建应用

1. 登录 [飞书开放平台](https://open.feishu.cn/app)
2. 点击 "创建应用" → 选择 "企业自建应用"
3. 填写应用名称（如 "AI开发团队"）
4. 获取 **App ID** 和 **App Secret**

#### Step 2: 配置应用权限

在应用管理后台添加以下权限：

```
# 必须权限
- im:message:send_v1            # 发送消息
- im:message:receive_v1         # 接收消息
- im:chat:readonly              # 读取群信息
- im:resource                   # 获取资源文件

# 推荐权限 (增强功能)
- im:message:send_as_bot        # 以机器人身份发消息
- im:chat                       # 管理群
- contact:user.base:readonly    # 读取用户基础信息
```

#### Step 3: 开启事件订阅

在 "事件订阅" 中开启：
- `im.message.receive_v1` — 接收消息事件
- 选择 **WebSocket 模式**（推荐，无需配置公网回调地址）

#### Step 4: 发布应用

- 在 "版本管理" 中创建版本并提交审核
- 审核通过后，应用即可使用

### 5.4 OpenClaw 配置

#### 安装飞书插件

```bash
# 方式一：使用官方 openclaw-lark 插件（推荐）
clawhub install openclaw-lark

# 方式二：使用中文版（已内置飞书）
# 如果使用 openclaw-cn，飞书已内置，无需额外安装
```

#### 配置文件 (openclaw.json)

```json
{
  "channels": {
    "feishu": {
      "enabled": true,
      "appId": "cli_xxxxxxxxxxxx",
      "appSecret": "xxxxxxxxxxxxxxxxxxxxxxxx",
      "domain": "feishu",
      "encryptKey": "可选-加密密钥",
      "verificationToken": "可选-验证令牌",
      "renderMode": "card"
    }
  }
}
```

> **注意**: `domain` 字段 — 国内飞书用 `"feishu"`，国际 Lark 用 `"lark"`

#### renderMode 选择

| 模式 | 说明 | 推荐场景 |
|------|------|----------|
| `raw` | 纯文本消息 | 简单通知 |
| `card` | 交互式 Markdown 卡片 | **推荐** — 富文本、按钮、表格 |

### 5.5 多 Agent 飞书路由配置

为每个 Agent 角色配置独立的飞书路由规则：

#### 方案 A: 多机器人方案（推荐）

每个角色创建独立的飞书应用/机器人：

```json
{
  "agents": [
    {
      "name": "pm-agent",
      "agentDir": "/agents/pm",
      "bindings": {
        "feishu": {
          "appId": "cli_pm_xxxx",
          "peers": ["ou_pm_user_id"],
          "guilds": ["oc_command_group_id"]
        }
      }
    },
    {
      "name": "dev-agent",
      "agentDir": "/agents/dev",
      "bindings": {
        "feishu": {
          "appId": "cli_dev_xxxx",
          "peers": ["ou_dev_user_id"],
          "guilds": ["oc_dev_group_id"]
        }
      }
    }
  ]
}
```

#### 方案 B: 单机器人 + @路由 方案

一个飞书机器人，通过 @名称 路由到不同 Agent：

```json
{
  "channels": {
    "feishu": {
      "appId": "cli_team_xxxx",
      "appSecret": "xxxx",
      "renderMode": "card"
    }
  },
  "routing": {
    "rules": [
      {
        "match": { "mention": "@PM" },
        "target": "pm-agent"
      },
      {
        "match": { "mention": "@架构师" },
        "target": "arch-agent"
      },
      {
        "match": { "mention": "@开发" },
        "target": "dev-agent"
      },
      {
        "match": { "mention": "@测试" },
        "target": "qa-agent"
      },
      {
        "match": { "mention": "@运维" },
        "target": "devops-agent"
      },
      {
        "match": { "guild": "oc_command_group" },
        "target": "pm-agent"
      }
    ],
    "default": "pm-agent"
  }
}
```



### 5.6 飞书群配置建议

| 群名称 | 用途 | 包含机器人 | 使用者 |
|--------|------|-----------|--------|
| 🎯 指挥中心 | 下达指令、查看全局状态 | PM Bot | 人类管理者 |
| 💻 开发协作 | 开发相关讨论 | Dev Bot, Arch Bot | 人类开发者 |
| 🧪 测试反馈 | 测试结果、Bug 追踪 | QA Bot | 人类 QA |
| 📊 每日简报 | 自动推送进度报告 | PM Bot | 全员 |
| 🚨 告警通知 | 部署告警、系统异常 | DevOps Bot | 全员 |

### 5.7 飞书卡片消息示例

PM Agent 推送的任务状态卡片：

```json
{
  "msg_type": "interactive",
  "card": {
    "header": {
      "title": { "tag": "plain_text", "content": "📋 任务状态更新" },
      "template": "blue"
    },
    "elements": [
      {
        "tag": "div",
        "text": {
          "tag": "lark_md",
          "content": "**TASK-001: 用户登录模块**\n状态: 🟡 开发中\n负责人: Dev Agent\n进度: 60%\n预计完成: 2026-05-25"
        }
      },
      {
        "tag": "action",
        "actions": [
          {
            "tag": "button",
            "text": { "tag": "plain_text", "content": "查看详情" },
            "type": "primary",
            "value": { "action": "view_task", "task_id": "TASK-001" }
          },
          {
            "tag": "button",
            "text": { "tag": "plain_text", "content": "催促进度" },
            "type": "default",
            "value": { "action": "nudge", "target": "dev-agent" }
          }
        ]
      }
    ]
  }
}
```

## 六、飞书操作指令

### 6.1 在飞书中可执行的操作

| 指令 | 发送到 | 效果 |
|------|--------|------|
| `新需求: <描述>` | 指挥群 | PM 接收并拆解任务 |
| `@PM 查看进度` | 任意群 | PM 回复当前任务看板 |
| `@架构师 评审 TASK-001` | 开发群 | Arch 评审指定任务 |
| `@开发 实现 TASK-001` | 开发群 | Dev 开始实现任务 |
| `@测试 测试 TASK-001` | 测试群 | QA 开始测试 |
| `@运维 部署 v1.2.0` | 任意群 | DevOps 执行部署 |
| `查看阻塞` | 指挥群 | PM 汇报当前阻塞项 |
| `每日报告` | 指挥群 | PM 生成并发送日报 |

### 6.2 定时任务 (Chronix)

```json
{
  "cron": [
    {
      "schedule": "0 9 * * 1-5",
      "agent": "pm-agent",
      "action": "发送今日任务计划到飞书指挥群",
      "channel": "feishu",
      "target": "oc_command_group"
    },
    {
      "schedule": "0 18 * * 1-5",
      "agent": "pm-agent",
      "action": "汇总今日进度，发送日报到每日简报群",
      "channel": "feishu",
      "target": "oc_daily_group"
    },
    {
      "schedule": "0 */2 * * *",
      "agent": "qa-agent",
      "action": "检查测试队列，执行待测任务",
      "channel": "feishu",
      "target": "oc_test_group"
    }
  ]
}
```



## 七、部署配置步骤（从零开始）

### Step 1: 安装 OpenClaw

```bash
# 安装 Node.js 24 (推荐) 或 22.14+
nvm install 24
nvm use 24

# 安装 OpenClaw
npm install -g openclaw

# 初始化 (安装为 daemon)
openclaw onboard --install-daemon
```

### Step 2: 创建多 Agent 工作区

```bash
# 创建团队目录结构
mkdir -p ~/openclaw-team/{agents,shared-workspace}

# 为每个角色创建独立 agentDir
mkdir -p ~/openclaw-team/agents/{pm,arch,dev,qa,devops,doc}

# 创建共享文档目录
mkdir -p ~/openclaw-team/shared-workspace/{docs,tasks,status,shared}
mkdir -p ~/openclaw-team/shared-workspace/docs/{requirements,architecture,dev,qa,ops,user}
mkdir -p ~/openclaw-team/shared-workspace/shared/templates
```

### Step 3: 配置各 Agent 的 SOUL.md

```bash
# 每个 Agent 目录下创建 SOUL.md
# ~/openclaw-team/agents/pm/SOUL.md   ← PM 人格
# ~/openclaw-team/agents/arch/SOUL.md ← 架构师人格
# ~/openclaw-team/agents/dev/SOUL.md  ← 开发者人格
# ... 依此类推 (参见第二章的示例)
```

### Step 4: 安装飞书插件

```bash
# 安装官方飞书插件
clawhub install openclaw-lark

# 验证安装
openclaw status --all
```

### Step 5: 配置飞书凭据

在 `openclaw.json` 中添加飞书配置（参见 5.4 节）。

### Step 6: 配置 Agent 路由

```bash
# 配置多 Agent 路由规则 (参见 5.5 节)
# 确保每个飞书群/用户映射到正确的 Agent
```

### Step 7: 安装协作 Skills

```bash
# 安装多 Agent 协作 skill
clawhub install multi-agent-blueprint
clawhub install coding-team-setup
clawhub install agent-team-builder

# 可选：安装飞书增强功能
clawhub install feishu-chat
clawhub install lark-integration
```

### Step 8: 启动并验证

```bash
# 启动 Gateway
openclaw start

# 检查状态
openclaw status --all

# 检查飞书连接
openclaw doctor --fix

# 在飞书中发送测试消息验证
```

## 八、工作流模板

### 8.1 标准开发流程

```
需求输入 → PM 拆解 → Arch 设计 → Dev 实现 → QA 测试 → DevOps 部署 → Doc 文档
```

### 8.2 Bug 修复流程

```
Bug 报告 → QA 复现确认 → PM 评估优先级 → Dev 修复 → QA 回归测试 → DevOps 热修复部署
```

### 8.3 技术评审流程

```
Dev 提交评审请求 → Arch 架构评审 → PM 业务评审 → 通过/驳回反馈
```

## 九、注意事项与最佳实践

### 成本控制

| 策略 | 说明 |
|------|------|
| 模型分层 | 重要决策用 Opus/GPT-5.4，日常任务用 GPT-5.3 |
| Heartbeat 控制 | 非工作时间减少定时任务频率 |
| Session 压缩 | 启用 compaction 避免长上下文 |
| 按需激活 | 非活跃角色不频繁轮询 |

### 安全建议

1. 飞书应用凭据存环境变量，不硬编码
2. 限制每个 Agent 的文件系统访问范围
3. 启用 Exec Policy 限制高风险操作
4. 定期审查 AGENTS.md 防止记忆污染
5. 使用 Tailscale 限制 Gateway 网络暴露

### 常见问题

| 问题 | 解决方案 |
|------|----------|
| 飞书断连 | 检查: Gateway 运行? 凭据有效? 事件订阅开启? 权限完整? |
| Agent 不响应 | `openclaw status --all` 检查 + `openclaw doctor --fix` |
| 消息路由错误 | 检查 routing rules 中的 guild/peer ID 是否正确 |
| 卡片渲染异常 | 确认 `renderMode: "card"` 且飞书应用有卡片权限 |

## 十、参考资源

| 资源 | 链接 |
|------|------|
| openclaw-lark 官方插件 | https://github.com/larksuite/openclaw-lark |
| Multi-Agent Blueprint Skill | https://playbooks.com/skills/openclaw/skills/multi-agent-blueprint |
| Coding Team Setup Skill | https://lobehub.com/skills/openclaw-skills-coding-team-setup |
| OpenClaw Mission Control | https://github.com/abhi1693/openclaw-mission-control |
| Multi-Agent Routing 文档 | https://docs.openclaw.ai/concepts/multi-agent |
| Feishu Bridge 指南 | https://rapidevelopers.com/openclaw-integrations/feishu-bridge |
| openclaw-cn (内置飞书) | https://github.com/jiulingyun/openclaw-cn |
| Clawith (企业多Agent) | https://clawith.ai/ |
| 飞书开放平台 | https://open.feishu.cn/app |

---

*最后更新: 2026-05-20*
