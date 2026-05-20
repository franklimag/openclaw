# Agentic Loop 学习笔记

> OpenClaw 的核心推理循环，将 "聊天机器人" 升级为 "自主 Agent"。

## 核心定位

Agentic Loop 是 OpenClaw 与普通聊天机器人的根本区别。它实现了 **Observe-Plan-Act** 循环模式，让 Agent 能够自主链式调用工具完成复杂任务，无需每步用户确认。

## 完整流程

```
Intake → Context Assembly → Model Inference → Tool Execution → Streaming Replies → Persistence
```

### Observe-Plan-Act 循环

```
┌──────────────────────────────────────────────────┐
│              Agentic Loop                        │
│                                                  │
│  ┌──────────┐    ┌───────────┐    ┌─────────┐  │
│  │ Observe  │───▶│   Plan    │───▶│   Act   │  │
│  │ 观察上下文│    │ LLM 推理  │    │ 执行工具 │  │
│  └──────────┘    └───────────┘    └────┬────┘  │
│       ▲                                 │       │
│       │         ┌──────────────┐        │       │
│       └─────────│   Result     │◀───────┘       │
│                 │  收集结果     │                 │
│                 └──────────────┘                 │
│                                                  │
│  终止条件: 任务完成 / 迭代上限 / 超时 / 用户中断   │
└──────────────────────────────────────────────────┘
```

## Agent Runtime — Pi Agent Core

Agentic Loop 运行在 **Pi Agent Core** 中，这是 OpenClaw 的推理引擎。

### 上下文组装

每次循环开始时组装完整上下文：
1. System Prompt
2. SOUL.md（人格定义）
3. AGENTS.md（操作指令 + 工作记忆）
4. Session History（对话历史）
5. Tool/Skill 定义
6. 当前环境信息

### 模型调度

- 支持多模型配置（主模型 + fallback 链）
- v2026.5.12: 默认通过 **Codex app-server runtime** 管理 agent turns
- 模型选择: GPT-5.x / Opus 4.6 / Grok / Gemini / Codex

```bash
openclaw models set openai/gpt-5.4     # 设置主模型
```

## 关键特性

### 1. 自主链式调用
- 不需要每步用户确认
- 自动串联多个工具调用完成复杂任务
- Agent 自行决定何时继续、何时终止

### 2. Compaction（上下文压缩）
- 长对话/多轮工具调用时自动压缩历史上下文
- 保留关键信息，丢弃冗余细节
- 防止超出模型上下文窗口限制

### 3. Runtime Fallbacks (v2026.5.12)
- 模型调用失败时自动切换到备选提供商
- 无缝切换，不中断当前任务
- 配置 fallback 链：主模型 → 备选1 → 备选2

### 4. Stalled-stream Recovery (v2026.5.12)
- 流式处理过程中如遇网络中断或 API 限速
- 自动检测流停滞
- 自动恢复，不丢失已处理数据
- 无需用户干预

### 5. Streaming
- 支持流式输出到通道
- Partial replies 机制
- 实时反馈执行进度

### 6. Tool Execution
- 工具调用有权限检查
- 支持沙箱隔离
- 执行结果反馈给下一轮循环

## 与 ReAct 模式的关系

OpenClaw 的 Agentic Loop 是 **ReAct (Reasoning + Acting)** 模式的工程化实现：

| ReAct 概念 | OpenClaw 实现 |
|-----------|--------------|
| Thought | LLM 推理阶段 (Plan) |
| Action | Tool 执行阶段 (Act) |
| Observation | 结果收集阶段 (Observe/Result) |

## Codex 集成 (2026.5+)

从 2026.5.12 开始，OpenClaw 默认使用 OpenAI Codex app-server runtime：
- Codex 拥有 workspace/edit/exec 工具
- 更好的工具集成和编程任务支持
- 通过 ChatGPT 订阅认证 (OAuth)

## Queueing + 并发控制

- 每个 session 的 agent turn 进入 Lane Queue
- 串行处理防止竞态
- 并行仅对低风险任务可选
- 前一个 turn 完成后才处理下一个

## 生命周期钩子

可在以下点拦截：
1. **Agent 启动前** — Gateway hooks
2. **Prompt 组装后** — Plugin hooks
3. **工具调用前后** — Tool hooks
4. **回复生成前** — Response shaping
5. **Session 结束** — Cleanup

## 问题与思考

- [x] 核心模式？→ Observe-Plan-Act (ReAct)
- [x] 模型 fallback？→ 2026.5.12 支持 runtime fallbacks
- [x] 流中断处理？→ Stalled-stream recovery
- [ ] 单次 loop 的最大迭代次数？
- [ ] Compaction 的具体压缩算法/策略？
- [ ] 并行工具调用是否支持？
- [ ] Token 计数和成本控制机制？
- [ ] ACP runtime options 映射细节？

## 学习记录

| 日期 | 内容 | 来源 |
|------|------|------|
| 2026-05-20 | 更新至 v2026.5.12，添加 Codex 集成、fallback、stream recovery | 多源 |

---
