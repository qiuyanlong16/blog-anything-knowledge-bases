# 腾讯 QClaw 代码层架构分析

> **分析日期**: 2026-05-13
> **产品版本**: 0.2.18
> **包名**: `@guanjia-openclaw/electron`
> **说明**: 本文档仅包含从源码中直接提取的信息，所有结论均有代码证据支撑。

---

## 一、源码可读性说明

| 文件/目录 | 状态 | 可读性 |
|-----------|------|--------|
| `package.json` | 源码 | 完全可读 |
| `out/main/index.cjs` | 源码 | 完全可读 (4行) |
| `out/main/openclaw-bootstrap.mjs` | 源码 | 完全可读 (87行) |
| `out/main/bytecode-loader.cjs` | 源码 | 完全可读 (73行) |
| `out/main/index.cjsc` | **V8字节码** (3.9MB) | 仅可提取字符串，不可反编译 |
| `out/main/chunks/*.cjsc` | **V8字节码** (453KB+810KB) | 仅可提取字符串 |
| `out/preload/*.cjs` | 源码 | 完全可读 (各4行) |
| `out/preload/*.cjsc` | **V8字节码** | 不可读 |
| `out/renderer/index.html` | 源码 | 完全可读 (19行) |
| `out/renderer/sdk/*.js` | 源码 | 完全可读 (263行) |
| `out/renderer/assets/*.js` | 源码 (Vite压缩) | 可提取标识符，不可读逻辑 |
| `out/renderer/assets/*.css` | 源码 | 可读样式 |

**核心业务逻辑 (Electron main process) 全部编译为 V8 bytecode (`.cjsc`)**，无法直接还原源码。
以下分析仅基于 bytecode 中提取的字符串常量和可读源码文件。

---

## 二、直接读取的源码分析

### 2.1 入口结构 (`out/main/index.cjs`)

```javascript
"use strict";
require("./bytecode-loader.cjs");
require("./index.cjsc");
```

仅 4 行：加载 bytecode 解析器 → 加载编译后的主程序。

### 2.2 Bytecode 加载机制 (`out/main/bytecode-loader.cjs`)

从代码中提取的确信息：

- 注册 `Module._extensions[".jsc"]` 和 `Module._extensions[".cjsc"]` 自定义扩展处理器
- V8 标志：`--no-lazy`、`--no-flush-bytecode`
- 字节码文件头结构：
  - offset 8: SOURCE_HASH (4 bytes) — 原始源码长度
  - offset 12: FLAG_HASH (4 bytes) — V8 兼容性标志
- 使用 `vm.Script` + `cachedData` 加载字节码
- 构造零宽空格 dummy code (`"​".repeat(length - 2)`) 匹配原始源码长度
- 注入自定义 `require` 函数，传入 `module.exports, require, module, filename, dirname, process, global`

### 2.3 Bootstrap 看门狗 (`out/main/openclaw-bootstrap.mjs`)

从代码中提取的确切信息：

- 环境变量：
  - `QCLAW_PARENT_PID` — 父进程 PID，用于看门狗监控
  - `QCLAW_REAL_ENTRY` — 真实入口文件路径
  - `QCLAW_MEMORY_FS_REGISTER` — 内存文件系统注册 URL
- 看门狗：每 **2000ms** 轮询 `process.kill(pid, 0)`，父进程死亡则 `process.exit(0)`
- 每 30 次检查记录一次心跳日志
- 加载完真实入口后清理三个环境变量
- 将 `process.argv[1]` 修补为真实入口路径

### 2.4 IMA JSAPI SDK (`out/renderer/sdk/ima-jsapi-sdk.js`, v1.0.0)

从代码中提取的 **15 个 API 方法**（均为源码直接可读）：

| 方法名 | 参数 | 返回值 |
|--------|------|--------|
| `getDeviceInfo()` | 无 | `{q36, qua}` |
| `getToken()` | 无 | `{token}` |
| `refreshToken()` | 无 | - |
| `addKnowledgeTask({medias})` | `{id, mediaType, title}[]` | - |
| `download({url})` | URL | - |
| `openBrowser({url})` | URL | - |
| `openMedia({id, mediaType})` | 文件ID+类型 | - |
| `openApp({schema, url})` | schema 或 URL | - |
| `checkAppInstalled()` | 无 | `{installed: boolean}` |
| `getAccountInfo()` | 无 | `{accountInfo}` |
| `encryptData({data})` | 明文字符串 | `{data, x_ima_cm, x_ima_ckey?, x_ima_ctk?}` |
| `decryptData({data})` | 密文字符串 | `{msg, data}` |
| `setCryptoToken({token, expire})` | token+过期时间 | - |
| `clearCryptoSession()` | 无 | - |
| `notifyAuthCode({authCode})` | 授权码 | - |
| `getSupportFileFormats()` | 无 | `{formats}` |

