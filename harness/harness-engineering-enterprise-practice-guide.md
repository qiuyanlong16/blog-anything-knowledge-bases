# Harness Engineering 企业实战指南

> 从概念到落地，一套可快速应用的 AI 智能体驾驭工程手册
>
> 参考视频：[最近爆火的Harness Engineering 到底是啥？一期讲透！](https://www.bilibili.com/video/BV1Zk9FBwELs/)

---

## 目录

1. [什么是 Harness Engineering](#1-什么是-harness-engineering)
2. [核心公式：Agent = Model + Harness](#2-核心公式agent--model--harness)
3. [六层架构总览](#3-六层架构总览)
4. [L1：信息边界层](#4-l1信息边界层)
5. [L2：工具与执行层](#5-l2工具与执行层)
6. [L3：工作流编排层](#6-l3工作流编排层)
7. [L4：记忆与上下文层](#7-l4记忆与上下文层)
8. [L5：反馈与评估层](#8-l5反馈与评估层)
9. [L6：约束与护栏层](#9-l6约束与护栏层)
10. [从简单到复杂：渐进式学习路径](#10-从简单到复杂渐进式学习路径)
11. [企业实战场景](#11-企业实战场景)
12. [常见问题与排错](#12-常见问题与排错)
13. [参考资源](#13-参考资源)

---

## 1. 什么是 Harness Engineering

Harness Engineering（驾驭工程 / 缰绳工程）是 2025-2026 年 AI Agent 开发领域兴起的一门新学科。

**一句话解释**：为 AI 智能体设计环境、约束和反馈回路，让它从"能聊天"变成"能干活"。

### 起源故事

视频开头讲了一个真实案例：一个团队用了最强的模型，迭代了上百版提示词，但 Agent 的任务成功率始终低于 70%。后来他们不再纠结于提示词本身，转而构建了一套**结构化的驾驭系统**，成功率大幅提升。

这个故事揭示了一个核心认知：

> **问题不在模型，而在你如何"驾驭"模型。**

### 为什么现在火了

- 2025 年底 OpenAI 发布 [Harness Engineering 官方文章](https://openai.com/index/harness-engineering/)，正式提出这一概念
- Anthropic 基于 Harness Design 理论框架构建多智能体编排
- OpenAI Codex CLI 用 Harness 方法构建了超过 100 万行代码
- 行业共识从"更好的提示词"转向"更好的驾驭系统"

---

## 2. 核心公式：Agent = Model + Harness

```
Agent = Model + Harness
```

| 组件 | 说明 | 类比 |
|------|------|------|
| **Model** | 大语言模型（GPT、Claude 等）提供原始智能 | 马的体力和智力 |
| **Harness** | 围绕模型构建的环境、工具、约束、反馈系统 | 缰绳、马鞍、跑道、训练方法 |

**关键认知**：Harness 架构可以让同一个模型的性能提升 **6 倍**。这不是夸张，多个团队已验证过。

---

## 3. 六层架构总览

Harness Engineering 的核心是**六层架构**，从内到外逐层构建：

```
┌─────────────────────────────────────┐
│  L6  约束与护栏 (Constraints)       │  ← 安全边界、权限控制、领域隔离
├─────────────────────────────────────┤
│  L5  反馈与评估 (Feedback)          │  ← 测试、CI/CD、代码审查、自修正
├─────────────────────────────────────┤
│  L4  记忆与上下文 (Memory)          │  ← CLAUDE.md、AGENTS.md、知识库
├─────────────────────────────────────┤
│  L3  工作流编排 (Orchestration)     │  ← SOP、任务分解、多智能体协调
├─────────────────────────────────────┤
│  L2  工具与执行 (Tools)             │  ← 文件系统、Bash 沙箱、MCP、搜索
├─────────────────────────────────────┤
│  L1  信息边界 (Information)         │  ← 上下文范围、项目结构、文档
└─────────────────────────────────────┘
```

**构建原则**：从 L1 开始逐层向外搭建，每一层都建立在前一层的基础上。

---

## 4. L1 信息边界层

### 定义

控制 Agent **能看到什么信息**。就像给 Agent 一张项目地图，告诉它"这里有什么，别的地方别去"。

### 为什么重要

没有信息边界的 Agent 就像把人扔进图书馆但不给目录——它会浪费时间在不相关的文件上，或者遗漏关键上下文。

### 实操方法

#### 4.1 项目上下文文件

在项目根目录创建 `CLAUDE.md`（或 `AGENTS.md`），告诉 Agent 项目的基本结构：

```markdown
# Project Context

This repo is a personal knowledge base — a place for scattered notes.

## How It Works
- Each top-level folder = one topic/module
- Notes are written in Markdown
- Static demo pages go inside a docs/ subfolder

## How to Maintain This Repo
1. Check if a relevant topic folder already exists
2. Keep filenames descriptive
3. Commit and push — this repo is personal, no PR process needed
```

这就是一个典型的信息边界：用 20 行文字定义了项目的"世界观"。

#### 4.2 模块化上下文

对于大型项目，每个模块放自己的上下文文件：

```
project/
├── CLAUDE.md              # 全局项目上下文
├── src/
│   ├── api/
│   │   └── CLAUDE.md      # API 模块上下文
│   ├── frontend/
│   │   └── CLAUDE.md      # 前端模块上下文
│   └── database/
│       └── CLAUDE.md      # 数据库模块上下文
```

#### 4.3 文件过滤

告诉 Agent **忽略什么**同样重要：

```markdown
## Files to Ignore
- node_modules/ — generated, do not modify
- dist/ — build output, never edit
- *.lock — auto-generated dependency files
```

### 企业应用

| 场景 | 信息边界内容 |
|------|-------------|
| 微服务项目 | 每个服务一个 AGENTS.md，描述服务职责、API 契约、依赖关系 |
| 遗留系统改造 | 标注"禁止修改"的模块、已知技术债、迁移计划中的阶段 |
| 多团队协作 | 明确每个团队的代码所有权边界，防止越界修改 |

---

## 5. L2 工具与执行层

### 定义

Agent 能使用哪些**工具和执行环境**。信息边界告诉 Agent"有什么"，工具层告诉 Agent"能做什么"。

### 核心工具集

```
┌──────────────────────────────────────────┐
│              工具执行环境                  │
├────────────┬──────────┬────────┬─────────┤
│ 文件系统    │ Bash沙箱  │ Web搜索 │  MCP   │
│ 读写文件    │ 命令执行  │ 信息检索 │ 外部API │
└────────────┴──────────┴────────┴─────────┘
```

#### 5.1 文件系统

Agent 需要结构化的文件访问权限：

```
project/
├── src/           # 源码 — Agent 可以修改
├── tests/         # 测试 — Agent 应该更新
├── docs/          # 文档 — Agent 可以补充
├── .claude/       # Agent 配置 — Agent 可以调整
└── secrets.env    # 密钥 — Agent 只能读取
```

#### 5.2 Bash 沙箱

安全的命令执行环境：

```markdown
# 允许的操作
- npm install, npm run build, npm test
- git status, git diff, git log
- python -c "import ..."
- curl http://localhost:3000/health

# 禁止的操作
- rm -rf / 或任何破坏性操作
- 修改 CI/CD 管道配置
- 推送代码到远程（除非明确授权）
```

#### 5.3 MCP（Model Context Protocol）

MCP 是 Agent 连接外部系统的标准化协议：

```
Agent ←→ MCP Server ←→ 外部系统
                   ├── 数据库
                   ├── GitHub API
                   ├── Jira
                   ├── Slack
                   └── 自定义内部 API
```

示例 MCP 配置：

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": { "GITHUB_PERSONAL_ACCESS_TOKEN": "..." }
    },
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/workspace"]
    }
  }
}
```

### 企业应用

| 场景 | L2 配置 |
|------|---------|
| CI/CD 集成 | 通过 MCP 连接 Jenkins/GitHub Actions，Agent 可以触发构建、查看状态 |
| 数据库操作 | MCP 连接只读数据库副本，Agent 能查询但不能直接修改生产库 |
| 内部工具对接 | 为内部 API 编写 MCP Server，Agent 可以操作工单、查看监控 |

---

## 6. L3 工作流编排层

### 定义

Agent **如何工作**——任务分解、执行步骤、多智能体协调。这是从"单步操作"到"流程驱动"的关键层。

### 核心概念

#### 6.1 SOP（标准操作流程）

为常见任务定义标准流程：

```
任务：添加新功能
  1. 阅读 CLAUDE.md 了解项目结构
  2. 探索相关模块的代码
  3. 设计实现方案（列出步骤）
  4. 编写代码
  5. 运行测试
  6. 更新文档
  7. 提交代码
```

#### 6.2 任务分解

大任务拆成可独立执行的小步骤：

```
用户请求："实现用户登录功能"

Agent 分解：
  Step 1: 创建登录 API 端点
  Step 2: 实现 JWT 生成逻辑
  Step 3: 添加密码验证中间件
  Step 4: 编写登录页面的前端组件
  Step 5: 编写集成测试
  Step 6: 更新 API 文档
```

#### 6.3 多智能体编排

复杂任务用多个专业 Agent 协作：

```
┌─────────────┐
│  协调者 Agent │
└──────┬──────┘
       │ 分解任务
  ┌────┼────────┐
  ▼    ▼        ▼
┌────┐┌────┐┌────────┐
│前端││后端││测试Agent│
│Agent││Agent│        │
└────┘└────┘└────────┘
  │     │        │
  └─────┴────────┘
        ▼
   合并结果 → 审查 → 输出
```

#### 6.4 Hooks 机制

在关键节点触发预定义行为：

```json
{
  "hooks": {
    "PreToolUse": [
      { "matcher": "Bash", "hook": "validate-command" }
    ],
    "PostToolUse": [
      { "matcher": "Edit", "hook": "run-linter" }
    ],
    "OnUserMessage": [
      { "hook": "load-context" }
    ]
  }
}
```

### 企业应用

| 场景 | L3 编排策略 |
|------|------------|
| 代码审查自动化 | Agent 收到 PR → 运行静态分析 → 检查安全漏洞 → 生成审查报告 |
| Bug 修复流程 | Agent 读取 Issue → 复现 Bug → 定位代码 → 修复 → 运行测试 → 生成 PR |
| 功能开发 | 协调者 Agent 分解任务 → 前端/后端/测试 Agent 并行执行 → 合并验证 |

---

## 7. L4 记忆与上下文层

### 定义

Agent **如何记住**之前的工作和学习成果。没有记忆的 Agent 每次都是全新的开始。

### 记忆类型

```
┌────────────────────────────────────────┐
│              记忆系统                    │
├──────────────┬─────────────────────────┤
│  短期记忆     │  长期记忆                │
│ (会话上下文)  │ (持久化存储)             │
│              │                          │
│ · 对话历史    │ · 项目知识 (CLAUDE.md)   │
│ · 当前任务    │ · 用户偏好 (memory/)     │
│ · 工具调用    │ · 反馈记录              │
│ · 中间状态    │ · 经验教训              │
└──────────────┴─────────────────────────┘
```

### 7.1 短期记忆

会话内的上下文管理：

- **上下文窗口管理**：在有限窗口内保留最关键信息
- **压缩策略**：将冗长对话压缩为摘要
- **主动遗忘**：丢弃不相关的中间状态

### 7.2 长期记忆

持久化的知识存储：

```
.claude/
├── CLAUDE.md              # 项目级持久记忆
├── settings.json          # Agent 行为配置
├── commands/              # 自定义命令
│   ├── review.md          # /review 命令
│   └── deploy.md          # /deploy 命令
└── memory/                # 语义记忆
    ├── MEMORY.md          # 记忆索引
    ├── user_role.md       # 用户信息
    ├── feedback_*.md      # 反馈记录
    ├── project_*.md       # 项目状态
    └── reference_*.md     # 外部资源引用
```

记忆文件示例：

```markdown
---
name: database preference
description: 集成测试必须使用真实数据库
type: feedback
---

集成测试必须使用真实数据库，不能使用 mock。
原因：之前的 mock/prod 差异掩盖了迁移失败的 bug。
适用场景：所有涉及数据库的代码变更。
```

### 7.3 记忆的读写策略

```
写记忆时机：
  - 用户明确要求"记住这个"
  - 学到新的用户偏好
  - 发现重要的项目决策原因
  - 记录失败的排查经验

读记忆时机：
  - 会话开始时加载 MEMORY.md
  - 收到相关任务时检索对应记忆
  - 做决策前检查反馈记录中的约束
```

### 企业应用

| 场景 | L4 实现方式 |
|------|------------|
| 新人 Onboarding | 将团队约定、代码规范、常见坑写入 CLAUDE.md，新成员使用 Agent 自动获得知识 |
| 跨会话知识积累 | Agent 从历史会话中提取经验教训，持久化到 memory 文件夹 |
| 团队共享经验 | 建立团队共享的记忆库，记录架构决策、技术选型原因 |

---

## 8. L5 反馈与评估层

### 定义

Agent **如何知道自己做得对不对**。这是从"盲盒输出"到"可控质量"的关键。

### 反馈回路

```
┌─────────────────────────────────────┐
│          反馈与评估循环              │
│                                     │
│  任务输入 → Agent 执行 → 评估       │
│                ▲          │         │
│                │          ▼         │
│           自修正 ←─── 反馈结果      │
└─────────────────────────────────────┘
```

### 8.1 自动化测试

最直接的反馈机制：

```
Agent 修改代码
  → 运行单元测试
  → 如果失败：分析错误 → 修复 → 重新测试
  → 如果通过：运行集成测试
  → 如果失败：分析错误 → 修复 → 重新测试
  → 全部通过：提交
```

### 8.2 代码审查循环

```
Agent 完成修改
  → 触发 /review 命令
  → 审查 Agent 检查：
    ├── 是否符合项目约定 (CLAUDE.md)
    ├── 是否有安全漏洞
    ├── 是否引入不必要的复杂度
    └── 测试是否覆盖
  → 生成审查报告
  → 如果有问题：返回修改
  → 如果没问题：标记完成
```

### 8.3 CI/CD 集成

```yaml
# .github/workflows/agent-review.yml
name: Agent Code Review
on: pull_request

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run Agent Review
        run: |
          npm install
          npm run lint
          npm test
          npm run build
```

### 8.4 可观测性

```
监控指标：
  - Agent 任务成功率
  - 平均修复尝试次数
  - 测试通过率
  - 代码审查通过率
  - 上下文命中率（是否读对了文件）
```

### 企业应用

| 场景 | L5 实现 |
|------|---------|
| 生产事故响应 | Agent 修复 → 自动化回归测试 → 监控指标验证 → 确认修复效果 |
| 合规审查 | Agent 生成代码 → 安全扫描 → 合规检查 → 人工审批 |
| 质量门禁 | 只有通过测试、lint、安全检查的代码才能合并 |

---

## 9. L6 约束与护栏层

### 定义

Agent **不能做什么**。这是最后一道防线，确保安全性和可控性。

### 约束类型

```
┌────────────────────────────────────┐
│          约束与护栏                 │
├──────────┬───────────┬─────────────┤
│ 权限约束  │ 领域约束   │ 行为约束    │
│          │           │             │
│ 能读不能改│ 只负责前端 │ 不跳过测试  │
│ 不能删文件│ 不碰后端   │ 不推送到主分支│
│ 不能执行rm│ 不碰数据库 │ 不发送消息  │
└──────────┴───────────┴─────────────┘
```

### 9.1 权限控制

```json
{
  "permissions": {
    "Read": "auto",
    "Edit": "auto",
    "Bash": "prompt",
    "Bash:*": "deny",
    "Bash:npm install": "auto",
    "Bash:npm test": "auto",
    "Bash:git commit": "auto",
    "Bash:git push": "deny",
    "Bash:rm *": "deny",
    "Bash:git reset *": "deny"
  }
}
```

### 9.2 领域隔离

```markdown
# 领域约束示例

## 允许修改的范围
- src/frontend/ — 前端代码
- src/styles/ — 样式文件
- public/ — 静态资源

## 禁止修改的范围
- src/backend/ — 后端代码（由后端 Agent 负责）
- src/database/ — 数据库迁移（由 DBA Agent 负责）
- infra/ — 基础设施配置（由运维 Agent 负责）
```

### 9.3 行为规则

```markdown
# Agent 行为约束

## 提交规则
- 提交信息必须遵循 Conventional Commits
- 每个提交只包含一个逻辑变更
- 不要在提交信息中引用内部工具调用

## 代码规则
- 不要引入新的依赖（除非用户明确要求）
- 优先修改现有代码，不要创建新文件（除非必要）
- 不要添加注释，除非逻辑非常隐蔽

## 安全规则
- 不要在代码中硬编码密钥
- 不要提交 .env 文件
- 不要在日志中输出敏感信息
```

### 企业应用

| 场景 | L6 约束 |
|------|---------|
| 多租户系统 | Agent 只能访问其所属租户的代码和数据 |
| 合规要求 | 金融/医疗行业：Agent 操作必须留下审计日志 |
| 权限分级 | 初级 Agent 只能修改非核心模块，核心模块需要高级 Agent 或人工审批 |

---

## 10. 从简单到复杂：渐进式学习路径

### Level 1：单 Agent 单文件修改（入门）

**目标**：让 Agent 正确修改一个文件

**步骤**：
1. 创建 `CLAUDE.md`，描述项目基本信息
2. 给一个简单任务，比如"把这个函数的命名改为 snake_case"
3. 观察 Agent 是否正确找到并修改文件

**检查点**：
- [ ] Agent 能正确定位文件
- [ ] Agent 的修改不引入 Bug
- [ ] Agent 的修改不破坏其他功能

### Level 2：单 Agent 多文件协调（L1+L2）

**目标**：让 Agent 在多个文件间正确修改

**步骤**：
1. 给一个需要同时修改前端和后端的任务
2. 在 CLAUDE.md 中添加模块依赖关系
3. 让 Agent 运行测试验证修改

**检查点**：
- [ ] Agent 理解文件间的依赖关系
- [ ] Agent 按照正确顺序修改（先接口定义，再实现）
- [ ] Agent 运行了相关测试

### Level 3：工作流编排（L1+L2+L3）

**目标**：定义标准流程，让 Agent 按 SOP 执行

**步骤**：
1. 为常见任务类型编写 SOP 文档
2. 配置 Hooks，在关键节点自动检查
3. 让 Agent 按照流程执行复杂任务

**检查点**：
- [ ] Agent 能分解大任务
- [ ] Agent 按照定义的流程执行
- [ ] Hooks 在正确的节点触发

### Level 4：记忆与知识积累（L1-L4）

**目标**：让 Agent 从历史经验中学习

**步骤**：
1. 设置 memory 目录和 MEMORY.md
2. 记录用户的反馈和偏好
3. 验证 Agent 在新会话中应用历史知识

**检查点**：
- [ ] Agent 不会重复犯已知的错误
- [ ] Agent 应用了用户的偏好
- [ ] 记忆不会过期或矛盾

### Level 5：自动化反馈（L1-L5）

**目标**：让 Agent 自我验证和修正

**步骤**：
1. 配置自动化测试和 lint
2. 让 Agent 在修改后自动运行验证
3. 测试失败时让 Agent 自动修复

**检查点**：
- [ ] Agent 能识别测试失败
- [ ] Agent 能定位失败原因
- [ ] Agent 能自动修复并重新测试

### Level 6：完整 Harness（L1-L6）

**目标**：所有层级协同工作的生产级系统

**步骤**：
1. 部署完整的六层架构
2. 配置权限约束和安全护栏
3. 接入 CI/CD 和监控系统
4. 在真实生产环境中验证

**检查点**：
- [ ] 所有层级都正确配置
- [ ] 约束不会被绕过
- [ ] 异常情况有降级策略
- [ ] 可观测性指标正常

### 学习时间参考

| 阶段 | 预计时间 | 产出 |
|------|---------|------|
| Level 1 | 30 分钟 | 第一个成功的 Agent 修改 |
| Level 2 | 1-2 小时 | 多文件协调修改 |
| Level 3 | 半天 | 标准工作流 SOP |
| Level 4 | 1 天 | 记忆系统上线 |
| Level 5 | 1-2 天 | 自动化反馈循环 |
| Level 6 | 1 周 | 生产级 Harness 系统 |

---

## 11. 企业实战场景

### 场景一：日常开发辅助

**背景**：开发团队 5 人，日常有大量 CRUD 功能和 API 端点需要实现。

**Harness 配置**：

```
L1: CLAUDE.md 包含项目架构、API 规范、数据库约定
L2: 文件系统读写 + npm 工具 + 本地测试环境
L3: SOP = "读文档 → 设计 → 编码 → 测试 → 提交"
L4: 记忆团队编码约定和历史踩坑记录
L5: 每次修改后运行单元测试 + lint + 类型检查
L6: 禁止修改基础设施配置，禁止推送主分支
```

**效果**：
- 简单功能开发时间从 2 小时缩短到 20 分钟
- Agent 成功率从 60% 提升到 95%
- 开发者只需要审查和微调，不再从零编写

### 场景二：遗留系统改造

**背景**：一个 5 年历史的后端系统，文档缺失，技术债积累严重。

**Harness 配置**：

```
L1: Agent 先做"代码考古"——阅读代码生成文档
    CLAUDE.md 记录已知的技术债和模块关系
L2: 只读模式探索 + 沙箱中运行测试
L3: SOP = "分析现有代码 → 生成文档 → 提出改造方案 → 小步重构"
L4: 记忆每个模块的实际行为（可能与文档不符）
L5: 每次重构后运行回归测试
L6: 严格禁止修改生产配置，所有变更需要 PR
```

**效果**：
- 2 周内自动生成了缺失的模块文档
- 识别出 15 个未使用的代码段
- 渐进式重构，每次改动都有测试保障

### 场景三：多智能体协作开发

**背景**：一个全栈功能需要前端、后端、测试、文档同时开发。

**Harness 架构**：

```
                    ┌─────────────────┐
                    │   协调者 Agent    │
                    │  (L3 编排层)     │
                    └────────┬────────┘
                             │ 分解任务
              ┌──────┬───────┼───────┬──────┐
              ▼      ▼       ▼       ▼      ▼
           ┌────┐┌────┐┌────┐┌────┐┌──────┐
           │前端││后端││测试││文档││DB迁移│
           │Agent││Agent││Agent││Agent││Agent │
           └────┘└────┘└────┘└────┘└──────┘
              │      │       │       │      │
              └──────┴───────┴───────┴──────┘
                             ▼
                    ┌─────────────────┐
                    │   审查 Agent     │
                    │  (L5 反馈层)     │
                    └────────┬────────┘
                             │ 通过？
                       ┌─────┴─────┐
                       ▼           ▼
                    合并 PR    返回修改
```

**各 Agent 的 Harness 配置**：

| Agent | L1 信息 | L2 工具 | L6 约束 |
|-------|---------|---------|---------|
| 前端 Agent | 组件库规范、样式约定 | 文件系统 + npm + 浏览器 | 不能改后端代码 |
| 后端 Agent | API 规范、数据模型 | 文件系统 + 数据库 | 不能改前端代码 |
| 测试 Agent | 测试框架、覆盖率要求 | 文件系统 + 测试运行器 | 只能改测试文件 |
| 文档 Agent | 文档格式、API 文档规范 | 文件系统 | 只读源码，只写文档 |
| DB 迁移 Agent | 数据库架构、迁移规范 | 数据库连接 | 只在迁移文件中修改 |

### 场景四：Code Review 自动化

**背景**：PR 量大，人工审查不过来，需要 Agent 自动做第一轮审查。

**Harness 配置**：

```
L1: CLAUDE.md 定义代码规范、架构约定
L2: git diff 读取变更 + 静态分析工具
L3: SOP = "读取 PR → 分析变更 → 检查规范 → 检查安全 → 生成报告"
L4: 记忆团队的代码审查偏好和历史问题
L5: 生成审查报告，标记需要人工关注的点
L6: 只能评论，不能直接合并或关闭 PR
```

**审查报告示例**：

```markdown
## Agent Review Report

### ✅ Passed
- 代码风格符合 ESLint 规则
- 单元测试覆盖率 > 80%
- 无已知的安全漏洞

### ⚠️ Needs Attention
- 在 user-service.ts:42 处，SQL 查询使用了字符串拼接，
  建议改用参数化查询（SQL 注入风险）
- 新增了 3 个依赖包，请确认是否需要
- 修改了认证中间件，建议安全负责人复核

### ℹ️ Info
- 本次变更涉及 5 个文件，+234/-56 行
- 影响模块：user-service, auth-middleware
```

---

## 12. 常见问题与排错

### Q1: Agent 总是忽略我的约束怎么办？

**排查步骤**：
1. 检查 CLAUDE.md 是否在正确的位置
2. 约束描述是否清晰明确（避免"尽量"、"最好"等模糊用词）
3. 在 L6 层面添加硬约束（权限配置），而非仅依赖文本描述

**常见原因**：
- 约束写在 Agent 不会读取的文件中
- 约束太长，被上下文压缩丢弃
- 约束与 Agent 的工具权限冲突

### Q2: Agent 的修改通过了测试但有 Bug 怎么办？

**说明**：这是 L5 反馈层不够强的信号。

**改进方向**：
1. 补充缺失的测试用例
2. 添加集成测试覆盖场景
3. 引入静态分析工具（eslint、sonarqube）
4. 在 CLAUDE.md 中记录常见 Bug 模式

### Q3: 多 Agent 协作时互相覆盖修改怎么办？

**解决方案**：
1. L6 层面：严格划分每个 Agent 的修改范围
2. L3 层面：协调者 Agent 控制执行顺序，避免并行冲突
3. L2 层面：使用 git 分支隔离，最后统一合并

### Q4: Agent 的"记忆"越来越多，影响性能怎么办？

**优化策略**：
1. 定期清理过期记忆（超过 3 个月未引用的）
2. 合并相似的记忆条目
3. 分层存储：高频记忆放 CLAUDE.md（每次加载），低频记忆放 memory/ 目录（按需加载）
4. 设置 MEMORY.md 行数限制（建议 200 行以内）

### Q5: 如何衡量 Harness 的效果？

**关键指标**：

| 指标 | 说明 | 目标值 |
|------|------|--------|
| 任务成功率 | Agent 一次完成正确的比例 | > 90% |
| 平均修复次数 | 需要自修正的次数 | < 2 次 |
| 上下文准确率 | 读对文件和代码的比例 | > 95% |
| 约束遵守率 | 不违反 L6 约束的比例 | 100% |
| 知识积累速度 | 新增有效记忆/周 | > 3 条 |

---

## 13. 参考资源

### 核心参考

- [最近爆火的Harness Engineering 到底是啥？一期讲透！（B站视频）](https://www.bilibili.com/video/BV1Zk9FBwELs/)
- [Harness Engineering - OpenAI 官方文章](https://openai.com/index/harness-engineering/)
- [一文搞懂Harness Engineering：六层架构、上下文管理与一线团队实战](https://javaguide.cn/ai/agent/harness-engineering.html)
- [The Six-Layer Architecture of Harness Engineering](https://docs.bswen.com/blog/2026-04-21-six-layer-harness-architecture/)
- [Harness Engineering 最佳实践：长运行多智能体的框架设计](https://zhuanlan.zhihu.com/p/2021965663103665310)
- [一文讲透如何构建Harness——六大组件全解析](https://cloud.tencent.com/developer/article/2648873)
- [Harness的六层架构学习思考 - LINUX DO](https://linux.do/t/topic/1923559)

### 延伸阅读

- [Harness Engineering 完全指南（2026）](https://www.nxcode.io/resources/news/harness-engineering-complete-guide-ai-agent-codex-2026)
- [AI Agent Harness 架构解析：7 种生产模式](https://medium.com/@wasowski.jarek/ai-agent-harness-architecture-7-patterns-that-control-autonomous-agents-in-production-d07a94a9cdcd)
- [Agent Harness 六架构模式](https://gerl.dev/blog/agent-harness-taxonomy)
- [智能体Harness工程指南 - OpenClaw](https://yeasy.gitbook.io/harness_engineering_guide)
- [Harness Engineering for AI Coding Agents](https://www.augmentcode.com/guides/harness-engineering-ai-coding-agents)
- [Modern Agent Harness Blueprint 2026](https://gist.github.com/amazingvince/52158d00fb8b3ba1b8476bc62bb562e3)

---

> **总结**：Harness Engineering 的核心思想是——**不要纠结于让模型更聪明，而是构建一个让模型能可靠工作的系统**。六层架构从信息边界到约束护栏，每一层都在增加 Agent 的可靠性。从 Level 1 开始，逐层构建，你会看到 Agent 的能力呈指数级增长。
