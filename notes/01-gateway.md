# Gateway 层学习笔记

> OpenClaw 的控制平面 — WebSocket 服务器，统管所有通道、Agent 和平台应用。

## 核心定位

Gateway 是 OpenClaw 的 **心脏** — 一个长期运行的 WebSocket 服务器，充当中央控制平面。
它不仅是消息路由器，更是整个系统的编排中枢。

## 技术规格

| 属性 | 值 |
|------|-----|
| 协议 | WebSocket (主) + HTTP (webhook/health) |
| 默认端口 | 18789 |
| 进程模型 | 单进程长驻 daemon (Node.js) |
| 安装 | `openclaw onboard --install-daemon` |
| 运行 | 系统服务 (managed local service) |

## 核心职责

### 1. Channel Adapters（通道适配器）

- 每个消息平台一个 adapter（内置 + 通道插件）
- 负责协议转换：平台特定格式 ↔ OpenClaw 内部消息格式
- 支持 50+ 平台
- 通过 ClawHub 可安装更多通道插件

### 2. Session 管理

- 一个 session = 一个独立对话上下文
- 存储为 **append-only transcript** + **mutable index**
- Session 间状态完全隔离
- 支持 compaction（压缩）和 pruning（修剪）

```bash
# CLI 管理
openclaw sessions list
```

### 3. Lane Queue（队列系统）

- 默认：同一 session 内**串行执行**
- 目的：防止并发修改共享状态的竞态条件
- 并行执行：仅对显式标记的低风险任务开放（opt-in）

### 4. 路由

- 根据消息来源和 session 分配给正确的 Agent 实例
- 支持多个 Agent 同时运行

### 5. Hooks 系统

两种 hook 类型：
- **Gateway Hooks (内部)**: 系统级拦截点
- **Plugin Hooks**: Agent + Gateway 生命周期回调

### 6. Cron / Scheduling — Chronix

- Chronix 调度引擎
- 支持定时任务
- 心跳 (Heartbeat) daemon
- 让 Agent 主动执行，无需等待用户消息

### 7. Webhook Router

- 接收外部 webhook 请求
- 路由到对应的 handler
- 支持第三方服务回调

### 8. Plugin System

- 通道插件：扩展消息平台支持
- 工具插件：扩展 Agent 能力
- 通过 `openclaw plugins` CLI 管理

## 架构图

```
              ┌──────────────────────────┐
              │       外部平台            │
              │ Telegram/Slack/Discord/...│
              └──────────┬───────────────┘
                         │ Webhook / WebSocket
                         ▼
┌────────────────────────────────────────────────┐
│              Gateway (port 18789)                │
│                                                  │
│  ┌────────────┐  ┌──────────────┐  ┌────────┐ │
│  │  Channel   │  │   Session    │  │  Lane  │ │
│  │  Adapters  │  │   Manager    │  │  Queue │ │
│  └────────────┘  └──────────────┘  └────────┘ │
│                                                  │
│  ┌────────────┐  ┌──────────────┐  ┌────────┐ │
│  │   Hooks    │  │   Chronix    │  │Webhook │ │
│  │  System    │  │  Scheduler   │  │ Router │ │
│  └────────────┘  └──────────────┘  └────────┘ │
│                                                  │
│  ┌──────────────────────────────────────────┐  │
│  │           Plugin System                    │  │
│  └──────────────────────────────────────────┘  │
└────────────────────────┬───────────────────────┘
                         │ 分发
                         ▼
              ┌──────────────────────┐
              │   Agent Runtime      │
              └──────────────────────┘
```

## 关键设计决策

1. **单进程 daemon**: 简化部署，避免分布式复杂性
2. **WebSocket 优先**: 低延迟、双向通信
3. **串行队列**: 安全性优先于吞吐量
4. **插件化通道**: 核心轻量，按需扩展

## 安全注意

- Gateway WebSocket 接口是 **最宽的集成面** — 主要攻击面之一
- 建议：限制网络暴露，使用 Tailscale 等安全隧道
- ClawJacked 漏洞即通过 Gateway 接口实现攻击

## 性能优化 (v2026.5.22) 🆕

v2026.5.22 对 Gateway 进行了重大性能优化：

### Model List API: 4,100x 提速

| 指标 | 优化前 | 优化后 |
|------|--------|--------|
| Model list 响应时间 | ~20 秒 | 5 毫秒 |
| 提升倍数 | — | **4,100x** |

**优化策略**:
- **Pre-warm auth states** — Gateway 启动时预热所有 provider 认证状态
- **热路径跳过 discovery** — 请求时跳过整个 plugin + CLI discovery 链
- 这是从"demo 级"到"生产级"的关键基础设施优化

### Channel Catalog 复用

- 跨进程复用 channel catalog 读取结果
- 避免每个请求重复读取通道配置
- 显著减少高吞吐场景下的 I/O 开销

### CPU Profile 轮转

- 自动轮转 CPU profile 数据
- 减少持续 profiling 对正常运行的资源消耗
- 便于事后诊断性能问题

### 启动时间优化

结合上述优化，Gateway 启动时间和首次响应延迟都大幅改善。

## 诊断命令

```bash
openclaw status --all       # 全面状态检查
openclaw doctor --fix       # 自动诊断修复
```

## 问题与思考

- [x] Gateway 监听什么端口？→ 18789
- [x] 使用什么协议？→ WebSocket + HTTP
- [x] 进程模型？→ 单进程 Node.js daemon
- [ ] Gateway hooks 的具体 API 签名？
- [ ] Lane Queue 的最大深度/超时配置？
- [ ] 多 Gateway 实例是否支持（高可用）？
- [ ] Chronix 的 cron 表达式格式？
- [ ] WebSocket 协议的消息帧格式？

## 学习记录

| 日期 | 内容 | 来源 |
|------|------|------|
| 2026-05-20 | 初始化笔记，整理 Gateway 核心职责和技术规格 | 多源搜索 |
| 2026-05-26 | 新增 v2026.5.22 性能优化：4,100x API 提速、catalog 复用、CPU profile 轮转 | release notes |

---