通信机制：
- 请求消息类型：`qclaw-ima-jsapi`
- 响应消息类型：`qclaw-ima-jsapi-response`
- 超时：30 秒
- 全局暴露：`window.QClawBridge` 和 `window.QClawJSBridge`

### 2.5 IMA SDK 包信息 (`out/renderer/sdk/package.json`)

```json
{
  "name": "@tencent/qclaw-ima-jsapi-sdk",
  "version": "1.1.2",
  "author": "xuankehuang",
  "publishConfig": { "registry": "https://mirrors.tencent.com/npm/" }
}
```

### 2.6 依赖清单 (`package.json`)

直接可读的依赖（非推断）：

| 依赖 | 版本 | 用途（从包名/文档推断） |
|------|------|------------------------|
| `@electron-toolkit/utils` | ^3.0.0 | Electron 工具 |
| `@guanjia-openclaw/galileo-otel` | workspace:* | 内部 OTel 封装 |
| `@guanjia-openclaw/report` | workspace:* | 内部报告库 |
| `@tencent-connect/qqbot-connector` | ^1.1.0 | QQ Bot 连接器 |
| `@tencent/docs-engine` | 0.0.1-beta.8 | 腾讯文档引擎 |
| `@tencent/smh-js-sdk` | ^1.0.6 | 安全/监控 SDK |
| `better-sqlite3` | ^12.8.0 | SQLite 数据库 |
| `electron-log` | ^5.4.3 | 日志框架 |
| `electron-updater` | ^6.6.2 | 自动更新 |
| `tencentcloud-sdk-nodejs-asr` | ^4.1.205 | 腾讯云语音识别 |
| `tencentcloud-sdk-nodejs-tts` | ^4.1.182 | 腾讯云语音合成 |
| `mint-filter` | ^4.0.3 | 敏感词过滤 |
| `pii-masker` | ^1.0.1 | PII 脱敏 |
| `node-machine-id` | ^1.1.12 | 设备指纹 |
| `adm-zip` | ^0.5.16 | ZIP 处理 |
| `archiver` | ^7.0.1 | 归档压缩 |
| `tar` | ^7.5.9 | TAR 处理 |
| `yauzl` | ^3.3.0 | ZIP 读取 |
| `json5` | ^2.2.3 | JSON 解析 |
| `jsonrepair` | ^3.13.3 | JSON 修复 |

---

## 三、从 Bytecode 字符串提取的信息

### 3.1 确认的外部 URL（按功能分类）

**官方与核心服务：**
| URL | 提取来源 | 用途 |
|-----|----------|------|
| `https://qclaw.qq.com/` | CSP 头 + 字符串 | 官方网站 |
| `https://guanjia.qq.com` | 字符串 | QQ 管家 |
| `https://ilinkai.weixin.qq.com` | 字符串 | 微信 AI 后端 |
| `https://cdn.m.qq.com/qclaw/lookfile/index.html?filepath=` | 字符串 | 文件预览 CDN |

**JPRX (精灵小助手) 后端：**
| URL | 提取来源 | 用途 |
|-----|----------|------|
| `https://jprx.m.qq.com/` | 字符串 | 生产环境 API |
| `https://jprx.woa.com/` | 字符串 | 内部环境 API |
| `http://jprx.beta.wsd.com/` | 字符串 | Beta 环境 |
| `jprx.sparta.html5.qq.com` | 字符串 | JPRX sparta 服务 |
| `/data/4142/forward`, `/data/4169/forward`, `/data/4235/forward`, `/data/4236/forward` | 字符串 | JPRX 数据转发路由 |

**安全与监控：**
| URL | 提取来源 | 用途 |
|-----|----------|------|
| `https://security.guanjia.qq.com/login` | 字符串 | 安全登录 |
| `https://pcmgrmonitor.3g.qq.com/datareport` | 字符串 | 数据上报 |
| `https://mmgrcalltoken.3g.qq.com/aizone/v1` | 字符串 | Agent 服务 |
| `wss://mmgrcalltoken.3g.qq.com/agentwss` | 字符串 | Agent WebSocket |
| `https://aegis.qq.com/collect/events` | 字符串 | Aegis 前端监控 |
| `https://galileotelemetry.tencent.com/collect` | 字符串 | Galileo 遥测 |
| `https://galileotelemetry.tencent.com/crashReport` | 字符串 | Galileo 崩溃报告 |

**腾讯云 ASR/TTS：**
| URL | 提取来源 | 用途 |
|-----|----------|------|
| `asr.tencentcloudapi.com` | 字符串 | 语音识别 API |
| `tts.tencentcloudapi.com` | 字符串 | 语音合成 API |

