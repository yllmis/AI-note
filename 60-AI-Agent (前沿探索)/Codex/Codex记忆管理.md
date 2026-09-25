---
title: Codex 记忆管理
date: 2026-09-24
tags:
  - Codex
  - AI-Agent
  - 工具
  - 记忆系统
related_project:
  - "[[60-AI-Agent (前沿探索)/Claude Code/Claude Code记忆管理]]"
---

# Codex 记忆管理

Codex（OpenAI）的记忆分两层：**AGENTS.md（人写的指令）** + **Memories（自动生成的记忆）**。核心原则：==强制规则写 AGENTS.md，记忆只是辅助召回层==。

## 两层记忆机制

| 机制 | 谁维护 | 生命周期 | 作用 |
|------|--------|----------|------|
| **AGENTS.md** | 人写 | 永久、可进仓库 | 项目/全局的稳定指令、约定、规范 |
| **Memories** | Codex 自动后台生成 | 永久、本地存储 | 从历史会话提炼的召回上下文 |
| **Session context** | 自动 | 单次会话 | 当前对话历史 |
| **Skills / MCP / Subagents** | 人配置 | 持久能力扩展 | 可复用工作流、外部工具、子代理 |

> [!tip] 选择法则
> - 团队必守的规则、构建/测试命令 → **AGENTS.md**（进 git）
> - 从历史工作里学到的上下文 → **Memories**（本地，不进 git）
> - 可复用的工作流 → **Skills**
> - 事件触发的自动行为 → hooks / Automations（记忆做不到）

---

## AGENTS.md — 相当于 CLAUDE.md

人写的持久指令，**分层发现、按路径合并**。

### 发现顺序

1. **全局**：`~/.codex/AGENTS.override.md` 优先，否则 `~/.codex/AGENTS.md`（只取第一个非空文件）
2. **项目**：从项目根（通常 Git root）走到当前工作目录，每层取 `AGENTS.override.md` > `AGENTS.md` > fallback 名
3. **合并**：从根到 cwd 顺序拼接，==越靠近 cwd 优先级越高==

```
~/.codex/AGENTS.override.md              # 全局覆盖
~/.codex/AGENTS.md                       # 全局默认
repo-root/AGENTS.md                      # 项目级
services/payments/AGENTS.override.md     # 子目录覆盖（最优先）
```

### 关键配置

| 配置项 | 默认 | 说明 |
|--------|------|------|
| `project_doc_max_bytes` | 32 KiB | 合并后的大小上限，超出截断 |
| `project_doc_fallback_filenames` | — | 备用指令文件名（如 `TEAM_GUIDE.md`） |
| `CODEX_HOME` | `~/.codex` | 可换 profile |

```toml
# ~/.codex/config.toml
project_doc_fallback_filenames = ["TEAM_GUIDE.md", ".agents.md"]
project_doc_max_bytes = 65536
```

### 什么时候更新 AGENTS.md

- agent **反复犯同一个错** → 加规则
- 读了很多无关文件 → 加**路由指引**（优先看哪些目录）
- **重复的 PR review 意见** → 固化进去
- GitHub PR 里 `@codex add this to AGENTS.md` 可委托云端会话更新

> [!important] 反馈循环
> agent 做错时，纠正它并让它**把纠正写回 AGENTS.md**，未来会话自动继承。配合 pre-commit / linter 强制执行。

---

## Memories — 自动生成的跨会话记忆

### 开启方式

**默认关闭**。App：**Settings > Personalization → Enable memories**；或 config：

```toml
# ~/.codex/config.toml
[features]
memories = true
```

### 存储结构

```
~/.codex/memories/
├── summaries            # 会话摘要
├── durable entries      # 持久条目
├── recent inputs        # 最近输入
└── supporting evidence  # 支撑证据
```

> [!warning] 当作生成状态
> 官方明确：这些文件**别手改当主要控制面**，只用于排查或分享前检查。强制规范放 AGENTS.md。

### 生成机制

| 特性 | 说明 |
|------|------|
| 生成时机 | 会话**空闲一段时间后后台生成**，不是结束立刻写 |
| 跳过规则 | 跳过进行中的/短会话 |
| 安全 | 自动脱敏 secrets |
| 额度保护 | rate-limit 剩余低于阈值时跳过生成 |

### 控制粒度

| 开关 | 作用 |
|------|------|
| `/memories` 命令 | 控制**当前会话**：能否读旧记忆 / 能否作为未来记忆来源 |
| `memories.use_memories` | 是否把已有记忆注入未来会话（读） |
| `memories.generate_memories` | 新会话是否作为记忆生成输入（写） |
| `memories.disable_on_external_context` | 用了 MCP/web search 的会话不进记忆 |
| `memories.extract_model` | 单会话抽取用的模型 |
| `memories.consolidation_model` | 全局合并用的模型 |

App / CLI / IDE 都可用 `/memories` 做会话级控制，不影响全局设置。

---

## Computer History（macOS 桌面端）

把跨应用活动（点击、输入、切应用）转成记忆和时间线，ChatGPT 和 Codex 可引用。

