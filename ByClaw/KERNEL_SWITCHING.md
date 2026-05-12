# QClaw Agent 内核切换 — 技术分析

> **调研日期**: 2026-05-12
> **参考**: QClaw Mac 产品截图 + OpenClaw + Hermes Agent 源码分析

---

## 一、产品界面观察

### 1.1 Agent 创建工坊

```
┌─────────────────────────────────────────────────────────────┐
│  欢迎来到 Agent 创建工坊!                                      │
│                                                             │
│  运行内核:  ● OpenClaw内核 (推荐)    ○ Hermes Agent内核      │
│            (Win端打磨中，敬请期待)                            │
│                                                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │ 无不言    │  │ 代可行    │  │ 林且慢    │  │ 自定义    │   │
│  │ 毒舌的自由 │  │ 热爱手搓的 │  │ 少年爹系的│  │ 创建全新   │   │
│  │ 撰稿人    │  │ 程序员    │  │ 辅导员    │  │ Agent     │   │
│  ──────────┘  └──────────  └──────────┘  └──────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 关键观察

| 特征 | 推断 |
|------|------|
| **内核切换是创建时决定的** | 选择 OpenClaw 或 Hermes，之后该 Agent 固定使用这个内核 |
| **"Win端打磨中"** | Mac/Ubuntu 上 Hermes 已可用，且**预装在本地** |
| **多个预设 Agent 模板** | "无不言"、"代可行"、"林且慢" = 不同的 system prompt + 内核配置 |
| **"创建Agent中，请稍候..."** | 创建时需要初始化内核进程/会话上下文 |
| **可同时存在多个 Agent** | 用户可切换不同 Agent 对话，每个 Agent 有独立内核和配置 |

---

## 二、架构推断

### 2.1 整体架构

```
┌─────────────────────────────────────────────────────────────────┐
│                         QClaw Electron 应用                       │
│                                                                 │
│  ───────────────────────────────────────────────────────────┐  │
│  │  Electron Main Process                                     │  │
│  │                                                           │  │
│  │  ┌─────────────────────────────────────────────────────┐  │  │
│  │  │  Agent Manager (Agent 管理器)                         │  │  │
│  │  │                                                      │  │  │
│  │  │  • 管理多个 Agent 实例                                │  │  │
│  │  │  • 每个 Agent 绑定一个内核 (OpenClaw 或 Hermes)        │  │  │
│  │  │  • Agent 间隔离 (会话/配置/内存独立)                   │  │  │
│  │  └─────────────────────────────────────────────────────┘  │  │
│  │                                                           │  │
│  │  ┌─────────────────────────────────────────────────────┐  │  │
│  │  │  Kernel Pool (内核进程池)                             │  │  │
│  │  │                                                      │  │  │
│  │  │  ┌──────────────┐  ┌──────────────┐                 │  │  │
│  │  │  │ OpenClaw     │  │ Hermes       │                 │  │  │
│  │  │  │ Gateway      │  │ ACP          │                 │  │  │
│  │  │  │ (Node.js)    │  │ (Python)     │                 │  │  │
│  │  │  │              │  │              │                 │  │  │
│  │  │  │ 多会话复用   │  │ 多会话复用    │                 │  │  │
│  │  │  │ 一个进程     │  │ 一个进程      │                 │  │  │
│  │  │  └──────────────┘  └──────────────                 │  │  │
│  │  └─────────────────────────────────────────────────────┘  │  │
│  │                                                           │  │
│  │  ┌─────────────────────────────────────────────────────┐  │  │
│  │  │  Unified Protocol Layer (统一协议层)                  │  │  │
│  │  │                                                      │  │  │
│  │  │  将不同内核的协议统一为内部消息格式:                     │  │  │
│  │  │  • OpenClaw: WebSocket JSON 帧 → NormalizedMessage   │  │  │
│  │  │  • Hermes: stdio JSON-RPC (ACP) → NormalizedMessage  │  │  │
│  │  └─────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  Agent 持久化 (~/.qclaw/agents/)                           │  │
│  │                                                           │  │
│  │  ├── main/                    ← 默认 Agent (OpenClaw)      │  │
│  │  │   ├── sessions/            ← 会话 JSONL                 │  │
│  │  │   │   ├── xxx.jsonl                                    │  │
│  │  │   │   └── yyy.jsonl                                    │  │
│  │  │   └── config.json          ← Agent 配置                 │  │
│  │  ├── custom-agent-1/          ← 用户自定义 Agent           │  │
│  │  │   ├── kernel: hermes                                   │  │
│  │  │   ├── sessions/                                        │  │
│  │  │   └── config.json                                      │  │
│  │  └── custom-agent-2/          ← 另一个 Agent              │  │
│  │      ├── kernel: openclaw                                 │  │
│  │      ├── sessions/                                        │  │
│  │      └── config.json                                      │  │
│  ───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 内核进程管理策略