**腾讯文档：**
| URL | 提取来源 | 用途 |
|-----|----------|------|
| `https://docs.qq.com/openapi/mcp` | 字符串 | MCP 开放 API |
| `https://docs.qq.com/api/v6/doc/mcp` | 字符串 | 文档 MCP |
| `https://docs.qq.com/api/v6/sheet/mcp` | 字符串 | 表格 MCP |
| `https://docs.qq.com/scenario/open-claw.html` | 字符串 | OpenClaw 场景页 |

**腾讯问卷：**
| URL | 提取来源 | 用途 |
|-----|----------|------|
| `https://wj.qq.com/api/v2/account/tokens/device-auth/init` | 字符串 | 设备认证初始化 |
| `https://wj.qq.com/oauth/authorize` | 字符串 | OAuth 授权 |
| `https://wj.qq.com/api/v2/account/tokens/device-auth/poll` | 字符串 | 设备认证轮询 |
| `https://wj.qq.com/api/v2/mcp` | 字符串 | 问卷 MCP |

**企业微信：**
| URL | 提取来源 | 用途 |
|-----|----------|------|
| `https://work.weixin.qq.com/ai/qc/generate` | 字符串 | 企微 AI 生成 |
| `https://work.weixin.qq.com/ai/qc/query_result` | 字符串 | 企微 AI 查询结果 |
| `https://work.weixin.qq.com/ai/qc/gen` | 字符串 | 企微 AI 生成 |
| `https://qyapi.weixin.qq.com/cgi-bin/aibot/cli/get_mcp_config` | 字符串 | 企微 AI Bot MCP 配置 |

**百度网盘：**
| URL | 提取来源 | 用途 |
|-----|----------|------|
| `https://openapi.baidu.com/oauth/2.0/token` | 字符串 | OAuth Token |

**微云：**
| URL | 提取来源 | 用途 |
|-----|----------|------|
| `https://www.weiyun.com/authorize/token` | 字符串 | 微云授权 |
| `https://www.weiyun.com/api/v3/mcp/token/code` | 字符串 | 微云 MCP Token |
| `https://www.weiyun.com/api/v3/mcpserver` | 字符串 | 微云 MCP Server |

**WPS：**
| URL | 提取来源 | 用途 |
|-----|----------|------|
| `https://api.wps.cn/office/v5/ai/skill_hub/users/callback` | 字符串 | WPS Skill Hub 回调 |
| `https://account.wps.cn/login` | 字符串 | WPS 登录 |
| `https://api.wps.cn/office/v5/ai/skill_hub/wps_auth/exchange` | 字符串 | WPS 授权交换 |
| `https://mcp-center.wps.cn/skill_hub/mcp` | 字符串 | WPS MCP Center |

**其他腾讯服务：**
| URL | 提取来源 | 用途 |
|-----|----------|------|
| `https://goatdee.qq.com/ppt/` | 字符串 | AI PPT 服务 |
| `https://api.zhihui.qq.com` | 字符串 | 智慧QQ设计服务 |
| `skillhub.tencent.com` | CSP 头 | Skill Hub |

**COS 存储：**
| URL | 提取来源 | 用途 |
|-----|----------|------|
| `.cos.ap-guangzhou.myqcloud.com` | 字符串 | 广州 COS |
| `.cos.ap-shanghai.myqcloud.com` | 字符串 | 上海 COS |
| `.cos.ap-beijing.myqcloud.com` | 字符串 | 北京 COS |
| `webcdn-75028.gzc.vod.tencent-cloud.com` | 字符串 | 腾讯云 CDN |
| `.s3-accelerate.amazonaws.com` | 字符串 | S3 加速端点 |

**本地服务：**
| URL | 提取来源 | 用途 |
|-----|----------|------|
| `http://localhost:4318/` | 字符串 | OpenTelemetry OTLP |
| `http://localhost:5175` | 字符串 | 开发服务器 |
| `http://127.0.0.1:28789/*` | CSP 头 | 本地服务端口 |
| `http://127.0.0.1:39099/*` | CSP 头 | 本地服务端口 |
| `https://doh.pub/dns-query` | 字符串 | 腾讯 DoH |
| `https://dns.alidns.com/dns-query` | 字符串 | 阿里 DoH |

### 3.2 CSP 策略（从 bytecode 字符串直接提取）

```
开发环境 CSP:
default-src 'self';
script-src 'self' 'unsafe-inline' 'unsafe-eval' https://res.wx.qq.com https://wwcdn.weixin.qq.com;
style-src 'self' 'unsafe-inline';
img-src 'self' data: https: http: local:;
connect-src 'self' ws: wss: http: https:;
font-src 'self' data:;
frame-src 'self' https://open.weixin.qq.com https://work.weixin.qq.com http://127.0.0.1:39099

生产环境 CSP:
default-src 'self';
script-src 'self' https://res.wx.qq.com https://wwcdn.weixin.qq.com;
style-src 'self' 'unsafe-inline';
img-src 'self' data: https: http: local:;
connect-src 'self' http: https:;
font-src 'self' data:;
frame-src 'self' https://open.weixin.qq.com https://work.weixin.qq.com http://127.0.0.1:39099
```