- 问"我刚才在干嘛"、"提案文档在哪"
- 从重复工作流里**建议生成 Skill**
- 不含截图、不含录音、不含无痕浏览
- 默认关闭，需单独授权（Business/Enterprise 需管理员先开通）

---

## 与 Claude Code 对比

| 维度 | Claude Code | Codex |
|------|-------------|-------|
| 人写指令 | `CLAUDE.md` | `AGENTS.md` |
| 自动记忆 | Auto memory（4 类） | Memories（摘要 + durable entries） |
| 存储位置 | `~/.claude/projects/<slug>/memory/` | `~/.codex/memories/` |
| **记忆作用域** | **按项目隔离**（换项目不互通） | **全局**（所有项目共享一份） |
| 索引机制 | `MEMORY.md` 索引 + 按需读正文 | 摘要/条目直接生成 |
| 默认状态 | 观察到就写 | **默认关闭** |
| 生成时机 | 对话中实时写 | 空闲后**后台批量生成** |
| 类型系统 | user / feedback / project / reference | 无显式分类 |
| 控粒度 | 说"记住/忘掉" | `/memories` + config 开关 |
| 配套能力 | Skills / hooks / subagents | Skills / MCP / Subagents / Computer History |

**一句话**：Codex 偏"指令分层 + 后台生成召回层"；Claude Code 偏"指令 + 分类明确的 auto memory"。

### 记忆作用域：全局 vs 按项目

Codex 的 Memories 是**整个 Codex 的全局个性化**，不是按项目隔离：

- 只有 `~/.codex/memories/` 一份，不按项目分目录
- 合并叫 **global consolidation**（`max_raw_memories_for_consolidation`）
- 项目级 `.codex/config.toml` 只能覆盖部分配置，**不能开独立记忆库**
- 在 A 项目学到的偏好，开 B 项目也会被注入

> [!note] 项目差异靠谁体现？
> Codex：Memories 管"你这个人的通用上下文"，项目差异靠 **AGENTS.md 分层**体现。
> Claude Code：auto memory 本身就按项目隔离，项目状态可以写进 project 类记忆。

全局生效的记忆参数：

| 参数 | 默认 | 说明 |
|------|------|------|
| `min_rollout_idle_hours` | 6h | 会话空闲多久后才生成记忆 |
| `max_rollout_age_days` | 30d | 只考虑 N 天内的会话 |
| `max_unused_days` | 30d | N 天没用到的记忆不参与合并 |
| `max_raw_memories_for_consolidation` | 256 | 全局合并最多保留的原始记忆条数 |

---

## 相关路径

- 全局指令：`~/.codex/AGENTS.md`
- 记忆目录：`~/.codex/memories/`
- 配置：`~/.codex/config.toml`
- 官方文档：[Memories](https://learn.chatgpt.com/docs/customization/memories) · [AGENTS.md](https://learn.chatgpt.com/docs/agent-configuration/agents-md) · [Customization](https://learn.chatgpt.com/docs/customization/overview)

---

## Q&A

## Q1：Codex 的记忆分哪两层？各存什么？

**答案**：
1. **AGENTS.md**：人写的持久指令，分层发现（全局 → 项目 → 子目录，近者优先），相当于 Claude Code 的 CLAUDE.md
2. **Memories**：Codex 自动生成的召回层，从历史会话提炼摘要和持久条目，存 `~/.codex/memories/`

**记忆**：**AGENTS.md 管规则，Memories 管召回；规则必须进指令文件**。

---

## Q2：Codex Memories 默认开启吗？什么时候生成？

**答案**：
- **默认关闭**，需 Settings > Personalization 开启或 config 里 `[features] memories = true`
- 不是会话结束立刻写，而是**空闲一段时间后后台生成**；跳过短会话、脱敏 secrets、额度低时跳过

**记忆**：**默认关、后台写、空闲才生成**。

---

## Q3：AGENTS.md 的发现顺序和优先级？

**答案**：
1. 全局 `~/.codex/`（`AGENTS.override.md` 优先于 `AGENTS.md`）
2. 从项目根走到 cwd，每层取 `AGENTS.override.md` > `AGENTS.md` > fallback
3. 从根到 cwd 拼接，==越靠近 cwd 越优先==
4. 合并上限默认 32 KiB

**记忆**：**全局到本地逐层拼，近者胜；32KB 封顶**。

---

## Q4：记忆能实现"每次 X 自动做 Y"吗？

**答案**：
不能。记忆只影响模型看到的上下文。事件触发的自动行为要用 hooks / Automations / Skills。
与 Claude Code 同理：==记忆管上下文，hooks 管自动行为==。

**记忆**：**记忆不触发动作，自动化找 hooks/Skills**。

---

## Q5：Codex 本地记忆是全局的还是按项目隔离？

**答案**：
**全局**。所有项目共享 `~/.codex/memories/` 一份，合并是 global consolidation。
换项目记忆照常注入；项目差异靠 AGENTS.md 分层体现。
（对比：Claude Code 的 auto memory 按项目目录隔离）

**记忆**：**Codex 记忆全局共享，项目差异交给 AGENTS.md**。
