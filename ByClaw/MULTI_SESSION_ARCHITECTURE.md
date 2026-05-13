# QClaw 多会话架构 — 实地验证（第二轮）

> **调研日期**: 2026-05-13
> **数据来源**: QClaw Mac 实际用户数据目录 + 安装包解压（`~/.qclaw` + `~/.qclaw-hermes` + `Resources/`）
> **App 版本**: 0.2.18 (`@guanjia-openclaw/electron`)

---

## 一、核心发现：OpenClaw 内核支持多 Agent

第一轮调研时，`~/.qclaw/agents/` 目录下只有一个 `main` agent。第二轮数据证实 OpenClaw 内核**完整支持多 Agent 架构**：

```
~/.qclaw/agents/
├── main/                          ← 默认 Agent（QClaw）
│   ├── agent/models.json          ← 该 Agent 的模型配置
│   └── sessions/
│       ├── sessions.json          ← 该 Agent 的会话索引
│       └── *.jsonl                ← 会话数据
└── agent-3d43b911/                ← 自定义 Agent（无不言）
    ├── agent/models.json          ← 该 Agent 的模型配置
    └── sessions/
        ├── sessions.json          ← 该 Agent 的会话索引
        └── *.jsonl                ← 会话数据
```

**每个 Agent 有独立的**：
- 模型配置（`agent/models.json`）
- 会话索引（`sessions/sessions.json`）
- 会话文件（`sessions/*.jsonl`）
- 技能列表（`skills` 数组，在 `openclaw.json` 中定义）
- 工作区目录（`workspace` 字段，默认 `~/.qclaw/workspace`，自定义 Agent 可指定 `~/.qclaw/workspace-{agentId}`）

**但共享**：
- 同一个 `openclaw.json` 配置文件（`agents.list` 数组定义所有 Agent）
- 同一个 OpenClaw Gateway 进程（端口 28789）
- 同一个本地模型代理（127.0.0.1:19000）
- 同一个 Skills 目录（`~/.qclaw/skills/`）
- 同一个 Plugins 配置（`plugins` 段）

### 1.1 openclaw.json 多 Agent 定义

```json
{
  "agents": {
    "defaults": {
      "model": { "primary": "qclaw/modelroute" },
      "workspace": "/Users/qiududu/.qclaw/workspace",
      "maxConcurrent": 3,
      "timeoutSeconds": 72000
    },
    "list": [
      {
        "id": "main",
        "name": "QClaw",
        "identity": { "name": "QClaw" },
        "skills": ["find-skills", "qclaw-rules", "qclaw-env", ...]
      },
      {
        "id": "agent-3d43b911",
        "name": "无不言",
        "workspace": "/Users/qiududu/.qclaw/workspace-agent-3d43b911",
        "identity": {
          "name": "无不言",
          "emoji": "📚",
          "avatar": "https://..."
        },
        "agentDir": "/Users/qiududu/.qclaw/agents/agent-3d43b911/agent",
        "skills": [
          "content-factory", "docx", "pptx", "pdf", "xlsx",
          "multi-search-engine", "self-improving", "browser-cdp", ...
        ]
      }
    ]
  }
}
```

### 1.2 Channel → Agent 绑定机制

```json
{
  "bindings": [
    {
      "agentId": "main",
      "match": {
        "channel": "openclaw-weixin",
        "accountId": "*"
      }
    }
  ]
}
```

**规则**：来自 `openclaw-weixin` channel 的所有消息自动路由到 `main` agent。未匹配 binding 的消息（如 webchat）走默认路由。

### 1.3 Channel 默认映射（channel-defaults.json）

```json
{
  "main": {
    "openclaw-weixin": {
      "to": "o9cq802Ri1lUwALNHdoLDKyrARE8@im.wechat"
    }
  }
}
```

记录每个 agent 的 channel 默认目标用户。

---

## 二、Hermes 内核：单 Agent 设计

Hermes 内核目前仅支持单 Agent（`agentId: "hermes_default"`）：

```
~/.qclaw-hermes/
├── config.yaml         ← 配置（YAML，由 QClaw 自动生成）
├── agent.json          ← 唯一的 Agent 元数据
├── SOUL.md             ← 角色设定
├── sessions/           ← 会话 JSON 文件
│   └── session_{id}.json
├── gateway_state.json  ← Gateway 进程状态
├── state.db            ← 状态存储
├── audit.db            ← 审计日志
├── response_store.db   ← 响应存储
└── ...
```

