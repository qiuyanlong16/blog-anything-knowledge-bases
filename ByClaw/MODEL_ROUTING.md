# QClaw 模型路由与多模型管理 — 技术调研

> **调研日期**: 2026-05-12
> **参考**: QClaw 产品实际运行截图 + 用户本地 `.qclaw` 目录结构分析

---

## 一、核心发现

### 1.1 实际配置 vs UI 展示的差异

**openclaw.json 中只配置了 2 个 Provider**：

| Provider | 模型 ID | 说明 |
|----------|---------|------|
| `qclaw` | `modelroute` | 走本地代理 `http://127.0.0.1:19000/proxy/llm` |
| `deepseek` | `deepseek-reasoner` | 直连 DeepSeek 官方 API |

**但 UI 中展示的模型远不止这些**：

```
Kimi-K2.6      (爆满)
Kimi-K2.5      (爆满)
GLM-5.1        (爆满)
GLM-5.0        (爆满)
GLM-5.0-Turbo
深度求索 (DeepSeek) - deepseek-reasoner
deepseek-reasoner
自定义大模型
```

**关键矛盾**: UI 展示的 Kimi/GLM 模型在配置中不存在，但用户能选能聊。

---

## 二、架构解析

### 2.1 本地代理路由网关

```
─────────────────────────────────────────────────────────────────┐
│                        QClaw 整体架构                            │
│                                                                 │
│  用户操作: 选择 "Kimi-K2.6" 发送消息                              │
│       │                                                         │
│       ▼                                                         │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Electron Main Process                                    │  │
│  │                                                           │  │
│  │  ┌────────────────────────────────────────────────────┐  │  │
│  │  │  本地模型路由代理 (127.0.0.1:19000)                 │  │  │
│  │  │                                                     │  │  │
│  │  │  功能:                                              │  │  │
│  │  │  1. 聚合多家 LLM 提供商的模型列表                    │  │  │
│  │  │  2. 动态查询模型状态 (可用/爆满)                     │  │  │
│  │  │  3. 路由转发请求到实际 LLM API                       │  │  │
│  │  │  4. 统一计费/限流/负载均衡                           │  │  │
│  │  │                                                     │  │  │
│  │  │  端点: /proxy/llm (OpenAI 兼容接口)                  │  │  │
│  │  │  端点: /proxy/llm/v1/models (返回聚合模型列表)       │  │  │
│  │  └────────────────────────────────────────────────────┘  │  │
│  │                                                           │  │
│  │  ┌────────────────────────────────────────────────────┐  │  │
│  │  │  OpenClaw Gateway (子进程)                           │  │  │
│  │  │                                                     │  │  │
│  │  │  openclaw.json:                                     │  │  │
│  │  │  {                                                  │  │  │
│  │  │    "providers": {                                   │  │  │
│  │  │      "qclaw": {                                     │  │  │
│  │  │        "baseUrl": "http://127.0.0.1:19000/proxy/llm",│ │ │
│  │  │        "api": "openai-completions",                  │  │  │
│  │  │        "models": [{ "id": "modelroute" }]            │  │  │
│  │  │      },                                              │  │  │
│  │  │      "deepseek": {                                   │  │  │
│  │  │        "baseUrl": "https://api.deepseek.com/",       │  │  │
│  │  │        "apiKey": "sk-c5f14cca...",                   │  │  │
│  │  │        "api": "openai-completions",                  │  │  │
│  │  │        "models": [{ "id": "deepseek-reasoner" }]     │  │  │
│  │  │      }                                               │  │  │
│  │  │    }                                                 │  │  │
│  │  │  }                                                   │  │  │
│  │  └────────────────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  实际 LLM 提供商 (路由目标)                                │  │
│  │                                                           │  │
│  │  modelroute (via 19000 代理):                             │  │
│  │    ├─ kimi-k2.6       → 腾讯/月之暗面 Kimi API            │  │
│  │    ├─ kimi-k2.5       → 月之暗面 Kimi API                 │  │
│  │    ├─ glm-5.1         → 智谱 GLM API                      │  │
│  │    ├─ glm-5.0         → 智谱 GLM API                      │  │
│  │    ├─ glm-5.0-turbo   → 智谱 GLM API                      │  │
│  │    ├─ pool-minimax-m2.7 → Minimax API                    │  │
│  │    └─ ... (动态扩展)                                       │  │
│  │                                                           │  │
│  │  deepseek (直连):                                         │  │
│  │    └─ deepseek-reasoner → https://api.deepseek.com/      │  │
│  │                                                           │  │
│  │  自定义模型:                                              │  │
│  │    └─ 用户自行配置的任意 OpenAI 兼容 API                    │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 模型请求流程

```
步骤 1: 用户打开模型选择下拉框
  │
  ▼
  前端请求 /proxy/llm/v1/models (或 Electron IPC)
  │
  ▼
  本地代理 127.0.0.1:19000/proxy/llm/v1/models
  │
  ├── 查询已配置的 provider (qclaw, deepseek)
  ├── 对 qclaw provider:
  │   └── 从内部配置/腾讯模型管理服务获取模型列表
  │       → [{ id: "kimi-k2.6", status: "full" }, ...]
  ├── 对 deepseek provider:
  │   └── 使用配置的 apiKey 调用 deepseek /v1/models
  │       → [{ id: "deepseek-reasoner", ... }]
  ├── 合并所有 provider 的模型列表
  └── 返回前端: 聚合后的完整模型列表
  │
