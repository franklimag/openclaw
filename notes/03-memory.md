# Memory 系统学习笔记

> OpenClaw 的记忆系统采用 Markdown-first 设计，透明、可编辑、可版本控制。

## 设计哲学

OpenClaw 记忆系统的核心理念：**透明优于性能，可审计优于便捷**。

拒绝不透明的二进制存储，所有记忆都是人类可读的纯文本 Markdown。
Agent 从文件中重建自身 — 每次启动都可以从文件恢复完整状态。

## 记忆文件体系

| 文件 | 用途 | 生命周期 | 可编辑 |
|------|------|----------|--------|
| `SOUL.md` | 人格定义、边界、语气 | 持久 | 用户可直接编辑 |
| `AGENTS.md` | 操作指令 + 工作记忆 | 持久（动态更新） | 用户 + Agent |
| `TOOLS.md` | 用户维护的工具使用说明/笔记 | 持久 | 用户 |
| `IDENTITY.md` | 身份定义 | 持久 | 用户 |
| `BOOTSTRAP.md` | 首次运行引导仪式 | 一次性（完成后删除） | 用户 |

### SOUL.md 示例结构

```markdown
# Soul

## Personality
- 简洁、专业、有帮助

## Boundaries
- 不执行破坏性操作
- 敏感操作前询问确认

## Tone
- 友好但不过度热情
- 技术问题直接回答
```

### AGENTS.md 作用

- Agent 的 "operating instructions"
- 包含工作记忆（Agent 自己记录的笔记）
- 在运行过程中可被 Agent 动态更新
- 相当于 Agent 的 "个人笔记本"

### BOOTSTRAP.md

- 首次运行时的引导脚本
- 完成后自动删除
- 用于初始配置、权限确认、偏好收集

## 记忆层次

### 1. Session Transcript（会话记录）
- **格式**: Append-only（只追加，不修改）
- **内容**: 完整对话记录 + 工具调用历史
- **管理**: 支持 compaction（压缩）和 pruning（修剪）
- **索引**: Mutable index 用于快速检索
- **范围**: 单个 session 内

### 2. Durable Memory（持久记忆）
- **格式**: Markdown 文件
- **内容**: 跨 session 的长期知识和偏好
- **存储**: 文件系统中的 .md 文件
- **管理**: 用户和 Agent 均可编辑
- **版本控制**: 可用 git 管理

### 3. SQLite Index（索引层）
- **用途**: 加速语义检索
- **技术**: SQLite 数据库
- **特点**: 对用户透明，自动维护

### 4. Activation/Decay 系统
- **概念**: 记忆有 "活跃度" 属性
- **Activation**: 频繁使用的记忆活跃度高，优先被召回
- **Decay**: 长期不用的记忆活跃度衰减
- **效果**: 模拟人类记忆的遗忘曲线

## 高级特性

### Knowledge Graph
- 支持知识图谱（3K+ facts）
- 结构化知识存储
- 关系推理能力

### Semantic Search
- 多语言语义搜索
- GPU 加速时延迟约 7ms
- 用于在大量记忆中找到相关内容

### Domain RAG
- 领域特定的检索增强生成
- 结合知识图谱 + 语义搜索
- 为 Agent 提供领域专业知识

### Continuity & Stability Plugins
- 记忆连续性插件
- 稳定性保障
- Graph-memory 插件

## 记忆与 Agent 重建

关键特性：**Agent 可以从文件重建自身**。

```
启动时：
  读取 SOUL.md → 恢复人格
  读取 AGENTS.md → 恢复操作指令和工作记忆
  读取 TOOLS.md → 恢复工具知识
  加载 SQLite Index → 恢复检索能力
  加载最近 Session → 恢复对话上下文
```

这意味着：
- 可以通过编辑文件来"修改" Agent 的记忆
- 可以用 git 版本控制记忆的变化
- 可以在不同机器间迁移 Agent 状态

## 安全考虑

- Memory 是中等风险的攻击面
- **风险**: 持久状态篡改（通过 prompt injection 写入恶意指令到 AGENTS.md）
- **BOOTSTRAP.md 攻击**: 注入恶意引导脚本实施隐蔽操纵
- **缓解**: 定期审查记忆文件内容

## Embedding 模型与 memorySearch 配置

### 是否需要设置专门的 Embedding 模型？

**结论**: 取决于使用场景 — 轻度使用可不设置（自动 fallback），重度使用或隐私敏感场景建议手动配置。

### 默认搜索插件: memory-core

默认内置的 `memory-core` 工作原理：
1. 将 Markdown 文件分块 (~400 token / 80 token overlap)
2. 对每个块生成 embedding 向量
3. 存入 SQLite 向量索引（sqlite-vec 加速）
4. 查询时做语义相似度搜索，返回 top-k 结果