### 3.3 IPC 通道（从 bytecode 字符串直接提取，共 90+ 个）

**app: 前缀 (32 个)：**
```
app:captureScreen          app:get-marketing-params
app:check-openclaw-compatibility  app:get-openclaw-version
app:checkSkillsInstalled   app:getScreenResolution
app:copySkillsToWorkspace  app:getSkillsInfo
app:downloadAndExtractWorkspace  app:getSystemHardware
app:downloadFile           app:get-version
app:downloadFileToPath     app:isRunningUnderARM64Translation
app:downloadProgress       app:openAISandbox
app:downloadSkill          app:openFileDialog
app:enableSkills           app:openFolderDialog
app:fetchUrl               app:openPath
app:get-channel            app:previewFile
app:get-first-channel      app:readMediaFile
app:getLocalFilePreviewUrl app:removeUserSkill
app:get-machine-id         app:reportPerfTiming
                           app:saveBase64Image
                           app:showItemInFolder
```

**credentials: 前缀 (31 个)：**
```
credentials:authorizedPlatforms  credentials:qqbotStopQrConnect
credentials:completeAuth         credentials:remove
credentials:disconnect           credentials:removeForAccount
credentials:getOAuthCallbackUrl  credentials:removeSkillEnv
credentials:has                  credentials:save
credentials:hasForAccount        credentials:saveForAccount
credentials:imaCheckInstalled    credentials:saveToken
credentials:imaWriteTokenToMcporter  credentials:tdocCheckAuth
credentials:initiateAuth         credentials:tdocDisconnect
credentials:installPluginAndRestart  credentials:tdocEnsureMcporter
credentials:listAccountIds       credentials:tdocSyncLocalToken
credentials:mcpSkillCheckAuth    credentials:tdocWaitAuth
credentials:mcpSkillDisconnect   credentials:toggleAuth
credentials:mcpSkillEnsureMcporter   credentials:wecomCliCheckAuth
credentials:mcpSkillWaitAuth     credentials:wecomCliDisconnect
credentials:openExternal         credentials:wecomCliGenerateQR
credentials:openUrlBySystem      credentials:wecomCliHasCredentials
credentials:qqbotStartQrConnect  credentials:wecomCliManualAuth
                                 credentials:wecomCliPollQR
                                 credentials:writeSkillEnv
```

**integration: 前缀 (22 个)：**
```
integration:getOAuthCallbackUrl   integration:weixinDisconnect
integration:getWeixinStatus       integration:weixinEnable
integration:imaCheckInstalled     integration:weixinLoginCancel
integration:imaWriteTokenToMcporter  integration:weixinLoginPoll
integration:mcpSkill* (4个)       integration:weixinLoginStart
integration:oauthCallback         integration:weixinSubmitVerifyCode
integration:openExternal          integration:yuanbaoLoginCancel
integration:openUrlBySystem       integration:yuanbaoLoginPoll
integration:qqbotQrExpired        integration:yuanbaoLoginStart
integration:qqbotQrUrl
integration:qqbotStartQrConnect
integration:qqbotStopQrConnect
integration:tdoc* (5个)
integration:wecomCli* (6个)
```

**gateway: 前缀 (6 个)：**
```
gateway.auth      gateway.reload
gateway.bind      gateway.remote
gateway.controlUi gateway.tailscale
gateway.port
```

**feedback: 前缀 (7 个)：**
```
feedback:isWindowOpen        feedback:resetForm
feedback:notifyScreenshotSuccess  feedback:screenshotSuccess
feedback:notifySubmitSuccess feedback:submitSuccess
feedback:openWindow
```

**其他通道：**
```
filePreview:openWindow          securityAudit:getAuditLogCount
filePreview:setData             securityAudit:getAuditLogs
ioaLogin:openWindow             securityAudit:getConfig
ioaLogin:closeWindow            securityAudit:getEnabled
ioaLogin:windowClosed           securityAudit:getFileProtectionMode
appearance:setNativeTheme       securityAudit:removeFileProtectionPaths
quit-and-install                file-system
get-downloaded-info             file-tracking
get-openclaw-version            stream-start
check-openclaw-compatibility    set-user-id
install-inspiration-skills      securityAudit
                                securityAudit:addFileProtectionPaths
```

### 3.4 OpenTelemetry 属性键（从 bytecode 字符串直接提取）