**与 OpenClaw 的关键差异**：

| 维度 | OpenClaw | Hermes |
|------|----------|--------|
| Agent 数量 | 多 Agent（`agents.list` 数组） | 单 Agent（`hermes_default`） |
| 配置格式 | JSON（`openclaw.json`） | YAML（`config.yaml`） |
| Agent 元数据 | `agents/{id}/agent/models.json` + `openclaw.json` list 段 | `agent.json`（包含 name/vibe/bio/skills） |
| 会话索引 | `agents/{id}/sessions/sessions.json`（per-agent） | `agent.json` 内嵌 sessionIds/sessionTitles/sessionUpdatedAts |
| 会话格式 | JSONL 逐行追加 | 完整 JSON 文件（含 system_prompt + tools + messages） |
| 角色设定 | workspace/ 下的 AGENTS.md、SOUL.md、IDENTITY.md 等文件 | agent.json bio 段 + SOUL.md |
| 工作区 | `~/.qclaw/workspace`（可 per-agent 指定） | `~/.qclaw-hermes/workspace` |
| Gateway | 共享进程 + 端口 28789 | 独立进程 + `gateway_state.json` 跟踪 |
| 插件系统 | `plugins` 段（wechat-access, lossless-claw, browser 等） | `plugins.enabled`（qclaw-plugin-hermes） |
| 技能系统 | `skills.entries` + `skills.extraDirs` | `agent.json` skills 数组 |
| 模型配置 | `agents.{id}/agent/models.json` per-agent | `config.yaml` model 段 |
| 消息队列 | `messages.queue.mode: "followup"` | 内建 |
| 长期记忆 | `memory/lossless/lcm.db`（lossless-claw 插件） | `memories/` 目录（当前为空） + 内置 memory 工具 |

---

## 三、会话索引格式详解

### 3.1 OpenClaw 会话索引（per-agent sessions.json）

**Session Key 格式**：`agent:{agentId}:{channel}:{chatType}:{from}`

```json
{
  "agent:main:openclaw-weixin:direct:o9cq802ri1luwalnhdoldkyrare8@im.wechat": {
    "origin": {
      "label": "o9cq802Ri1lUwALNHdoLDKyrARE8@im.wechat",
      "provider": "openclaw-weixin",
      "chatType": "direct",
      "from": "o9cq802Ri1lUwALNHdoLDKyrARE8@im.wechat",
      "to": "o9cq802Ri1lUwALNHdoLDKyrARE8@im.wechat",
      "accountId": "23911165a2ad-im-bot"
    },
    "sessionId": "8bc4b975-3cc3-41b2-98da-74a0d8177911",
    "updatedAt": 1778591521938,
    "systemSent": true,
    "chatType": "direct",
    "deliveryContext": {
      "channel": "openclaw-weixin",
      "to": "o9cq802Ri1lUwALNHdoLDKyrARE8@im.wechat",
      "accountId": "23911165a2ad-im-bot"
    },
    "lastChannel": "openclaw-weixin",
    "sessionFile": "/Users/qiududu/.qclaw/agents/main/sessions/8bc4b975-....jsonl",
    "skillsSnapshot": {
      "prompt": "...<available_skills>...",
      "skills": [{"name": "find-skills"}, ...],
      "skillFilter": ["find-skills", "qclaw-rules", ...],
      "resolvedSkills": [{"name": "find-skills", "filePath": "...", "source": "openclaw-extra"}, ...],
      "version": 0
    },
    "status": "done",
    "startedAt": 1778591501538,
    "endedAt": 1778591521875,
    "runtimeMs": 20337,
    "modelProvider": "qclaw",
    "model": "modelroute",
    "contextTokens": 200000,
    "systemPromptReport": {
      "workspaceDir": "/Users/qiududu/.qclaw/workspace",
      "injectedWorkspaceFiles": [
        {"name": "AGENTS.md", "path": "...", "rawChars": 7727, ...},
        {"name": "SOUL.md", "path": "...", "rawChars": 20, ...},
        ...
      ],
      "skills": {"promptChars": 4668, "entries": [...]},
      "tools": {"schemaChars": 21487, "entries": [...]}
    }
  },
  "agent:main:main": {
    "updatedAt": 1778591501083,
    "sessionId": "64a72002-ed29-4396-9176-7edd0700dcaf"
  }
}
```