### Embedding Provider 自动 Fallback 链

```
local (本地模型) → OpenAI → Gemini → Voyage → Mistral
```

- 有 OpenAI API key → 自动使用 OpenAI embedding，**零配置**
- 无任何 key → memorySearch 不可用（只剩关键词匹配）
- 可强制本地: `memorySearch.provider = "local"` 或配置 Ollama

### 默认搜索的局限

社区普遍反馈：
- ❌ 记忆量大时效果差 — "记住一切但理解不了"
- ❌ 分块策略简单 — 400 token 固定切分，语义割裂
- ❌ 早期版本几乎是关键词匹配 — 搜 "CI/CD" 找不到 "部署流水线"
- ✅ 小量记忆（< 50 文件）工作正常

### 推荐配置方案

| 场景 | 方案 | 配置 |
|------|------|------|
| 已有 OpenAI key + 不在意隐私 | 零配置 | 自动使用 OpenAI embedding |
| 隐私优先 / 本地优先 | Ollama + nomic-embed-text | `openclaw models memory set ollama/nomic-embed-text` |
| 大量记忆 / 复杂检索 | QMD 混合搜索 | `memory.backend = "qmd"` |
| 企业级 / 多 Agent | LanceDB Pro | Vector + BM25 + Cross-Encoder Rerank |

### 方案详解

#### 方案 A: Ollama 本地 Embedding（推荐）

```bash
# 安装 Ollama + 拉取 embedding 模型
ollama pull nomic-embed-text

# 配置 OpenClaw 使用 Ollama
openclaw models memory set ollama/nomic-embed-text
```

优势：免费、隐私保护、离线可用
适合：个人使用、多 Agent 团队

#### 方案 B: QMD 混合搜索引擎

```bash
# 安装 QMD
bun install -g https://github.com/tobi/qmd

# OpenClaw 配置
# memory.backend = "qmd"
```

QMD 提供：
- Vector 语义搜索 (本地 GGUF embedding)
- BM25 关键词搜索（双路并行）
- Reciprocal Rank Fusion 合并结果
- 本地 LLM re-ranking（精排）

#### 方案 C: LanceDB Pro（企业级）

```
Hybrid Retrieval (Vector + BM25) → Cross-Encoder Rerank → Multi-Scope Isolation
```

适合多 Agent 团队的隔离记忆管理。

### 飞书多 Agent 团队场景建议

对于 `notes/07-team-setup.md` 中描述的多 Agent 团队：
- shared-workspace 中会积累大量文档（需求/设计/测试/运维）
- **强烈建议配置 Ollama + nomic-embed-text** 或 QMD
- 确保各 Agent 能在大量文档中准确检索相关信息
- 隐私安全：本地 embedding 避免文档内容泄漏给第三方

### 隐私注意

⚠️ 如果使用**云端 embedding**（OpenAI/Gemini），所有记忆内容会被发送给模型提供商进行向量化。对于包含敏感信息（代码、业务数据、客户信息）的场景，**必须使用本地 embedding**。

### 配置参考（官方文档）

- Memory configuration reference: https://docs.openclaw.ai/reference/memory-config
- 支持配置项: embedding cache、hybrid search、session memory search (experimental)、sqlite-vec 加速
- 支持的 embedding providers: OpenAI、Gemini (native + Embedding 2 preview)、Voyage、Mistral、Ollama、自定义 OpenAI-compatible endpoint

## 问题与思考

- [x] 记忆文件的格式？→ Markdown (SOUL.md, AGENTS.md 等)
- [x] 如何持久化？→ 文件系统 + SQLite Index
- [x] Agent 如何更新记忆？→ 动态写入 AGENTS.md
- [x] 需要设置 embedding 模型吗？→ 视场景：轻度不需要（自动 fallback）；重度/隐私敏感建议 Ollama
- [x] 默认存储路径？→ `~/.openclaw/workspace`
- [ ] SOUL.md 的完整格式规范（YAML frontmatter?）
- [ ] Activation/Decay 的具体衰减公式？
- [ ] 知识图谱的存储格式和查询语言？
- [ ] 多 Agent 场景下 memory 如何共享/隔离？
- [ ] Session transcript 的最大大小限制？
- [ ] QMD 与 memory-core 的性能对比数据？

## 学习记录

| 日期 | 内容 | 来源 |
|------|------|------|
| 2026-05-20 | 全面更新：Activation/Decay、Knowledge Graph、安全风险 | 多源 |
| 2026-05-22 | 新增 Embedding 配置调研：自动 fallback 链、Ollama 方案、QMD 混合搜索、隐私建议 | 社区教程 + 官方文档 + GitHub issues |

---
