# Electron 自动更新学习文档

> 本文档汇总 electron-builder + electron-updater 的完整知识体系，用于团队学习和升级方案决策。
>
> 参考来源：[掘金 - 从0到1实现企业级AppUpdater组件](https://juejin.cn/post/7535006257344577588) 及多源资料

---

## 目录

1. [两个核心工具的角色定位](#1-两个核心工具的角色定位)
2. [更新的完整流程](#2-更新的完整流程)
3. [更新源 Provider 选择](#3-更新源-provider-选择)
4. [electron-builder 核心配置](#4-electron-builder-核心配置)
5. [electron-updater 企业级实现](#5-electron-updater-企业级实现)
6. [latest.yml 详解](#6-latestyml-详解)
7. [关键配置项汇总](#7-关键配置项汇总)
8. [企业级方案要点](#8-企业级方案要点)
9. [常见问题 & 踩坑](#9-常见问题--踩坑)
10. [决策参考](#10-决策参考)

---

## 1. 两个核心工具的角色定位

| 工具 | 定位 | 安装位置 | 核心职责 |
|------|------|----------|----------|
| **electron-builder** | 打包工具 | `devDependencies` | 打包成 `.exe` / `.dmg` / `.AppImage`，**同时生成更新元数据文件**（`latest.yml`） |
| **electron-updater** | 更新库 | `dependencies` | 应用运行时检查更新、下载、安装新安装包 |

两者同属 electron-userland 项目，通过 `publish` 配置联动。**electron-builder 负责在打包时生成元数据，electron-updater 负责在运行时读取元数据判断是否有更新。**

---

## 2. 更新的完整流程

### 2.1 打包阶段

```
源码 + Electron 运行时 ──► electron-builder build
         │
         ├─► MyApp Setup 1.0.0.exe   (安装包)
         └─► latest.yml              (更新元数据)
                ├── version: 1.0.0
                ├── files[].url: 安装包文件名
                ├── files[].sha512: 校验值
                ├── files[].size: 文件大小
                └── releaseDate: 发布日期
```

### 2.2 运行时阶段

```
应用启动
  │
  ▼
autoUpdater.checkForUpdates()
  │
  ▼
GET https://your-server.com/latest.yml
  │
  ├─ 版本一致 ──► 无更新
  │
  └─ 远端版本更高 ──► update-available 事件
        │
        ▼
    downloadUpdate() ──► download-progress 事件（进度条）
        │
        ▼
    update-downloaded 事件
        │
        ▼
    quitAndInstall() ──► 应用重启，运行新版本
```

---

## 3. 更新源 Provider 选择

### 3.1 Generic（自建服务器）—— 企业最常用

```yaml
# electron-builder 配置
publish:
  provider: generic
  url: "https://your-server.com/updates"
```

**服务器需要存放：**
- 安装包文件：`MyApp Setup 1.2.0.exe`
- 元数据文件：`latest.yml`（打包时自动生成，需手动上传到对应目录）

**优点**：完全可控，适合企业内部
**缺点**：需要自己维护更新服务器和 CDN

### 3.2 GitHub Releases

```yaml
publish:
  provider: github
  owner: your-org
  repo: your-app
```

适合开源项目，更新元数据和安装包都作为 GitHub Release 附件。国内网络不稳定，企业场景不推荐。

### 3.3 S3 / 对象存储

```yaml
publish:
  provider: s3
  bucket: my-bucket
  region: us-east-1
```

把 `latest.yml` 和安装包上传到对象存储，通过 CDN 分发。国内可使用阿里云 OSS、腾讯云 COS 替代（需配合 Generic provider）。

### 3.4 对比

| Provider | 适用场景 | 运维成本 | 国内可用性 |
|----------|---------|---------|-----------|
| Generic | 企业内部 / 自有服务器 | 中（需维护服务器） | 好 |
| GitHub | 开源项目 | 低 | 差（网络不稳定） |
| S3 / 云存储 | 云服务用户 | 低-中 | 好（用国内 OSS 替代） |
| Snapcraft | Linux Snap 包 | 低 | — |

---

## 4. electron-builder 核心配置

### 4.1 完整配置示例

```yaml
# electron-builder.yml
appId: com.yourcompany.appname
productName: YourAppName

directories:
  output: dist
  buildResources: build

files:
  - "**/*"
  - "!**/*.ts"
  - "!*.code-workspace"
  - "!src/**/*"

win:
  target:
    - target: nsis
      arch:
        - x64

nsis:
  oneClick: false          # 关闭一键安装，允许用户选择安装路径
  perMachine: true         # 为所有用户安装
  allowElevation: true     # 请求管理员权限
  allowToChangeInstallationDirectory: true  # 允许自定义安装路径
  createDesktopShortcut: true
  createStartMenuShortcut: true
  shortcutName: YourAppName
  installerIcon: build/icon.ico
  uninstallerIcon: build/icon.ico

publish:
  provider: generic
  url: "https://your-server.com/updates"

# 代码签名（Windows 更新需要签名匹配）
win:
  sign: "./sign.js"
  certificateFile: "cert.pfx"
  certificatePassword: "your-password"
```

### 4.2 NSIS 关键配置说明

| 配置项 | 说明 | 推荐值 |
|--------|------|--------|
| `oneClick` | 是否一键安装 | `false`（企业场景让用户选路径） |
| `perMachine` | 是否为所有用户安装 | `true` |
| `allowElevation` | 请求管理员权限 | `true` |
| `allowToChangeInstallationDirectory` | 允许改安装路径 | `true`（当 `oneClick: false` 时生效） |
| `differentialPackage` | 是否生成增量包 | `false`（默认，全量更新更稳妥） |

---

## 5. electron-updater 企业级实现

### 5.1 主进程 UpdateController

```typescript
// updateController.ts
import { autoUpdater } from "electron-updater";
import { BrowserWindow, dialog, ipcMain } from "electron";

export class UpdateController {
  private mainWindow: BrowserWindow;

  constructor(mainWindow: BrowserWindow) {
    this.mainWindow = mainWindow;
    this.initAutoUpdater();
  }

  private initAutoUpdater() {
    // 关键配置
    autoUpdater.autoDownload = false;          // 先通知用户再下载，避免带宽浪费
    autoUpdater.autoInstallOnAppQuit = true;   // 退出时自动安装
    // autoUpdater.allowPrerelease = false;    // 不包含预发布版本（默认 false）

    this.bindEvents();
  }

  private bindEvents() {
    // 检查更新中
    autoUpdater.on("checking-for-update", () => {
      this.sendToRenderer("checking-for-update");
    });

    // 无可用更新
    autoUpdater.on("update-not-available", (info) => {
      this.sendToRenderer("update-not-available", info);
    });

    // 有可用更新
    autoUpdater.on("update-available", (info) => {
      this.sendToRenderer("update-available", {
        version: info.version,
        releaseNotes: info.releaseNotes,
      });
    });

    // 下载进度
    autoUpdater.on("download-progress", (progress) => {
      this.sendToRenderer("download-progress", {
        percent: Math.round(progress.percent),
        speed: progress.bytesPerSecond,
        transferred: progress.transferred,
        total: progress.total,
      });
    });

    // 下载完成
    autoUpdater.on("update-downloaded", (info) => {
      this.sendToRenderer("update-downloaded", info);
      // 弹窗提示用户重启安装
      dialog.showMessageBox(this.mainWindow, {
        type: "info",
        title: "更新已就绪",
        message: `新版本 v${info.version} 已下载完成`,
        detail: "重启应用后即可使用新版本",
        buttons: ["立即重启", "稍后"],
      }).then((result) => {
        if (result.response === 0) {
          autoUpdater.quitAndInstall();
        }
      });
    });

    // 更新错误
    autoUpdater.on("error", (err) => {
      this.sendToRenderer("update-error", err.message);
      console.error("Auto update error:", err);
    });
  }

  // 供渲染进程调用：开始检查更新
  public checkForUpdates() {
    autoUpdater.checkForUpdates();
  }

  // 下载更新
  public downloadUpdate() {
    autoUpdater.downloadUpdate();
  }

  // 重启并安装
  public quitAndInstall() {
    autoUpdater.quitAndInstall();
  }

  private sendToRenderer(channel: string, data?: unknown) {
    if (this.mainWindow && !this.mainWindow.isDestroyed()) {
      this.mainWindow.webContents.send("auto-update", {
        channel,
        data,
      });
    }
  }
}
```

### 5.2 主进程入口集成

```typescript
// main.ts
import { app, BrowserWindow, ipcMain } from "electron";
import * as path from "path";
import { UpdateController } from "./updateController";

let mainWindow: BrowserWindow;
let updateController: UpdateController;

function createWindow() {
  mainWindow = new BrowserWindow({
    width: 1200,
    height: 800,
    webPreferences: {
      nodeIntegration: false,
      contextIsolation: true,
      preload: path.join(__dirname, "preload.js"),
    },
  });

  // 初始化更新控制器
  updateController = new UpdateController(mainWindow);

  // 应用启动后延迟检查更新（避免阻塞启动）
  setTimeout(() => {
    updateController.checkForUpdates();
  }, 3000);
}

// IPC 处理渲染进程的更新请求
ipcMain.handle("check-for-updates", () => {
  updateController.checkForUpdates();
});

ipcMain.handle("download-update", () => {
  updateController.downloadUpdate();
});

ipcMain.handle("quit-and-install", () => {
  updateController.quitAndInstall();
});

app.whenReady().then(createWindow);
```

### 5.3 preload 安全通信

```typescript
// preload.ts
import { contextBridge, ipcRenderer } from "electron";

contextBridge.exposeInMainWorld("electronAPI", {
  checkForUpdates: () => ipcRenderer.invoke("check-for-updates"),
  downloadUpdate: () => ipcRenderer.invoke("download-update"),
  quitAndInstall: () => ipcRenderer.invoke("quit-and-install"),
  onUpdate: (callback: (message: any) => void) => {
    ipcRenderer.on("auto-update", (_event, message) => callback(message));
  },
});
```

### 5.4 渲染进程交互（React/Vue 通用）

```typescript
// 渲染进程中

// 监听主进程发来的更新事件
window.electronAPI.onUpdate((message) => {
  const { channel, data } = message;

  switch (channel) {
    case "checking-for-update":
      console.log("正在检查更新...");
      // 显示 loading 状态
      break;

    case "update-not-available":
      console.log("当前已是最新版本");
      // 提示用户无更新
      break;

    case "update-available":
      console.log(`发现新版本: ${data.version}`);
      // 显示更新弹窗，展示 releaseNotes
      // 用户点击确认后调用 window.electronAPI.downloadUpdate()
      break;

    case "download-progress":
      console.log(`下载进度: ${data.percent}%`);
      // 更新 UI 进度条
      break;

    case "update-downloaded":
      console.log("下载完成，准备重启");
      // 显示重启确认弹窗
      break;

    case "update-error":
      console.error("更新失败:", data);
      // 提示用户更新失败，可手动下载
      break;
  }
});

// 用户手动触发检查更新
function handleCheckForUpdates() {
  window.electronAPI.checkForUpdates();
}

// 用户确认下载
function handleConfirmDownload() {
  window.electronAPI.downloadUpdate();
}

// 用户确认重启
function handleConfirmRestart() {
  window.electronAPI.quitAndInstall();
}
```

---

## 6. latest.yml 详解

### 6.1 文件结构

```yaml
version: 1.2.0
files:
  - url: MyApp-Setup-1.2.0.exe
    sha512: aBcDeFgHiJkLmNoPqRsTuVwXyZ0123456789...
    size: 123456789
path: MyApp-Setup-1.2.0.exe
sha512: aBcDeFgHiJkLmNoPqRsTuVwXyZ0123456789...
releaseDate: "2026-05-12T10:00:00.000Z"
```

### 6.2 各字段含义

| 字段 | 说明 |
|------|------|
| `version` | 新版本号（必须高于当前版本，否则不触发更新） |
| `files[].url` | 安装包文件名 |
| `files[].sha512` | 安装包的 SHA-512 校验值，下载后自动校验确保完整性 |
| `files[].size` | 安装包大小（字节） |
| `path` | 安装包的相对路径 |
| `sha512` | 元数据文件本身的校验值 |
| `releaseDate` | 发布日期（ISO 8601 格式） |

> **注意**：`latest.yml` 由 electron-builder 在打包时自动生成在 `dist/` 目录中，需要将其和安装包一起上传到更新服务器。

---

## 7. 关键配置项汇总

### 7.1 autoUpdater 行为控制

| 属性 | 默认值 | 说明 |
|------|--------|------|
| `autoDownload` | `true` | 发现更新后是否自动下载。企业建议设为 `false`，让用户先确认再下载 |
| `autoInstallOnAppQuit` | `true` | 下载完成后，用户退出应用时是否自动安装 |
| `allowPrerelease` | `false` | 是否包含预发布版本（如 `1.0.0-alpha.1`） |
| `disableWebInstaller` | `false` | 是否禁用 Web 安装模式 |

### 7.2 事件列表

| 事件 | 触发时机 | 携带数据 |
|------|---------|---------|
| `checking-for-update` | 开始检查更新时 | — |
| `update-available` | 发现新版本 | `{version, releaseNotes, releaseDate, releaseName}` |
| `update-not-available` | 已是最新版本 | `{version}` |
| `download-progress` | 下载过程中 | `{percent, bytesPerSecond, transferred, total}` |
| `update-downloaded` | 下载完成 | `{version, releaseNotes}` |
| `error` | 发生错误 | `Error` 对象 |
| `update-cancelled` | 用户取消更新 | — |

---

## 8. 企业级方案要点

### 8.1 推荐更新策略

```
1. 应用启动 3 秒后 ──► 静默检查更新
2. 有新版本 ──► 弹窗告知用户，显示更新日志（releaseNotes）
3. 用户确认 ──► 后台下载，显示进度条
4. 下载完成 ──► 提示用户重启应用
5. 用户拒绝 ──► 不强制，下次启动再次检查
```

### 8.2 灰度发布（高级）

electron-updater 本身**不支持**灰度发布，但可通过以下方式实现：

- **方案 A**：服务器端根据用户 ID / 分组返回不同的 `latest.yml`
- **方案 B**：使用 `autoUpdater.setFeedURL()` 动态切换更新源
- **方案 C**：自定义更新服务器，在 API 层控制灰度比例

### 8.3 强制更新

对于安全补丁等场景需要强制更新：

```typescript
autoUpdater.on("update-available", (info) => {
  // 检查 releaseNotes 中是否包含 [FORCE] 标记
  const isForceUpdate = info.releaseNotes?.includes("[FORCE]");

  if (isForceUpdate) {
    dialog.showMessageBox({
      type: "warning",
      title: "重要更新",
      message: "检测到重要安全更新，必须重启应用才能继续使用",
      buttons: ["立即更新"],
      cancelable: false,  // 不允许取消
    }).then(() => {
      autoUpdater.downloadUpdate();
    });
  } else {
    // 普通更新，用户可自选
    // ... 常规流程
  }
});
```

### 8.4 离线环境适配

企业内部可能有离线用户：

```typescript
autoUpdater.on("error", (err) => {
  if (err.message.includes("net::ERR_INTERNET_DISCONNECTED")) {
    // 网络不可用，静默失败或下次启动再试
    console.log("网络不可用，跳过更新检查");
  } else if (err.message.includes("404")) {
    // 服务器上无更新文件
    console.log("服务器上无更新文件");
  } else {
    // 其他错误
    console.error("更新错误:", err);
  }
});
```

### 8.5 定时检查更新

除了启动时检查，还可以在运行期间定时检查：

```typescript
// 每 4 小时检查一次
setInterval(() => {
  updateController.checkForUpdates();
}, 4 * 60 * 60 * 1000);
```

---

## 9. 常见问题 & 踩坑

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| 检查更新返回 `null` | 未配置 `publish` 或 `latest.yml` 无法访问 | 确认更新源 URL 可访问，且 `latest.yml` 存在 |
| 版本号没变化 | 远端版本 <= 本地版本 | 确保 `latest.yml` 中的 `version` 严格高于当前版本 |
| Windows 安装时报签名错误 | 未配置代码签名或签名不匹配 | 配置 `certificateFile`，或使用测试证书 |
| 下载进度卡住 | CDN 或网络问题 | 设置超时，增加重试机制 |
| 热更新（不重启） | electron-updater **不支持**热更新 | 需要自行实现或使用 `electron-hot-reload` 等方案 |
| macOS Gatekeeper 拦截 | 未做 Apple Developer ID 签名 | 使用 Developer ID 证书签名，并在系统设置中允许 |
| 更新后配置丢失 | 安装路径改变 | 配置数据使用 `app.getPath('userData')` 存储，与安装路径无关 |
| 开发环境触发更新检查 | 开发模式下没有 `publish` 配置 | 通过 `app.isPackaged` 判断，仅在生产环境检查更新 |

### 开发环境跳过更新

```typescript
if (!app.isPackaged) {
  console.log("开发模式，跳过更新检查");
} else {
  updateController.checkForUpdates();
}
```

---

## 10. 决策参考

基于以上分析，针对企业场景推荐的方案组合：

```
打包工具:        electron-builder
更新库:          electron-updater
更新源:          Generic（自建服务器 / Nginx + CDN）
安装方式:        NSIS（非一键安装，允许选路径）
更新体验:        检查 → 弹窗 → 下载 → 提示重启（不强制）
签名:            Windows 代码签名证书
增量更新:        初期不开启（全量更新更稳定）
灰度发布:        服务器端控制（可选，二期实现）
定时检查:        启动时 + 运行期间每 4 小时
```

### 实施步骤

1. **Phase 1**：基础自动更新（检查 → 下载 → 安装）
2. **Phase 2**：用户体验优化（更新日志展示、进度条、强制更新标记）
3. **Phase 3**：灰度发布 + 增量更新（按需）