```
┌─────────────────────────────────────────────────────────────────┐
│                    内核进程池管理                                 │
│                                                                 │
│  场景 1: 用户只有一个 OpenClaw Agent                             │
│  ─────────────────────────────────────────────────────────────  │
│  OpenClaw Gateway 进程: 1 个                                     │
│  └── 多个会话共享同一个 Gateway 实例                              │
│      (通过 session ID 区分上下文)                                  │
│                                                                 │
│  场景 2: 用户有 2 个 OpenClaw Agent + 1 个 Hermes Agent          │
│  ─────────────────────────────────────────────────────────────  │
│  OpenClaw Gateway 进程: 1 个 (所有 OpenClaw Agent 共享)           │
│  Hermes ACP 进程: 1 个 (所有 Hermes Agent 共享)                   │
│                                                                 │
│  场景 3: 用户切换 Agent 对话                                      │
│  ─────────────────────────────────────────────────────────────  │
│  前端: 发送 switchAgent(agentId) → IPC                           │
│  后端: 根据 agentId 查找对应的 kernel 类型                        │
│        → 路由消息到对应的内核进程                                  │
│        → 内核进程通过 session ID 隔离上下文                        │
│                                                                 │
│  关键: 不是为每个 Agent 启动独立进程，而是进程复用 + 会话隔离      │
└─────────────────────────────────────────────────────────────────┘
```

---

## 三、统一协议层设计

### 3.1 协议转换

```
┌─────────────────────────────────────────────────────────────────┐
│                    统一协议层 (Normalized Protocol)               │
│                                                                 │
│  OpenClaw WebSocket 帧:                                          │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ {                                                            ││
│  │   "type": "message",                                         ││
│  │   "data": { "role": "assistant", "content": "...",           ││
│  │             "stream": true, "done": false }                  ││
│  │ }                                                            ││
│  └─────────────────────────────────────────────────────────────┘│
│                        │                                         │
│                        ▼ 转换                                    │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ NormalizedMessage {                                          ││
│  │   agentId: "main",                                           ││
│  │   sessionId: "xxx",                                          ││
│  │   role: "assistant",                                         ││
│  │   content: "...",                                            ││
│  │   isStreaming: true,                                         ││
│  │   isDone: false,                                             ││
│  │   metadata: { modelId, provider, usage, ... }               ││
│  │ }                                                            ││
│  └─────────────────────────────────────────────────────────────┘│
│                        │                                         │
│                        ▼ 分发                                    │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ IPC → Renderer: electronAPI.onAgentStream(data)              ││
│  ─────────────────────────────────────────────────────────────┘│
│                                                                 │
│  Hermes ACP JSON-RPC:                                           │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ stdin → { "jsonrpc": "2.0", "id": 1, "method": "chat.send",  ││
│  │           "params": { "message": "..." } }                   ││
│  │                                                              ││
│  │ stdout ← { "jsonrpc": "2.0", "id": 1, "result": {           ││
│  │             "content": "...", "done": true } }               ││
│  └─────────────────────────────────────────────────────────────┘│
│                        │                                         │
│                        ▼ 同样的转换 → NormalizedMessage           │
─────────────────────────────────────────────────────────────────┘
```

### 3.2 统一接口定义