步骤 2: 用户选择 "kimi-k2.6" 发送消息
  │
  ▼
  前端发送消息 + modelId="kimi-k2.6"
  │
  ▼
  Electron IPC → OpenClaw Gateway
  │
  ▼
  OpenClaw 查找 modelId 对应的 provider:
  │
  ├── kimi-k2.6 → 匹配 "qclaw" provider
  │   └── 请求转发到 baseUrl: http://127.0.0.1:19000/proxy/llm
  │       └── 本地代理收到请求，识别 model=kimi-k2.6
  │           └── 转发到实际的 Kimi API 端点
  │
  └── deepseek-reasoner → 匹配 "deepseek" provider
      └── 直连 https://api.deepseek.com/v1/chat/completions
  │
步骤 3: 响应返回 + 会话持久化
  │
  ▼
  Session JSONL 中记录: "modelId": "kimi-k2.6"
```

### 2.3 Session 中 modelId 的存储机制

用户本地 `.qclaw/agents/main/sessions/` 目录下，每个会话是一个 JSONL 文件：

```jsonl
// xxx.session.jsonl
{"type": "session", "version": 3, "id": "c8c2efeb-...", ...}
{"type": "model_change", "id": "1a9f8928", "modelId": "pool-minimax-m2.7", ...}
{"type": "custom", "customType": "model_snapshot", "data": {
  "modelId": "pool-minimax-m2.7", "provider": "qclaw"
}}
{"type": "message", "id": "7b0735e0", "message": {"role": "user", ...}}
{"type": "message", "id": "8eba12ce", "message": {"role": "assistant", ...},
 "model": "pool-minimax-m2.7", "provider": "qclaw", ...}
```

**关键发现**:
- `modelId` 原样存储，不做校验或映射
- 每条消息都记录了实际使用的 `model` 和 `provider`
- `model_change` 事件记录了切换历史
- `model_snapshot` 用于会话恢复

---

## 三、技术实现推断

### 3.1 本地代理的启动方式

```typescript
// QClaw Electron 主进程启动流程

import { spawn } from 'child_process';
import path from 'path';

// 1. 启动本地模型路由代理
const proxyProcess = spawn(
  getProxyExecutable(),  // 可能是独立的 Go/Rust 二进制，或 Node.js 服务
  ['--port', '19000', '--config', proxyConfigPath],
  {
    stdio: 'pipe',
    windowsHide: true,
  }
);

// 2. 等待代理就绪
await waitForProxy('http://127.0.0.1:19000/health');

// 3. 注入环境变量
process.env.QCLAW_LLM_BASE_URL = 'http://127.0.0.1:19000/proxy/llm';
// QCLAW_LLM_API_KEY 可能由代理内部管理