**`openclaw.*` 前缀属性 (30+ 个)：**
```
openclaw.run_id                openclaw.llm_proxy.client_url
openclaw.session_id            openclaw.llm_proxy.upstream_url
openclaw.session_key           openclaw.llm_proxy.upstream_host
openclaw.llm.seq               openclaw.llm_proxy.method
openclaw.scence_type           openclaw.llm_proxy.type
openclaw.channel_id            openclaw.llm_proxy.trace_source
openclaw.trace_id              openclaw.conversation_id
openclaw.client_trace_id       openclaw.conversation_request_id
openclaw.query_scene           openclaw.agent.id
openclaw.prompt                openclaw.agent.name
openclaw.system_prompt         openclaw.is_first_turn
openclaw.tool_name             openclaw.session.completed
openclaw.app_version           openclaw.session.interrupted
openclaw.source_terminal       openclaw.error_message
openclaw.openclaw_version      openclaw.user_message_length
openclaw.prompt_id             openclaw.assistant_response_length
openclaw.wechat_session_id     openclaw.finish_reason
openclaw.usage_inspiration     openclaw.http_status
openclaw.channel_request_id    openclaw.duration_ms
openclaw.channel_session_id    openclaw.request_body_length
openclaw.backend               openclaw.response_content_length
openclaw.platform              openclaw.task_id
openclaw.sender_id             openclaw.tool.args_preview
openclaw.llm_proxy.status      openclaw.tool.result_length
openclaw.llm_proxy.duration_sec openclaw.tool.result_truncated
openclaw.llm_proxy.error_message
```

### 3.5 RUM 事件名（从 bytecode 字符串直接提取，50+ 个）

```
RUM_EVENT_APP_LAUNCH                    RUM_EVENT_OPENCLAW_SUPERVISOR_RECOVER_FAILED
RUM_EVENT_APP_QUIT                      RUM_EVENT_OPENCLAW_SUPERVISOR_RESTART_ATTEMPT
RUM_EVENT_CHILD_PROCESS_GONE             RUM_EVENT_OPENCLAW_UNEXPECTED_EXIT
RUM_EVENT_CONFIG_FILE_PERMISSION         RUM_EVENT_OPENCLAW_VERSION_INFO
RUM_EVENT_HERMES_SERVICE_START           RUM_EVENT_PID_GUARD_ALLOW
RUM_EVENT_HERMES_TAR_EXTRACT             RUM_EVENT_PID_GUARD_BLOCK
RUM_EVENT_HERMES_UNEXPECTED_EXIT         RUM_EVENT_PID_GUARD_SKIP
RUM_EVENT_JWT_GUARD_ALLOW                RUM_EVENT_RENDERER_PROCESS_GONE
RUM_EVENT_JWT_GUARD_BLOCK                RUM_EVENT_SERVICE_START
RUM_EVENT_LOOP_DETECTION_CONFIG_APPLY    RUM_EVENT_SERVICE_STOP
RUM_EVENT_LOOP_DETECTION_CONFIG_FETCH    RUM_EVENT_TURING_SHIELD_INIT
RUM_EVENT_MAC_TAR_EXTRACT                RUM_EVENT_UPDATER_DOWNLOAD_FAILED
RUM_EVENT_MAIN_UNCAUGHT_EXCEPTION        RUM_EVENT_UPDATER_DOWNLOAD_START
RUM_EVENT_MAIN_UNHANDLED_REJECTION       RUM_EVENT_UPDATER_DOWNLOAD_SUCCESS
RUM_EVENT_OPENCLAW_CIRCUIT_OPEN          RUM_EVENT_UPDATER_INSTALL_TRIGGERED
RUM_EVENT_OPENCLAW_CONFIG_RECOVERY_EXHAUSTED  RUM_EVENT_UPDATER_INSUFFICIENT_DISK_SPACE
RUM_EVENT_OPENCLAW_FIELD_RESTORE_FALLBACK     RUM_EVENT_WIN_TAR_BOOT_RECOVERY
RUM_EVENT_OPENCLAW_HEALTH_RESTART             RUM_EVENT_WIN_TAR_RUNTIME_RECOVERY
RUM_EVENT_OPENCLAW_INVALID_CONFIG             RUM_EVENT_WORKSPACE_MEMORY_STATUS
RUM_EVENT_OPENCLAW_LOCK_CLEANUP
RUM_EVENT_OPENCLAW_SUPERVISOR_RECOVER
RUM_EVENT_AGENT_ZIP_DOWNLOAD
RUM_EVENT_AGENT_ZIP_EXTRACT
```

### 3.6 代理路由（从 bytecode 字符串直接提取）

```
/proxy/qclaw-cos        - 腾讯云 COS 代理
/proxy/prosearch        - 搜索代理
/proxy/oauth-callback   - OAuth 回调代理
/proxy/internal/report  - 内部报告代理
```

### 3.7 LLM 相关端点（从 bytecode 字符串直接提取）

```
/chat/completions     - OpenAI 兼容聊天补全
/v1/responses         - Hermes Agent 响应端点
qclaw/modelroute      - 模型路由
```

### 3.8 数据库与存储（从 bytecode 字符串提取）