**关键发现**：
- Session Key 包含完整的 channel 信息（`agent:{agentId}:{channel}:{chatType}:{from}`）
- `skillsSnapshot` 记录了技能解析的完整快照（prompt、resolved skills、skillFilter）
- `systemPromptReport` 记录了 system prompt 生成的详细信息（注入的文件、字符数、工具数）
- 每个 agent 有独立的 `sessions.json`

### 3.2 Hermes 会话索引（agent.json 内嵌）

```json
{
  "agentId": "hermes_default",
  "name": "林且慢",
  "vibe": "少年爹系的辅导员",
  "bio": [
    {"title": "经历", "desc": "大学辅导员，管理184个学生..."},
    {"title": "风格", "desc": "温润如玉..."}
  ],
  "skills": ["niuamaxia-scheduler", "tencent-meeting-mcp", "docx", ...],
  "sessionIds": ["04a1cd54-48bb-4f5b-8877-727d5dc8a30d"],
  "sessionTitles": {
    "04a1cd54-48bb-4f5b-8877-727d5dc8a30d": "你是 hermes agent的机器人 [c8a30d]"
  },
  "sessionUpdatedAts": {
    "04a1cd54-48bb-4f5b-8877-727d5dc8a30d": 1778591428511
  },
  "createdAt": 1778591307048,
  "updatedAt": 1778591428511
}
```

### 3.3 OpenClaw JSONL 会话数据格式

```jsonl
{"type":"session","version":3,"id":"8bc4b975-...","timestamp":"2026-05-12T13:06:48.601Z","cwd":"/Users/qiududu/.qclaw/workspace"}
{"type":"model_change","id":"8512ab77","parentId":null,"timestamp":"...","provider":"qclaw","modelId":"modelroute"}
{"type":"thinking_level_change","id":"ddbf9369","parentId":"8512ab77","timestamp":"...","thinkingLevel":"off"}
{"type":"custom","customType":"model-snapshot","data":{"timestamp":...,"provider":"qclaw","modelApi":"openai-completions","modelId":"modelroute"},"id":"fed5b6ba","parentId":"ddbf9369","timestamp":"..."}
{"type":"message","id":"086e1013","parentId":"fed5b6ba","timestamp":"...","message":{"role":"user","content":[{"type":"text","text":"...你好"}],"timestamp":1778591208640}}
{"type":"message","id":"71997456","parentId":"086e1013","timestamp":"...","message":{"role":"assistant","content":[{"type":"thinking","thinking":"...","thinkingSignature":"reasoning_content"},{"type":"text","text":"你好！我是 QClaw..."}],"api":"openai-completions","provider":"qclaw","model":"modelroute","usage":{"input":0,"output":0,...},"stopReason":"stop","timestamp":1778591208655,"responseId":"fb9811b72a2b46aa"}}
```

### 3.4 Hermes JSON 会话数据格式

```json
{
  "session_id": "b2725658-95c4-4041-9a1c-313340d911b1",
  "model": "modelroute",
  "base_url": "http://127.0.0.1:19000/proxy/llm",
  "platform": "api_server",
  "session_start": "2026-05-12T21:10:28.612689",
  "last_updated": "2026-05-12T21:10:36.994506",
  "system_prompt": "# 身份\n\n你的名称是「林且慢」\n\n> 少年爹系的辅导员\n...",
  "tools": [
    {"type": "function", "function": {"name": "browser_back", ...}},
    {"type": "function", "function": {"name": "browser_click", ...}},
    ...
  ],
  "messages": [
    {"role": "user", "content": "你好"},
    {"role": "assistant", "content": "...", "reasoning": "...", "reasoning_content": "..."}
  ]
}
```

---

## 四、Plugin 插件系统

### 4.1 OpenClaw 插件配置

```json
{
  "plugins": {
    "enabled": true,
    "allow": ["wechat-access", "qclaw-plugin", "lossless-claw", "browser", "openclaw-weixin"],
    "slots": {
      "contextEngine": "lossless-claw"
    },
    "load": {
      "paths": ["/Users/qiududu/Library/Application Support/QClaw/openclaw/config/extensions"]
    },
    "entries": {
      "wechat-access": {
        "enabled": true,
        "config": {}
      },
      "lossless-claw": {
        "enabled": true,
        "config": {
          "dbPath": "/Users/qiududu/.qclaw/memory/lossless/lcm.db",
          "contextThreshold": 0.6,
          "freshTailCount": 64,
          "incrementalMaxDepth": 5,
          "freshTailMaxTokens": 40000
        }
      },
      "openclaw-weixin": {
        "enabled": true,
        "config": {}
      }
    }
  }
}
```

