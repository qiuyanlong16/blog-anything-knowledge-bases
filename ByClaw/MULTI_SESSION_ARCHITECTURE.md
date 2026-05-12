# QClaw 多会话架构 — 实地验证

> **调研日期**: 2026-05-12
> **数据来源**: QClaw Mac 实际用户数据目录（`~/.qclaw` + `~/.qclaw-hermes`）

---

## 一、核心发现：完全独立的数据目录

QClaw 没有用统一目录管理多内核，而是**每个内核有完全独立的数据目录**：

```
用户数据根目录 (~)
├── .qclaw/              ← OpenClaw 内核的完整数据目录
│   ├── openclaw.json    ← OpenClaw 核心配置
│   ├── qclaw.json       ← QClaw 产品配置（auth/端口/CLI 路径）
│   ├── agents/          ← OpenClaw Agent 目录
│   │   ├── main/        ← 默认 Agent（OpenClaw 内核）
│   │   │   ├── agent/   ← Agent 元数据（models.json 等）
│   │   │   └── sessions/← 会话 JSONL 文件
│   │   └── __creating__/← Agent 创建中的临时目录
│   ├── skills/          ← OpenClaw Skills
│   ├── plugins/         ← OpenClaw Plugins
│   ├── memory/          ← 长期记忆（SQLite）
│   ├── workspace/       ← 工作区文件
│   ├── flows/           ← 流程注册表（SQLite）
│   └── ...
│
└── .qclaw-hermes/       ← Hermes 内核的完整数据目录
    ├── config.yaml      ← Hermes 配置
    ├── agent.json       ← Hermes Agent 元数据
    ├── SOUL.md          ← 角色设定
    ├── sessions/        ← 会话 JSON 文件（每个会话一个 JSON）
    ├── skills/          ← Hermes Skills
    ├── state.db         ← 状态 SQLite
    ├── audit.db         ← 审计 SQLite
    ├── response_store.db← 响应存储 SQLite
    └── ...
```

### 关键验证点

| 发现 | 证据 |
|------|------|
| **两个内核数据完全隔离** | `.qclaw/` 和 `.qclaw-hermes/` 互不共享任何文件 |
| **配置格式不同** | OpenClaw: `openclaw.json` (JSON)；Hermes: `config.yaml` (YAML) |
| **会话存储格式不同** | OpenClaw: JSONL 逐行追加；Hermes: 完整 JSON 文件 |
| **Agent 元数据不同** | OpenClaw: `agents/main/agent/models.json`；Hermes: `agent.json` |
| **角色设定方式不同** | OpenClaw: skills 注入（qclaw-rules 等）；Hermes: `SOUL.md` 文件 |
| **共享本地代理** | 两者都走 `http://127.0.0.1:19000/proxy/llm` |

---

## 二、详细数据格式对比

### 2.1 QClaw 产品配置（.qclaw/qclaw.json）

```json
{
  "authGatewayBaseUrl": "http://127.0.0.1:19000/proxy",
  "sharedParams": {
    "guid": "be8acdd423c6...",
    "appVersion": "0.2.18",
    "appChannel": "5001",
    "platform": "Qclaw_MAC_ARM",
    "sessionId": "e508de21-7481-4e78-8d76-e9305db83cc6"
  },
  "cli": {
    "nodeBinary": "/Applications/QClaw.app/Contents/Resources/node/node",
    "openclawMjs": "/Users/yangjihong/Library/Application Support/QClaw/openclaw/node_modules/openclaw/openclaw.mjs",
    "pid": 9712
  },
  "stateDir": "/Users/yangjihong/.qclaw",
  "configPath": "/Users/yangjihong/.qclaw/openclaw.json",
  "port": 28789,
  "platform": "darwin"
}
```

**关键信息**:
- `stateDir`: OpenClaw 数据目录路径
- `cli.nodeBinary`: QClaw 内置 Node.js 路径
- `cli.openclawMjs`: QClaw 内置 OpenClaw 入口
- `port: 28789`: OpenClaw Gateway 端口（不是默认 18789）
- `authGatewayBaseUrl`: 本地代理基础地址

### 2.2 OpenClaw Agent 元数据（.qclaw/agents/main/agent/models.json）