```typescript
// packages/shared/src/types/agent.ts

// 统一的内核类型
export type KernelType = 'openclaw' | 'hermes';

// 统一的 Agent 定义
export interface AgentConfig {
  id: string;
  name: string;
  kernel: KernelType;
  systemPrompt: string;       // 性格/语气 (注入为 system message)
  modelId?: string;           // 默认模型
  avatar?: string;
  createdAt: string;
}

// 统一的消息格式
export interface NormalizedMessage {
  id: string;
  agentId: string;
  sessionId: string;
  role: 'user' | 'assistant' | 'system';
  content: string;
  modelId?: string;
  provider?: string;
  isStreaming: boolean;
  isDone: boolean;
  metadata?: Record<string, any>;
  timestamp: number;
}

// 统一的内核接口
export interface AgentKernel {
  type: KernelType;

  // 生命周期
  start(config: KernelConfig): Promise<void>;
  stop(): Promise<void>;
  isRunning(): boolean;

  // 消息
  sendMessage(sessionId: string, message: string, options?: {
    systemPrompt?: string;
    modelId?: string;
    stream?: boolean;
  }): Promise<ReadableStream<NormalizedMessage>>;

  // 模型管理
  getModels(): Promise<ModelInfo[]>;

  // 会话管理
  createSession(agentId: string): Promise<string>;
  deleteSession(sessionId: string): Promise<void>;

  // 事件
  onStream(sessionId: string, callback: (msg: NormalizedMessage) => void): () => void;
}
```

---

## 四、Mac 上的具体实现

### 4.1 双内核预装结构

```
QClaw.app/Contents/Resources/
│
├── openclaw/                 # OpenClaw Node.js 运行时
│   ├── node/                 # 内置 Node.js 22+
│   │   ├── bin/node
│   │   └── lib/node_modules/
│   ├── node_modules/         # OpenClaw + 依赖
│   ├── gateway.js            # Gateway 启动入口
│   └── package.json
│
├── hermes/                   # Hermes Python 运行时
│   ├── venv/                 # 预装的 Python 虚拟环境
│   │   ├── bin/python3
│   │   ├── bin/hermes
│   │   └── lib/python3.11/site-packages/
│   └── requirements.txt
│
├── app.asar                  # Electron 业务代码
└── icons/
```

### 4.2 内核启动命令

```bash
# OpenClaw Gateway
/Users/.../QClaw.app/Contents/Resources/openclaw/node/bin/node \
  /Users/.../QClaw.app/Contents/Resources/openclaw/gateway.js \
  --config ~/.qclaw/openclaw.json

# Hermes ACP
/Users/.../QClaw.app/Contents/Resources/hermes/venv/bin/python3 \
  -m hermes acp
```

### 4.3 内核管理器实现

