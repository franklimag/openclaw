# 安全机制学习笔记

> OpenClaw 的安全架构：快速增长中暴露的攻击面与持续加固的安全措施。

## 安全现状概述

OpenClaw 在 2026 年经历了多次严重安全事件，促使项目从 "轻量安全" 转向更严格的安全态势。
创始人在官方博客中承认需要 "更少 core 中的 magic，更清晰的插件边界，更好的扫描和发布卫生"。

## 攻击面分析

根据学术研究 (arxiv:2603.27517 - A Systematic Taxonomy of Security Vulnerabilities)：

### 攻击面层次

| 攻击面 | 风险等级 | 说明 |
|--------|----------|------|
| **Exec Policy Engine** | 最高 | 最主要的攻击入口，控制工具执行 |
| **Gateway WebSocket** | 高 | 最宽的集成面，暴露外部接口 |
| **Container Boundary** | 高 | 沙箱逃逸风险 |
| **Agent Runtime** | 中 | 推理过程被操纵 |
| **Memory & Knowledge** | 中 | 持久状态篡改 |
| **LLM Provider** | 中 | 通过 prompt injection 控制 |
| **Local Execution** | 中 | 本地代码执行环境 |

### 系统性弱点模式

1. **Exec Policy Engine** — 最主要的攻击入口
2. **Gateway WebSocket Interface** — 最宽的集成面
3. **Container Boundary** — 高关注面

## 已知安全事件 (2026)

### ClawJacked (2026 Q1)
- **发现者**: Oasis Security
- **严重程度**: High
- **影响**: 任何网站可静默接管开发者的 AI Agent
- **无需**: 插件、扩展或用户交互
- **修复**: OpenClaw 团队 24 小时内修复
- **启示**: Gateway 暴露面需要严格限制

### Claw Chain (2026-05)
- **发现者**: Cyera Research
- **影响范围**: 245,000 AI Agent 服务器
- **漏洞链**: 四个链式漏洞
  1. 沙箱逃逸 (Container Boundary breach)
  2. 凭证窃取 (Credential theft)
  3. 输入验证绕过 (Input validation bypass)
  4. 持久控制建立 (Persistent control)
- **修复**: ✅ v2026.5.18 中**全部修补确认**
- **状态**: 已关闭，四个漏洞均已在稳定版中修复

### ClawHub 投毒
- **发现者**: Bitdefender Labs
- **方式**: 通过公共 Skills 注册表分发恶意包
- **影响**: 企业网络中的 OpenClaw 实例
- **缓解**: 加强 ClawHub 扫描和审核

### BOOTSTRAP.md 注入
- **论文**: arxiv:2603.19974v1
- **方式**: 通过注入恶意 BOOTSTRAP.md 内容操纵 Agent
- **特点**: 隐蔽、持久、难以检测

### Prompt Injection 攻击
- **案例**: Nvidia 沙箱 OpenClaw Agent 被突破
- **问题**: 传统防御无法预测 LLM 驱动应用的运行时行为
- **教训**: 自主 Agent 的安全边界比传统软件更模糊

## 安全层次设计

### 1. Exec Policy Engine (执行策略引擎)
- 决定哪些工具调用被允许
- 参数验证和限制
- 黑名单/白名单机制

### 2. Container Boundary (容器边界)
- 可选的沙箱隔离
- 限制工具执行的系统访问
- 防止逃逸到宿主

### 3. 权限系统
- 工具/Skill 级别的权限管理
- 敏感操作需要用户确认
- 分级权限模型

### 4. ClawHub 安全
- A-F 安全等级评分
- 上架前安全扫描
- 恶意包检测

### 5. 网络隔离
- 推荐使用 Tailscale 等安全隧道
- 限制 Gateway 端口暴露
- 不建议将 18789 端口直接暴露到公网

### 6. Local-first 隐私
- 数据不离开本地（除 LLM API 调用外）
- 无云端数据存储
- 用户完全控制数据

## 安全加固方向 (2026.5+)

根据官方博客和社区动态：