```json
{
  "providers": {
    "ollama": {
      "baseUrl": "http://127.0.0.1:11434",
      "api": "ollama",
      "models": [{ "id": "deepseek-r1:latest", "name": "deepseek-r1:latest", "reasoning": true, ... }]
    },
    "qclaw": {
      "baseUrl": "http://127.0.0.1:19000/proxy/llm",
      "apiKey": "__QCLAW_AUTH_GATEWAY_MANAGED__",
      "api": "openai-completions",
      "models": [{ "id": "modelroute", "name": "modelroute", "input": ["text", "image"], ... }]
    }
  }
}
```

### 2.3 OpenClaw 会话列表（.qclaw/agents/main/sessions/sessions.json）

```json
{
  "agent:main:session-1778580648753-gix2sb": {
    "sessionId": "033db0cf-f43e-4bb5-92cb-9c49edc1a744",
    "updatedAt": 1778580696605,
    "label": "如何安装openclaw [gix2sb]",
    "systemSent": true,
    "abortedLastRun": false,
    "chatType": "direct",
    "deliveryContext": { "channel": "webchat" },
    "lastChannel": "webchat",
    "origin": { "provider": "webchat", "surface": "webchat", "chatType": "direct" },
    "sessionFile": "/Users/yangjihong/.qclaw/agents/main/sessions/033db0cf-f43e-4bb5-92cb-9c49edc1a744.jsonl",
    "skillsSnapshot": { ... },
    "status": "done",
    "startedAt": 1778580676252,
    "endedAt": 1778580696539,
    "runtimeMs": 20287,
    "modelProvider": "qclaw",
    "model": "modelroute"
  }
}
```

**关键信息**:
- `sessionId`: UUID，对应 sessions 目录下的 JSONL 文件名
- `label`: 会话标题（自动生成）
- `sessionFile`: 完整文件路径
- `modelProvider` / `model`: 使用的模型

### 2.4 OpenClaw 会话数据（JSONL 格式）

```jsonl
{"type":"session","version":3,"id":"033db0cf-...","timestamp":"2026-05-12T10:11:16.228Z","cwd":"/Users/yangjihong/.qclaw/workspace"}
{"type":"model_change","id":"a64b25b9","parentId":null,"timestamp":"...","provider":"qclaw","modelId":"modelroute"}
{"type":"thinking_level_change","id":"fa967722","parentId":"a64b25b9","timestamp":"...","thinkingLevel":"off"}
{"type":"custom","customType":"model-snapshot","data":{"timestamp":1778580676239,"provider":"qclaw","modelApi":"openai-completions","modelId":"modelroute"},"id":"bb35414e","parentId":"fa967722","timestamp":"..."}
{"type":"message","id":"79288a1e","parentId":"bb35414e","timestamp":"...","message":{"role":"user","content":[{"type":"text","text":"Sender (untrusted metadata):\n...\n[Tue 2026-05-12 18:11 GMT+8] 如何安装openclaw"}],"timestamp":1778580676250}}
{"type":"message","id":"8eba12ce","parentId":"79288a1e","timestamp":"...","message":{"role":"assistant","content":[{"type":"text","text":"..."}]},...}
```

**格式特征**:
- JSONL（每行一个 JSON 对象），逐行追加
- 类型化事件：`session`, `model_change`, `thinking_level_change`, `message`, `custom`
- 链式结构：`parentId` 指向父事件
- `content` 是数组格式：`[{ "type": "text", "text": "..." }]`

### 2.5 Hermes Agent 元数据（.qclaw-hermes/agent.json）

```json
{
  "agentId": "hermes_default",
  "name": "代可行",
  "vibe": "热爱手搓的程序员",
  "avatar": "",
  "emoji": "",
  "bio": [
    { "title": "经历", "desc": "计算机出身，毕业后在大厂做了四年后端开发..." },
    { "title": "风格", "desc": "极度务实，只关注能不能解决问题..." }
  ],
  "skills": ["github-skill", "coding-agent", "mcporter", "browser-cdp", ...],
  "sessionIds": ["b56fdc35-c846-4f0f-b646-c3b15a1c08f9"],
  "sessionTitles": {
    "b56fdc35-c846-4f0f-b646-c3b15a1c08f9": "你好 [1c08f9]"
  },
  "sessionUpdatedAts": {
    "b56fdc35-c846-4f0f-b646-c3b15a1c08f9": 1778580295090
  },
  "createdAt": 1778579966869,
  "updatedAt": 1778580295090
}
```

