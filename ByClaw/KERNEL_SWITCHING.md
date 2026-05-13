# QClaw Agent 内核切换 — 技术分析

> **调研日期**: 2026-05-13（第二轮验证）
> **参考**: QClaw Mac 实际用户数据目录 + 安装包解压 + OpenClaw/Hermes 源码分析
> **App 版本**: 0.2.18 (`@guanjia-openclaw/electron`)

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

### 2.1 整体架构（经实际数据验证修正）

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
│  │  │  从 openclaw.json 读取 agents.list:                    │  │  │
│  │  │  • main (QClaw) → kernel: openclaw                   │  │  │
│  │  │  • agent-3d43b911 (无不言) → kernel: openclaw        │  │  │
│  │  │  从 .qclaw-hermes/agent.json 读取:                    │  │  │
│  │  │  • hermes_default (林且慢) → kernel: hermes          │  │  │
│  │  │                                                      │  │  │
│  │  │  bindings: channel → agentId 路由规则                  │  │  │
│  │  │  (openclaw-weixin → main)                             │  │  │
│  │  └─────────────────────────────────────────────────────┘  │  │
│  │                                                           │  │
│  │  ┌─────────────────────────────────────────────────────┐  │  │
│  │  │  Kernel Pool (内核进程池)                             │  │  │
│  │  │                                                      │  │  │
│  │  │  ┌──────────────────┐  ┌──────────────────┐         │  │  │
│  │  │  │ OpenClaw Gateway │  │ Hermes Gateway   │         │  │  │
│  │  │  │ (Node.js)        │  │ (Python)         │         │  │  │
│  │  │  │ 端口 28789       │  │ gateway run      │         │  │  │
│  │  │  │ 多 Agent 共享    │  │ 单 Agent          │         │  │  │
│  │  │  │ (main + 自定义)  │  │ (hermes_default)  │         │  │  │
│  │  │  └──────────────────┘  └──────────────────┘         │  │  │
│  │  └─────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  数据目录 (~)                                              │  │
│  │                                                           │  │
│  │  ├── .qclaw/                    ← OpenClaw 内核            │  │
│  │  │   ├── openclaw.json          ← agents.list 注册表       │  │
│  │  │   └── agents/                ← per-Agent 目录           │  │
│  │  │       ├── main/              ← 默认 Agent              │  │
│  │  │       │   ├── agent/models.json                        │  │
│  │  │       │   └── sessions/*.jsonl                         │  │
│  │  │       └── agent-{uuid}/      ← 自定义 Agent             │  │
│  │  │           ├── agent/models.json                        │  │
│  │  │           └── sessions/*.jsonl                         │  │
│  │  │                                                       │  │
│  │  └── .qclaw-hermes/             ← Hermes 内核 (完全独立)   │  │
│  │      ├── config.yaml                                      │  │
│  │      ├── agent.json                                         │  │
│  │      └── sessions/session_*.json                          │  │
│  ───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

**关键修正**：
- Agent 注册表在 `openclaw.json` 的 `agents.list` 段，不是独立的 `agents.json`
- OpenClaw 支持多 Agent，每个 Agent 有独立的 `~/.qclaw/agents/{id}/` 目录
- Hermes 仅支持单 Agent（`hermes_default`），数据在 `~/.qclaw-hermes/`
- Session Key 包含完整 channel 信息：`agent:{agentId}:{channel}:{chatType}:{from}`

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

### 4.1 双内核预装结构（经安装包验证）

```
QClaw.app/Contents/Resources/
│
├── app.asar                    ← Electron 主进程代码 (V8 字节码编译, 107MB)
│   └── out/main/index.cjsc     ← 字节码编译后的主进程代码
│       └── bytecode-loader.cjs ← V8 bytecode 反序列化加载器
│
├── app.asar.unpacked/
│   └── node_modules/
│       ├── @tencent/            ← 腾讯 SDK (QQ Bot 等)
│       └── better-sqlite3/      ← SQLite 原生模块
│
├── openclaw.tar.gz              ← OpenClaw 运行时 (162MB, 安装时解压)
│   └── openclaw/                ← 解压后目录
│       ├── node/                ← 内置 Node.js 22+
│       └── node_modules/        ← OpenClaw + 依赖
│
├── hermes.tar.gz                ← Hermes 运行时 (126MB, 安装时解压)
│   └── hermes/                  ← 解压后目录
│       └── hermes               ← Hermes 二进制
│
├── hermes-plugins/              ← Hermes 插件 (Python 包)
│   └── qclaw-plugin-hermes/
│       ├── adapter/             ← 适配层 (dispatch.py, register.py)
│       ├── core_kit/            ← 核心工具包
│       └── plugin.yaml
│
├── node/node                    ← 内置 Node.js 二进制 (110MB)
├── scripts/
│   ├── pack-qclaw.cjs           ← 问题反馈打包脚本
│   └── unpack-openclaw.cjs      ← 安装时 OpenClaw tar 解压脚本
├── channel.json                 ← 当前渠道 ({"channel": 5001})
└── icon.icns
```

### 4.2 内核启动命令（经实际数据验证）

```bash
# OpenClaw Gateway (从 qclaw.json 中的 cli 字段推断)
/Applications/QClaw.app/Contents/Resources/node/node \
  /Users/qiududu/Library/Application Support/QClaw/openclaw/node_modules/openclaw/openclaw.mjs \
  gateway --config /Users/qiududu/.qclaw/openclaw.json --port 28789

# Hermes Gateway (从 gateway_state.json argv 字段)
/Users/qiududu/Library/Application Support/QClaw/hermes/hermes \
  gateway run --replace \
  --config /Users/qiududu/.qclaw-hermes/config.yaml
```

### 4.3 代码保护机制

QClaw 使用 **V8 字节码编译** 保护主进程代码：

```javascript
// out/main/index.cjs
"use strict";
require("./bytecode-loader.cjs");
require("./index.cjsc");
```

`.cjsc` 文件是 V8 编译后的字节码，通过自定义 `Module._extensions[".cjsc"]` 加载器反序列化。加载器使用 `vm.Script` + `cachedData` 机制，设置 `--no-lazy` 和 `--no-flush-bytecode` V8 标志。

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

### 5.1 实际目录结构（第二轮验证，完整修正）

**OpenClaw 和 Hermes 使用完全独立的数据目录，不是统一目录下的子目录。**

**关键修正**：OpenClaw 内核支持多 Agent，`openclaw.json` 的 `agents.list` 段就是注册表。

```
~/.qclaw/                              ← OpenClaw 数据目录
├── openclaw.json                      ← OpenClaw 核心配置 (JSON)
│                                      #   - agents.defaults: 全局默认 (model, workspace, maxConcurrent)
│                                      #   - agents.list: [多 Agent 定义数组] ← 这就是注册表
│                                      #   - bindings: channel → agent 绑定规则
│                                      #   - skills: extraDirs + entries 开关
│                                      #   - models: providers + merge 模式
│                                      #   - plugins: 插件系统 (wechat-access, lossless-claw, ...)
│                                      #   - channels: channel 配置
│                                      #   - gateway: 端口/模式/认证
├── qclaw.json                         ← QClaw 产品配置
│   ├── authGatewayBaseUrl: http://127.0.0.1:19000/proxy
│   ├── stateDir: ~/.qclaw
│   ├── port: 28789
│   └── cli: { nodeBinary, openclawMjs, pid }
├── channel-defaults.json              ← channel 默认目标映射
│                                      #   - {agentId: {channel: {to: userId}}}
│
├── agents/                            ← Agent 目录（多 Agent 支持）
│   ├── main/                          ← 默认 Agent (QClaw)
│   │   ├── agent/
│   │   │   └── models.json            ← Agent 专属模型配置
│   │   └── sessions/
│   │       ├── sessions.json          ← 会话索引 (per-agent)
│   │       │                          #   - Key: "agent:{agentId}:{channel}:{chatType}:{from}"
│   │       │                          #   - Value: {sessionId, origin, deliveryContext,
│   │       │                          #             skillsSnapshot, systemPromptReport, ...}
│   │       └── *.jsonl                ← 会话数据 (JSONL, 逐行追加)
│   │
│   └── agent-{uuid}/                  ← 自定义 Agent (如: 无不言)
│       ├── agent/
│       │   └── models.json            ← 该 Agent 的模型配置
│       └── sessions/
│           ├── sessions.json          ← 该 Agent 的会话索引
│           └── *.jsonl                ← 该 Agent 的会话数据
│
├── skills/                            ← OpenClaw Skills
├── plugins/                           ← OpenClaw Plugins
├── memory/lossless/lcm.db             ← 长期记忆 (SQLite, lossless-claw 插件)
├── flows/registry.sqlite              ← 流程注册表 (SQLite)
├── workspace/                         ← 默认工作区 (main agent)
│   ├── AGENTS.md / SOUL.md / TOOLS.md / IDENTITY.md / USER.md
└── workspace-agent-{id}/             ← 自定义 Agent 工作区


~/.qclaw-hermes/                       ← Hermes 数据目录 (完全独立)
├── config.yaml                        ← Hermes 配置 (YAML)
│   ├── model.default: modelroute
│   ├── model.base_url: http://127.0.0.1:19000/proxy/llm
│   └── plugins.enabled: [qclaw-plugin-hermes]
├── agent.json                         ← 唯一的 Agent 元数据 (JSON)
│   ├── agentId: hermes_default
│   ├── name, vibe, avatar, emoji
│   ├── bio: [{title, desc}, ...]
│   ├── skills: [技能名数组]
│   ├── sessionIds: [会话ID数组]
│   ├── sessionTitles: {id: title}
│   └── sessionUpdatedAts: {id: timestamp}
├── SOUL.md                            ← 角色设定 (Markdown)
├── sessions/
│   └── session_{uuid}.json            ← 会话数据 (完整 JSON)
│       ├── session_id, model, base_url
│       ├── system_prompt: 完整 Markdown (含身份设定)
│       ├── tools: [function calling schema]
│       └── messages: [{role, content, reasoning?, ...}]
├── state.db / audit.db / response_store.db  ← SQLite 数据库
├── gateway_state.json                 ← Gateway 状态
│   ├── pid, kind: "hermes-gateway"
│   ├── gateway_state: "running"
│   ├── platforms: {api_server: {state: "connected"}}
├── gateway.pid / gateway.lock         ← 进程管理
├── channel_directory.json             ← 频道目录
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

### 6.1 核心设计决策（第二轮实际数据修正）

| 决策 | 方案 | 原因 | 实际验证 |
|------|------|------|----------|
| **数据目录隔离** | OpenClaw: `~/.qclaw/`，Hermes: `~/.qclaw-hermes/` | 两个内核各自管理自己的数据，互不干扰 | ✅ QClaw 实际如此 |
| **配置格式差异** | OpenClaw JSON，Hermes YAML | 各用各的原生格式，不强求统一 | ✅ QClaw 实际如此 |
| **会话格式差异** | OpenClaw JSONL，Hermes JSON | 各用各的存储方式 | ✅ QClaw 实际如此 |
| **Agent 注册** | `openclaw.json` 的 `agents.list` 段 | 集中配置，不需要额外文件 | ✅ 第二轮验证修正 |
| **多 Agent 支持** | OpenClaw 支持多 Agent，Hermes 仅单 Agent | OpenClaw 已实现 per-agent 目录隔离 | ✅ agents/main/ + agents/agent-3d43b911/ |
| **Channel 绑定** | `bindings` 数组定义 channel → agent 路由 | 自动将不同渠道消息路由到指定 Agent | ✅ openclaw.json 验证 |
| **共享本地代理** | 两者都走 127.0.0.1:19000 | 统一管理模型路由、计费、状态 | ✅ 两者 baseUrl 都是 19000 |
| **角色设定** | OpenClaw: workspace 文件，Hermes: SOUL.md | 各自用各自的方式 | ✅ 实际验证 |
| **内核进程** | OpenClaw 默认启动，Hermes 按需启动 | 节省资源 | ✅ gateway_state.json 证明 |
| **代码保护** | V8 字节码编译 (.cjsc) | 防止逆向工程 | ✅ bytecode-loader.cjs 验证 |
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

## 七、总结（第二轮实际数据修正）

QClaw 的多内核管理本质上是：

1. **双目录隔离** — OpenClaw (`~/.qclaw/`) 和 Hermes (`~/.qclaw-hermes/`) 各自有完全独立的数据目录
2. **OpenClaw 多 Agent** — `openclaw.json` 的 `agents.list` 段定义多个 Agent，每个 Agent 有独立目录 (`~/.qclaw/agents/{id}/`)
3. **Hermes 单 Agent** — 目前仅支持 `hermes_default`，数据在 `~/.qclaw-hermes/`
4. **Session Key 含 Channel** — 格式为 `agent:{agentId}:{channel}:{chatType}:{from}`，完整标识会话来源
5. **Channel → Agent 绑定** — `bindings` 数组定义路由规则（如 openclaw-weixin → main）
6. **配置格式不同** — OpenClaw 用 JSON，Hermes 用 YAML；会话存储前者用 JSONL 后者用完整 JSON
7. **共享本地代理** — 两者都走 `127.0.0.1:19000` 同一个模型路由代理
8. **角色设定方式不同** — OpenClaw 用 workspace/ 下的 AGENTS.md、SOUL.md 等文件注入，Hermes 用 `SOUL.md` 文件 + `agent.json` bio 段
9. **Gateway 状态管理** — Hermes 有完整的 `gateway_state.json`（含 pid/platforms 状态），OpenClaw 依赖 qclaw.json 中的 pid
10. **代码保护** — 主进程代码使用 V8 字节码编译（.cjsc），防止逆向
11. **Plugin 插件系统** — OpenClaw 有完整的插件系统（wechat-access, lossless-claw 记忆等），Hermes 有 Python 插件（qclaw-plugin-hermes）
12. **前端统一抽象** — 主进程负责格式转换，前端只看到 `UnifiedSession`

> 详细的会话数据格式对比、前端加载流程、主进程转换代码见 **[MULTI_SESSION_ARCHITECTURE.md](MULTI_SESSION_ARCHITECTURE.md)**。
> 模型路由与多模型管理见 **[MODEL_ROUTING.md](MODEL_ROUTING.md)**。
> 长期记忆架构对比（OpenClaw lossless-claw vs Hermes 内置 memory）见 **[LONG_TERM_MEMORY.md](LONG_TERM_MEMORY.md)**。