**关键插件**：
| 插件 | 用途 |
|------|------|
| `wechat-access` | 微信接入（WebSocket） |
| `lossless-claw` | 长期记忆系统（压缩对话历史 + 语义检索） |
| `browser` | 浏览器自动化 |
| `openclaw-weixin` | 微信本地传输 |
| `qclaw-plugin` | QClaw 产品插件 |

### 4.2 Hermes 插件

```yaml
plugins:
  enabled:
    - qclaw-plugin-hermes
```

Hermes 插件以 Python 包形式打包在 `Resources/hermes-plugins/qclaw-plugin-hermes/`：
```
qclaw-plugin-hermes/
├── adapter/          ← 适配层（dispatch.py, register.py）
├── core_kit/         ← 核心工具包（config_center, ctx, logger, telemetry_bridge, types）
├── modules/          ← 功能模块
└── plugin.yaml       ← 插件描述
```

---

## 五、安装包结构

```
QClaw.app/Contents/Resources/
├── app.asar                    ← Electron 主进程代码（V8 字节码编译，107MB）
├── app.asar.unpacked/
│   └── node_modules/
│       ├── @tencent/           ← 腾讯 SDK（QQ Bot 等）
│       └── better-sqlite3/     ← SQLite 原生模块
├── app-update.yml              ← 自动更新配置
├── channel.json                ← 当前渠道（{"channel": 5001}）
├── hermes.tar.gz               ← Hermes 运行时（126MB）
├── hermes-plugins/             ← Hermes 插件（Python 包）
│   └── qclaw-plugin-hermes/
├── openclaw.tar.gz             ← OpenClaw 运行时（162MB）
├── node/node                   ← 内置 Node.js 二进制（110MB）
├── scripts/
│   ├── pack-qclaw.cjs          ← 问题反馈打包脚本
│   └── unpack-openclaw.cjs     ← 安装时 OpenClaw tar 解压脚本
├── icon.icns                   ← 应用图标
└── oauth-assets/               ← OAuth 认证资源
```

### 5.1 代码保护：V8 字节码编译

```javascript
// out/main/index.cjs
"use strict";
require("./bytecode-loader.cjs");
require("./index.cjsc");
```

主进程代码使用 V8 字节码编译（`.cjsc` 扩展），通过自定义 `Module._extensions` 加载器反序列化。字节码加载器（`bytecode-loader.cjs`）使用 `vm.Script` + `cachedData` 机制加载编译后的代码。

### 5.2 OpenClaw Bootstrap 机制

```typescript
// openclaw-bootstrap.mjs
// 1. 启动父进程看门狗（通过 QCLAW_PARENT_PID 环境变量）
// 2. 动态导入真正的 Gateway 入口（通过 QCLAW_REAL_ENTRY 环境变量）
// 3. 支持 MemoryFS 注册（通过 QCLAW_MEMORY_FS_REGISTER 环境变量）
```

### 5.3 运行时打包与解压

- **安装时**：`unpack-openclaw.cjs` 解压 `openclaw.tar.gz` 到 `resources/openclaw/` 目录
- **支持多分片格式**：`openclaw_0.tar` ~ `openclaw_N.tar` + `openclaw_manifest.json`
- **问题反馈**：`pack-qclaw.cjs` 将 `~/.qclaw` + `~/.qclaw-hermes` + 日志打包为 ZIP

---

## 六、双内核完整目录结构

### 6.1 OpenClaw 数据目录（~/.qclaw/）