**关键信息**:
- `agentId`: 固定为 `hermes_default`（Hermes 目前只支持单 Agent）
- `name` / `vibe`: 角色名称和风格标签
- `bio`: 经历 + 风格的详细描述（注入为 system prompt）
- `sessionIds`: 会话 ID 列表
- `sessionTitles`: 会话 ID → 标题映射
- `sessionUpdatedAts`: 会话 ID → 最后更新时间映射

### 2.6 Hermes 配置（.qclaw-hermes/config.yaml）

```yaml
# Hermes Agent 配置 (由 QClaw 自动生成和管理)
# model/terminal 段由 QClaw 同步，其他段可手动编辑

model:
  default: "modelroute"
  base_url: "http://127.0.0.1:19000/proxy/llm"
  api_key: "__QCLAW_AUTH_GATEWAY_MANAGED__"
  provider: "custom"

terminal:
  cwd: "/Users/yangjihong/.qclaw-hermes/workspace"

plugins:
  enabled:
    - qclaw-plugin-hermes
```

### 2.7 Hermes 角色设定（.qclaw-hermes/SOUL.md）

```markdown
# 身份

你的名称是「代可行」

> 热爱手搓的程序员

# 经历

计算机出身，毕业后在大厂做了四年后端开发。代码review从不废话，批注从不解释为什么，默认你能看懂。下班后爱打游戏，段位甚高。

# 风格

极度务实，只关注能不能解决问题，以结果为导向。沉默寡言但并非冷漠，对自己要求严苛，极度自律。冷静理性，几乎不会情绪化，擅长寻找对策。
```

### 2.8 Hermes 会话数据（JSON 格式）

```json
{
  "session_id": "2c0ee4f9-7ea5-4c9c-9f71-6b756fff386e",
  "model": "modelroute",
  "base_url": "http://127.0.0.1:19000/proxy/llm",
  "platform": "api_server",
  "session_start": "2026-05-12T18:04:55.214933",
  "last_updated": "2026-05-12T18:05:03.607598",
  "system_prompt": "# 身份\n\n你的名称是「代可行」\n\n> 热爱手搓的程序员\n\n...",
  "tools": [
    { "type": "function", "function": { "name": "cronjob", "description": "...", "parameters": {...} } },
    { "type": "function", "function": { "name": "delegate_task", ... } },
    { "type": "function", "function": { "name": "execute_code", ... } },
    // ... 约 20 个工具定义
  ],
  "message_count": 5,
  "messages": [
    { "role": "user", "content": "你好" },
    { "role": "user", "content": "你好" },
    {
      "role": "assistant",
      "content": "你好。有什么需要我帮忙的？",
      "reasoning": "The user is greeting me in Chinese...",
      "finish_reason": "stop",
      "reasoning_content": "..."
    },
    { "role": "user", "content": "这个hermes agent 你的配置文件在哪里" },
    { "role": "assistant", "content": "Hermes 的配置文件通常在以下位置：...", "reasoning": "...", "finish_reason": "stop", "reasoning_content": "..." }
  ]
}
```

**格式特征**:
- 单个完整 JSON 文件（不是 JSONL）
- 包含完整的工具定义（function calling schema）
- `system_prompt` 是完整的 Markdown 文本（SOUL.md + memory 提示）
- `reasoning` / `reasoning_content`: 推理过程（思维链）
- 每个会话一个独立文件，大小约 47KB

---

## 三、Gateway 状态管理

### 3.1 Hermes Gateway 状态（.qclaw-hermes/gateway_state.json）