从字符串中提取的数据库操作痕迹：

- SQLite 查询模式：`updated_at = excluded.updated_at` (UPSERT)
- 会话表：`state.db.sessions`
- 响应存储：`response_store.conversations`
- 线程绑定：`threadBindings` (含 `idleHours`, `ttlHours`)
- 会话清理：`scheduleSessionCleanup`, `SESSION_LINGER_MS`
- Agent 配置：`agent.json`, `openclaw.json`, `openclaw.tar.gz`
- 插件配置：`openclaw.plugin.json`

### 3.9 安全审计模块（从 bytecode 字符串提取）

```
securityAudit              - 审计模块
securityAudit:addFileProtectionPaths   - 添加文件保护路径
securityAudit:getAuditLogCount         - 获取审计日志数量
securityAudit:getAuditLogs             - 获取审计日志
securityAudit:getConfig                - 获取审计配置
securityAudit:getEnabled               - 获取审计启用状态
securityAudit:getFileProtectionMode    - 获取文件保护模式
securityAudit:removeFileProtectionPaths - 移除文件保护路径
```

相关常量：
```
TURING_SHIELD_IPC          - 图灵盾 IPC 命名空间
TURING_SHIELD_PRODUCT_NAME - 图灵盾产品名
TURING_SHIELD_CHANNEL_ID   - 图灵盾通道 ID
PROTECTED_CONFIG_PATHS     - 受保护配置路径
SENSITIVE_FIELD_KEYWORDS   - 敏感字段关键词
SENSITIVE_VAR_PATTERNS     - 敏感变量模式
SENSITIVE_WORDS            - 敏感词列表
EXTRA_PII_RULES            - 额外 PII 规则
```

### 3.10 从 Renderer JS 提取的 UI 组件标识符

**Agent 相关：**
```
AgentApiService, AgentAvatarList, AgentInspirations, AgentInteraction,
AgentInternalContent, AgentList, AgentManager, AgentPreference,
AgentRead, AgentSystemLines, AgentSystemMessageByText, AgentUnreadCount,
AgentUnreadCounts, AgentInCache, AgentData, agenter, agentCount, agentId,
agentIds, agentMap, agents, AgentApiService
```

**Chat 相关：**
```
ChatState, ChatMessage, ChatDedup, ChatInterrupted, ChatPageReady,
ChatMainLayoutMounted, chatMessageCallback, chatCallbackWarned,
chatCodeBg, chatCodeBorder, chatCodeHeaderBg, chatCodeText,
chatBubbleUserBg, chatBubbleUserText, chatBlockquoteBg, chatBg,
chatWsUrl, chatVersion, chattype, chatVersion
```

**文件相关：**
```
FilePreviewWindow, FileSpaceSwitchConfig, DownloadSwitchConfig,
DownloadProgress, FileToCOS, fileResourceUpload, filesGet, filesList,
filesSet, AvatarUploadError
```

**IMA 相关：**
```
imaAddKnowledge, imaCheckAuth, imaCreateMedia, imaDownloadMedias,
imaGetAuthState, imaGetKnowledgeBaseList, imaGetKnowledgeList,
imaGetSkillApikey, imaRevoke, imaSearchKnowledge
```

**其他 UI 组件：**
```
ConfigProvider, ConfigService, ConfigChange, Configs, configKey, configData
ConfirmDialog, DialogTitle, DialogContent, DialogMask, DialogWrap
ButtonProps, ButtonSize, ButtonGroup, buttonPaddingHorizontal
Avatar, AvatarList, AvatarUploadError, avatarUrl
Backup, backupPath
Channel, channels, channelID, channelSource, channelToken
JprxUrl, jprxGateway, jprxSign
AgentBaseUrl, openclawApiService, openclawChannelToken, openClawToken
```

---

## 四、架构图

