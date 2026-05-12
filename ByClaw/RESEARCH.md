# 跨平台桌面 AI Agent 产品 — 技术调研与项目初始化文档

> **调研日期**: 2026-05-12
> **项目定位**: 类 QClaw 桌面产品，集成 OpenClaw + Hermes Agent 双引擎
> **目标平台**: macOS / Windows / Ubuntu Linux
> **最终选型**: Electron

---

## 目录

1. [需求梳理](#一需求梳理)
2. [竞品分析](#二竞品分析)
3. [候选框架对比](#三候选框架对比)
4. [最终选型：Electron](#四最终选型electron)
5. [整体架构设计](#五整体架构设计)
6. [安全设计（四层防御体系）](#六安全设计四层防御体系)
7. [项目目录结构](#七项目目录结构)
8. [核心实现方案](#八核心实现方案)
9. [打包与分发](#九打包与分发)
10. [OpenClaw 升级策略](#十openclaw-升级策略)
11. [Hermes Agent 集成](#十一hermes-agent-集成)
11. [模型路由与多模型管理](#十一附模型路由与多模型管理) — 详见 [MODEL_ROUTING.md](MODEL_ROUTING.md)
11. [Agent 内核切换](#十一附二agent-内核切换) — 详见 [KERNEL_SWITCHING.md](KERNEL_SWITCHING.md)
11. [多会话架构](#十一附三多会话架构) — 详见 [MULTI_SESSION_ARCHITECTURE.md](MULTI_SESSION_ARCHITECTURE.md)
12. [实施计划](#十二实施计划)
13. [参考项目](#十三参考项目)

---

## 一、需求梳理

### 1.1 核心诉求

| 需求 | 说明 |
|------|------|
| **开箱即用** | 用户安装后无需单独安装 Node.js、Python 等运行时 |
| **OpenClaw 内置** | 选定版本的 OpenClaw 随桌面应用一起打包，包含团队深度二次开发（模型切换、多 Channel、埋点、网关守护等） |
| **Hermes Agent 集成** | Mac/Ubuntu 开箱即用；Windows 用户点击按钮后在 WSL2 中部署 |
| **Agent 切换** | 用户可在 UI 中切换使用 OpenClaw 或 Hermes |
| **本地 IPC 通信** | 优先使用 IPC（stdio），而非 HTTP 端口 |
| **Web 团队全栈** | 充分利用团队 Web 开发能力 |

### 1.2 两个 Agent 的技术特征

**OpenClaw Gateway**:
- Node.js/TypeScript 项目，需要 Node.js >= 22.16
- 默认 WebSocket 服务器（端口 18789），可配置
- 提供 HTTP REST API（`POST /v1/chat/completions` OpenAI 兼容，默认关闭）
- 自定义 JSON-RPC 风格的 WebSocket 帧协议
- 插件系统：支持自定义 Channel、Tool、Hook
- 配置存储在 `~/.openclaw/openclaw.json`
- 状态管理：`~/.openclaw/workspace/MEMORY.md`、SQLite 向量存储
- 团队已对其做大量二次开发

**Hermes Agent (Nous Research)**:
- Python 项目，需要 Python >= 3.11
- 多种运行模式：CLI/TUI、Gateway、ACP（JSON-RPC over stdio）
- **ACP 模式专为编辑器/桌面集成设计**：`hermes acp` 通过 stdin/stdout 交换 JSON-RPC 消息
- 可启用 OpenAI 兼容 HTTP API
- 支持自学习循环（skills from experience）
- 100,000+ GitHub Stars

---

## 二、竞品分析

### 2.1 QClaw（腾讯）

QClaw 是腾讯电脑管家团队基于 OpenClaw 打造的本地 AI 助手，支持微信远程操控。

| 维度 | 详情 |
|------|------|
| **框架** | **Electron** |
| **前端** | Vue 3 + TypeScript |
| **打包** | Electron Builder + NSIS (Windows) |
| **核心卖点** | 一键安装、自动处理依赖、数据不出本机、微信远程控制 |
| **架构** | Electron 壳 + OpenClaw 核心 + 插件系统 |

QClaw 的处理方式：
- 安装包内预构建 OpenClaw 运行时
- 自动检测环境（Node.js 版本、WSL on Windows）
- 自动安装缺失依赖
- 预打包完整文档处理技能（PDF、XLSX、DOCX）

### 2.2 同类项目参考

| 项目 | 团队 | 框架 | 特点 |
|------|------|------|------|
| **LobsterAI** | 网易有道 | Electron + electron-builder | 预构建 OpenClaw 运行时到 `Resources/cfmind`，`package.json` 固定版本 |
| **ClawX** | ValueCell | Electron + Vite + electron-builder | OpenClaw Gateway GUI 集成 |
| **EasyClaw** | 社区 | Electron | OpenClaw 一键安装器 |

**关键发现**: 所有类 OpenClaw 桌面产品均选择 Electron，无一例外。

---

## 三、候选框架对比

### 3.1 候选框架概览

| 框架 | 后端语言 | 打包体积 | 空闲内存 | 安全模型 | Web 团队覆盖度 |
|------|----------|----------|----------|----------|----------------|
| **Electron** | Node.js/TS | 80-200 MB | 150-400 MB | 可配置 | **100%** |
| **Tauri v2** | Rust | 3-10 MB | 30-80 MB | Capabilities 系统 | ~40% |
| **Wails v2** | Go | 8-15 MB | 50-100 MB | 开发者自控 | ~60% |
| Neutralinojs | 无 | 1-2 MB | 30-80 MB | 基础 | ~70% |
| NW.js | Node.js/TS | 80-180 MB | 150-350 MB | 弱 | 100% |
| Flutter Desktop | Dart | 15-50 MB | 80-200 MB | 平台沙箱 | 0% (非 Web) |

### 3.2 三个核心候选的深度对比

#### Electron

**架构**: 内置完整 Chromium + Node.js，主进程（Node.js）与渲染进程通过 IPC 通信。

**优势**:
- Web 团队 100% 全栈，一个团队搞定所有代码
- 可直接 `require()` OpenClaw 内部模块（零通信开销）
- 如果 OpenClaw 不支持 stdio，可直接用其原生 WebSocket API
- 最成熟的生态系统（npm 周下载 ~500 万）
- QClaw、LobsterAI、ClawX 等已踩完所有坑
- 自定义 Hook/插件/埋点全部在同一 TS 代码库中

**劣势**:
- 打包体积大（80-200 MB + OpenClaw 依赖）
- 内存占用高（150-400 MB 空闲）
- 安全配置不当容易导致 XSS-to-RCE

#### Tauri v2

**架构**: 系统内置 WebView + Rust 后端，前后端通过类型化 IPC 桥接。

**优势**:
- 极小包体积（3-10 MB）
- 极低内存占用（30-80 MB）
- 最强安全模型（Capabilities 系统）

**劣势**:
- **深度二次开发场景下，所有定制逻辑需 Rust 重写**
- 只能通过 IPC 调用 OpenClaw 暴露的接口，无法直接访问内部模块
- Web 团队只能做前端（约 40% 工作量），需要 Rust 工程师
- OpenClaw 每次升级，Rust 适配层也需同步修改

#### Wails v2

**架构**: 系统内置 WebView + Go 后端，双向绑定暴露 Go 方法为 JS 函数。

**优势**:
- Go 学习曲线比 Rust 平缓（Web 开发者 1-2 周可上手）
- 打包体积适中（8-15 MB）
- 前端调用最简单（自动生成 TS 绑定）

**劣势**:
- 仍需 Go 后端代码（约 40% 工作量非 Web 团队可覆盖）
- 没有 Tauri 的 Capabilities 安全系统
- 无法直接 `require()` OpenClaw 模块

### 3.3 针对本项目的关键维度

| 维度 | Electron | Tauri v2 | Wails v2 |
|------|:---:|:---:|:---:|
| OpenClaw 深度定制 | 直接 require | 需 Rust 重写全部逻辑 | 需 Go 重写全部逻辑 |
| 自定义 Hook/插件 | 同一 TS 代码库 | 需通过 IPC 间接调用 | 需通过 IPC 间接调用 |
| Agent 切换逻辑 | TS 实现 | Rust 实现 | Go 实现 |
| 埋点/遥测 | TS 实现 | Rust 实现 | Go 实现 |
| 网关守护进程 | TS 实现 | Rust 实现 | Go 实现 |
| Hermes WSL2 部署 | TS 实现 | Rust 实现 | Go 实现 |
| OpenClaw 升级影响 | 只改适配层 | 改适配层 + IPC 层 | 改适配层 + IPC 层 |
| 团队组成 | 全 Web 团队 | Web + Rust 工程师 | Web + Go 工程师 |

---

## 四、最终选型：Electron

### 4.1 决定性因素

1. **QClaw 本身就是 Electron** — 腾讯、网易等所有类 OpenClaw 桌面产品均基于 Electron
2. **深度二次开发** — 团队对 OpenClaw 做了大量定制，Electron 可直接 `require()` OpenClaw 内部模块
3. **Web 团队全栈** — 所有代码（桌面壳 + 网关守护 + 插件 + 埋点）都是 TypeScript
4. **不动源码** — 通过 OpenClaw 插件系统（Hook/Channel/Event）注入定制逻辑
5. **升级隔离** — 适配层设计确保 OpenClaw 升级时只改一层代码
6. **生态验证** — 大量 OpenClaw 桌面化项目已验证此路径

### 4.2 技术栈确定

| 层级 | 技术选择 |
|------|----------|
| **桌面框架** | Electron 37+ (Chromium 138+, Node.js 22+) |
| **前端框架** | Vue 3 + TypeScript |
| **构建工具** | Vite 6 + pnpm workspaces + electron-builder 26 |
| **OpenClaw** | npm 依赖，版本固定在 package.json |
| **Hermes** | Mac/Ubuntu 预装 Python venv，Windows WSL2 按需部署 |
| **打包格式** | Windows: NSIS / macOS: DMG / Ubuntu: AppImage + deb |
| **语言** | 100% TypeScript |

---

## 五、整体架构设计

### 5.1 架构总览

```
┌─────────────────────────────────────────────────────────────────┐
│                          Electron 应用                           │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  Electron Main Process (Node.js / TypeScript)             │  │
│  │                                                           │  │
│  │  ┌─────────────┐  ┌──────────────┐  ┌─────────────────┐  │  │
│  │  │ 插件管理层   │  │ 网关守护进程  │  │  埋点/遥测层    │  │  │
│  │  │ PluginMgr   │  │ Guardian     │  │  Telemetry      │  │  │
│  │  └──────┬──────┘  └──────┬───────┘  └────────┬────────┘  │  │
│  │         │                │                    │           │  │
│  │  ┌──────┴────────────────┴────────────────────┴────────┐  │  │
│  │  │          插件适配层 (Plugin Adapter Layer)           │  │  │
│  │  │                                                     │  │  │
│  │  │  OpenClawAdapter  ─── 不修改 OpenClaw 源码          │  │  │
│  │  │  HermesAdapter    ─── 通过 API / Hook 注入定制逻辑   │  │  │
│  │  └──────────────────────┬──────────────────────────────┘  │  │
│  │                         │                                   │  │
│  │  ┌──────────────────────┴──────────────────────────────┐  │  │
│  │  │         OpenClaw Core (npm 依赖, 版本固定)           │  │  │
│  │  │                                                     │  │  │
│  │  │  const { Gateway } = require('openclaw')            │  │  │
│  │  │                                                     │  │  │
│  │  │  → 模型路由 / Channel 管理 / 会话管理 / 技能执行     │  │  │
│  │  └─────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  Electron Renderer Process (Vue 3 + TypeScript)           │  │
│  │                                                           │  │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  │  │
│  │  │ 聊天页面  │  │ 模型配置  │  │ Agent切换 │  │ 埋点面板  │  │  │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘  │  │
│  │                                                           │  │
│  │  ◄───── ipcRenderer / contextBridge ─────►                │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  Hermes Agent (独立子进程, stdio ACP 模式)                 │  │
│  │  Mac/Ubuntu: 预装 + hermes acp                            │  │
│  │  Windows: WSL2 按需部署                                    │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 通信链路

```
用户操作
  │
  ▼
Vue 组件 (Renderer, packages/web/src/)
  │  window.electronAPI.xxx()  (通过 contextBridge 暴露的安全 API)
  ▼
preload.ts  (contextBridge, packages/shell/src/preload/)
  │  ipcRenderer.invoke() / ipcRenderer.on()
  ▼
Electron Main Process (Node.js, packages/shell/src/main/)
  │  ipcMain.handle() / mainWindow.webContents.send()
  ▼
OpenClawAdapter / HermesAdapter
  │  require('openclaw')  或  spawn('hermes', ['acp'])
  ▼
OpenClaw Gateway / Hermes Agent
```

### 5.3 安全模型

> **设计原则**: 性能优先 > 安全分级防御。以下安全措施中，标记 ⚡ 的为零性能损耗，标记 🔥 的为有微小开销但可接受，标记 ⏱️ 的为有可测量开销（按需启用）。

```
┌────────────────────────────────────────────────────────────────┐
│                      四层安全防御体系                            │
│                                                                │
│  第一层: 进程隔离 (⚡ 零开销)                                    │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  sandbox: true           → 渲染进程操作系统级沙箱          │  │
│  │  nodeIntegration: false   → 禁止 Node.js 注入             │  │
│  │  contextIsolation: true   → 上下文隔离防原型链污染         │  │
│  │  webSecurity: true        → 同源策略 + CSP 强制执行       │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                │
│  第二层: 通信防护 (⚡ 零开销)                                    │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  contextBridge 最小 API 表面                               │  │
│  │  IPC 参数白名单校验 (zod/schema)                           │  │
│  │  禁止 event.sender.send 动态调用                           │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                │
│  第三层: 资源保护 (⚡ 构建时处理, ⏱️ 运行时微小开销)              │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  ASAR 归档 + AES-256 加密                                  │  │
│  │  ASAR 完整性校验 (asarIntegrity)                          │  │
│  │  CSP 策略: script-src 'self'                               │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                │
│  第四层: 运行时加固 (🔥 微小开销, 可按需启用)                     │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  禁用 DevTools (生产环境)                                  │  │
│  │  禁用 allowRunningInsecureContent                          │  │
│  │  安全导航控制 (禁止外部链接加载)                            │  │
│  │  代码混淆 (JSC/obfuscator, 按需)                           │  │
│  └──────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────┘
```

---

## 六、安全设计（四层防御体系）

> **核心原则**: 性能第一优先，安全分级防御。所有零开销 (⚡) 措施全部启用，有开销 (🔥/⏱️) 的措施按需选择。
> 参考: [Electron 代码防泄露全攻略](https://juejin.cn/post/7626891132237103131)、Electron 官方安全指南

### 6.1 第一层：进程沙箱隔离（⚡ 零性能开销）

这是 Electron 安全的基础，也是防御代码注入、权限滥用的第一道防线。**必须全部启用**。

```typescript
// packages/shell/src/main/index.ts — BrowserWindow 安全配置

mainWindow = new BrowserWindow({
  width: 1200,
  height: 800,
  webPreferences: {
    // ⚡ 启用沙箱 — 渲染进程与操作系统隔离
    sandbox: true,

    // ⚡ 禁用 Node.js 集成 — 渲染进程无法 require('fs')
    nodeIntegration: false,

    // ⚡ 上下文隔离 — 防止原型链污染和原型劫持
    contextIsolation: true,

    // ⚡ 启用 Web 安全 — 强制执行同源策略和 CSP
    webSecurity: true,

    // ⚡ 禁止加载非本地资源
    allowRunningInsecureContent: false,

    // ⚡ 禁止自动打开 DevTools (生产环境)
    devTools: process.env.NODE_ENV === 'development',

    // preload 是唯一的安全桥接通道
    preload: path.join(__dirname, '../preload/index.js'),

    // ⚡ 禁用实验性特性
    experimentalFeatures: false,

    // ⚡ 禁用 Node.js 模块加载 (额外保险)
    nodeIntegrationInSubFrames: false,
    nodeIntegrationInWorker: false,
  },
});
```

**关键说明**：

| 配置项 | 安全影响 | 性能影响 | 默认值 | 我们的设置 |
|--------|----------|----------|--------|------------|
| `sandbox` | 阻止 RCE（远程代码执行） | 零 | false | **true** |
| `contextIsolation` | 防止原型链攻击 | 零 | true | **true** |
| `nodeIntegration` | 阻止直接访问 Node API | 零 | false | **false** |
| `webSecurity` | 强制执行 CSP 和同源策略 | 零 | true | **true** |
| `allowRunningInsecureContent` | 阻止混合内容攻击 | 零 | false | **false** |
| `experimentalFeatures` | 减少攻击面 | 零 | false | **false** |
| `devTools` | 生产环境禁止打开调试 | 零 | true (可开) | **production=false** |

### 6.2 第二层：IPC 通信安全（⚡ 零性能开销）

#### 6.2.1 最小 API 表面

preload 只暴露**最小必要**的 API，每个方法都经过白名单校验：

```typescript
// packages/shell/src/main/preload/index.ts

import { contextBridge, ipcRenderer, IpcRendererEvent } from 'electron';

contextBridge.exposeInMainWorld('electronAPI', {
  // Agent 生命周期 (仅允许这两个值)
  startAgent: () => ipcRenderer.invoke('agent:start'),
  switchAgent: (target: 'openclaw' | 'hermes') =>
    // ⚡ 白名单校验在 main 进程侧执行，这里不传任意字符串
    ipcRenderer.invoke('agent:switch', target),

  // 聊天 (限制单次消息长度)
  sendChat: (message: string) => ipcRenderer.invoke('agent:chat', message),

  // 模型 (只允许传入预定义的 modelId)
  switchModel: (modelId: string) => ipcRenderer.invoke('agent:switchModel', modelId),
  getModels: () => ipcRenderer.invoke('agent:getModels'),

  // 配置 (不暴露任意文件读写)
  getConfig: () => ipcRenderer.invoke('config:get'),
  setConfig: (patch: object) => ipcRenderer.invoke('config:set', patch),

  // Hermes
  installHermes: () => ipcRenderer.invoke('hermes:install'),
  getHermesStatus: () => ipcRenderer.invoke('hermes:status'),

  // 事件监听 (返回清理函数)
  onAgentStream: (cb: (data: any) => void) => {
    const listener = (_e: IpcRendererEvent, d: any) => cb(d);
    ipcRenderer.on('agent:stream', listener);
    return () => ipcRenderer.removeListener('agent:stream', listener);
  },
});
```

#### 6.2.2 Main 进程 IPC 参数校验

所有 IPC handler 接收的参数必须经过白名单/Schema 校验，防止注入攻击：

```typescript
// packages/shell/src/main/ipc/chat.ts

import { ipcMain } from 'electron';

// 消息长度限制 (防止 DoS 和资源耗尽)
const MAX_MESSAGE_LENGTH = 50000; // 50KB

ipcMain.handle('agent:chat', async (event, message: string) => {
  // ⚡ 输入校验
  if (typeof message !== 'string') {
    throw new Error('Invalid message type');
  }
  if (message.length > MAX_MESSAGE_LENGTH) {
    throw new Error('Message exceeds maximum length');
  }
  if (message.trim().length === 0) {
    throw new Error('Empty message');
  }

  // 使用 event.sender (安全的 WebContents 引用) 而不是全局引用
  const webContents = event.sender;
  if (!webContents.isDestroyed()) {
    // 业务逻辑...
  }
});

ipcMain.handle('agent:switch', async (event, target: string) => {
  // ⚡ 白名单校验 — 只允许预定义的值
  const validAgents = ['openclaw', 'hermes'] as const;
  if (!validAgents.includes(target as typeof validAgents[number])) {
    throw new Error(`Invalid agent: ${target}`);
  }
  // ... 切换逻辑
});
```

#### 6.2.3 禁止动态 IPC 调用

```typescript
// ❌ 危险 — 允许任意 IPC channel 调用
ipcMain.on('dynamic-invoke', (event, channel, args) => {
  // 攻击者可以构造: ipcRenderer.send('dynamic-invoke', 'fs:readFile', '/etc/passwd')
});

// ✅ 安全 — 每个 channel 独立注册，固定参数类型
ipcMain.handle('agent:chat', validateAndHandleChat);
ipcMain.handle('agent:switch', validateAndHandleSwitch);
```

### 6.3 第三层：内容与资源保护

#### 6.3.1 内容安全策略 CSP（⚡ 零运行开销）

通过 CSP 限制资源加载来源，防止 XSS 和恶意脚本注入：

```html
<!-- packages/web/index.html -->
<head>
  <meta http-equiv="Content-Security-Policy"
        content="default-src 'self';
                 script-src 'self';
                 style-src 'self' 'unsafe-inline';
                 img-src 'self' data: https:;
                 connect-src 'self';
                 font-src 'self';
                 media-src 'self';
                 frame-src 'none';
                 object-src 'none';
                 worker-src 'self' blob:;">
</head>
```

| 指令 | 值 | 说明 |
|------|-----|------|
| `default-src` | `'self'` | 默认只允许同源资源 |
| `script-src` | `'self'` | 禁止内联脚本（除非有 nonce） |
| `style-src` | `'self' 'unsafe-inline'` | 允许内联样式（Vue 需要） |
| `frame-src` | `'none'` | 禁止 iframe 嵌入 |
| `object-src` | `'none'` | 禁止 Flash/Java 等插件 |
| `connect-src` | `'self'` | 限制 XHR/fetch/WebSocket 目标 |

#### 6.3.2 ASAR 归档 + 加密（⚡ 构建时处理，⏱️ 启动时微小开销）

见 Section 8.10 ASAR 打包与加密。三重保护：
1. **ASAR 归档** — 源码打包为 `.asar` 二进制文件
2. **ASAR 完整性校验** — SHA-256 哈希校验防篡改
3. **AES-256 加密** — 运行时解密加载

#### 6.3.3 代码混淆（⏱️ 构建时开销，按需启用）

对 ASAR 加密仍有顾虑时，可叠加代码混淆：

```json
// packages/shell/package.json
{
  "scripts": {
    "obfuscate": "javascript-obfuscator dist/main --output dist/main-obfuscated --compact true --control-flow-flattening true --dead-code-injection true"
  },
  "devDependencies": {
    "javascript-obfuscator": "^4.1.0"
  }
}
```

> **注意**: 混淆会增加构建时间 2-5 倍，且可能影响调试。建议只在生产发布时启用。

### 6.4 第四层：运行时加固（🔥 微小开销）

#### 6.4.1 安全导航控制

禁止页面跳转到外部 URL，防止钓鱼和恶意重定向：

```typescript
// packages/shell/src/main/index.ts

// 禁止导航到非预期的 URL
mainWindow.webContents.on('will-navigate', (event, url) => {
  const allowedHosts = ['localhost', '127.0.0.1'];
  try {
    const parsed = new URL(url);
    if (!allowedHosts.includes(parsed.hostname)) {
      event.preventDefault();
    }
  } catch {
    event.preventDefault();
  }
});

// 禁止创建新窗口
mainWindow.webContents.setWindowOpenHandler(() => ({ action: 'deny' }));

// 禁止超链接打开外部 URL
mainWindow.webContents.on('new-window', (event) => {
  event.preventDefault();
});
```

#### 6.4.2 证书透明度验证

对于 HTTPS 请求，验证证书透明度：

```typescript
mainWindow.webContents.on('certificate-transparency', (event, url) => {
  // 记录证书透明度事件，用于安全审计
  console.log(`[CT] Certificate transparency for: ${url}`);
});
```

### 6.5 安全防御总结

| 层级 | 措施 | 开销 | 状态 |
|------|------|------|------|
| **第一层** | sandbox + contextIsolation + nodeIntegration=false | ⚡ 零 | **必须启用** |
| **第一层** | webSecurity=true + 禁用实验特性 | ⚡ 零 | **必须启用** |
| **第一层** | 生产环境禁用 DevTools | ⚡ 零 | **必须启用** |
| **第二层** | contextBridge 最小 API 表面 | ⚡ 零 | **必须启用** |
| **第二层** | IPC 参数白名单/Schema 校验 | ⚡ 零 | **必须启用** |
| **第二层** | 禁止动态 IPC 调用 | ⚡ 零 | **必须启用** |
| **第三层** | CSP 策略 | ⚡ 零 | **必须启用** |
| **第三层** | ASAR + AES-256 加密 | ⏱️ 启动时 | **推荐启用** |
| **第三层** | 代码混淆 | ⏱️ 构建时 | 按需 |
| **第四层** | 安全导航控制 | 🔥 微小 | **推荐启用** |
| **第四层** | 证书透明度审计 | 🔥 微小 | 按需 |

---

## 七、项目目录结构

采用 **pnpm workspaces monorepo** 架构，前端（Vue 应用）和 Electron 壳子（主进程 + preload）作为独立 package 分别构建，最终由 electron-builder 统一打包。参考 oneclaw/oneclaw 的项目组织方式。

```
qclaw-next/
│
├── pnpm-workspace.yaml             # pnpm workspaces 声明
├── package.json                    # Root package (workspace 根)
├── tsconfig.base.json              # 共享 TypeScript 基础配置
├── electron-builder.yml            # electron-builder 打包配置
│
├── packages/
│   ├── shell/                      # Electron 壳子 package
│   │   ├── package.json            # shell 独立依赖 (electron, electron-builder)
│   │   ├── tsconfig.json           # 继承 tsconfig.base.json
│   │   ├── vite.main.config.ts     # 主进程 Vite 构建配置
│   │   ├── vite.preload.config.ts  # Preload Vite 构建配置
│   │   │
│   │   ├── src/
│   │   │   ├── main/               # Electron 主进程
│   │   │   │   ├── index.ts        # 入口: 创建窗口、注册 IPC、启动守护
│   │   │   │   ├── preload.ts      # 预加载脚本: contextBridge
│   │   │   │   │
│   │   │   │   ├── core/           # 核心业务层
│   │   │   │   │   ├── guardian.ts         # 网关守护 (健康检查 + 自动重启)
│   │   │   │   │   ├── plugin-manager.ts   # 插件加载与管理
│   │   │   │   │   ├── model-switcher.ts   # 模型切换
│   │   │   │   │   ├── channel-manager.ts  # 多 Channel 管理
│   │   │   │   │   ├── telemetry.ts        # 埋点/遥测系统
│   │   │   │   │   └── hermes-manager.ts   # Hermes Agent 子进程管理
│   │   │   │   │
│   │   │   │   ├── adapters/       # 适配层 (不动 OpenClaw 源码的关键)
│   │   │   │   │   ├── openclaw-adapter.ts # OpenClaw 适配层
│   │   │   │   │   └── hermes-adapter.ts   # Hermes 适配层
│   │   │   │   │
│   │   │   │   ├── ipc/            # IPC 处理器 (按功能域拆分)
│   │   │   │   │   ├── chat.ts     # 聊天: send, stream, history
│   │   │   │   │   ├── config.ts   # 配置: get, set, reset
│   │   │   │   │   └── agent.ts    # Agent: switch, status, install
│   │   │   │   │
│   │   │   │   └── utils/
│   │   │   │       ├── paths.ts          # 跨平台路径处理
│   │   │   │       ├── process.ts        # 进程管理工具
│   │   │   │       ├── logger.ts         # 日志
│   │   │   │       └── asar-decrypt.ts   # ASAR 解密加载器
│   │   │   │
│   │   │   └── preload/            # Preload 脚本 (独立目录)
│   │   │       └── index.ts        # contextBridge 暴露安全 API
│   │   │
│   │   └── dist/                   # 主进程构建产物 (vite build 输出)
│   │       ├── main/index.js
│   │       └── preload/index.js
│   │
│   ├── web/                        # 前端 Vue package (独立构建)
│   │   ├── package.json            # web 独立依赖 (vue, vue-router, pinia, UI库)
│   │   ├── tsconfig.json           # 继承 tsconfig.base.json
│   │   ├── vite.config.ts          # 前端 Vite 构建配置
│   │   ├── index.html
│   │   │
│   │   ├── src/
│   │   │   ├── main.ts             # Vue 应用入口
│   │   │   ├── App.vue             # 根组件
│   │   │   │
│   │   │   ├── pages/              # 页面组件
│   │   │   │   ├── ChatPage.vue        # 聊天页面
│   │   │   │   ├── ConfigPage.vue      # 模型配置页面
│   │   │   │   ├── AgentSwitchPage.vue # Agent 切换页面
│   │   │   │   └── TelemetryPage.vue   # 埋点面板
│   │   │   │
│   │   │   ├── hooks/              # Composition API hooks
│   │   │   │   └── useAgent.ts     # Agent 通信 Hook
│   │   │   │
│   │   │   ├── components/         # 共享组件
│   │   │   │   ├── ChatInput.vue
│   │   │   │   ├── MessageList.vue
│   │   │   │   ├── AgentSelector.vue
│   │   │   │   └── ModelDropdown.vue
│   │   │   │
│   │   │   ├── stores/             # Pinia 状态管理
│   │   │   │   ├── agent.ts        # Agent 状态
│   │   │   │   ├── config.ts       # 配置状态
│   │   │   │   └── telemetry.ts    # 埋点状态
│   │   │   │
│   │   │   ├── router/             # Vue Router
│   │   │   │   └── index.ts
│   │   │   │
│   │   │   ├── types/              # 类型定义
│   │   │   │   └── electron.d.ts   # window.electronAPI 类型声明
│   │   │   │
│   │   │   └── styles/             # 全局样式
│   │   │       └── main.css
│   │   │
│   │   └── dist/                   # 前端构建产物 (vite build 输出)
│   │       ├── index.html
│   │       ├── assets/
│   │       └── ...
│   │
│   └── shared/                     # 共享类型和工具 (可选)
│       ├── package.json
│       ├── src/
│       │   ├── types/              # 前后端共享类型
│       │   │   ├── agent.ts        # Agent 类型定义
│       │   │   ├── chat.ts         # 聊天消息类型
│       │   │   └── config.ts       # 配置类型
│       │   └── constants.ts        # 共享常量
│       └── dist/                   # 构建产物
│
├── plugins/                        # OpenClaw 自定义插件 (通过插件系统注入)
│   ├── wechat-channel/             # 微信 Channel 插件
│   ├── model-xxx/                  # 自定义模型插件
│   └── telemetry-custom/           # 自定义埋点插件
│
├── scripts/
│   ├── bundle-openclaw.cjs         # 打包前准备 OpenClaw
│   ├── bundle-hermes.cjs           # 打包前准备 Hermes
│   └── post-build.cjs              # 构建后处理 (注入 OpenClaw 运行时)
│
├── resources/
│   ├── hermes/                     # Hermes Python 环境 (Mac/Ubuntu 预装)
│   │   ├── requirements.txt
│   │   └── install.sh
│   └── icons/                      # 应用图标
│       ├── icon.ico
│       ├── icon.icns
│       └── icon.png
│
└── release/                        # electron-builder 输出目录
```

### 6.1 构建管线说明

```
pnpm build
  │
  ├── pnpm --filter @app/web build        # 1. 先构建前端 Vue → packages/web/dist/
  │
  ├── pnpm --filter @app/shell build      # 2. 构建 Electron 主进程 → packages/shell/dist/
  │   ├── vite build --config vite.main.config.ts
  │   └── vite build --config vite.preload.config.ts
  │
  └── electron-builder                    # 3. 打包为 DMG/NSIS/AppImage
      │
      ├── 读取 packages/shell/dist/main/  → 主进程代码
      ├── 读取 packages/shell/dist/preload/ → preload 脚本
      └── 读取 packages/web/dist/          → 前端静态资源
```

### 6.2 关键设计决策

| 决策 | 说明 |
|------|------|
| **pnpm workspaces** | 原生 monorepo 支持，比 npm/yarn workspaces 更严格的依赖边界 |
| **前端独立构建** | Vue 应用可以独立开发、独立测试、独立部署 (`pnpm --filter @app/web dev`) |
| **壳子独立构建** | 主进程和 preload 单独用 Vite 构建为 Node.js 目标 |
| **shared 包** | 前后端共享类型（聊天消息格式、配置类型），避免重复定义 |
| **Vite 而非 tsc** | 主进程也用 Vite 构建，支持 ESM/CJS 兼容、更快的构建速度 |

---

## 八、核心实现方案

### 7.1 OpenClaw 适配层（不修改源码）

**原则**: 所有定制逻辑通过 OpenClaw 的插件系统（Hook、Channel、Event）注入，不修改 OpenClaw 源码。

```typescript
// packages/shell/src/main/adapters/openclaw-adapter.ts

import { Gateway, GatewayConfig } from 'openclaw';
import { EventEmitter } from 'events';
import path from 'path';
import { app } from 'electron';

export class OpenClawAdapter extends EventEmitter {
  private gateway: Gateway | null = null;

  async start(config: Partial<GatewayConfig> = {}): Promise<void> {
    const userConfig = this.loadUserConfig();
    const mergedConfig: GatewayConfig = {
      ...userConfig,
      ...config,
      hooks: this.buildCustomHooks(),
      channels: this.buildCustomChannels(),
    };

    this.gateway = new Gateway(mergedConfig);
    this.registerTelemetry();
    await this.gateway.start();
    this.emit('started');
  }

  async sendMessage(message: string, options?: {
    model?: string;
    channel?: string;
    stream?: boolean;
  }): Promise<any> {
    if (!this.gateway) throw new Error('Gateway not started');
    if (options?.model) this.gateway.setModel(options.model);
    return this.gateway.processMessage(message, {
      channel: options?.channel,
      stream: options?.stream,
    });
  }

  switchModel(modelId: string): void {
    this.gateway?.setModel(modelId);
    this.emit('model:switched', { modelId });
  }

  getConfig(): GatewayConfig {
    return this.gateway?.getConfig() ?? {};
  }

  async updateConfig(patch: Partial<GatewayConfig>): Promise<void> {
    await this.gateway?.updateConfig(patch);
  }

  async stop(): Promise<void> {
    await this.gateway?.stop();
    this.emit('stopped');
  }

  // ===== 以下是不动源码的注入方式 =====

  private buildCustomHooks() {
    return {
      'beforeProcess': async (msg: any) => {
        this.emit('telemetry:message_received', msg);
      },
      'beforeModelSelect': async (candidates: any[]) => {
        this.emit('telemetry:model_select', { candidates });
      },
      'afterProcess': async (result: any) => {
        this.emit('telemetry:message_completed', result);
      },
      'onError': async (error: Error) => {
        this.emit('telemetry:error', { error: error.message });
      },
    };
  }

  private buildCustomChannels() {
    return [
      require(path.join(app.getAppPath(), 'plugins', 'wechat-channel')),
    ];
  }

  private registerTelemetry() {
    if (!this.gateway) return;
    this.gateway.on('message', (msg: any) => {
      this.emit('telemetry:message', msg);
    });
    this.gateway.on('error', (err: Error) => {
      this.emit('telemetry:error', { error: err.message });
    });
    this.gateway.on('model:change', (model: string) => {
      this.emit('telemetry:model_change', { model });
    });
  }

  private loadUserConfig(): GatewayConfig {
    const configPath = path.join(app.getPath('userData'), 'config.json');
    try { return require(configPath); } catch { return {} as GatewayConfig; }
  }
}
```

### 7.2 网关守护进程

```typescript
// packages/shell/src/main/core/guardian.ts

import { OpenClawAdapter } from '../adapters/openclaw-adapter';
import { EventEmitter } from 'events';

export class Guardian extends EventEmitter {
  private adapter: OpenClawAdapter;
  private healthCheckInterval: NodeJS.Timeout | null = null;
  private restartCount = 0;
  private maxRestarts = 5;
  private isRunning = false;

  constructor(adapter: OpenClawAdapter) {
    super();
    this.adapter = adapter;
  }

  async start(): Promise<void> {
    this.isRunning = true;
    this.restartCount = 0;

    try {
      await this.adapter.start();
      this.emit('guardian:started');
    } catch (err) {
      this.emit('guardian:start_failed', err);
      return;
    }

    this.startHealthCheck();

    this.adapter.on('stopped', () => {
      if (this.isRunning) {
        this.emit('guardian:unexpected_stop');
        this.handleCrash();
      }
    });
  }

  private startHealthCheck(): void {
    this.healthCheckInterval = setInterval(async () => {
      try {
        const config = this.adapter.getConfig();
        if (!config) throw new Error('Gateway not responsive');
      } catch {
        this.emit('guardian:health_check_failed');
        this.handleCrash();
      }
    }, 30000);
  }

  private async handleCrash(): Promise<void> {
    if (!this.isRunning || this.restartCount >= this.maxRestarts) {
      this.emit('guardian:max_restarts_reached');
      this.isRunning = false;
      return;
    }

    this.restartCount++;
    const delay = 1000 * Math.pow(2, this.restartCount - 1);
    this.emit('guardian:restarting', { attempt: this.restartCount, delay });

    await new Promise(r => setTimeout(r, delay));

    try {
      await this.adapter.start();
      this.emit('guardian:restarted', { attempt: this.restartCount });
    } catch (err) {
      this.emit('guardian:restart_failed', err);
      this.handleCrash();
    }
  }

  async stop(): Promise<void> {
    this.isRunning = false;
    if (this.healthCheckInterval) clearInterval(this.healthCheckInterval);
    await this.adapter.stop();
    this.emit('guardian:stopped');
  }
}
```

### 7.3 IPC 通信层

```typescript
// packages/shell/src/main/index.ts — 主进程入口

import { app, BrowserWindow, ipcMain } from 'electron';
import path from 'path';
import { loadAsarDecrypt } from './utils/asar-decrypt';
import { OpenClawAdapter } from './adapters/openclaw-adapter';
import { Guardian } from './core/guardian';
import { HermesManager } from './core/hermes-manager';
import { ModelSwitcher } from './core/model-switcher';

// ASAR 加密启用时，启动时自动解密 ASAR 到内存
if (process.env.ASAR_ENCRYPTION_KEY) {
  loadAsarDecrypt();
}

let mainWindow: BrowserWindow;
const openclawAdapter = new OpenClawAdapter();
const guardian = new Guardian(openclawAdapter);
const hermesManager = new HermesManager();
const modelSwitcher = new ModelSwitcher(openclawAdapter);
let currentAgent: 'openclaw' | 'hermes' | null = null;

app.whenReady().then(async () => {
  mainWindow = new BrowserWindow({
    width: 1200, height: 800,
    webPreferences: {
      preload: path.join(__dirname, 'preload.js'),
      contextIsolation: true,
      nodeIntegration: false,
      sandbox: true,
    },
  });

  if (process.env.NODE_ENV === 'development') {
    mainWindow.loadURL('http://localhost:5173');
  } else {
    mainWindow.loadFile(path.join(__dirname, '../renderer/dist/index.html'));
  }

  // === IPC 注册 ===

  // 启动 OpenClaw (应用启动时默认)
  ipcMain.handle('agent:start', async () => {
    await guardian.start();
    currentAgent = 'openclaw';
    return { agent: currentAgent };
  });

  // 聊天消息
  ipcMain.handle('agent:chat', async (_event, message: string) => {
    if (currentAgent === 'openclaw') {
      return openclawAdapter.sendMessage(message, { stream: true });
    }
    return hermesManager.sendMessage('chat.send', { message });
  });

  // 模型切换
  ipcMain.handle('agent:switchModel', (_event, modelId: string) => {
    modelSwitcher.switch(modelId);
    return { model: modelId };
  });

  // 获取可用模型
  ipcMain.handle('agent:getModels', () => {
    return modelSwitcher.getAvailableModels();
  });

  // Agent 切换
  ipcMain.handle('agent:switch', async (_event, target: string) => {
    await guardian.stop();
    if (target === 'hermes') {
      await hermesManager.start();
      currentAgent = 'hermes';
    } else {
      await guardian.start();
      currentAgent = 'openclaw';
    }
    return { agent: currentAgent };
  });

  // Hermes 安装 (Windows WSL2)
  ipcMain.handle('hermes:install', async () => {
    return hermesManager.installOnWSL();
  });

  // Hermes 状态检查
  ipcMain.handle('hermes:status', async () => {
    return { installed: await hermesManager.isInstalled() };
  });

  // 配置读写
  ipcMain.handle('config:get', () => openclawAdapter.getConfig());
  ipcMain.handle('config:set', async (_event, patch: any) => {
    await openclawAdapter.updateConfig(patch);
  });
});

// 应用退出时清理
app.on('before-quit', async (event) => {
  event.preventDefault();
  await guardian.stop();
  await hermesManager.stop();
  app.quit();
});
```

```typescript
// packages/shell/src/main/preload/index.ts

import { contextBridge, ipcRenderer } from 'electron';

contextBridge.exposeInMainWorld('electronAPI', {
  // Agent 生命周期
  startAgent: () => ipcRenderer.invoke('agent:start'),
  switchAgent: (target: string) => ipcRenderer.invoke('agent:switch', target),

  // 聊天
  sendChat: (message: string) => ipcRenderer.invoke('agent:chat', message),

  // 模型
  switchModel: (modelId: string) => ipcRenderer.invoke('agent:switchModel', modelId),
  getModels: () => ipcRenderer.invoke('agent:getModels'),

  // 配置
  getConfig: () => ipcRenderer.invoke('config:get'),
  setConfig: (patch: any) => ipcRenderer.invoke('config:set', patch),

  // Hermes
  installHermes: () => ipcRenderer.invoke('hermes:install'),
  getHermesStatus: () => ipcRenderer.invoke('hermes:status'),

  // 流式事件 (main → renderer)
  onAgentStream: (cb: (data: any) => void) => {
    const listener = (_e: any, d: any) => cb(d);
    ipcRenderer.on('agent:stream', listener);
    return () => ipcRenderer.removeListener('agent:stream', listener);
  },

  // 守护进程事件
  onGuardianEvent: (cb: (event: string, data: any) => void) => {
    const listener = (_e: any, d: any) => cb(d.event, d.data);
    ipcRenderer.on('guardian:event', listener);
    return () => ipcRenderer.removeListener('guardian:event', listener);
  },
});
```

### 7.4 Hermes Agent 管理

```typescript
// packages/shell/src/main/core/hermes-manager.ts

import { spawn, ChildProcess } from 'child_process';
import path from 'path';

export class HermesManager {
  private child: ChildProcess | null = null;
  private buffer = '';
  private messageId = 0;
  private pendingCallbacks = new Map<number, (result: any) => void>();

  async isInstalled(): Promise<boolean> {
    if (process.platform === 'win32') return this.checkWSLHermes();
    try {
      await this.execCommand('python3', ['-m', 'hermes', '--version']);
      return true;
    } catch { return false; }
  }

  async installOnWSL(): Promise<boolean> {
    const { stdout } = await this.execCommand('wsl', ['--list', '--verbose']);
    if (!stdout.includes('Ubuntu')) {
      throw new Error('WSL2 with Ubuntu is required');
    }
    await this.execCommand('wsl', ['bash', '-c',
      'python3 -m venv ~/.hermes-venv && ' +
      '~/.hermes-venv/bin/pip install --upgrade pip && ' +
      '~/.hermes-venv/bin/pip install hermes-agent'
    ]);
    return true;
  }

  async start(): Promise<void> {
    if (this.child) await this.stop();

    const [command, args] = process.platform === 'win32'
      ? ['wsl', ['bash', '-c', '~/.hermes-venv/bin/python3 -m hermes acp']]
      : ['python3', ['-m', 'hermes', 'acp']];

    this.child = spawn(command, args, {
      stdio: ['pipe', 'pipe', 'pipe'],
      windowsHide: true,
    });

    this.child.stdout.on('data', (chunk: Buffer) => {
      this.buffer += chunk.toString();
      this.processBuffer();
    });

    this.child.stderr.on('data', (chunk: Buffer) => {
      console.error('[Hermes stderr]:', chunk.toString());
    });
  }

  async sendMessage(method: string, params: any): Promise<any> {
    return new Promise((resolve, reject) => {
      if (!this.child?.stdin) return reject(new Error('Hermes not running'));
      const id = ++this.messageId;
      this.pendingCallbacks.set(id, resolve);
      const req = JSON.stringify({ jsonrpc: '2.0', id, method, params }) + '\n';
      this.child.stdin.write(req, 'utf8', (err) => {
        if (err) { this.pendingCallbacks.delete(id); reject(err); }
      });
    });
  }

  async stop(): Promise<void> {
    if (this.child) {
      this.child.stdin?.end();
      this.child.kill('SIGTERM');
      await new Promise<void>(r => {
        this.child!.on('exit', () => r());
        setTimeout(() => r(), 5000);
      });
      this.child = null;
    }
  }

  private processBuffer(): void {
    const lines = this.buffer.split('\n');
    this.buffer = lines.pop() || '';
    for (const line of lines) {
      if (!line.trim()) continue;
      try {
        const msg = JSON.parse(line);
        if (msg.id !== undefined && this.pendingCallbacks.has(msg.id)) {
          this.pendingCallbacks.get(msg.id)!(msg);
          this.pendingCallbacks.delete(msg.id);
        }
      } catch { /* 非 JSON 输出忽略 */ }
    }
  }

  private async checkWSLHermes(): Promise<boolean> {
    try {
      await this.execCommand('wsl', ['bash', '-c', '~/.hermes-venv/bin/python3 -m hermes --version']);
      return true;
    } catch { return false; }
  }

  private execCommand(cmd: string, args: string[]): Promise<{ stdout: string }> {
    return new Promise((resolve, reject) => {
      const child = spawn(cmd, args, { windowsHide: true });
      let stdout = '', stderr = '';
      child.stdout.on('data', (d) => stdout += d.toString());
      child.stderr.on('data', (d) => stderr += d.toString());
      child.on('close', (code) => {
        if (code === 0) resolve({ stdout });
        else reject(new Error(stderr));
      });
    });
  }
}
```

### 7.5 ASAR 解密加载器

```typescript
// packages/shell/src/main/utils/asar-decrypt.ts

import { app } from 'electron';
import { existsSync, readFileSync, writeFileSync, renameSync } from 'fs';
import { join, dirname } from 'path';
import { createDecipheriv } from 'crypto';

/**
 * 启动时解密被加密的 ASAR 文件
 * 仅在 ASAR_ENCRYPTION_KEY 环境变量存在时调用
 */
export function loadAsarDecrypt(): void {
  const encryptionKey = process.env.ASAR_ENCRYPTION_KEY;
  if (!encryptionKey) return;

  try {
    const key = Buffer.from(encryptionKey, 'hex');

    // 定位 ASAR 文件 (不同平台路径不同)
    const appPath = app.getAppPath();
    let asarPath: string;

    if (appPath.endsWith('.asar')) {
      asarPath = appPath;
    } else {
      // 开发模式: 未打包，跳过解密
      return;
    }

    const encryptedData = readFileSync(asarPath);
    if (encryptedData.length < 17) return; // IV + 至少 1 字节密文

    // 提取 IV (前 16 字节)
    const iv = encryptedData.slice(0, 16);
    const encrypted = encryptedData.slice(16);

    // 解密
    const decipher = createDecipheriv('aes-256-cbc', key, iv);
    const decrypted = Buffer.concat([decipher.update(encrypted), decipher.final()]);

    // 写回原始文件 (ASAR 被解密后，Electron 可正常加载)
    const tmpPath = asarPath + '.decrypted';
    writeFileSync(tmpPath, decrypted);
    renameSync(tmpPath, asarPath);
  } catch (err) {
    console.error('[ASAR Decrypt] Failed to decrypt ASAR:', err);
    // 解密失败时不阻塞启动，让 Electron 尝试加载原始 ASAR
  }
}
```

### 7.6 前端 Hook

```typescript
// packages/web/src/hooks/useAgent.ts

import { ref, onMounted, onUnmounted } from 'vue';

export function useAgent() {
  const currentAgent = ref<string | null>(null);
  const messages = ref<any[]>([]);
  const streamingContent = ref('');
  const models = ref<string[]>([]);
  let streamCleanup: (() => void) | null = null;

  // 监听流式响应
  onMounted(() => {
    streamCleanup = window.electronAPI.onAgentStream((data) => {
      if (data.done) {
        const idx = messages.value.findIndex(m => m.isStreaming);
        if (idx !== -1) {
          messages.value[idx].content = streamingContent.value;
          messages.value[idx].isStreaming = false;
        }
        streamingContent.value = '';
      } else {
        streamingContent.value += (data.token || '');
        const idx = messages.value.findIndex(m => m.isStreaming);
        if (idx !== -1) {
          messages.value[idx].content = streamingContent.value;
        }
      }
    });

    window.electronAPI.getModels().then((m) => { models.value = m; });
  });

  onUnmounted(() => { streamCleanup?.(); });

  const sendMessage = async (text: string) => {
    messages.value.push({ role: 'user', content: text });
    messages.value.push({ role: 'assistant', content: '', isStreaming: true });
    streamingContent.value = '';
    await window.electronAPI.sendChat(text);
  };

  const switchAgent = async (target: 'openclaw' | 'hermes') => {
    if (target === 'hermes') {
      const status = await window.electronAPI.getHermesStatus();
      if (!status.installed) {
        await window.electronAPI.installHermes();
      }
    }
    const result = await window.electronAPI.switchAgent(target);
    currentAgent.value = result.agent;
  };

  const switchModel = async (modelId: string) => {
    await window.electronAPI.switchModel(modelId);
  };

  return {
    currentAgent, messages, streamingContent, models,
    sendMessage, switchAgent, switchModel,
  };
}
```

---

## 九、打包与分发

### 9.1 electron-builder.yml (根目录)

```yaml
appId: com.yourcompany.product
productName: Your Product

directories:
  output: release
  buildResources: resources

files:
  - from: packages/shell/dist/main
    to: dist/main
    filter:
      - "**/*"
  - from: packages/shell/dist/preload
    to: dist/preload
    filter:
      - "**/*"
  - from: packages/web/dist
    to: dist/web
    filter:
      - "**/*"

extraResources:
  - from: resources/hermes
    to: hermes
    filter:
      - "**/*"
      - "!**/__pycache__"
  - from: plugins
    to: plugins
    filter:
      - "**/*"

asar: true
asarIntegrity: true
asarUnpack:
  - "plugins/**/*"
  - "resources/hermes/**/*"

win:
  target:
    - target: nsis
      arch:
        - x64
        - arm64
  icon: resources/icons/icon.ico
  asarIntegrity: true

nsis:
  oneClick: false
  allowToChangeInstallationDirectory: true
  perMachine: true

mac:
  target:
    - target: dmg
      arch:
        - x64
        - arm64
  category: public.app-category.developer-tools
  hardenedRuntime: true
  gatekeeperAssess: false

linux:
  target:
    - target: deb
      arch:
        - x64
        - arm64
    - target: AppImage
      arch:
        - x64
  category: Development
  icon: resources/icons/icon.png
  maintainer: "Your Team <your@email.com>"
  desktop:
    Name: Your Product
    Comment: Local AI Agent Desktop
    Categories: Development;Utility;

afterPack: scripts/post-build.cjs
```

### 9.2 根 package.json

```json
{
  "name": "qclaw-next",
  "private": true,
  "version": "1.0.0",
  "scripts": {
    "dev:web": "pnpm --filter @app/web dev",
    "dev:shell": "pnpm --filter @app/shell dev",
    "dev": "concurrently \"pnpm dev:web\" \"pnpm dev:shell\"",
    "build:web": "pnpm --filter @app/web build",
    "build:shell": "pnpm --filter @app/shell build",
    "build": "pnpm build:web && pnpm build:shell",
    "prebundle": "node scripts/bundle-openclaw.cjs && node scripts/bundle-hermes.cjs",
    "bundle": "npm run prebundle && npm run build && electron-builder --config electron-builder.yml",
    "postinstall": "electron-builder install-app-deps"
  },
  "devDependencies": {
    "concurrently": "^9.0.0",
    "electron": "^42.0.0",
    "electron-builder": "^26.0.0"
  }
}
```

### 9.3 pnpm-workspace.yaml

```yaml
packages:
  - 'packages/*'
```

### 9.4 packages/shell/package.json

```json
{
  "name": "@app/shell",
  "version": "1.0.0",
  "private": true,
  "main": "dist/main/index.js",
  "scripts": {
    "dev": "concurrently \"vite build --config vite.main.config.ts --watch\" \"vite build --config vite.preload.config.ts --watch\"",
    "build": "vite build --config vite.main.config.ts && vite build --config vite.preload.config.ts"
  },
  "dependencies": {
    "openclaw": "2026.5.12"
  },
  "devDependencies": {
    "electron": "^42.0.0",
    "typescript": "^5.7.0",
    "vite": "^6.0.0",
    "@app/shared": "workspace:*"
  }
}
```

### 9.5 packages/web/package.json

```json
{
  "name": "@app/web",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "dev": "vite",
    "build": "vue-tsc --noEmit && vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "vue": "^3.5.0",
    "vue-router": "^4.5.0",
    "pinia": "^2.3.0"
  },
  "devDependencies": {
    "@vitejs/plugin-vue": "^5.0.0",
    "typescript": "^5.7.0",
    "vite": "^6.0.0",
    "vue-tsc": "^2.2.0",
    "@app/shared": "workspace:*"
  }
}
```

### 9.6 shell 主进程 Vite 配置

```typescript
// packages/shell/vite.main.config.ts
import { defineConfig } from 'vite';
import { resolve } from 'path';

export default defineConfig({
  build: {
    outDir: 'dist/main',
    lib: {
      entry: resolve(__dirname, 'src/main/index.ts'),
      formats: ['cjs'],
      fileName: () => 'index.js',
    },
    rollupOptions: {
      external: ['electron', 'openclaw'],
    },
    target: 'node22',
    sourcemap: true,
  },
});
```

```typescript
// packages/shell/vite.preload.config.ts
import { defineConfig } from 'vite';
import { resolve } from 'path';

export default defineConfig({
  build: {
    outDir: 'dist/preload',
    lib: {
      entry: resolve(__dirname, 'src/main/preload/index.ts'),
      formats: ['cjs'],
      fileName: () => 'index.js',
    },
    external: ['electron'],
    target: 'node22',
    sourcemap: true,
  },
});
```

### 9.7 web 前端 Vite 配置

```typescript
// packages/web/vite.config.ts
import { defineConfig } from 'vite';
import vue from '@vitejs/plugin-vue';
import { resolve } from 'path';

export default defineConfig({
  plugins: [vue()],
  build: {
    outDir: 'dist',
    rollupOptions: {
      input: resolve(__dirname, 'index.html'),
    },
  },
  base: './',
});
```

### 9.8 开发模式联动

开发时需要同时启动 Web 和 Shell 的热更新：

```bash
# 方式一：concurrently (推荐)
pnpm dev

# 方式二：手动分终端
终端1: pnpm --filter @app/web dev        # 启动 Vite dev server :5173
终端2: pnpm --filter @app/shell dev      # 监听主进程/preload变化
```

Shell 的 `index.ts` 开发模式加载 Vite dev server：

```typescript
// packages/shell/src/main/index.ts
if (process.env.NODE_ENV === 'development') {
  mainWindow.loadURL('http://localhost:5173');
} else {
  mainWindow.loadFile(path.join(__dirname, '../web/index.html'));
}
```

### 9.9 打包脚本

```javascript
// scripts/bundle-openclaw.cjs
const { execSync } = require('child_process');
const fs = require('fs');
const path = require('path');

const pkg = require('../packages/shell/package.json');
const openclawVersion = pkg.dependencies.openclaw;

console.log(`Bundling OpenClaw ${openclawVersion}...`);

execSync(`pnpm install --filter @app/shell openclaw@${openclawVersion} --no-save`, { stdio: 'inherit' });

const gatewayPath = path.join(__dirname, '..', 'packages', 'shell', 'node_modules', 'openclaw');
if (!fs.existsSync(gatewayPath)) {
  throw new Error('OpenClaw not found after install');
}

console.log('OpenClaw bundled successfully');
```

### 9.10 ASAR 打包与加密

electron-builder 默认启用 ASAR 归档（`asar: true`）。为增强安全性，添加两层保护：

| 层级 | 机制 | 说明 |
|------|------|------|
| **ASAR 归档** | `asar: true` | 将源码打包为 `.asar` 二进制文件，防止直接读取 |
| **ASAR 完整性校验** | `asarIntegrity: true` | 为 ASAR 文件生成 SHA-256 哈希，运行时校验是否被篡改 |
| **ASAR 加密** | `afterPack` 自定义脚本 | 对 ASAR 内容进行 AES-256 加密，运行时解密加载 |

#### 9.10.1 ASAR 加密脚本

```javascript
// scripts/post-build.cjs
const fs = require('fs');
const path = require('path');
const crypto = require('crypto');
const { execSync } = require('child_process');

/**
 * electron-builder afterPack hook
 * 在 electron-builder 完成 ASAR 打包后执行 ASAR 加密
 */
module.exports = async function(context) {
  const { appOutDir, packager, electronPlatformName } = context;

  // ASAR 文件路径 (不同平台路径略有差异)
  const appName = packager.appInfo.productFilename;
  let asarPath;

  if (electronPlatformName === 'win32') {
    asarPath = path.join(appOutDir, 'resources', 'app.asar');
  } else if (electronPlatformName === 'darwin') {
    asarPath = path.join(appOutDir, `${appName}.app`, 'Contents', 'Resources', 'app.asar');
  } else {
    asarPath = path.join(appOutDir, 'resources', 'app.asar');
  }

  if (!fs.existsSync(asarPath)) {
    console.log(`[ASAR Encrypt] ASAR not found at ${asarPath}, skipping`);
    return;
  }

  console.log(`[ASAR Encrypt] Encrypting: ${asarPath}`);

  // 1. 读取原始 ASAR
  const asarContent = fs.readFileSync(asarPath);

  // 2. AES-256-CBC 加密
  const encryptionKey = process.env.ASAR_ENCRYPTION_KEY;
  if (!encryptionKey) {
    throw new Error(
      'ASAR_ENCRYPTION_KEY environment variable is required (32-byte hex string for AES-256)'
    );
  }

  const key = Buffer.from(encryptionKey, 'hex');
  const iv = crypto.randomBytes(16);
  const cipher = crypto.createCipheriv('aes-256-cbc', key, iv);

  const encrypted = Buffer.concat([
    cipher.update(asarContent),
    cipher.final(),
  ]);

  // 3. 写入加密后的 ASAR (iv + encrypted data)
  const encryptedAsar = Buffer.concat([iv, encrypted]);
  fs.writeFileSync(asarPath, encryptedAsar);

  console.log(`[ASAR Encrypt] Done: ${asarPath} (${(encryptedAsar.length / 1024 / 1024).toFixed(1)} MB)`);

  // 4. 注入解密 loader (修改 Electron 入口)
  injectDecryptLoader(appOutDir, electronPlatformName, appName);
};

/**
 * 注入解密代码到 Electron 主进程入口
 * 使得 Electron 启动时先解密 ASAR 到内存，再加载执行
 */
function injectDecryptLoader(appOutDir, platform, appName) {
  // 将解密逻辑写入 asar 的初始化脚本
  // 这里使用 Electron 的 --require 参数在启动时加载解密模块
  const decryptScript = path.join(appOutDir, 'resources', 'decrypt.js');

  const decryptCode = `
(function() {
  const fs = require('fs');
  const crypto = require('crypto');
  const path = require('path');

  // 读取加密的 ASAR
  const asarPath = __filename.replace('decrypt.js', 'app.asar');
  const encryptedData = fs.readFileSync(asarPath);

  // 提取 IV (前 16 字节)
  const iv = encryptedData.slice(0, 16);
  const encrypted = encryptedData.slice(16);

  // 解密
  const key = Buffer.from(process.env.ASAR_ENCRYPTION_KEY, 'hex');
  const decipher = crypto.createDecipheriv('aes-256-cbc', key, iv);
  const decrypted = Buffer.concat([decipher.update(encrypted), decipher.final()]);

  // 在内存中注册 ASAR (覆盖原始文件)
  fs.writeFileSync(asarPath + '.tmp', decrypted);
  fs.renameSync(asarPath + '.tmp', asarPath);
})();
`;

  fs.writeFileSync(decryptScript, decryptCode);
}
```

#### 9.10.2 加密密钥管理

```bash
# 生成 32 字节的 AES-256 密钥 (64 个 hex 字符)
node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"

# 构建时传入密钥 (CI/CD 环境变量)
ASAR_ENCRYPTION_KEY=your-64-char-hex-key pnpm bundle

# Windows PowerShell
$env:ASAR_ENCRYPTION_KEY="your-64-char-hex-key"; pnpm bundle
```

#### 9.10.3 CI/CD 集成 (GitHub Actions 示例)

```yaml
- name: Build & Package
  env:
    ASAR_ENCRYPTION_KEY: ${{ secrets.ASAR_ENCRYPTION_KEY }}
  run: pnpm bundle
```

### 9.11 打包体积预估

| 组件 | 大小 | 说明 |
|------|------|------|
| Electron 运行时 | ~120 MB | Chromium + Node.js |
| OpenClaw + node_modules | ~50-150 MB | 取决于依赖，可用 `--production` 裁剪 |
| Hermes Python 环境 | ~100-300 MB | 仅 Mac/Ubuntu 预装，Windows 按需下载 |
| 业务代码 | ~5-20 MB | 取决于 UI 复杂度 |
| **总计 (不含 Hermes)** | **~180-300 MB** | Windows 安装包 |
| **总计 (含 Hermes)** | **~280-600 MB** | Mac/Ubuntu 安装包 |

**优化方向**:
- OpenClaw 依赖裁剪：`npm ci --production`，移除 devDependencies
- 启用 electron-builder 的 7z 压缩
- Hermes 在 Windows 上改为按需下载安装包而非预装

---

## 十、OpenClaw 升级策略

### 9.1 三层隔离架构

```
┌───────────────────────────────────────┐
│  IPC 层 + 前端 UI                      │  ← 稳定，不随 OpenClaw 变化
│  (agent:chat, agent:switchModel, ...)  │
└──────────────┬────────────────────────┘
               │ 稳定接口
┌──────────────▼────────────────────────┐
│  OpenClaw Adapter 层                   │  ← 版本升级时唯一需要改的地方
│                                       │
│  • 封装 OpenClaw 的 Gateway API        │
│  • 封装 Hook / Channel 注册            │
│  • 向上层保持接口不变                  │
└──────────────┬────────────────────────┘
               │ 可能变化的接口
┌──────────────▼────────────────────────┐
│  OpenClaw Core (npm 依赖, 版本固定)    │  ← 只改 package.json 版本号
│                                       │
│  • 升级时更新 dependencies 中的版本号   │
│  • 然后验证 + 调整 adapter 层          │
└───────────────────────────────────────┘
```

### 9.2 升级流程

1. 更新 `package.json` 中 `openclaw` 版本号
2. 运行 `npm install` 获取新版本
3. 运行测试套件，验证 `OpenClawAdapter` 层是否有 API 变化
4. 如果有变化，只修改 `openclaw-adapter.ts` 这一个文件
5. 其他所有代码（聊天页面、守护进程、IPC 处理器、前端 UI）不需要改

### 9.3 版本锁定

```json
{
  "dependencies": {
    "openclaw": "2026.5.12"
  }
}
```

使用精确版本号（不用 `^` 或 `~`），确保每次构建使用完全相同的 OpenClaw 版本。

---

## 十一、Hermes Agent 集成

### 11.1 各平台策略

| 平台 | 策略 | 说明 |
|------|------|------|
| **macOS** | 预装 Python venv | 安装包包含 Hermes 依赖，应用启动可直接使用 |
| **Ubuntu** | 预装 Python venv | 同上 |
| **Windows** | WSL2 按需部署 | 不预装，用户点击"安装 Hermes"按钮后在 WSL2 中部署 |

### 11.2 Windows WSL2 部署流程

```
用户点击"安装 Hermes"
       │
       ▼
  检查 WSL2 + Ubuntu
       │
       ├── 未安装 → 弹出引导页面，引导用户安装 WSL2
       │
       └── 已安装
              │
              ▼
         wsl bash -c 'python3 -m venv ~/.hermes-venv &&
                      pip install hermes-agent'
              │
              ▼
         验证安装: wsl ~/.hermes-venv/bin/python3 -m hermes --version
              │
              └── 成功 → 更新 UI 状态为"可用"
```

### 11.3 打包策略

| 组件 | macOS/Ubuntu | Windows |
|------|-------------|---------|
| Hermes 依赖 | 预装在 `resources/hermes/` | 不预装 |
| 安装时机 | 安装包内置 | 用户点击按钮后下载部署 |
| 运行时命令 | `python3 -m hermes acp` | `wsl bash -c '~/.hermes-venv/bin/python3 -m hermes acp'` |

---

## 十一（附）、模型路由与多模型管理

详细技术调研见独立文档 **[MODEL_ROUTING.md](MODEL_ROUTING.md)**。

### 核心要点

```
┌─────────────────────────────────────────────────────────────┐
│  本地模型路由代理 (127.0.0.1:19000)                           │
│                                                             │
│  聚合: Kimi / GLM / Minimax / DeepSeek / 自定义              │
│  统一接口: /proxy/llm/v1/chat/completions                   │
│  模型列表: /proxy/llm/v1/models (返回聚合+状态)              │
│                                                             │
│  OpenClaw 只感知到代理: baseUrl = http://127.0.0.1:19000    │
│  Hermes 同样走代理: LLM_BASE_URL = http://127.0.0.1:19000   │
│  双引擎共享同一套路由和计费逻辑                                │
└─────────────────────────────────────────────────────────────┘
```

| 组件 | 实现方式 | 说明 |
|------|----------|------|
| **本地代理** | Node.js `http` 模块内嵌在主进程中 | 端口 19000，仅监听 127.0.0.1 |
| **Provider 混合** | `proxy` (走代理) + `direct` (直连) + `custom` (自定义) | 三类并存 |
| **模型聚合** | 代理从各 Provider 拉取模型列表，缓存 + 定期刷新 | "爆满" 状态从上游获取 |
| **请求转发** | modelId → 路由表 → 实际 API 端点 | 支持 SSE 流式透传 |
| **Session 记录** | 原样存储 modelId + provider | 不做映射，保持透明 |
| **环境变量** | 主进程动态注入 `QCLAW_LLM_BASE_URL` | 不在系统级暴露 |

**实现文件**: `packages/shell/src/main/core/model-proxy.ts`

---

## 十一（附二）、Agent 内核切换

详细技术调研见独立文档 **[KERNEL_SWITCHING.md](KERNEL_SWITCHING.md)**。

### 核心要点

QClaw 的内核切换设计：

```
┌─────────────────────────────────────────────────────────────┐
│  Kernel Pool (内核进程池)                                     │
│                                                             │
│  OpenClaw Gateway (Node.js) ← 默认启动，多会话共享            │
│  Hermes ACP (Python)      ← 按需启动，多会话共享             │
│                                                             │
│  Agent 创建时决定内核类型，之后固定                            │
│  System Prompt 注入实现"性格/语气"差异化                      │
│  统一协议层将不同内核协议转换为 NormalizedMessage              │
└─────────────────────────────────────────────────────────────┘
```

| 设计 | 方案 | 说明 |
|------|------|------|
| **进程复用** | 同类型内核共享进程，session ID 隔离 | 避免 N 个 Agent 启动 N 个进程 |
| **按需启动** | OpenClaw 默认启动，Hermes 首次使用时启动 | 节省启动时间和内存 |
| **协议统一** | NormalizedMessage 接口 | 前端只对接一套 IPC API |
| **System Prompt** | Agent 创建时定义，每次发消息前注入 | 实现"性格/语气" |
| **Mac 双内核预装** | OpenClaw + Hermes 都内置 | 用户可自由切换 |

**实现文件**: `packages/shell/src/main/core/kernel-manager.ts`

---

## 十一（附三）、多会话架构

详细技术调研见独立文档 **[MULTI_SESSION_ARCHITECTURE.md](MULTI_SESSION_ARCHITECTURE.md)**。

### 核心要点

QClaw 的多会话管理基于**双独立数据目录**设计，而非统一的抽象层：

| 维度 | OpenClaw (`~/.qclaw/`) | Hermes (`~/.qclaw-hermes/`) |
|------|------------------------|----------------------------|
| **配置文件** | `openclaw.json` (JSON) | `config.yaml` (YAML) |
| **会话索引** | `agents/main/sessions/sessions.json` | `agent.json` (sessionIds/sessionTitles/sessionUpdatedAts) |
| **会话存储** | JSONL 逐行追加 | 完整 JSON 文件 |
| **消息格式** | `{"type":"message","message":{"role":"user","content":[{"type":"text","text":"..."}]}}` | `{"messages":[{"role":"user","content":"你好","reasoning":"..."}]}` |
| **模型记录** | `sessions.json` 中 `modelProvider` + `model` 字段 | 消息中直接存储 `model` 字段 |

```
┌─────────────────────────────────────────────────────────────┐
│  前端 SessionManager                                         │
│                                                             │
│  1. probeDirectories() → 检测 .qclaw/ 和 .qclaw-hermes/     │
│  2. 分别解析两种格式的会话索引                                │
│  3. 归一化为 UnifiedSession 接口                             │
│  4. 通过 IPC 按需加载会话内容                                 │
└─────────────────────────────────────────────────────────────┘
```

**关键设计决策**：
- 前端主动探测双目录，主进程不做格式转换
- 两种内核共享同一本地代理 (127.0.0.1:19000)，但会话数据完全独立
- Agent 创建时绑定内核类型，会话文件归属由内核决定

---

## 十二、实施计划

### Phase 1: 项目初始化（1 周）

| 任务 | 产出 |
|------|------|
| 初始化 pnpm monorepo (shell + web + shared) | 可运行的空项目 |
| 配置 Electron + Vite + Vue 3 + TypeScript | 三端独立构建 |
| 配置 electron-builder | 三平台打包脚本 (NSIS/DMG/deb+AppImage) |
| 固定 OpenClaw 版本并集成到 shell 包 | `packages/shell/package.json` 锁定版本 |
| 实现 `OpenClawAdapter` 基础版 | 可启动/停止 Gateway |

### Phase 2: 核心功能（2-3 周）

| 任务 | 产出 |
|------|------|
| 完善 `OpenClawAdapter` | Hook 注入、Channel 注册、事件监听 |
| 实现网关守护进程 `Guardian` | 健康检查 + 指数退避重启 |
| 实现本地模型路由代理 `ModelProxy` | 端口 19000，聚合多 Provider 模型列表 |
| 实现模型切换 `ModelSwitcher` | 可用模型列表 + 切换 |
| 实现 IPC 通信层 | 前端 ↔ 后端完整通道 |
| 实现聊天页面 UI (Vue 3) | 消息列表 + 输入框 + 流式渲染 |
| 实现模型配置页面 (Vue 3) | 表单 + 保存 + 回读 |

### Phase 3: 定制功能（2-3 周）

| 任务 | 产出 |
|------|------|
| 实现内核管理器 `KernelManager` | OpenClaw/Hermes 进程池 + 统一协议层 |
| 实现 Agent 注册表 + 持久化 | `agents.json` + 多 Agent 切换 |
| 迁移现有自定义插件 | `src/plugins/` 目录 |
| 实现多 Channel 管理 | Channel 列表 + 启用/禁用 |
| 实现埋点/遥测系统 | 事件采集 + 上报 |
| 集成 Hermes Agent | Mac/Ubuntu 预装 + Windows WSL2 |
| Agent 切换功能 | UI + 后端切换逻辑 |

### Phase 4: 打包与测试（1-2 周）

| 任务 | 产出 |
|------|------|
| 配置跨平台打包 | DMG / NSIS / AppImage |
| 三平台集成测试 | 功能验证 |
| Hermes 跨平台行为测试 | Mac/Ubuntu/WSL2 验证 |
| OpenClaw 升级验证 | 版本切换测试 |
| 性能优化 | 包大小、启动速度、内存 |

**总计**: 6-9 周

---

## 十三、参考项目

### 竞品架构参考

| 项目 | 仓库 | 参考价值 |
|------|------|----------|
| **QClaw** | 腾讯电脑管家团队 | Electron + OpenClaw 深度集成 |
| **LobsterAI** | github.com/netease-youdao/lobsterai | electron-builder 打包 OpenClaw 运行时 |
| **ClawX** | github.com/ValueCell-ai/ClawX | Vite + electron-builder + OpenClaw GUI |
| **EasyClaw** | github.com/ybgwon96/easyclaw | OpenClaw 一键安装器 |

### 技术文档参考

| 资源 | 地址 |
|------|------|
| OpenClaw 官方文档 | https://docs.openclaw.ai |
| OpenClaw 插件系统 | https://openclaw-openclaw.mintlify.app/plugins/introduction |
| OpenClaw Gateway 配置 | https://github.com/openclaw/openclaw/blob/main/docs/gateway/configuration.md |
| Hermes Agent | https://github.com/NousResearch/hermes-agent |
| Hermes ACP 协议 | https://hermes-agent.nousresearch.com/docs/user-guide/features/acp |
| Electron 官方文档 | https://www.electronjs.org/docs |
| electron-builder | https://www.electron.build |

---

## 附录：框架选型决策树

```
是否需要深度定制 OpenClaw (Hook/插件/埋点/守护)?
├── 是 → 团队是否希望 Web 团队全栈开发?
│        ├── 是 → Electron (推荐)
│        └── 否 → Tauri (需 Rust 工程师)
│
└── 否 → 是否对包大小有极致要求?
         ├── 是 → Tauri
         └── 否 → Electron
```

对于本项目（深度定制 + Web 团队全栈 + 类 QClaw 产品），**Electron 是唯一合理的选择**。