```json
{
  "pid": 10188,
  "kind": "hermes-gateway",
  "argv": ["/Users/yangjihong/Library/Application Support/QClaw/hermes/hermes", "gateway", "run", "--replace"],
  "start_time": null,
  "gateway_state": "running",
  "exit_reason": null,
  "restart_requested": false,
  "active_agents": 0,
  "platforms": {
    "api_server": { "state": "connected", "error_code": null, "error_message": null, "updated_at": "2026-05-12T09:59:35.622966+00:00" }
  },
  "updated_at": "2026-05-12T09:59:35.623176+00:00"
}
```

### 3.2 启动命令推断

```bash
# Hermes Gateway
/Applications/QClaw.app/Contents/Resources/hermes/hermes \
  gateway run --replace \
  --config /Users/yangjihong/.qclaw-hermes/config.yaml

# OpenClaw Gateway（从 qclaw.json 推断）
/Applications/QClaw.app/Contents/Resources/node/node \
  /Users/yangjihong/Library/Application Support/QClaw/openclaw/node_modules/openclaw/openclaw.mjs \
  gateway --config /Users/yangjihong/.qclaw/openclaw.json --port 28789
```

---

## 四、前端会话加载架构

### 4.1 前端如何加载会话列表

```
前端加载流程:
─────────────────────────────────────────────────────────────────┐
│  用户打开 QClaw → 选择 Agent（OpenClaw / Hermes）                │
│       │                                                         │
│       ▼                                                         │
│  前端 IPC: getAgentSessions(agentKernel)                        │
│       │                                                         │
│       ▼                                                         │
│  Electron Main Process:                                         │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │  if (kernel === 'openclaw'):                               │ │
│  │    • 读取 .qclaw/agents/main/sessions/sessions.json        │ │
│  │    • 解析 JSON → 提取 sessionId, label, updatedAt, model   │ │
│  │    • 返回: [{ id, title, updatedAt, model, kernel: 'oc' }] │ │
│  │                                                           │ │
│  │  if (kernel === 'hermes'):                                 │ │
│  │    • 读取 .qclaw-hermes/agent.json                         │ │
│  │    • 提取 sessionIds, sessionTitles, sessionUpdatedAts     │ │
│  │    • 返回: [{ id, title, updatedAt, model, kernel: 'hermes'}]││
│  └───────────────────────────────────────────────────────────┘ │
│       │                                                         │
│       ▼                                                         │
│  前端 UI: 渲染会话列表                                           │
│  [{ id: "033db0cf", title: "如何安装openclaw", updatedAt: ..., │
│     kernel: "openclaw" }]                                       │
─────────────────────────────────────────────────────────────────┘
```

### 4.2 前端如何加载会话详情

```
前端加载会话详情:
─────────────────────────────────────────────────────────────────┐
│  用户点击某个会话                                                │
│       │                                                         │
│       ▼                                                         │
│  前端 IPC: openSession(kernel, sessionId)                       │
│       │                                                         │
│       ▼                                                         │
│  Electron Main Process:                                         │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │  if (kernel === 'openclaw'):                               │ │
│  │    • 读取 .qclaw/agents/main/sessions/{sessionId}.jsonl    │ │
│  │    • 逐行解析 JSONL → 过滤 type='message' 的行             │ │
│  │    • 转换为统一格式:                                       │ │
│  │      [{ id, role, content, modelId, timestamp }]           │ │
│  │                                                           │ │
│  │  if (kernel === 'hermes'):                                 │ │
│  │    • 读取 .qclaw-hermes/sessions/session_{id}.json         │ │
│  │    • 解析 JSON → 提取 messages 数组                         │ │
│  │    • 转换为统一格式:                                       │ │
│  │      [{ id, role, content, reasoning, timestamp }]         │ │
│  └───────────────────────────────────────────────────────────┘ │
│       │                                                         │
│       ▼                                                         │
│  前端 UI: 渲染消息列表                                           │
│  [{ role: 'user', content: '如何安装openclaw' },                │
│   { role: 'assistant', content: '...', reasoning: '...' }]     │
─────────────────────────────────────────────────────────────────┘
```

### 4.3 统一消息格式（主进程转换后返回前端）

