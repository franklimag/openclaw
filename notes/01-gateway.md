# Gateway 层学习笔记

> OpenClaw 的控制平面，负责消息接入、路由与会话管理。

## 核心职责

1. **Channel Adapters** — 为每个消息平台提供适配器，统一消息格式
2. **Session 管理** — 每个对话维持独立会话
3. **Lane Queue** — 串行队列，防止竞态条件
4. **路由** — 消息分发给正确的 Agent 实例
5. **Hooks** — Gateway hooks 用于拦截和扩展

## 关键概念

### Channel Adapter
- 每个平台一个 adapter
- 负责协议转换：平台特定格式 → OpenClaw 内部消息格式
- 输出时反向转换

### Session
- 一个 session = 一个独立对话上下文
- session 隔离状态，互不干扰
- session 持有自己的 memory 和 history

### Lane Queue
- 默认串行执行（同一 session 内）
- 防止并发修改共享状态
- 并行执行仅对显式标记的低风险任务开放

## 源码位置（待确认）

```
TBD - 需要查看源码确认具体文件路径
```

## 问题与思考

- [ ] Gateway 如何处理消息优先级？
- [ ] 多个 channel adapter 同时收到消息时的处理策略？
- [ ] Session 超时和清理机制？
- [ ] Gateway hooks 的具体 API 接口？

## 学习记录

| 日期 | 内容 | 来源 |
|------|------|------|
| | | |

---