1. **更少 core magic** — 减少核心代码中的隐式行为
2. **更清晰的插件边界** — 隔离插件权限
3. **更好的扫描** — ClawHub 和本地 skills 的安全扫描
4. **更严格的发布卫生** — 防止恶意包上架
5. **改进的安全态势** — 系统性安全增强
6. **安全路线图** (2026-05-15 发布) — Jesse Merhi 主导

## 新增安全机制 (v2026.5.19-5.20)

### Plaintext Secret Checks (v2026.5.20) 🆕

**问题背景**: Snyk 扫描发现 ClawHub 中约 7.1% 的 Skills（283/4000）存在凭据明文暴露问题 — SKILL.md 指令要求 Agent "使用此 API key"，导致密钥被保存在 LLM 上下文中，可能泄漏给模型提供商或出现在日志明文中。

**v2026.5.20 的修复**:
- 新增**运行时明文密钥检测** — 自动扫描 SKILL.md 和上下文中的明文凭据
- 检测到明文密钥时发出警告/拒绝加载
- 引导使用环境变量或 secret store 替代硬编码
- 对开发者的影响：SKILL.md 中**不应再硬编码任何 API key、token 或 password**

**正确做法**:
```markdown
# SKILL.md (错误 ❌)
Use API key: sk-xxxxxxxxxxxx

# SKILL.md (正确 ✅)
Read the API key from environment variable $MY_API_KEY
```

### Skill 安装信任边界 (v2026.5.19)

- 上传的 zip-backed skill archives 现在需要显式开关: `skills.install.allowUploadedArchives`
- Symlinked skill targets 保持 opt-in
- Linked plugin runtime facades 可加载但不削弱默认隔离模型

### Cron Failover 安全分类 (v2026.5.19)

- Cron failover 现在可以**分类结构化的 `server_error` payloads**
- 根据错误类型决定重试策略（而非盲目重试）
- 防止因持续重试恶意/损坏的 cron 任务导致资源耗尽

### Runtime 错误处理增强 (v2026.5.19)

- 畸形的 session-kill 路径返回**确定性 400 错误**（而非不可预测的行为）
- 失败的工具调用在声称成功后会**显示简洁警告**
- scoped background exec/process 引用在 **compaction 后存活**（不会因上下文压缩丢失正在运行的后台任务引用）

## 最佳安全实践

### 部署建议
```bash
# 使用 Tailscale 限制网络暴露
# 不要将 Gateway 端口暴露到公网
# 使用强密码/token 认证

# 定期更新到最新版本
openclaw update

# 运行安全诊断
openclaw doctor --fix
```

### 日常使用
1. 定期审查 SOUL.md 和 AGENTS.md 内容
2. 注意 ClawHub 安装的 Skills 的安全评级
3. 监控 Agent 的工具调用日志
4. 限制高风险工具的使用范围
5. 关注官方安全公告

## 问题与思考

- [x] 主要攻击面？→ Exec Policy Engine > Gateway WebSocket > Container Boundary
- [x] 已知严重漏洞？→ ClawJacked, Claw Chain (已全部修补), ClawHub 投毒, BOOTSTRAP 注入
- [x] 安全方向？→ 更少 magic, 更清晰边界, 更好扫描
- [x] 凭据泄漏防护？→ v2026.5.20 plaintext secret checks（自动检测 SKILL.md 中的明文密钥）
- [ ] Exec Policy Engine 的配置语法？
- [ ] Container Boundary 的具体技术实现？
- [ ] 如何监控/审计 Agent 的所有工具调用？
- [ ] 多用户场景下的隔离保障？
- [ ] 是否有 bug bounty 计划？

## 学习记录

| 日期 | 内容 | 来源 |
|------|------|------|
| 2026-05-20 | 全面更新：安全事件梳理、攻击面分析、加固方向 | arxiv + 新闻 + 官方博客 |
| 2026-05-22 | 补充：plaintext secret checks、Claw Chain 修补确认、Cron failover 安全分类、Skill 信任边界 | v2026.5.18-5.20 release notes |

---