```typescript
// packages/shared/src/types/session.ts

// 前端收到的统一消息格式
export interface UnifiedMessage {
  id: string;
  role: 'user' | 'assistant' | 'system';
  content: string;              // 纯文本（从不同格式中提取）
  reasoning?: string;           // Hermes 特有
  modelId?: string;             // OpenClaw 特有
  provider?: string;            // OpenClaw 特有
  timestamp: number;
  isStreaming?: boolean;
}

// 前端收到的统一会话格式
export interface UnifiedSession {
  id: string;
  title: string;
  kernel: 'openclaw' | 'hermes';
  updatedAt: number;
  messages: UnifiedMessage[];
}
```

---

## 五、完整目录结构验证

```
~/.qclaw/                              # OpenClaw 数据目录
├── openclaw.json                      # OpenClaw 核心配置 (JSON)
├── openclaw.json.bak                  # 自动备份
├── openclaw.json.last-good            # 最后良好配置
├── qclaw.json                         # QClaw 产品配置 (JSON)
│                                      #   - authGatewayBaseUrl: 127.0.0.1:19000
│                                      #   - stateDir: ~/.qclaw
│                                      #   - port: 28789
├── agents/                            # Agent 目录
│   ├── main/                          # 默认 Agent
│   │   ├── agent/
│   │   │   └── models.json            # Agent 元数据 (JSON)
│   │   │                              #   - providers: { qclaw, ollama, deepseek }
│   │   └── sessions/
│   │       ├── sessions.json          # 会话索引 (JSON)
│   │       │                          #   - { sessionKey: { sessionId, label, ... } }
│   │       ├── 033db0cf-....jsonl     # 会话数据 (JSONL)
│   │       └── ...
│   └── __creating__/                  # Agent 创建中
│       └── sessions/
│           └── sessions.json
├── skills/                            # OpenClaw Skills
│   ├── find-skills/
│   ├── online-search/
│   ├── qclaw-rules/
│   └── ...
├── plugins/                           # OpenClaw Plugins
│   ├── prompt-optimizer/
│   └── wechat-access/
├── memory/lossless/lcm.db             # 长期记忆 (SQLite)
├── flows/registry.sqlite              # 流程注册表 (SQLite)
├── devices/paired.json                # 已配对设备
├── workspace/                         # 工作区
│   ├── AGENTS.md
│   ├── SOUL.md
│   ├── TOOLS.md
│   ├── IDENTITY.md
│   └── USER.md
├── canvas/index.html                  # Canvas 页面
├── logs/                              # 日志
│   ├── config-audit.jsonl
│   └── config-health.json
├── backups/                           # 配置备份
│   └── openclaw.2026-05-12T09-56-38-519Z.json
└── compile-cache/                     # V8 编译缓存
    └── v22.16.0-arm64-.../


~/.qclaw-hermes/                       # Hermes 数据目录
├── config.yaml                        # Hermes 配置 (YAML)
│                                      #   - model.default: modelroute
│                                      #   - model.base_url: 127.0.0.1:19000
│                                      #   - plugins.enabled: [qclaw-plugin-hermes]
├── agent.json                         # Agent 元数据 (JSON)
│                                      #   - agentId: hermes_default
│                                      #   - name: 代可行
│                                      #   - vibe: 热爱手搓的程序员
│                                      #   - bio: [{ title, desc }, ...]
│                                      #   - sessionIds: [...]
│                                      #   - sessionTitles: { id: title }
│                                      #   - sessionUpdatedAts: { id: timestamp }
├── SOUL.md                            # 角色设定 (Markdown)
│                                      #   - 身份 / 经历 / 风格
├── .skills_prompt_snapshot.json       # Skills 快照
├── sessions/                          # 会话目录
│   └── session_2c0ee4f9-....json      # 会话数据 (JSON)
│                                      #   - session_id, model, base_url
│                                      #   - system_prompt: 完整 Markdown
│                                      #   - tools: [function calling schema]
│                                      #   - messages: [{ role, content, reasoning }]
├── state.db                           # 状态存储 (SQLite)
├── audit.db                           # 审计日志 (SQLite)
├── response_store.db                  # 响应存储 (SQLite)
├── gateway_state.json                 # Gateway 状态
│                                      #   - pid, kind: hermes-gateway
│                                      #   - gateway_state: running
│                                      #   - platforms: { api_server: { state: connected } }
├── gateway.pid                        # Gateway PID
├── gateway.lock                       # Gateway 锁
├── logs/                              # 日志
│   ├── agent.log
│   └── errors.log
├── skills/                            # Hermes Skills
├── memories/                          # 记忆
├── platforms/                         # 平台集成
│   └── pairing/
├── cron/                              # 定时任务
│   ├── .tick.lock
│   └── output/
├── bin/tirith                         # 二进制工具
├── channel_directory.json             # 频道目录
│                                      #   - platforms: { telegram, discord, ... }
└── workspace/                         # 工作区
```