基于以上从代码中直接提取的信息，绘制如下架构图：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        启动层 (Bootstrap)                                │
│                                                                         │
│  [外部 Launcher] ──设置 QCLAW_PARENT_PID──→ [openclaw-bootstrap.mjs]     │
│                     QCLAW_REAL_ENTRY                                      │
│                     QCLAW_MEMORY_FS_REGISTER                              │
│                                    │                                      │
│                                    ├─ 看门狗: 每2s检查父进程存活           │
│                                    ├─ MemoryFS 注册 (可选)                │
│                                    └─ 动态 import 真实入口                │
└────────────────────────────────────┼─────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    Electron Main Process (.cjsc 字节码)                   │
│                                                                         │
│  ┌──────────────────────┐  ┌──────────────────────────────────────────┐│
│  │  Electron 应用初始化    │  │          本地服务                       ││
│  │                      │  │                                          ││
│  │  • Single Instance   │  │  ┌─────────────┐  ┌──────────────────┐  ││
│  │  • qclaw:// 协议注册 │  │  │ AuthGateway │  │ Gateway Router   │  ││
│  │  • CSP (开发/生产)   │  │  │ Server      │  │                  │  ││
│  │  • 自定义证书验证     │  │  │             │  │  /chat/completions│  ││
│  │  • DoH DNS 解析      │  │  │ auth.*      │  │  /v1/responses    │  ││
│  │  • GPU 降级处理      │  │  │ bind        │  │  /proxy/qclaw-cos │  ││
│  │  • ARM 翻译检测      │  │  │ reload      │  │  /proxy/prosearch │  ││
│  │                      │  │  │ remote      │  │  /proxy/oauth-cb  │  ││
│  │                      │  │  │ tailscale   │  │  /proxy/internal/ │  ││
│  │                      │  │  │ controlUi   │  │  /report          │  ││
│  │                      │  │  │ port        │  │  qclaw/modelroute │  ││
│  └──────────┬───────────┘  │  └─────────────┘  └──────────────────┘  ││
│             │               └──────────────────────────────────────────┘│
│             ▼                                                           │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │  BrowserWindow (主窗口) + Tray + 辅助窗口                         │  │
│  │                                                                  │  │
│  │  ┌───────────────┐  ┌────────────────────────────────────────┐  │  │
│  │  │ Preload       │  │  Renderer (Vue 3 + Ant Design Vue)     │  │  │
│  │  │ (.cjsc)       │  │                                        │  │  │
│  │  │               │  │  组件: Chat, Agent, File, Config,      │  │  │
│  │  │ Context       │  │  IMA, Channel, Backup, JPRX...         │  │  │
│  │  │ Isolation     │  │                                        │  │  │
│  │  │               │  │  外部 SDK:                              │  │  │
│  │  │ IPC Bridge    │  │  • wxLogin.js (微信登录)               │  │  │
│  │  └───────┬───────┘  │  • wecom-aibot-sdk (企微AI Bot)        │  │  │
│  │          │          │  • ima-jsapi-sdk (IMA JSAPI)           │  │  │
│  │          │          └────────────────────────────────────────┘  │  │
│  └──────────┼──────────────────────────────────────────────────────┘  │
│             │                                                         │
│             ▼                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │  IPC 通信层 (90+ 通道)                                           │ │
│  │                                                                  │ │
│  │  app:* (32)     credentials:* (31)   integration:* (22)          │ │
│  │  gateway:* (6)  feedback:* (7)      filePreview:* (2)           │ │
│  │  ioaLogin:* (3) appearance:* (1)    securityAudit:* (6)         │ │
│  │  quit-and-install  get-downloaded-info  file-system  etc.       │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                       │
│  ┌───────────────────────┐  ┌───────────────────────────────────────┐│
│  │  腾讯服务集成          │  │  第三方服务集成                        ││
│  │                       │  │                                       ││
│  │  jprx.m.qq.com        │  │  docs.qq.com (腾讯文档 MCP)           ││
│  │  jprx.woa.com         │  │  work.weixin.qq.com (企业微信)        ││
│  │  ilinkai.weixin.qq.com│  │  wj.qq.com (腾讯问卷)                 ││
│  │  guanjia.qq.com       │  │  weiyun.com (微云)                    ││
│  │  qclaw.qq.com         │  │  api.wps.cn (WPS Skill Hub)          ││
│  │  goatdee.qq.com       │  │  openapi.baidu.com (百度网盘)         ││
│  │  api.zhihui.qq.com    │  │  skillhub.tencent.com                 ││
│  │                       │  │  github.com (GitHub)                  ││
│  │  mmgrcalltoken.3g.qq  │  │  COS: ap-guangzhou/shanghai/beijing  ││
│  │  .com (Agent WSS)     │  │  S3: s3-accelerate.amazonaws.com     ││
│  │                       │  │                                       ││
│  │  asr.tencentcloudapi  │  │  webcdn-75028.gzc.vod               ││
│  │  .com (ASR)           │  │  .tencent-cloud.com (CDN)            ││
│  │  tts.tencentcloudapi │  │  objects.githubusercontent.com        ││
│  │  .com (TTS)           │  │                                       ││
│  │  security.guanjia.qq  │  │                                       ││
│  │  .com (安全登录)       │  │                                       ││
│  │  pcmgrmonitor.3g.qq   │  │                                       ││
│  │  .com (数据上报)       │  │                                       ││
│  │  aegis.qq.com         │  │                                       ││
│  │  (前端监控)            │  │                                       ││
│  │  galileotelemetry.    │  │                                       ││
│  │  tencent.com (遥测)   │  │                                       ││
│  └───────────────────────┘  └───────────────────────────────────────┘│
│                                                                       │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │  可观测性 (OpenTelemetry + Galileo)                               │ │
│  │                                                                  │ │
│  │  OTLP Exporter: http://localhost:4318                            │ │
│  │                                                                  │ │
│  │  RUM 事件: 50+ 事件 (app_launch, app_quit, hermes_*, updater_*,  │ │
│  │            circuit_open, supervisor_*, pid_guard_*, jwt_guard_*, │ │
│  │            unexpected_exit, health_restart, tar_extract 等)       │ │
│  │                                                                  │ │
│  │  Trace 属性: 30+ openclaw.* 属性                                 │ │
│  │            (session_id, run_id, trace_id, llm_proxy.*,           │ │
│  │             tool.*, http_status, duration_ms 等)                  │ │
│  │                                                                  │ │
│  │  Crash Handler: CrashHandler + CRASH_REPORT_DIR                  │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                       │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │  安全模块                                                        │ │
│  │                                                                  │ │
│  │  • Turing Shield (TURING_SHIELD_IPC/PRODUCT_NAME/CHANNEL_ID)     │ │
│  │  • PII Masker (EXTRA_PII_RULES)                                 │ │
│  │  • 敏感词过滤 (SENSITIVE_WORDS)                                  │ │
│  │  • 文件保护 (PROTECTED_CONFIG_PATHS, SENSITIVE_FIELD_KEYWORDS)   │ │
│  │  • SecurityAudit (6 IPC 通道, 审计日志, 文件保护模式)            │ │
│  │  • 代码签名验证 (verifyAuthenticodeSignature)                    │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                       │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │  数据持久化                                                      │ │
│  │                                                                  │ │
│  │  • SQLite (bytecode 中可见 UPSERT 模式, sessions 表)             │ │
│  │  • agent.json / openclaw.json / openclaw.plugin.json             │ │
│  │  • 备份系统 (backupPath, BACKUP_SYSTEM_EVENT)                    │ │
│  │  • 配置管理 (ConfigService, ConfigProvider)                       │ │
│  └──────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────┘

                         外部通信
                            │
           ┌────────────────┼────────────────┐
           ▼                ▼                ▼
    ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
    │ IMA iframe   │ │ 微信登录     │ │ 企微 AI Bot  │
    │              │ │              │ │              │
    │ postMessage  │ │ wxLogin.js   │ │ wecom-aibot  │
    │ ima-jsapi-   │ │ open.weixin  │ │ sdk          │
    │ sdk          │ │ .qq.com      │ │              │
    │ 15 API方法   │ │              │ │              │
    └──────────────┘ └──────────────┘ └──────────────┘