```
~/.qclaw/
├── openclaw.json                      # 核心配置（JSON）
│                                      #   - agents.defaults: 全局默认
│                                      #   - agents.list: [多 Agent 定义数组]
│                                      #   - skills: extraDirs + entries 开关
│                                      #   - models: providers 定义 + merge 模式
│                                      #   - plugins: 插件系统 + slots
│                                      #   - channels: channel 配置
│                                      #   - bindings: channel → agent 绑定规则
│                                      #   - browser: 浏览器自动化配置
│                                      #   - gateway: 端口/模式/认证
│                                      #   - session: 重置策略 + 维护策略
│                                      #   - meta: 版本追踪
├── openclaw.json.bak                  # 自动备份
├── openclaw.json.last-good            # 最后良好配置
├── qclaw.json                         # QClaw 产品配置
│                                      #   - authGatewayBaseUrl: 127.0.0.1:19000
│                                      #   - sharedParams: guid, appVersion, platform
│                                      #   - cli: nodeBinary, openclawMjs, pid
│                                      #   - stateDir, port
├── channel-defaults.json              # channel 默认目标映射
│                                      #   - {agentId: {channel: {to: userId}}}
├── agent-skill-usage.json             # Agent 技能使用统计
│
├── agents/                            # Agent 目录（多 Agent 支持）
│   ├── main/                          # 默认 Agent
│   │   ├── agent/
│   │   │   └── models.json            # Agent 专属模型配置
│   │   │                              #   - providers: {qclaw, deepseek, ...}
│   │   │                              #   - 每个 provider 有 baseUrl, apiKey, models[]
│   │   └── sessions/
│   │       ├── sessions.json          # 会话索引（per-agent）
│   │       │                          #   - Key: "agent:{agentId}:{channel}:{chatType}:{from}"
│   │       │                          #   - Value: {sessionId, origin, deliveryContext,
│   │       │                          #             skillsSnapshot, systemPromptReport, ...}
│   │       └── *.jsonl                # 会话数据（JSONL 逐行追加）
│   │
│   └── agent-{uuid}/                  # 自定义 Agent
│       ├── agent/
│       │   └── models.json            # 该 Agent 的模型配置
│       └── sessions/
│           ├── sessions.json          # 该 Agent 的会话索引
│           └── *.jsonl                # 该 Agent 的会话数据
│
├── skills/                            # Skills 目录
│   ├── find-skills/
│   ├── online-search/
│   ├── qclaw-rules/
│   ├── qclaw-env/
│   └── ...
├── plugins/                           # Plugins 目录
│   ├── prompt-optimizer/
│   └── wechat-access/
│
├── memory/lossless/lcm.db             # 长期记忆（SQLite）
├── flows/registry.sqlite              # 流程注册表（SQLite）
├── devices/paired.json                # 已配对设备
│
├── workspace/                         # 默认工作区（main agent）
│   ├── AGENTS.md
│   ├── SOUL.md
│   ├── TOOLS.md
│   ├── IDENTITY.md
│   ├── USER.md
│   └── HEARTBEAT.md
├── workspace-agent-{id}/             # 自定义 Agent 工作区
│   └── ...
│
├── canvas/index.html                  # Canvas 页面
├── logs/                              # 配置审计日志
│   ├── config-audit.jsonl
│   └── config-health.json
├── backups/                           # 配置备份
│   └── openclaw.{timestamp}.json
├── compile-cache/                     # V8 编译缓存
│   └── v{version}-{arch}-{hash}/
└── .auto-memory/                      # 自动记忆文件
    └── cursor-agent_{id}_{session}.json
```

### 6.2 Hermes 数据目录（~/.qclaw-hermes/）

