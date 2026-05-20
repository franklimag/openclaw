# OpenClaw 版本更新追踪

> 跟踪 OpenClaw 的版本发布、重要更新和社区动态。

## 版本命名规则

OpenClaw 使用 `YYYY.M.DD` 格式的版本号（如 `2026.5.12`）。
发布节奏极其激进 — 2026 Q1-Q2 期间每周多个版本，有时 24 小时内发布两个版本。

## 版本历史

### 2026.5.x 系列

#### 2026.5.18 (May 18, 2026)
- Android 客户端切换到基于 Gateway relay 的实时语音 sessions
- 完全解锁多模型配置支持
- 修复 Claw Chain 四个链式安全漏洞

#### 2026.5.16 Beta 6
- **Browser Dialogs**: 可在快照中展示待处理/已处理的浏览器模态对话框
- 新增 `blockedByDialog` 状态
- 支持按 dialog id 回答待处理对话框
- **Plugin 工具链**: `defineToolPlugin` + `openclaw plugins build/validate/init`
- 降低将内部脚本转为安全 OpenClaw 工具的成本

#### ⭐ 2026.5.12 (当前稳定版)
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

## 关注的功能方向

- [x] Codex 集成（已完成 2026.5）
- [x] Runtime fallbacks（已完成 2026.5.12）
- [x] Plugin 工具链 defineToolPlugin（已完成 2026.5.16）
- [x] Agent-browser 默认集成（已完成）
- [x] 多模型配置（已完成 2026.5.18）
- [ ] Android 语音增强
- [ ] 安全框架持续加固
- [ ] 更多 Channel 插件
- [ ] 性能持续优化

## 如何追踪更新

1. **GitHub Releases**: https://github.com/openclaw/openclaw/releases
2. **CHANGELOG.md**: https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md
3. **官方博客**: https://openclaw.ai/blog
4. **PatchBot**: https://patchbot.io/ai/openclaw
5. **社区**: Discord / GitHub Issues

---

*最后更新: 2026-05-20*
