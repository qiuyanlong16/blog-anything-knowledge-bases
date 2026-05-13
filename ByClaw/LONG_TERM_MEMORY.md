# QClaw 长期记忆架构 — 实地验证

> **调研日期**: 2026-05-13
> **数据来源**: lcm.db SQLite 数据库结构 + Hermes 工具定义 + openclaw.json 配置

---

## 一、核心结论：两套完全独立的记忆系统

QClaw 没有为两个内核设计统一的记忆层，而是各用各的：

| 维度 | OpenClaw | Hermes |
|------|----------|--------|
| **实现方式** | `lossless-claw` 插件 | Hermes 内置 `memory` 工具 |
| **触发机制** | 自动压缩（上下文 > 60% 阈值） | Agent 自主判断保存 |
| **存储格式** | SQLite（lcm.db，32 张表） | 未知（`memory` 工具内部，`memories/` 为空） |
| **记忆类型** | 对话压缩 + 多层摘要 + FTS 全文检索 | 声明式事实（facts） |
| **搜索能力** | 6 个记忆工具（lcm_grep/describe/expand/memory_search/memory_get） | `memory` + `session_search` |
| **上下文注入** | 自动（context_items + bootstrap） | 每轮自动注入 |

---

## 二、OpenClaw 长期记忆：lossless-claw 插件

### 2.1 配置

```json
// openclaw.json
{
  "plugins": {
    "slots": {
      "contextEngine": "lossless-claw"    // 记忆引擎挂载为上下文引擎 slot
    },
    "entries": {
      "lossless-claw": {
        "enabled": true,
        "config": {
          "dbPath": "/Users/qiududu/.qclaw/memory/lossless/lcm.db",
          "contextThreshold": 0.6,           // 上下文使用率达到 60% 时触发压缩
          "freshTailCount": 64,              // 保留最近 64 条消息不参与压缩
          "incrementalMaxDepth": 5,          // 增量检索最大摘要深度
          "freshTailMaxTokens": 40000        // 尾部最大 token 数
        }
      }
    }
  }
}
```

**关键设计**：
- 记忆引擎以"上下文引擎"（contextEngine）的形式挂载到插件系统的 `slots` 中
- 可替换：通过修改 `plugins.slots.contextEngine` 字段可换用其他引擎
- 配置可精细调优：阈值、保留条数、token 上限、检索深度

### 2.2 数据库结构（lcm.db，32 张表）

#### 核心数据表

| 表名 | 主键 | 用途 | 关键字段 |
|------|------|------|---------|
| `conversations` | `conversation_id` | 会话元数据 | `session_id`, `session_key`, `active`, `archived_at`, `title` |
| `messages` | `message_id` | 压缩后的消息 | `conversation_id`, `seq`, `role`, `content`, `token_count`, `identity_hash` |
| `summaries` | `summary_id` | 多层对话摘要 | `conversation_id`, `kind`, `depth`, `content`, `token_count`, `earliest_at`, `latest_at`, `descendant_count`, `descendant_token_count`, `model` |
| `message_parts` | `part_id` | 消息细节拆分 | `message_id`, `part_type`, `text_content`, `tool_name`, `tool_status`, `tool_input`, `tool_output`, `subtask_desc`, `step_cost`, `step_tokens_in/out` |
| `context_items` | (conversation_id, ordinal) | 上下文注入项 | `item_type` (message/summary), `message_id`, `summary_id` |
| `large_files` | `file_id` | 大文件缓存 | `conversation_id`, `file_name`, `mime_type`, `byte_size`, `exploration_summary` |

#### 状态管理表

| 表名 | 用途 |
|------|------|
| `conversation_bootstrap_state` | 会话引导状态：`session_file_path`, `last_processed_offset`, `last_processed_entry_hash` |
| `conversation_compaction_telemetry` | 压缩遥测：`cache_state`, `consecutive_cold_observations`, `retention`, `provider`, `model` |
| `conversation_compaction_maintenance` | 压缩维护队列：`pending`, `reason`, `token_budget`, `current_token_count` |
| `lcm_migration_state` | 迁移状态追踪 |

#### 全文搜索表

