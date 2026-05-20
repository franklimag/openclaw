# Agentic Loop 学习笔记

> OpenClaw 的核心推理循环，将 "聊天机器人" 升级为 "自主 Agent"。

## 核心流程

```
Intake → Context Assembly → Model Inference → Tool Execution → Streaming Replies → Persistence
```

### Observe-Plan-Act 循环

```
┌─────────┐    ┌──────┐    ┌─────┐
│ Observe │───▶│ Plan │───▶│ Act │
└─────────┘    └──────┘    └──┬──┘
     ▲                        │
     └────────── Result ──────┘
```

- **Observe**: 观察当前上下文（消息、记忆、工具结果）
- **Plan**: LLM 推理，决定下一步
- **Act**: 执行工具调用或生成回复
- **循环**: 直到任务完成或达到迭代限制

## 关键特性

### 自主链式调用
- 不需要每步用户确认
- 自动串联多个工具调用完成复杂任务

### Compaction（上下文压缩）
- 长对话时压缩历史上下文
- 保留关键信息，丢弃冗余

### Retries（重试机制）
- 工具调用失败时自动重试
- 有最大重试次数限制

### Streaming
- 支持流式输出
- partial replies 机制

## 与 ReAct 模式的关系

OpenClaw 的 Agentic Loop 本质上是 ReAct（Reasoning + Acting）模式的实现：
- Reasoning: LLM 推理阶段
- Acting: Tool 执行阶段
- Observation: 结果收集阶段

## 源码位置（待确认）

```
TBD - 需要查看源码确认
```

## 问题与思考

- [ ] 单次 loop 的最大迭代次数是多少？
- [ ] Compaction 算法的具体策略？
- [ ] 如何处理工具调用超时？
- [ ] 并行工具调用是否支持？
- [ ] Loop 终止条件有哪些？

## 学习记录

| 日期 | 内容 | 来源 |
|------|------|------|
| | | |

---