// 4. 启动 OpenClaw Gateway (继承父进程环境变量)
const openclawProcess = spawn(
  getNodeExecutable(),
  ['--', path.join(__dirname, 'resources', 'openclaw', 'gateway.js')],
  {
    env: process.env,  // ← 包含 QCLAW_LLM_BASE_URL
    stdio: 'pipe',
  }
);
```

### 3.2 本地代理的模型聚合逻辑

```typescript
// 本地代理 /v1/models 端点伪代码

async function getAggregatedModels(): Promise<ModelInfo[]> {
  const models: ModelInfo[] = [];

  // 1. 从腾讯模型管理服务获取代理路由的模型
  const proxyModels = await fetch('https://internal-qclaw-models.tencent.com/v1/models', {
    headers: { 'Authorization': `Bearer ${INTERNAL_TOKEN}` }
  });
  for (const m of proxyModels) {
    models.push({
      id: m.id,                    // "kimi-k2.6"
      provider: 'qclaw',           // 映射到 openclaw.json 中的 qclaw provider
      baseUrl: 'http://127.0.0.1:19000/proxy/llm',
      status: m.status,            // "available" / "full" (爆满)
      cost: m.costMultiplier,      // x1.6 积分
    });
  }

  // 2. 从配置的直连 provider 获取模型
  const providers = loadOpenclawConfig().providers;
  for (const [name, config] of Object.entries(providers)) {
    if (name === 'qclaw') continue; // 已处理
    const remoteModels = await fetch(`${config.baseUrl}/v1/models`, {
      headers: { 'Authorization': `Bearer ${config.apiKey}` }
    });
    for (const m of remoteModels) {
      models.push({
        id: m.id,
        provider: name,            // "deepseek"
        baseUrl: config.baseUrl,
        status: 'available',
      });
    }
  }

  // 3. 加载用户自定义模型
  const customModels = loadCustomModels(); // 从 qclaw.json 或本地存储
  for (const m of customModels) {
    models.push({
      id: m.id,
      provider: 'custom',
      baseUrl: m.baseUrl,
      status: 'available',
    });
  }

  return models;
}
```

### 3.3 本地代理的请求转发

```typescript
// 本地代理 /proxy/llm/v1/chat/completions 端点