| 表名 | 用途 |
|------|------|
| `messages_fts` + 6 张辅助表 | 消息全文搜索（标准英文分词） |
| `summaries_fts` + 6 张辅助表 | 摘要全文搜索（标准英文分词） |
| `summaries_fts_cjk` + 6 张辅助表 | 摘要全文搜索（**中文分词**） |

### 2.3 记忆工具（6 个）

| 工具 | 用途 | 说明 |
|------|------|------|
| `lcm_grep` | 用正则/全文搜索压缩的对话历史 | 支持多字段搜索，含 metadata 过滤 |
| `lcm_describe` | 按 ID 查找 LCM 项目的元数据和内容 | 快速查看特定会话/摘要的概览 |
| `lcm_expand` | 展开压缩的对话摘要 | 将 summary 还原为详细消息 |
| `lcm_expand_query` | 用 LCM 扩展回答焦点自然语言问题 | 针对摘要提出具体问题 |
| `memory_search` | 语义搜索 MEMORY.md + memory/*.md | 跨会话的纯文本记忆搜索 |
| `memory_get` | 从 MEMORY.md 或 memory/*.md 读取片段 | 精确读取特定记忆条目 |

### 2.4 `.auto-memory/` 辅助文件

```json
// .auto-memory/cursor-agent_main_openclaw-weixin_direct_o9cq802ri1luwalnhdoldkyrare8_im_wechat.json
{"lastIndex": 0, "anchorHash": "", "updatedAt": 1778591217992}

// .auto-memory/cursor-agent_agent-3d43b911_session-1778591236009-xv3xlw.json
{"lastIndex": 0, "anchorHash": "", "updatedAt": 1778591263671}
```

**命名规则**：`cursor-agent_{agentId}_{sessionKey}.json`

**用途**：跟踪每个 agent+channel 的增量处理进度（lastIndex + anchorHash），避免重复处理已压缩的消息。

### 2.5 工作流程

```
┌──────────────────────────────────────────────────────────────┐
│  OpenClaw lossless-claw 记忆流程                               │
│                                                              │
│  1. 对话进行中                                                 │
│     消息逐行追加到 agents/{id}/sessions/{id}.jsonl            │
│                                                              │
│  2. 触发压缩（满足任一条件）                                     │
│     • 上下文 token > contextWindow × contextThreshold (60%)    │
│     • compaction_maintenance.pending = true                   │
│                                                              │
│  3. 压缩执行                                                   │
│     • 保留 fresh tail（最近 64 条消息，≤ 40000 tokens）        │
│     • 之前的消息按 depth 分层压缩为 summaries                   │
│     • 提取 tool_call 等细节到 message_parts                    │
│     • 写入 lcm.db（messages + summaries + context_items）      │
│     • 更新 FTS 索引（英文 + 中文分词）                          │
│     • 记录 telemetry（cache_state, provider, model）          │
│                                                              │
│  4. 下次对话时                                                 │
│     • bootstrap 注入 conversation 摘要                        │
│     • 根据 context_items 表组装上下文                           │
│     • AI 可通过 lcm_grep/lcm_describe 等工具按需检索历史         │
│                                                              │
│  5. 维护                                                       │
│     • compaction_maintenance 队列驱动后台压缩                   │
│     • telemetry 追踪缓存命中率，优化压缩策略                     │
└──────────────────────────────────────────────────────────────┘
```

### 2.6 多层摘要结构

```
Session Messages (原始对话)
    │
    ├── Fresh Tail (最近 64 条，保留原始)
    │
    └── Compressed (之前的消息)
         │
         ├── Summary depth=1 (近层摘要，覆盖最近 N 条)
         │    ├── descendant_count: 覆盖的消息数
         │    └── descendant_token_count: 覆盖的 token 数
         │
         ├── Summary depth=2 (中层摘要，聚合 depth=1)
         │
         └── Summary depth=3+ (远层摘要，高度压缩)
```

**关键参数**：
- `depth`：摘要层级，越高越压缩
- `descendant_count`：该摘要覆盖的原始消息数
- `descendant_token_count`：覆盖的原始 token 数
- `source_message_token_count`：被压缩的原始消息 token 数
- `model`：生成摘要使用的模型

---

## 三、Hermes 长期记忆：Agent 内置 memory 工具

### 3.1 记忆工具定义

Hermes 内置 26 个工具，与记忆相关的：

| 工具 | 用途 |
|------|------|
| `memory` | 保存持久化信息到跨会话记忆 |
| `session_search` | 搜索过去的对话记录 |
| `skill_manage` | 管理技能（创建/更新/删除技能） |
| `skill_view` | 查看技能内容 |
| `skills_list` | 列出可用技能 |

### 3.2 `memory` 工具行为

来自 Hermes 系统提示（system_prompt）：

```
You have persistent memory across sessions. Save durable facts using the memory tool:

WHEN TO SAVE (proactive, don't wait to be asked):
- User corrects you or says "remember this" / "don't do that again"
- User shares a preference, habit, or personal detail (name, role, timezone, coding style)
- You discover something about the environment (OS, installed tools, project structure)
- You learn a convention, API quirk, or workflow specific to this user's setup
- You identify a stable fact that will be useful again in future sessions

PRIORITY: User preferences and corrections > environment facts > procedural knowledge

Do NOT save:
- Task progress, session outcomes, completed-work logs, or temporary TODO state
- Procedures and workflows (those belong in skills, not memory)

FORMAT: Write memories as declarative facts, not instructions
  "User prefers concise responses" ✓
  "Always respond concisely" ✗
```

**关键设计原则**：
- **主动保存**：Agent 不等用户要求，自己判断是否需要保存
- **事实声明**：记忆是 declarative facts，不是指令
- **跨会话持久化**：每次对话开始时记忆自动注入
- **紧凑优先**：优先保存能减少用户重复指导的信息

### 3.3 数据存储

| 位置 | 状态 |
|------|------|
| `~/.qclaw-hermes/memories/` | 目录存在，当前为空（新 Agent 尚未触发记忆保存） |
| `~/.qclaw-hermes/state.db` | SQLite 数据库，无表结构（可能是 WAL 模式或尚未初始化） |
| `~/.qclaw-hermes/audit.db` | SQLite 数据库，无表结构 |
| `~/.qclaw-hermes/response_store.db` | SQLite 数据库，无表结构 |

**推测**：Hermes 的记忆可能由 `hermes` 二进制内部管理，可能：
1. 存储在 `memories/` 目录下（尚未触发）
2. 或通过 `state.db` 存储（WAL 模式在关闭后可能清空）
3. 或内嵌在 session JSON 的 `system_prompt` 中（每次对话注入）

### 3.4 工作流程

```
┌──────────────────────────────────────────────────────────────┐
│  Hermes Agent 自主记忆流程                                     │
│                                                              │
│  1. 对话进行中                                                 │
│     Agent 判断是否需要保存记忆：                                │
│     • 用户纠正了 AI 的行为                                      │
│     • 用户分享了个人偏好                                       │
│     • 发现了环境相关信息                                       │
│                                                              │
│  2. 调用 memory 工具保存                                       │
│     → 写入内部存储                                             │
│                                                              │
│  3. 下次对话时                                                 │
│     → 记忆自动注入到 system_prompt                             │
│     → Agent 可通过 session_search 搜索历史对话                  │
│     → 复杂技能可保存为 skill（skill_manage 工具）               │
└──────────────────────────────────────────────────────────────┘
```

---

## 四、两套系统的详细对比

### 4.1 架构对比

| 维度 | OpenClaw (lossless-claw) | Hermes (内置 memory) |
|------|--------------------------|---------------------|
| **归属** | 插件系统 (`plugins.slots.contextEngine`) | Agent 原生能力（内置工具） |
| **触发** | 被动自动（token 阈值触发） | 主动自主（Agent 判断） |
| **存储** | SQLite（lcm.db，32 张表） | 未知（Hermes 内部管理） |
| **记忆内容** | 压缩的对话历史 + 多层摘要 | 声明式事实（facts） |
| **检索** | FTS 全文搜索（英文 + 中文分词） | `session_search` 工具 |
| **注入方式** | context_items 表 + bootstrap 文件 | 每轮自动注入 system_prompt |
| **可扩展性** | 插件可替换（`plugins.slots.contextEngine`） | 固化在 Hermes 二进制中 |
| **开发者可见性** | 高（数据库结构完全可见） | 低（内部实现不透明） |
| **中文支持** | 有（`summaries_fts_cjk` 中文分词表） | 未知 |
| **工具数量** | 6 个 | 1 个（memory）+ 1 个（session_search） |

### 4.2 记忆内容对比

| 记忆类型 | OpenClaw | Hermes |
|----------|----------|--------|
| 用户偏好 | memory_search / memory_get | ✅ memory 工具主动保存 |
| 环境信息 | memory_search / memory_get | ✅ memory 工具主动保存 |
| 对话历史 | ✅ LCM 压缩 + 多层摘要 | ✅ session_search 搜索 |
| 程序性知识 | ❌（属于 skills） | ❌（保存到 skill_manage） |
| 工具调用记录 | ✅ message_parts 表记录 | ❌（不保存） |
| 任务进度 | ❌（不保存） | ❌（明确不保存） |

### 4.3 数据流对比

```
OpenClaw 记忆流:
用户消息 → JSONL → LCM 引擎自动压缩 → lcm.db (SQLite)
                                            ↓
                              context_items + bootstrap 注入
                                            ↓
                              AI 通过 lcm_grep 等工具检索


Hermes 记忆流:
用户消息 → Agent 自主判断 → memory 工具 → 内部存储
                                            ↓
                              每轮自动注入 system_prompt
                                            ↓
                              AI 通过 session_search 搜索
```

---

## 五、对我们构建产品的启示

### 5.1 设计建议

| 设计点 | 推荐方案 | 原因 |
|--------|---------|------|
| **记忆层独立性** | 设计独立的记忆服务，两个内核共享 | 避免数据割裂，支持跨 Agent 记忆 |
| **触发机制** | 混合模式：自动压缩 + 主动保存 | OpenClaw 太被动（只在上下文满时压缩），Hermes 依赖 AI 判断 |
| **存储格式** | SQLite + FTS + JSON 混合 | LCM 的 32 张表太复杂，但 FTS + 摘要压缩是好的设计 |
| **检索能力** | 语义搜索 + FTS 全文搜索 | LCM 有中文 FTS 支持，Hermes 只有工具搜索 |
| **记忆分类** | 分三层：事实记忆、对话摘要、程序性知识 | 当前两套系统都无法清晰区分 |

### 5.2 参考架构

```
┌──────────────────────────────────────────────────────────────┐
│  统一记忆服务 (MemoryService)                                 │
│                                                              │
│  ├── 事实记忆层 (Fact Memory)                                 │
│  │   • 用户偏好、环境信息、约定规则                             │
│  │   • 类似 Hermes 的 memory 工具                              │
│  │   • 存储：轻量 KV / SQLite                                 │
│  │   • 每轮自动注入                                            │
│  │                                                           │
│  ├── 对话压缩层 (Conversation Compression)                     │
│  │   • 类似 OpenClaw 的 LCM 引擎                               │
│  │   • 自动压缩历史对话为多层摘要                               │
│  │   • 存储：SQLite + FTS（含中文分词）                         │
│  │   • 工具：grep/describe/expand                              │
│  │                                                           │
│  └── 技能记忆层 (Skill Memory)                                 │
│      • 类似 Hermes 的 skill_manage                             │
│      • 程序性知识保存为可执行技能                               │
│      • 存储：文件系统 / 数据库                                  │
└──────────────────────────────────────────────────────────────┘
```

### 5.3 关键取舍

| 取舍 | OpenClaw 方式 | Hermes 方式 | 推荐 |
|------|-------------|------------|------|
| 压缩触发 | 自动（阈值） | 手动（AI 判断） | **混合**：自动压缩对话 + AI 主动保存事实 |
| 检索方式 | 工具调用 | 系统自动注入 | **混合**：事实自动注入，历史按需检索 |
| 记忆粒度 | 消息级 | 事实级 | **分层**：事实（小粒度）+ 摘要（大粒度） |
| 可扩展性 | 插件可替换 | 固化 | **插件化**：记忆引擎可替换 |
