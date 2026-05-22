# OpenClaw 版本更新追踪

> 跟踪 OpenClaw 的版本发布、重要更新和社区动态。

## 版本命名规则

OpenClaw 使用 `YYYY.M.DD` 格式的版本号（如 `2026.5.12`）。
发布节奏极其激进 — 2026 Q1-Q2 期间每周多个版本，有时 24 小时内发布两个版本。

## 版本历史

### 2026.5.x 系列

#### ⭐ 2026.5.20 (May 20, 2026) — 最新版本 🆕
- **Discord voice follows users**: Discord 语音现在跟随用户移动
- **Plaintext secret checks**: 明文密钥检查（安全增强），防止凭据以明文暴露在 SKILL.md 和上下文中
- **Model status clarification**: 模型状态行为更清晰，改进错误提示
- **Windows install fixes**: 修复 Windows 安装问题
- 修复超过 100 处隐藏的 "认知偏差" 和 "系统黑洞"

#### 2026.5.19 (May 19, 2026) — 稳定性大版本
- **90+ 稳定性修复**: "Fixes 部分远大于 Changes 部分"，移除大量潜在故障点
- **Browser dialog 增强**: 更好的浏览器阻断证明 (proof around browser blockers)
- **Plugin 创建优化**: 更清晰的插件创建流程
- **Runtime parity checks**: 更强的运行时一致性检查
- **Mac 设置优化**: macOS 配置更稳定
- **Gateway 重启报告**: Gateway 重启或 Agent 跨通道工作时更谨慎的报告
- **运行时恢复**: scoped background exec/process 引用在 compaction 后存活
- **错误处理**: session-kill 路径返回确定性 400 错误，失败的工具调用显示简洁警告
- **Cron 容错**: cron failover 可分类结构化 server_error 用于重试策略

#### 2026.5.18 (May 18, 2026) — 稳定滚动更新
- **xAI/Grok OAuth 修复**: 修复 Grok 认证流程关键问题
- **Android Realtime Talk Mode**: 通过 Gateway relay 实现实时语音（流式麦克风输入 + 实时音频播放）
- **Telegram 修复**: 媒体和 forum-topic 投递可靠性提升
- **Browser dialog handling**: 浏览器对话框处理增强
- **Claw Chain 修复**: 四个链式安全漏洞全部修补
- **多模型配置**: 完全解锁多模型配置支持
- **GPT-5 全面支持**: 解锁 GPT-5 系列模型

#### 2026.5.16 Beta 6
- **Browser Dialogs**: 可在快照中展示待处理/已处理的浏览器模态对话框
- 新增 `blockedByDialog` 状态
- 支持按 dialog id 回答待处理对话框
- **Plugin 工具链**: `defineToolPlugin` + `openclaw plugins build/validate/init`
- 降低将内部脚本转为安全 OpenClaw 工具的成本

#### 2026.5.12
- **Codex login default**: 默认使用 OpenAI Codex 作为登录方式
- **Runtime fallbacks**: 模型调用失败时自动切换备选提供商
- **Stalled-stream recovery**: 流处理中断自动恢复，无数据丢失
- **Faster startup**: lean installs 的启动速度优化

#### 2026.5.7
- 紧急修复版本（具体内容待确认）

### 2026.4.x 系列

- Codex app-server 作为默认 agent turn 管理运行时
- ChatGPT subscription access 开放给 OpenClaw (3.2M 用户)
- GPT-5.4 通过 Codex endpoint 可用

### 2026.3.x 系列

#### 2026.3.22
- 转变为完整 "agent operating system"
- 插件系统
- 多模型支持增强
- 安全性加强

#### 2026.3.13-beta.1
- Chrome DevTools MCP 作为 first-class existing-session driver
- 支持连接已认证的浏览器 session

### 2026.2.x 系列

- OpenClaw 继续高速迭代
- Peter Steinberger 加入 OpenAI（2026年2月）
- 安全危机：ClawJacked 漏洞被发现并修复
- GitHub Stars 突破 200k

### 2025 Q4（初始发布）

- OpenClaw 开源发布
- 迅速成为 GitHub 增长最快的项目之一
- 创始人 Peter Steinberger 接受 Lex Fridman 采访（600k+ views）

## 里程碑事件

| 时间 | 事件 |
|------|------|
| 2025 Q4 | OpenClaw 开源发布（原名 Clawdbot/Moltbot）|
| 2025 Q4 | Peter Steinberger 接受 Lex Fridman 采访 |
| 2026-02 | Peter Steinberger 加入 OpenAI |
| 2026-02 | ClawJacked 安全漏洞事件 |
| 2026-03 | GitHub Stars 突破 200k |
| 2026-04 | ChatGPT 订阅用户可通过 Codex 使用 OpenClaw |
| 2026-05 | GitHub Stars 突破 350k |
| 2026-05 | v2026.5.12 发布 — Codex default + reliability 提升 |
| 2026-05 | Claw Chain 安全漏洞披露与修复 |
| 2026-05 | Peter Steinberger 100 个 Codex 实例 30 天 $1.3M API 费用 |
| 2026-05-18 | v2026.5.18 — Android 实时语音 + Grok OAuth 修复 + Claw Chain 修补 |
| 2026-05-19 | v2026.5.19 — 90+ 稳定性修复，运算符级版本 |
| 2026-05-20 | v2026.5.20 — Discord 语音增强 + 明文密钥检查 + Windows 修复 |
| 2026-05-22 | xAI 宣布 Grok 正式集成 OpenClaw (SuperGrok/X Premium) |

## 关注的功能方向

- [x] Codex 集成（已完成 2026.5）
- [x] Runtime fallbacks（已完成 2026.5.12）
- [x] Plugin 工具链 defineToolPlugin（已完成 2026.5.16）
- [x] Agent-browser 默认集成（已完成）
- [x] 多模型配置（已完成 2026.5.18）
- [x] Android 语音增强（已完成 2026.5.18 — Realtime Talk Mode）
- [x] 安全框架持续加固（已完成 2026.5.20 — plaintext secret checks）
- [x] Grok/xAI 集成（已完成 2026.5.18 OAuth 修复 + 5.22 正式集成）
- [ ] LTS 长期支持版本（官方已宣布 5 月后期发布）
- [ ] 更多 Channel 插件
- [ ] 性能持续优化
- [ ] Core 瘦身（将可选功能移至 ClawHub）

## 如何追踪更新

1. **GitHub Releases**: https://github.com/openclaw/openclaw/releases
2. **CHANGELOG.md**: https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md
3. **官方博客**: https://openclaw.ai/blog
4. **PatchBot**: https://patchbot.io/ai/openclaw
5. **社区**: Discord / GitHub Issues

---

*最后更新: 2026-05-22*