```
~/.qclaw-hermes/
├── config.yaml                        # Hermes 配置（YAML）
│                                      #   - model: default, base_url, api_key, provider
│                                      #   - terminal: cwd
│                                      #   - plugins.enabled: [qclaw-plugin-hermes]
│                                      # 注：model/terminal 段由 QClaw 同步生成
│
├── agent.json                         # 唯一的 Agent 元数据（JSON）
│                                      #   - agentId: "hermes_default"
│                                      #   - name, vibe, avatar, emoji
│                                      #   - bio: [{title, desc}, ...]
│                                      #   - skills: [技能名数组]
│                                      #   - sessionIds: [会话ID数组]
│                                      #   - sessionTitles: {id: title}
│                                      #   - sessionUpdatedAts: {id: timestamp}
│                                      #   - createdAt, updatedAt
│
├── SOUL.md                            # 角色设定（Markdown）
│                                      #   - 身份 / 经历 / 风格
│
├── .skills_prompt_snapshot.json       # Skills 提示快照
│
├── sessions/                          # 会话目录
│   └── session_{uuid}.json            # 会话数据（完整 JSON）
│                                      #   - session_id, model, base_url, platform
│                                      #   - session_start, last_updated
│                                      #   - system_prompt: 完整 Markdown（含身份设定）
│                                      #   - tools: [function calling schema]
│                                      #   - messages: [{role, content, reasoning?, ...}]
│
├── state.db                           # 状态存储（SQLite）
├── audit.db                           # 审计日志（SQLite）
├── response_store.db                  # 响应存储（SQLite）
│
├── gateway_state.json                 # Gateway 进程状态
│                                      #   - pid, kind: "hermes-gateway"
│                                      #   - argv: [启动命令]
│                                      #   - gateway_state: "running"
│                                      #   - active_agents: 0
│                                      #   - platforms: {api_server: {state: "connected"}}
├── gateway.pid                        # Gateway PID
├── gateway.lock                       # Gateway 锁
│
├── logs/                              # 日志
│   ├── agent.log
│   └── errors.log
│
├── skills/                            # Hermes Skills
├── memories/                          # 记忆
├── platforms/                         # 平台集成
│   └── pairing/
├── cron/                              # 定时任务
│   ├── .tick.lock
│   └── output/
├── bin/tirith                         # 二进制工具
├── channel_directory.json             # 频道目录
│                                      #   - platforms: {telegram: [], discord: [], ...}
└── workspace/                         # 工作区
```

---

## 七、共享本地模型代理

两个内核**共享同一个本地代理**：

| 配置项 | OpenClaw | Hermes |
|--------|----------|--------|
| Provider | `qclaw` | `custom` |
| Base URL | `http://127.0.0.1:19000/proxy/llm` | `http://127.0.0.1:19000/proxy/llm` |
| API Key | `__QCLAW_AUTH_GATEWAY_MANAGED__` | `__QCLAW_AUTH_GATEWAY_MANAGED__` |
| 默认模型 | `qclaw/modelroute` | `modelroute` |

**关键设计**：本地代理（127.0.0.1:19000）由 Electron 主进程启动和管理，聚合多个上游 LLM Provider（Kimi/GLM/Minimax/DeepSeek）。两个 Agent 内核对此代理完全透明，不感知上游 Provider 的存在。

---

## 八、前端会话加载架构

### 8.1 Agent 发现

```
前端 Agent 发现流程:
─────────────────────────────────────────────────────────────┐
│  1. 读取 openclaw.json → agents.list                       │
│     → [{ id: 'main', name: 'QClaw', kernel: 'openclaw' }, │
│        { id: 'agent-3d43b911', name: '无不言',            │
│          kernel: 'openclaw' }]                             │
│                                                            │
│  2. 读取 .qclaw-hermes/agent.json                          │
│     → [{ id: 'hermes_default', name: '林且慢',            │
│          kernel: 'hermes' }]                               │
│                                                            │
│  3. 合并 → 前端 Agent 选择器                                │
└─────────────────────────────────────────────────────────────┘
```

### 8.2 会话加载

```
前端加载会话列表:
─────────────────────────────────────────────────────────────┐
│  for each agent in agentList:                              │
│                                                            │
│  if (kernel === 'openclaw'):                               │
│    → 读取 ~/.qclaw/agents/{agentId}/sessions/sessions.json │
│    → 解析 JSON → 提取 sessionId, label/skillsSnapshot,     │
│      modelProvider, model, updatedAt                      │
│                                                            │
│  if (kernel === 'hermes'):                                 │
│    → 读取 ~/.qclaw-hermes/agent.json                       │
│    → 提取 sessionIds, sessionTitles, sessionUpdatedAts     │
│                                                            │
│  → 归一化为 UnifiedSession 格式返回前端                     │
└─────────────────────────────────────────────────────────────┘
```

### 8.3 消息发送路由

```
前端发送消息:
─────────────────────────────────────────────────────────────┐
│  sendMessage(agentId, kernel, sessionId, content)          │
│       │                                                    │
│       ▼                                                    │
│  if (kernel === 'openclaw'):                               │
│    → 检查 bindings 规则 → 确定目标 agentId                 │
│    → 路由到 OpenClaw Gateway（端口 28789）                 │
│    → 写入 JSONL 到 agents/{agentId}/sessions/{id}.jsonl   │
│                                                            │
│  if (kernel === 'hermes'):                                 │
│    → 路由到 Hermes Gateway 进程                            │
│    → 写入 JSON 到 sessions/session_{id}.json              │
└─────────────────────────────────────────────────────────────┘
```