---

## 六、前端实现参考

### 6.1 IPC 层定义

```typescript
// packages/shared/src/types/ipc.ts

interface ElectronAPI {
  // === Agent 管理 ===
  getAgents(): Promise<AgentInfo[]>;
  // 返回: [{ id: 'main', name: 'OpenClaw', kernel: 'openclaw' },
  //        { id: 'hermes_default', name: '代可行', kernel: 'hermes' }]

  // === 会话列表 ===
  getSessions(kernel: 'openclaw' | 'hermes'): Promise<SessionListItem[]>;
  // OpenClaw: 读取 sessions.json → 转换
  // Hermes: 读取 agent.json → 提取 sessionIds + sessionTitles

  // === 会话详情 ===
  openSession(kernel: 'openclaw' | 'hermes', sessionId: string): Promise<UnifiedSession>;
  // OpenClaw: 读取 JSONL → 过滤 message 类型 → 转换
  // Hermes: 读取 JSON → 提取 messages → 转换

  // === 发送消息 ===
  sendMessage(kernel: 'openclaw' | 'hermes', sessionId: string, content: string): Promise<void>;
  // 路由到对应内核进程

  // === 流式事件 ===
  onStream(kernel: 'openclaw' | 'hermes', sessionId: string, cb: (msg: UnifiedMessage) => void): () => void;
}
```

### 6.2 主进程会话管理器

```typescript
// packages/shell/src/main/core/session-manager.ts

import { readFileSync, readdirSync, appendFileSync } from 'fs';
import { join } from 'path';
import { app } from 'electron';

export class SessionManager {
  private get openclawDir(): string {
    return join(app.getPath('home'), '.qclaw');
  }

  private get hermesDir(): string {
    return join(app.getPath('home'), '.qclaw-hermes');
  }

  // === OpenClaw 会话加载 ===

  async getOpenClawSessions(): Promise<SessionListItem[]> {
    const sessionsIndexPath = join(this.openclawDir, 'agents', 'main', 'sessions', 'sessions.json');
    const raw = JSON.parse(readFileSync(sessionsIndexPath, 'utf8'));

    return Object.values(raw).map((entry: any) => ({
      id: entry.sessionId,
      title: entry.label,
      updatedAt: entry.updatedAt,
      model: entry.model,
      modelProvider: entry.modelProvider,
      kernel: 'openclaw',
    }));
  }

  async loadOpenClawSession(sessionId: string): Promise<UnifiedSession> {
    const filePath = join(this.openclawDir, 'agents', 'main', 'sessions', `${sessionId}.jsonl`);
    const content = readFileSync(filePath, 'utf8');
    const lines = content.trim().split('\n');

    const messages: UnifiedMessage[] = [];
    let modelId: string | undefined;

    for (const line of lines) {
      const event = JSON.parse(line);

      if (event.type === 'model_change') {
        modelId = event.modelId;
      }

      if (event.type === 'message') {
        const msg = event.message;
        // content 是数组格式: [{ type: 'text', text: '...' }]
        const text = Array.isArray(msg.content)
          ? msg.content
              .filter((c: any) => c.type === 'text')
              .map((c: any) => c.text)
              .join('')
          : msg.content;

        messages.push({
          id: event.id,
          role: msg.role,
          content: text,
          modelId: event.model || modelId,
          provider: event.provider,
          timestamp: event.timestamp,
        });
      }
    }

    // 从 sessions.json 获取标题
    const sessionsIndexPath = join(this.openclawDir, 'agents', 'main', 'sessions', 'sessions.json');
    const raw = JSON.parse(readFileSync(sessionsIndexPath, 'utf8'));
    const sessionKey = Object.keys(raw).find(k => raw[k].sessionId === sessionId);
    const title = sessionKey ? raw[sessionKey].label : '新会话';

    return { id: sessionId, title, kernel: 'openclaw', updatedAt: Date.now(), messages };
  }

  // === Hermes 会话加载 ===

  async getHermesSessions(): Promise<SessionListItem[]> {
    const agentPath = join(this.hermesDir, 'agent.json');
    const agent = JSON.parse(readFileSync(agentPath, 'utf8'));

    return (agent.sessionIds || []).map((id: string) => ({
      id,
      title: agent.sessionTitles?.[id] || '新会话',
      updatedAt: agent.sessionUpdatedAts?.[id] || 0,
      model: 'modelroute',
      kernel: 'hermes',
    }));
  }

  async loadHermesSession(sessionId: string): Promise<UnifiedSession> {
    const filePath = join(this.hermesDir, 'sessions', `session_${sessionId}.json`);
    const session = JSON.parse(readFileSync(filePath, 'utf8'));

    const messages: UnifiedMessage[] = session.messages.map((msg: any, idx: number) => ({
      id: `hermes-${sessionId}-${idx}`,
      role: msg.role,
      content: msg.content,
      reasoning: msg.reasoning,
      timestamp: new Date(session.session_start).getTime() + idx * 1000,
    }));

    return {
      id: sessionId,
      title: `session_${sessionId}`,
      kernel: 'hermes',
      updatedAt: new Date(session.last_updated).getTime(),
      messages,
    };
  }

  // === 统一入口 ===

  async getSessions(kernel: 'openclaw' | 'hermes'): Promise<SessionListItem[]> {
    return kernel === 'openclaw'
      ? this.getOpenClawSessions()
      : this.getHermesSessions();
  }

  async openSession(kernel: 'openclaw' | 'hermes', sessionId: string): Promise<UnifiedSession> {
    return kernel === 'openclaw'
      ? this.loadOpenClawSession(sessionId)
      : this.loadHermesSession(sessionId);
  }
}
```

