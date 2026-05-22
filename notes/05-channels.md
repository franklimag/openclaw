# Channels 消息通道学习笔记

> OpenClaw 支持 50+ 消息平台接入，通过内置通道 + 通道插件实现 "无处不在"。

## 通道架构

OpenClaw 的通道系统分为两类：
- **内置通道 (Built-in)**: 核心支持，随 OpenClaw 安装
- **通道插件 (Channel Plugins)**: 社区贡献，通过 ClawHub 安装

### 架构图

```
┌──────────────────┐     ┌──────────────────────┐     ┌─────────────┐
│   外部平台        │────▶│   Channel Adapter     │────▶│   Gateway   │
│ (Telegram/Slack) │◀────│   (内置 or 插件)       │◀────│ (port 18789)│
└──────────────────┘     └──────────────────────┘     └─────────────┘
```

## 支持的平台

### 即时通讯
| 平台 | 类型 | 备注 |
|------|------|------|
| WhatsApp | 内置 | |
| Telegram | 内置 | |
| Signal | 内置 | |
| iMessage | 内置 | 推荐通过 BlueBubbles 接入 |

### 协作工具
| 平台 | 类型 | 备注 |
|------|------|------|
| Slack | 内置 | |
| Discord | 内置/插件 | |
| Microsoft Teams | 内置 | |
| Google Chat | 插件 | |

### 开源协议
| 平台 | 类型 | 备注 |
|------|------|------|
| Matrix | 插件 | 开源通信协议 |

### 区域平台
| 平台 | 类型 | 备注 |
|------|------|------|
| Zalo | 插件 | 越南 |
| 飞书 (Feishu) | 插件 | 中文版内置 |

### 其他
| 平台 | 类型 | 备注 |
|------|------|------|
| Email (SMTP/IMAP) | 内置 | |
| Calendar | 内置 | |
| Web UI | 内置 | 浏览器界面 |
| Android 语音 | 内置 | v2026.5.18+ 基于 Gateway relay |
| CRM | 插件 | 多种 CRM 集成 |

## Channel Adapter 职责

每个 adapter 负责：

1. **协议对接** — 处理平台特定的 API / Webhook / WebSocket
2. **消息标准化** — 将平台消息转为 OpenClaw 内部格式
3. **回复格式化** — 将 Agent 回复转为平台特定格式
4. **媒体处理** — 图片、音频、视频、文件的转换
5. **状态同步** — 已读回执、typing indicator 等
6. **连接管理** — 保持与平台的连接/重连

## 通道配置

通过 OpenClaw 配置文件启用/禁用通道：

```yaml
# 概念示例
channels:
  telegram:
    enabled: true
    token: "BOT_TOKEN"
  slack:
    enabled: true
    app_token: "xapp-..."
  whatsapp:
    enabled: false
```

## 多通道特性

- 同一用户可通过多个通道与同一 Agent 交互
- Session 可以跨通道延续
- **无需触发** — 持续监听所有已配置通道
- **通道无关** — Agent 核心逻辑不依赖特定通道

## iMessage 接入 (BlueBubbles)

OpenClaw 推荐使用 **BlueBubbles** 作为 iMessage 通道：
- 需要一台 macOS 设备运行 BlueBubbles server
- OpenClaw 通过 BlueBubbles API 收发 iMessage
- 截至 2026.4，这是官方推荐路径

## 语音能力 (v2026.5.18+)

### Android Realtime Talk Mode

v2026.5.18 引入完整的 Android 实时语音支持：

- **架构**: 通过 Gateway relay 实现语音中继
- **输入**: 流式麦克风输入 (streamed microphone input)
- **输出**: 实时音频播放 (real-time audio playback)
- **模型**: 完全解锁多模型配置（GPT-5 / Opus / Grok）
- **事件**: Codex-compatible realtime event aliases

### Discord 语音增强 (v2026.5.20)

- **Voice follows users**: 语音会话跟随用户在频道间移动
- **Speaker-turn diagnostics**: 说话者轮次诊断
- **Playback reset tracking**: 播放重置追踪
- **Barge-in analysis**: 打断分析
- **Audio cutoff investigation**: 音频截断调查
- **Safer playback queuing**: 更安全的播放队列，适应嘈杂的直播环境

### 语音通道总结

| 平台 | 语音能力 | 版本 |
|------|----------|------|
| Android | Realtime Talk Mode (双向流式) | v2026.5.18+ |
| Discord | Voice follows users + 诊断工具 | v2026.5.20+ |
| Telegram | 语音消息处理（非实时） | 已有 |

## 近期通道修复 (v2026.5.18-5.20)

### Telegram 增强
- 媒体文件投递可靠性提升
- Forum-topic 消息投递修复
- No-response DM turns 保持静默
- Partial-stream deltas 不产生重复预览
- 投递最终确认答案而非重播 pre-tool 文本
- Select-button callbacks 优先于 raw fallbacks

### xAI/Grok 集成 (v2026.5.18 + 2026-05-22 正式发布)

xAI 于 2026-05-22 正式宣布 Grok 集成 OpenClaw：

- **认证**: SuperGrok 或 X Premium 订阅即可使用
- **OAuth 修复**: v2026.5.18 修复了 Grok OAuth 认证流程
- **推理能力**: Grok 支持 `/think` 端点用于推理型任务
- **覆盖平台**: 通过 OpenClaw 在 WhatsApp、Telegram、Slack、Discord 等平台使用 Grok
- **配置**: `openclaw models auth login --provider xai`

## 安全注意

- Channel adapters 是外部暴露面
- 需要妥善保管各平台的认证凭据
- 建议使用环境变量存储 tokens
- 限制 Gateway 的网络暴露（Tailscale 推荐）

## 问题与思考

- [x] 支持哪些平台？→ 50+ (内置 + 插件)
- [x] iMessage 如何接入？→ BlueBubbles
- [x] 有语音支持吗？→ Android Realtime Talk (5.18+) + Discord voice follows (5.20+)
- [x] Grok 支持？→ v2026.5.18 OAuth 修复 + 2026-05-22 正式集成
- [ ] 如何开发自定义 channel adapter 插件？
- [ ] 跨通道 session 的合并策略？
- [ ] 各平台的 rate limiting 处理？
- [ ] 媒体文件的临时存储和清理机制？
- [ ] Web UI 的功能范围？

## 学习记录

| 日期 | 内容 | 来源 |
|------|------|------|
| 2026-05-20 | 全面更新：通道分类、BlueBubbles、Android 语音、插件机制 | 多源 |
| 2026-05-22 | 补充：Discord 语音增强、Telegram 修复细节、xAI/Grok 正式集成 | v2026.5.18-5.20 release notes |

---