```typescript
// packages/shell/src/main/core/kernel-manager.ts

import { spawn, ChildProcess } from 'child_process';
import path from 'path';
import { app } from 'electron';
import { EventEmitter } from 'events';

export class KernelManager extends EventEmitter {
  private openclawProcess: ChildProcess | null = null;
  private hermesProcess: ChildProcess | null = null;
  private agentRegistry = new Map<string, AgentConfig>();
  private sessionToAgent = new Map<string, string>();

  async start(): Promise<void> {
    // 启动 OpenClaw Gateway (默认内核)
    await this.startOpenClaw();

    // Hermes 按需启动 (首次切换到 Hermes Agent 时)
    // this.startHermes();
  }

  private async startOpenClaw(): Promise<void> {
    const resourcesPath = path.join(app.getAppPath(), '..', 'Resources');
    const nodePath = path.join(resourcesPath, 'openclaw', 'node', 'bin', 'node');
    const gatewayPath = path.join(resourcesPath, 'openclaw', 'gateway.js');

    this.openclawProcess = spawn(nodePath, [gatewayPath], {
      env: {
        ...process.env,
        OPENCLAW_CONFIG: path.join(app.getPath('userData'), 'openclaw.json'),
      },
      stdio: ['pipe', 'pipe', 'pipe'],
    });

    // 解析 WebSocket 帧 → NormalizedMessage
    this.openclawProcess.stdout.on('data', (chunk) => {
      const frames = this.parseWebSocketFrames(chunk);
      for (const frame of frames) {
        const normalized = this.normalizeOpenClawMessage(frame);
        const agentId = this.sessionToAgent.get(normalized.sessionId);
        if (agentId) {
          this.emit(`agent:stream:${agentId}`, normalized);
        }
      }
    });
  }

  private async startHermes(): Promise<void> {
    const resourcesPath = path.join(app.getAppPath(), '..', 'Resources');
    const pythonPath = path.join(resourcesPath, 'hermes', 'venv', 'bin', 'python3');

    this.hermesProcess = spawn(pythonPath, ['-m', 'hermes', 'acp'], {
      stdio: ['pipe', 'pipe', 'pipe'],
    });

    // 解析 JSON-RPC → NormalizedMessage
    this.hermesProcess.stdout.on('data', (chunk) => {
      const messages = this.parseJSONRPCMessages(chunk);
      for (const msg of messages) {
        const normalized = this.normalizeHermesMessage(msg);
        const agentId = this.sessionToAgent.get(normalized.sessionId);
        if (agentId) {
          this.emit(`agent:stream:${agentId}`, normalized);
        }
      }
    });
  }

  // Agent 切换
  async switchAgent(agentId: string): Promise<void> {
    const agent = this.agentRegistry.get(agentId);
    if (!agent) throw new Error(`Agent not found: ${agentId}`);

    // 确保对应内核已启动
    if (agent.kernel === 'hermes' && !this.hermesProcess) {
      await this.startHermes();
    }

    // 创建新会话
    const sessionId = await this.createSession(agent);
    this.sessionToAgent.set(sessionId, agentId);

    return { agentId, sessionId };
  }

  // 发送消息 (自动路由到对应内核)
  async sendMessage(sessionId: string, message: string): Promise<void> {
    const agentId = this.sessionToAgent.get(sessionId);
    const agent = this.agentRegistry.get(agentId);

    if (agent.kernel === 'openclaw') {
      await this.sendToOpenClaw(sessionId, message, agent);
    } else {
      await this.sendToHermes(sessionId, message, agent);
    }
  }

  // System Prompt 注入 (性格/语气)
  private async sendToOpenClaw(
    sessionId: string, message: string, agent: AgentConfig
  ): Promise<void> {
    // 在消息前注入 system prompt
    const payload = {
      type: 'message',
      sessionId,
      messages: [
        { role: 'system', content: agent.systemPrompt },
        { role: 'user', content: message },
      ],
    };
    this.openclawProcess.stdin.write(JSON.stringify(payload) + '\n');
  }

  private async sendToHermes(
    sessionId: string, message: string, agent: AgentConfig
  ): Promise<void> {
    const id = ++this.messageId;
    const payload = {
      jsonrpc: '2.0',
      id,
      method: 'chat.send',
      params: {
        sessionId,
        system: agent.systemPrompt,
        message,
      },
    };
    this.hermesProcess.stdin.write(JSON.stringify(payload) + '\n');
  }
}
```

---

## 五、Agent 持久化设计（经实际数据验证修正）

### 5.1 实际目录结构

**OpenClaw 和 Hermes 使用完全独立的数据目录**，不是统一目录下的子目录。