### 8.4 统一数据结构

```typescript
// packages/shared/src/types/session.ts

// 前端收到的统一 Agent 格式
export interface AgentInfo {
  id: string;                // 'main' | 'agent-3d43b911' | 'hermes_default'
  name: string;              // 'QClaw' | '无不言' | '林且慢'
  kernel: 'openclaw' | 'hermes';
  vibe?: string;             // Hermes 特有：风格标签
  emoji?: string;            // Hermes 特有
  avatar?: string;           // OpenClaw/Hermes 都有
  skills: string[];          // 已启用的技能列表
}

// 前端收到的统一会话格式
export interface UnifiedSession {
  id: string;
  title: string;
  agentId: string;           // 所属 Agent
  kernel: 'openclaw' | 'hermes';
  updatedAt: number;
  model?: string;
  modelProvider?: string;
  channel?: string;          // OpenClaw 特有
  messages: UnifiedMessage[];
}

// 前端收到的统一消息格式
export interface UnifiedMessage {
  id: string;
  role: 'user' | 'assistant' | 'system';
  content: string;
  reasoning?: string;
  modelId?: string;
  provider?: string;
  timestamp: number;
  isStreaming?: boolean;
}
```

---

## 九、与第一轮推断的对比修正

| 之前推断 | 第二轮验证 | 修正状态 |
|----------|-----------|---------|
| OpenClaw 单 Agent（agents/main/） | OpenClaw 多 Agent（agents.list 数组 + per-agent 目录） | 更新 |
| Session Key: `agent:main:session-xxx` | Session Key: `agent:{agentId}:{channel}:{chatType}:{from}` | 更新 |
| 不存在 agents.json 注册表 | 注册表在 openclaw.json 的 `agents.list` 段 | 部分修正 |
| Hermes 单 Agent | 确认：Hermes 仅支持单 Agent | 不变 |
| 共享本地代理 | 确认：两者都走 127.0.0.1:19000 | 不变 |
| JSONL vs JSON 格式差异 | 确认：格式完全不同 | 不变 |
| 前端探测双目录 | 改为：解析 openclaw.json agents.list + 读取 agent.json | 修正 |
| skillsSnapshot 结构 | 完整快照：prompt + skills + skillFilter + resolvedSkills | 新增 |
| systemPromptReport 结构 | 完整报告：注入文件 + 字符数 + 技能 + 工具 | 新增 |
| Plugin 插件系统 | lossless-claw 提供长期记忆能力 | 新增 |
| Channel 绑定机制 | bindings 数组定义 channel → agent 路由规则 | 新增 |
| App 代码保护 | V8 字节码编译（.cjsc） | 新增 |
| Hermes 插件 | Python 包（qclaw-plugin-hermes） | 新增 |

### 核心修正结论

**QClaw 的多会话管理不是"双 Agent 并存"，而是"OpenClaw 多 Agent + Hermes 单 Agent"**：

```
前端 Agent 选择器:
┌─────────────────────────────────────────────────────────────┐
│  ● QClaw (main)                                              │
│    内核: OpenClaw (Node.js)                                  │
│    数据: ~/.qclaw/agents/main/sessions/sessions.json         │
│    技能: 7 个（find-skills, qclaw-rules, qclaw-env, ...）   │
│    Channel: openclaw-weixin（binding 规则）                  │
│                                                              │
│  ● 无不言 (agent-3d43b911)                                   │
│    内核: OpenClaw (Node.js)                                  │
│    数据: ~/.qclaw/agents/agent-3d43b911/sessions/sessions.json│
│    技能: 24 个（docx, pptx, pdf, xlsx, browser-cdp, ...）   │
│    Channel: webchat                                          │
│                                                              │
│  ○ 林且慢 (hermes_default)                                   │
│    内核: Hermes (Python)                                     │
│    数据: ~/.qclaw-hermes/agent.json                          │
│    技能: 15 个（niuamaxia-scheduler, tencent-meeting, ...） │
│    Channel: api_server                                       │
└─────────────────────────────────────────────────────────────┘
```

> 详细的长期记忆架构对比（OpenClaw lossless-claw vs Hermes 内置 memory）见 **[LONG_TERM_MEMORY.md](LONG_TERM_MEMORY.md)**。