```

---

## 五、信息提取统计

| 提取方式 | 文件 | 获取内容 |
|----------|------|----------|
| **直接阅读源码** | `package.json` | 20 个依赖，包名，版本，入口 |
| **直接阅读源码** | `out/main/index.cjs` | 入口模式：bytecode-loader + cjsc |
| **直接阅读源码** | `out/main/bytecode-loader.cjs` | V8 bytecode 加载机制 (73行完整逻辑) |
| **直接阅读源码** | `out/main/openclaw-bootstrap.mjs` | 看门狗机制，3个环境变量，2s心跳 |
| **直接阅读源码** | `out/preload/index.cjs` | 同 main，bytecode 包装入口 |
| **直接阅读源码** | `out/renderer/index.html` | Vue 3 + Vite, 2个外部 SDK, CSP |
| **直接阅读源码** | `out/renderer/sdk/ima-jsapi-sdk.js` | 15个 API, postMessage 通信, 30s超时 |
| **直接阅读源码** | `out/renderer/sdk/package.json` | SDK v1.1.2, 作者 xuankehuang |
| **Bytecode 字符串** | `out/main/index.cjsc` (3.9MB) | 90+ IPC 通道, 50+ RUM 事件, 40+ URL, 30+ OTel 属性, 4 代理路由 |
| **Bytecode 字符串** | `out/renderer/assets/*.js` (830KB) | UI 组件标识符 (Agent*, Chat*, IMA*, Config* 等) |
| **不可获取** | 所有 `.cjsc` 文件 | 业务逻辑代码，函数实现，控制流 |

---

## 六、不可获取的信息（代码层限制）

由于主进程代码被编译为 V8 Bytecode，以下信息 **无法从代码层直接获取**：

1. **具体的函数实现** — 只能看到函数名/标识符的字符串，看不到函数体
2. **控制流逻辑** — if/else/循环/条件判断无法从字符串还原
3. **数据结构定义** — TypeScript interface/type 编译后消失
4. **类型信息** — 编译产物中不包含类型
5. **注释和文档** — 编译时被剥离
6. **精确的调用关系** — 无法从字符串判断谁调用了谁
7. **状态机流转** — 只能看到状态名称，看不到转换条件
8. **算法细节** — 加密算法具体参数、备份策略具体实现等

以上分析 **仅基于代码中存在的字符串常量和可读源码**，未做任何推断或联想。