```
~/.qclaw/                              ← OpenClaw 数据目录
├── openclaw.json                      ← OpenClaw 核心配置 (JSON)
├── qclaw.json                         ← QClaw 产品配置
│   ├── authGatewayBaseUrl: http://127.0.0.1:19000/proxy
│   ├── stateDir: ~/.qclaw
│   ├── port: 28789
│   └── cli: { nodeBinary, openclawMjs, pid }
├── agents/
│   ├── main/                          ← 默认 Agent (OpenClaw 内核)
│   │   ├── agent/
│   │   │   └── models.json            ← Agent 元数据 (providers 配置)
│   │   └── sessions/
│   │       ├── sessions.json          ← 会话索引: { sessionKey: { sessionId, label, model, ... } }
│   │       ├── 033db0cf-....jsonl     ← 会话数据 (JSONL, 逐行追加)
│   │       └── ...
│   └── __creating__/                  ← Agent 创建中临时目录
├── skills/                            ← OpenClaw Skills
├── plugins/                           ← OpenClaw Plugins
├── memory/lossless/lcm.db             ← 长期记忆 (SQLite)
├── flows/registry.sqlite              ← 流程注册表 (SQLite)
├── workspace/                         ← 工作区 (AGENTS.md, SOUL.md, TOOLS.md...)
└── ...

~/.qclaw-hermes/                       ← Hermes 数据目录 (完全独立)
├── config.yaml                        ← Hermes 配置 (YAML)
│   ├── model.default: modelroute
│   ├── model.base_url: http://127.0.0.1:19000/proxy/llm
│   └── plugins.enabled: [qclaw-plugin-hermes]
├── agent.json                         ← Agent 元数据 (JSON)
│   ├── agentId: hermes_default
│   ├── name: 代可行
│   ├── vibe: 热爱手搓的程序员
│   ├── bio: [{ title: "经历", desc: "..." }, { title: "风格", desc: "..." }]
│   ├── sessionIds: [...]
│   ├── sessionTitles: { id: title }
│   └── sessionUpdatedAts: { id: timestamp }
├── SOUL.md                            ← 角色设定 (Markdown)
├── sessions/
│   └── session_xxx.json               ← 会话数据 (完整 JSON)
│       ├── session_id, model, base_url
│       ├── system_prompt: 完整 Markdown (SOUL.md + memory 提示)
│       ├── tools: [function calling schema] (约 20 个工具)
│       └── messages: [{ role, content, reasoning, finish_reason }]
├── state.db                           ← 状态 (SQLite)
├── audit.db                           ← 审计 (SQLite)
├── response_store.db                  ← 响应存储 (SQLite)
├── gateway_state.json                 ← Gateway 状态 (pid, running, platforms)
├── gateway.pid / gateway.lock         ← 进程管理
├── skills/                            ← Hermes Skills
├── memories/                          ← 记忆
├── logs/                              ← 日志
└── workspace/                         ← 工作区
```

### 5.2 关键差异对比

| 维度 | OpenClaw | Hermes |
|------|----------|--------|
| 数据目录 | `~/.qclaw/` | `~/.qclaw-hermes/` |
| 配置文件 | `openclaw.json` (JSON) | `config.yaml` (YAML) |
| Agent 元数据 | `agents/main/agent/models.json` | `agent.json` |
| 会话索引 | `sessions/sessions.json` | `agent.json.sessionIds` |
| 会话数据 | JSONL 逐行追加 | 完整 JSON 文件 |
| 角色设定 | Skills 系统 (qclaw-rules 等) | `SOUL.md` 文件 |
| Agent 数量 | 支持多 Agent (`agents/main/`, `agents/agent-2/`) | 目前只支持单 Agent (`hermes_default`) |
| 消息格式 | `content: [{ type: 'text', text: '...' }]` | `content: string` + `reasoning: string` |
| 工具定义 | OpenClaw 内置工具 (read/edit/exec...) | Function calling schema (约 20 个) |
| 记忆存储 | `memory/lossless/lcm.db` (SQLite) | `memories/` + `state.db` |
| 状态管理 | `gateway_state.json` (无) | `gateway_state.json` (有完整状态) |

### 5.3 共享组件

| 组件 | 说明 |
|------|------|
| 本地代理 (127.0.0.1:19000) | 两者都走同一个模型路由代理 |
| `__QCLAW_AUTH_GATEWAY_MANAGED__` | 两者都用同样的 API Key 占位符 |
| modelroute | 两者都用同样的模型路由 ID |

---

## 六、对我们要构建的产品的启示

### 6.1 核心设计决策（经实际数据修正）