async function proxyChatCompletion(request: ChatRequest): Promise<Response> {
  const modelId = request.model;

  // 1. 查找 modelId 对应的实际 API 端点
  const route = modelRouteTable.get(modelId);
  if (!route) throw new Error(`Unknown model: ${modelId}`);

  // 2. 构建目标请求
  const targetUrl = route.actualBaseUrl + '/v1/chat/completions';
  const targetHeaders = {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${route.apiKey}`,
  };

  // 3. 转发请求 (支持流式)
  const response = await fetch(targetUrl, {
    method: 'POST',
    headers: targetHeaders,
    body: JSON.stringify({
      ...request,
      model: route.actualModelId, // 可能需要映射 modelId
    }),
  });

  // 4. 记录计费/用量
  await recordUsage(modelId, request, response);

  // 5. 返回响应 (原样透传)
  return response;
}

// 路由表示例
const modelRouteTable = new Map([
  ['kimi-k2.6',       { actualBaseUrl: 'https://api.moonshot.cn/v1', actualModelId: 'moonshot-v1-128k', apiKey: '...' }],
  ['glm-5.1',         { actualBaseUrl: 'https://open.bigmodel.cn/api/paas/v4', actualModelId: 'glm-4-plus', apiKey: '...' }],
  ['pool-minimax-m2.7', { actualBaseUrl: 'https://api.minimax.chat/v1', actualModelId: 'abab6.5-chat', apiKey: '...' }],
  ['deepseek-reasoner', { actualBaseUrl: 'https://api.deepseek.com', actualModelId: 'deepseek-reasoner', apiKey: '...' }],
]);
```

---

## 四、对我们要构建的产品的启示

### 4.1 架构建议

| 组件 | 实现方式 | 说明 |
|------|----------|------|
| **本地代理** | Electron 主进程内嵌 HTTP 服务 (port 19000) | 用 Node.js `http` 模块即可，无需额外二进制 |
| **模型配置** | `qclaw.json` 或独立 `models.json` | 用户可通过 UI 管理，持久化到本地 |
| **Provider 管理** | 支持 `proxy` (本地代理) + `direct` (直连) + `custom` (自定义) | 三类并存 |
| **模型列表** | 聚合所有 Provider 的模型，缓存 + 定期刷新 | 避免每次打开都调用远程 API |
| **路由表** | modelId → 实际 API 端点的映射 | 支持 modelId 别名映射 |
| **状态管理** | 模型可用性/配额/限流信息 | "爆满" 状态从上游获取 |

### 4.2 本地代理实现方案（Node.js 内嵌）

```typescript
// packages/shell/src/main/core/model-proxy.ts

import http from 'http';
import { OpenClawAdapter } from '../adapters/openclaw-adapter';

export class ModelProxy {
  private server: http.Server | null = null;
  private port: number;
  private routeTable: Map<string, ModelRoute>;

  constructor(port = 19000) {
    this.port = port;
    this.routeTable = new Map();
  }

  async start(): Promise<void> {
    this.server = http.createServer((req, res) => {
      if (req.url === '/proxy/llm/v1/models') {
        return this.handleModelsList(res);
      }
      if (req.url === '/proxy/llm/v1/chat/completions') {
        return this.handleChatCompletion(req, res);
      }
      res.writeHead(404);
      res.end();
    });

    this.server.listen(this.port, '127.0.0.1');
    console.log(`Model proxy started on 127.0.0.1:${this.port}`);
  }

  private async handleModelsList(res: http.ServerResponse): Promise<void> {
    const models = await this.aggregateModels();
    res.writeHead(200, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({ data: models }));
  }

  private async handleChatCompletion(
    req: http.IncomingMessage,
    res: http.ServerResponse
  ): Promise<void> {
    // 解析请求体，获取 modelId
    // 查找路由表，转发到实际 API
    // 支持 SSE 流式响应
  }
}
```

### 4.3 与 OpenClaw 的集成

```json
// openclaw.json — Provider 配置
{
  "providers": {
    "local-proxy": {
      "baseUrl": "http://127.0.0.1:19000/proxy/llm",
      "api": "openai-completions",
      "models": [{
        "id": "modelroute",
        "name": "模型路由",
        "input": ["text", "image"]
      }]
    },
    "deepseek": {
      "baseUrl": "https://api.deepseek.com/",
      "apiKey": "${DEEPSEEK_API_KEY}",
      "api": "openai-completions",
      "models": [{
        "id": "deepseek-reasoner",
        "name": "DeepSeek Reasoner"
      }]
    }
  }
}
```

### 4.4 与 Hermes Agent 的统一

Hermes Agent 同样可以通过本地代理访问模型：

```
Hermes Agent
  │
  ├── 配置指向本地代理: LLM_BASE_URL=http://127.0.0.1:19000/proxy/llm
  │
  ── 与 OpenClaw 共享同一套模型路由和计费逻辑
```

---

## 五、目录结构

```
.qclaw/
├── qclaw.json              # QClaw 产品配置 (包含模型路由配置)
├── openclaw.json           # OpenClaw 核心配置 (包含 provider 定义)
├── agents/                 # Agent 数据
│   ├── main/
│   │   └── sessions/       # 会话 JSONL 文件
│   │       ├── xxx.jsonl   # 每条消息记录 model/provider
│   │       └── yyy.jsonl
│   └── ...
├── memory/                 # 记忆数据
├── plugins/                # 插件
├── skills/                 # 技能
└── workspace/              # 工作区
```

---

## 六、总结

QClaw 的模型管理核心是一个 **本地代理路由层**：

1. **本地代理 (127.0.0.1:19000)** 聚合多家 LLM 提供商，对外提供统一接口
2. **OpenClaw 只感知到代理**，不关心背后有多少家模型提供商
3. **Provider 可混合** — `qclaw` 走代理、`deepseek` 直连、用户自定义任意 API
4. **Session 原样记录 modelId** — 不做映射，保持透明
5. **环境变量由主进程动态注入** — 不在系统级暴露

这种设计实现了模型管理的 **解耦 + 扩展性 + 安全性**，非常值得在我们的产品中复用。