---

## 七、与之前推断的对比修正

| 之前推断 | 实际情况 | 修正 |
|----------|----------|------|
| Agent 创建时选择内核 | 两个内核完全独立的数据目录 | 不是"切换"，是"两个独立 Agent" |
| 内核进程池共享 | OpenClaw: 单进程多会话；Hermes: 单进程单 Agent | Hermes 目前只支持一个 Agent |
| 统一 agents.json 注册表 | 不存在注册表，前端通过探测两个目录发现 Agent | 前端需要分别读取 `.qclaw/agents/main/` 和 `.qclaw-hermes/agent.json` |
| System Prompt 注入 | OpenClaw: skills 系统注入；Hermes: SOUL.md 文件 | 两种方式完全不同 |
| Session 格式统一 | OpenClaw: JSONL + sessions.json 索引；Hermes: 完整 JSON | 两种格式完全不同 |
| 每个 Agent 独立目录 | OpenClaw 支持多 Agent（`agents/main/`, `agents/agent-2/`）；Hermes 只有 `hermes_default` | OpenClaw 多 Agent 已实现，Hermes 单 Agent |

### 核心修正结论

**QClaw 的多会话管理不是"内核切换"，而是"双 Agent 并存"**:

```
前端 Agent 选择器:
┌─────────────────────────────────────────────────────────────┐
│  ○ OpenClaw (默认)                                           │
│    数据来源: ~/.qclaw/agents/main/sessions/sessions.json    │
│    读取: JSONL 逐行追加                                      │
│    配置: openclaw.json (JSON)                               │
│                                                              │
│  ○ Hermes (代可行)                                           │
│    数据来源: ~/.qclaw-hermes/agent.json                     │
│    读取: 完整 JSON 文件                                      │
│    配置: config.yaml (YAML)                                 │
└─────────────────────────────────────────────────────────────┘
```

前端只需要知道 `kernel: 'openclaw' | 'hermes'`，主进程根据 kernel 类型选择对应的数据目录和解析器，返回统一的 `UnifiedSession` 格式。