| 决策 | 方案 | 原因 | 实际验证 |
|------|------|------|----------|
| **数据目录隔离** | OpenClaw: `~/.qclaw/`，Hermes: `~/.qclaw-hermes/` | 两个内核各自管理自己的数据，互不干扰 | ✅ QClaw 实际如此 |
| **配置格式差异** | OpenClaw JSON，Hermes YAML | 各用各的原生格式，不强求统一 | ✅ QClaw 实际如此 |
| **会话格式差异** | OpenClaw JSONL，Hermes JSON | 各用各的存储方式 | ✅ QClaw 实际如此 |
| **Agent 注册** | 无统一注册表，前端探测两个目录 | 简化设计，不需要额外维护注册表 | ✅ QClaw 无 agents.json |
| **共享本地代理** | 两者都走 127.0.0.1:19000 | 统一管理模型路由、计费、状态 | ✅ 两者 baseUrl 都是 19000 |
| **角色设定** | OpenClaw: skills，Hermes: SOUL.md | 各自用各自的方式 | ✅ 实际验证 |
| **内核进程** | OpenClaw 默认启动，Hermes 按需启动 | 节省资源 | ✅ gateway_state.json 证明 |
| **前端统一** | 主进程转换后返回 UnifiedSession | 前端无感知底层差异 | 建议方案 |

### 6.2 内核切换流程图

```
用户操作: 选择 "林且慢" (Hermes 内核)
        │
        ▼
    前端 → IPC: switchAgent("agent-linqieman")
        │
        ▼
    KernelManager:
    ├── 查找 agent-linqieman → kernel: hermes
    ├── 检查 hermesProcess 是否运行
    │   ├── 未运行 → startHermes() (首次启动)
    │   └── 已运行 → 复用现有进程
    │
    ├── 创建新 session → sessionId
    ├── sessionToAgent.set(sessionId, "agent-linqieman")
    │
    └── 返回 { agentId, sessionId }
        │
        ▼
    用户发送消息: "你好"
        │
        ▼
    KernelManager.sendMessage(sessionId, "你好")
    ├── 查找 sessionId → agentId: "agent-linqieman"
    ├── 查找 agent → kernel: hermes
    ├── 获取 agent.systemPrompt: "你是一个少年爹系的辅导员..."
    │
    ── hermesProcess.stdin.write({
          jsonrpc: "2.0",
          method: "chat.send",
          params: {
            sessionId,
            system: "你是一个少年爹系的辅导员...",
            message: "你好"
          }
        })
        │
        ▼
    Hermes stdout → 解析 JSON-RPC → NormalizedMessage
        │
        ▼
    emit("agent:stream:agent-linqieman", normalizedMessage)
        │
        ▼
    IPC → Renderer → 前端 UI 显示
```

### 6.3 目录规划

```
~/.your-product/
├── agents/
│   ├── agents.json              # Agent 注册表
│   ├── default/                 # 默认 Agent
│   │   ├── config.json
│   │   └── sessions/
│   ── user-created-1/          # 用户创建的 Agent
│       ├── config.json
│       └── sessions/
├── openclaw.json                # OpenClaw 配置
── models.json                  # 模型路由配置
└── qclaw.json                   # 产品配置
```

---

## 七、总结（经实际数据修正）

QClaw 的多内核管理本质上是：

1. **双目录隔离** — OpenClaw (`~/.qclaw/`) 和 Hermes (`~/.qclaw-hermes/`) 各自有完全独立的数据目录
2. **配置格式不同** — OpenClaw 用 JSON，Hermes 用 YAML；会话存储前者用 JSONL 后者用完整 JSON
3. **共享本地代理** — 两者都走 `127.0.0.1:19000` 同一个模型路由代理
4. **无统一注册表** — 前端通过探测两个目录来发现 Agent，不存在 `agents.json` 注册表
5. **角色设定方式不同** — OpenClaw 用 skills 系统注入，Hermes 用 `SOUL.md` 文件
6. **Gateway 状态管理** — Hermes 有完整的 `gateway_state.json`，OpenClaw 依赖 qclaw.json 中的 pid
7. **前端统一抽象** — 主进程负责格式转换，前端只看到 `UnifiedSession`
8. **Hermes 单 Agent** — 目前 Hermes 只支持 `hermes_default` 一个 Agent，OpenClaw 支持多 Agent

> 详细的会话数据格式对比、前端加载流程、主进程转换代码见 **[MULTI_SESSION_ARCHITECTURE.md](MULTI_SESSION_ARCHITECTURE.md)**。
