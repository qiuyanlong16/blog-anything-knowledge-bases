# Claude Code 记忆系统完全指南：从入门到企业级实践

> **摘要**：Claude Code 拥有两套互补的记忆系统——CLAUDE.md（手动指令）和 Auto Memory（自动学习记忆）。本文将从零基础出发，深入剖析记忆系统的架构原理、文件结构、工作流程，并结合企业开发场景给出最佳实践。

---

## 一、为什么 Claude Code 需要记忆系统？

想象一下这样的场景：

你让 Claude 帮你写代码，它第一次用了你不喜欢的格式。你纠正了它："以后用 snake_case 命名变量。" 第二天你开了新会话，它又忘了，又用了 camelCase。你不得不重复解释同样的偏好——**这种体验就像每天都在教一个"失忆"的实习生**。

Claude Code 的记忆系统就是为了解决这个问题而设计的。它让 Claude 能够：

- **跨会话记住**你的编码风格、技术偏好和工作习惯
- **跨会话记住**项目的架构决策、技术栈和业务背景
- **自动学习**从你的纠正和反馈中积累知识
- **渐进式进化**随着合作时间增长，越来越懂你和你的项目

---

## 二、记忆系统的两层架构

Claude Code 的记忆系统由 **两层互补的机制** 组成：

### 2.1 CLAUDE.md — 手动记忆（你写的规则）

这是**你主动编写**的指令文件，告诉 Claude "应该怎么工作"。比如：

```markdown
# 项目约定
- 所有 API 返回统一使用 { code, data, message } 格式
- 禁止在代码中使用 eval()
- 测试覆盖率要求 ≥ 80%
```

### 2.2 MEMORY.md / Auto Memory — 动态记忆（Claude 自己学的）

这是 **Claude 自主写入**的学习记录，记录它"从交互中学到了什么"。比如：

- 你纠正过它不要用 mock 数据库 → 它记住这个反馈
- 它发现项目的鉴权中间件有特殊处理逻辑 → 它记住这个模式
- 它了解到你是前端工程师、不熟悉 Rust → 它调整解释方式

**核心区别**：

| 维度 | CLAUDE.md | MEMORY.md / Auto Memory |
|------|-----------|------------------------|
| 作者 | 你（人类） | Claude（AI） |
| 用途 | 规定规则 | 记录学到的知识 |
| 更新方式 | 手动编辑 | Claude 自动写入，也可手动编辑 |
| 提交到 Git | 是（团队协作） | 否（机器本地） |
| 内容类型 | "应该怎么做" | "发生了什么 / 用户偏好是什么" |

---

## 三、目录结构全景图

理解记忆系统，首先要认识 `.claude/` 目录。它分为 **项目级** 和 **用户级** 两个位置：

### 3.1 项目级（在代码仓库根目录）

```
your-project/
├── .claude/
│   ├── CLAUDE.md                # 项目级指令（提交到 Git）
│   ├── CLAUDE.local.md          # 个人本地覆盖配置（不提交）
│   ├── settings.json            # 设置（hooks、权限等）
│   ├── settings.local.json      # 本地设置（含密钥）
│   ├── rules/                   # 模块化规则文件
│   │   ├── testing.md           # 测试相关规则
│   │   ├── git.md               # Git 工作流规则
│   │   └── api-conventions.md   # API 约定
│   ├── skills/                  # 自定义技能
│   ├── subagents/               # 子代理配置
│   └── memory/                  # 项目级 Auto Memory
│       ├── MEMORY.md            # 记忆索引文件
│       ├── user_preferences.md  # 用户偏好记忆
│       └── project_context.md   # 项目上下文记忆
```

### 3.2 用户级（在用户主目录下）

```
~/.claude/
├── CLAUDE.md                    # 全局指令（所有项目通用）
├── CLAUDE.local.md              # 全局本地覆盖
├── settings.json                # 全局设置
├── skills/                      # 全局技能
├── projects/                    # 按项目隔离的记忆存储
│   └── <项目名>/
│       └── memory/
│           ├── MEMORY.md        # 该项目专属记忆索引
│           ├── user_*.md        # 用户偏好
│           ├── feedback_*.md    # 纠正反馈
│           ├── project_*.md     # 项目上下文
│           └── reference_*.md   # 引用标记
└── memory/                      # 全局 Auto Memory（跨项目）
    └── MEMORY.md                # 全局记忆索引
```

### 3.3 两个关键概念

1. **项目级 vs 用户级**：项目级 `.claude/` 在仓库里，适合团队共享；用户级 `~/.claude/` 在本地机器上，适合个人偏好。

2. **普通文件 vs local 文件**：`CLAUDE.md` 可以提交到 Git 让团队共用；`CLAUDE.local.md` 应该加入 `.gitignore`，存放个人专属设置。

---

## 四、Auto Memory 的四种记忆类型

Auto Memory 是 Claude Code 记忆系统的核心创新。它自动将学习到的内容分类为 **四种类型**：

### 4.1 User Memory（用户记忆）

**存储什么**：你的角色、专业水平、编码偏好、工作风格。

**示例文件**：`~/.claude/memory/user_experience.md`

```markdown
---
name: User Experience
description: User's technical background and preferences
type: user
---

用户是一名有 8 年经验的 Python 后端工程师，对前端不太熟悉。
解释前端概念时应该用后端类比（如把 React state 比作 Django model）。

用户偏好函数式编程风格，不喜欢类继承层级。
```

**何时写入**：当你提到自己的角色、经验水平、偏好时。

**作用范围**：全局（所有项目通用）。

### 4.2 Feedback Memory（反馈记忆）

**存储什么**：你对 Claude 的纠正和确认。这是最有价值的记忆类型。

**示例文件**：`~/.claude/memory/feedback_testing.md`

```markdown
---
name: Testing Corrections
description: Database testing rules
type: feedback
---

集成测试必须使用真实数据库，不能用 mock。
**Why:** 之前发生过 mock 和真实行为不一致，导致测试通过但上线失败的事故。
**How to apply:** 所有涉及数据库读写的测试，使用 testcontainers 启动真实 DB。
```

**何时写入**：当你纠正 Claude（"别再这样做了"）或确认它的做法（"这样很好，以后都这样"）时。

**作用范围**：全局。

### 4.3 Project Memory（项目记忆）

**存储什么**：特定项目的架构决策、正在进行的工作、发现的代码模式。

**示例文件**：`~/.claude/projects/my-blog/memory/project_auth.md`

```markdown
---
name: Auth Service Architecture
description: Current auth service decisions and context
type: project
---

认证服务正在从 Session 迁移到 JWT。
**Why:** 法律合规要求改进 session token 的存储方式。
**How to apply:** 新的 API 端点使用 JWT，旧的 Session 端点保持兼容直到迁移完成。
```

**何时写入**：当 Claude 了解到项目特有的决策、背景或正在进行的工作时。

**作用范围**：按项目隔离。

### 4.4 Reference Memory（引用记忆）

**存储什么**：你标记为重要参考的外部资源、文件路径、工具链接。

**示例文件**：`~/.claude/memory/reference_monitoring.md`

```markdown
---
name: Monitoring Dashboard
description: Grafana monitoring setup
type: reference
---

Grafana 监控面板地址：grafana.internal/d/api-latency
**用途**：oncall 团队监控 API 延迟。
**何时查看**：修改请求路径相关代码时检查此面板。
```

**何时写入**：当你标记某个文件或链接为重要参考时。

**作用范围**：可以是全局或项目级。

---

## 五、MEMORY.md 索引文件

### 5.1 它是什么

MEMORY.md 是一个**目录/索引文件**，不是记忆内容本身。它的作用类似一本书的目录，告诉 Claude "有哪些记忆文件、它们分别讲什么"。

### 5.2 文件格式

```markdown
# User Memory
- [User Role and Preferences](user_role.md) — 偏好 TypeScript，避免类继承
- [Working Style](working_style.md) — 喜欢详细 code review，要求先写测试

# Feedback Memory
- [Testing Corrections](feedback_testing.md) — 集成测试必须用真实数据库
- [Style Preferences](feedback_formatting.md) — 文件名始终用 kebab-case

# Project Memory
- [Auth Service Architecture](project_auth.md) — 当前架构和决策
- [Dashboard Development](project_dashboard.md) — 正在进行的面板功能

# Reference Memory
- [API Documentation](ref_api_docs.md) — 关键 REST API 端点和模式
```

### 5.3 格式规则

- 每条格式：`- [标题](文件名.md) — 一句话描述`
- 每条不超过 **150 个字符**
- **没有 YAML frontmatter** — 纯 Markdown
- 分区标题对应四种记忆类型
- 文件名用描述性主题命名（不是自动生成的哈希值）

### 5.4 200 行硬性限制

MEMORY.md 有 **200 行 / 25KB 的硬性上限**。只有前 200 行会在每次会话启动时加载，超出的内容会被**静默截断**（无错误提示）。这就是为什么它必须是索引——详细内容存在单独的主题文件中。

---

## 六、记忆加载顺序和优先级

每次 Claude Code 会话启动时，文件按以下顺序加载（从通用到具体）：

| 优先级 | 文件 | 作用范围 | 提交到 Git？ |
|--------|------|----------|-------------|
| 1 | `/etc/claude-code/CLAUDE.md` | 系统级（管理员） | N/A |
| 2 | `~/.claude/CLAUDE.md` | 所有项目 | 可选 |
| 3 | `~/.claude/CLAUDE.local.md` | 所有项目（本地） | **否** |
| 4 | `project/.claude/CLAUDE.md` | 当前项目 | **是** |
| 5 | `project/.claude/CLAUDE.local.md` | 当前项目（本地） | **否** |
| 6 | `.claude/rules/*.md` | 模块化规则 | **是** |
| 7 | Auto Memory（MEMORY.md 索引） | 学习到的上下文 | **否** |

**关键规则**：

- **更具体的覆盖更通用的**：项目级 CLAUDE.md 会覆盖全局 CLAUDE.md
- **rules 文件全文加载**：`.claude/rules/*.md` 中所有文件都会在启动时完整加载，消耗上下文窗口 token。保持精简
- **Auto Memory 索引加载，主题文件按需读取**：MEMORY.md 启动时加载，具体主题文件在 Claude 需要时才读取

---

## 七、记忆的生命周期：从创建到使用

### 7.1 自动创建流程

```
你与 Claude 交互
       │
       ▼
你纠正 Claude："不要用 mock 数据库"
       │
       ▼
Claude 判断这是值得记住的反馈
       │
       ▼
Claude 创建/更新 feedback 记忆文件
       │
       ▼
Claude 更新 MEMORY.md 索引
       │
       ▼
下次会话启动时，该记忆被自动加载
```

### 7.2 Claude 何时会保存记忆？

| 触发场景 | 记忆类型 | 示例 |
|----------|----------|------|
| 你提到角色/经验 | User | "我是前端转后端，对分布式不太熟" |
| 你纠正做法 | Feedback | "别再写 Python 2 风格的代码了" |
| 你确认做法正确 | Feedback | "对，这种格式正是我想要的" |
| 发现项目特有模式 | Project | "这个项目的中间件用了自定义错误处理" |
| 你标记重要参考 | Reference | "监控面板在 grafana.internal/d/xxx" |
| 了解到截止日期/冻结 | Project | "周四之后非关键合入冻结" |

### 7.3 Claude 不保存什么

- **代码模式、架构、文件路径** — 这些可以直接读代码获取
- **Git 历史、谁改了什么** — `git log` / `git blame` 是权威来源
- **调试方案、修复记录** — 修复已经在代码中，commit message 有上下文
- **CLAUDE.md 已有的内容** — 不要重复
- **瞬时任务细节** — 当前会话的进展、临时状态

### 7.4 手动管理记忆

在 Claude Code 会话中，你可以：

- **查看当前记忆**：Claude 可以列出所有已加载的记忆文件
- **手动编辑**：记忆文件是纯 Markdown，可以直接编辑
- **删除记忆**：删除对应的 `.md` 文件即可"遗忘"
- **更新 MEMORY.md**：直接在索引中添加/删除条目

---

## 八、记忆系统与企业开发组件的关系

在企业开发环境中，记忆系统与其他 `.claude/` 组件协同工作：

```
┌─────────────────────────────────────────────────┐
│              企业开发协作环境                      │
├─────────────────────────────────────────────────┤
│                                                 │
│  CLAUDE.md  ←─ 团队约定（提交 Git，全员共享）     │
│       │                                         │
│       ▼                                         │
│  rules/*.md  ←─ 模块化规则（测试、Git、API...）    │
│       │                                         │
│       ▼                                         │
│  MEMORY.md  ←─ 个人学习积累（不提交，机器本地）    │
│       │                                         │
│       ▼                                         │
│  settings.json  ←─ 行为配置（hooks、权限...）     │
│       │                                         │
│       ▼                                         │
│  skills/  ←─ 可复用能力（调试、TDD、审查...）      │
│       │                                         │
│       ▼                                         │
│  subagents/  ←─ 专用代理（代码审查、探索...）      │
│                                                 │
└─────────────────────────────────────────────────┘
```

### 各组件职责对比

| 组件 | 谁写 | 提交 Git | 作用 | 加载时机 |
|------|------|----------|------|----------|
| CLAUDE.md | 人 | 是 | 团队规则 | 会话启动 |
| CLAUDE.local.md | 人 | 否 | 个人覆盖 | 会话启动 |
| rules/*.md | 人 | 是 | 模块化规则 | 会话启动 |
| MEMORY.md | Claude | 否 | 记忆索引 | 会话启动 |
| memory/*.md | Claude | 否 | 记忆主题 | 按需加载 |
| settings.json | 人 | 是 | 行为配置 | 会话启动 |
| skills/ | 人/Claude | 可选 | 可复用能力 | 手动调用 |

---

## 九、企业级最佳实践

### 9.1 团队初始化：从零搭建

新仓库启动时，建议团队一起创建基础结构：

```bash
# 1. 创建项目级目录
mkdir -p .claude/rules

# 2. 创建项目 CLAUDE.md（团队共享规则）
# 3. 创建模块化规则文件
# 4. 将 .claude/CLAUDE.md 和 .claude/rules/ 提交到 Git
# 5. 将 .claude/CLAUDE.local.md 加入 .gitignore
```

**推荐的 .gitignore 条目**：

```
# Claude Code 本地文件
.claude/CLAUDE.local.md
.claude/settings.local.json
.claude/memory/
```

### 9.2 CLAUDE.md 编写模板

```markdown
# 项目名称

## 技术栈
- 后端：Python 3.11 + FastAPI
- 前端：TypeScript + React 18
- 数据库：PostgreSQL 15

## 代码规范
- 使用 snake_case 命名（Python）/ camelCase（TypeScript）
- 所有函数必须有类型注解
- 错误处理使用自定义异常类

## Git 工作流
- 功能分支命名：feature/简短描述
- commit message 格式：type(scope): description
- 合并前必须通过 CI 和代码审查

## 测试要求
- 单元测试覆盖率 ≥ 80%
- 集成测试使用 testcontainers 启动真实依赖
- 禁止 mock 数据库行为
```

### 9.3 rules 文件拆分策略

当 CLAUDE.md 超过 150 行时，拆分为 `.claude/rules/` 下的模块文件：

```
.claude/rules/
├── testing.md          # 测试策略和约定
├── git.md              # Git 分支、commit、PR 规则
├── api-conventions.md  # API 设计、错误格式、鉴权
├── database.md         # 数据库迁移、ORM 使用约定
└── security.md         # 安全要求、OWASP 规则
```

每个文件专注一个主题，保持 50-100 行以内。

### 9.4 管理 Auto Memory

1. **定期审查记忆**：用 `/memory` 命令检查 Claude 记住了什么，纠正不准确的条目
2. **清理过时记忆**：项目方向改变后，删除不再相关的记忆文件
3. **不要记忆可从代码获取的内容**：Claude 可以实时读代码、查 git 历史，不需要记忆这些
4. **记忆人，不记忆代码**：重点记录用户偏好、决策背景、工作方式，而非文件结构

### 9.5 Token 预算控制

所有启动时加载的文件都消耗上下文窗口 token：

| 文件 | 建议大小 | 原因 |
|------|----------|------|
| CLAUDE.md | ≤ 150 行 | 每会话加载 |
| 每个 rules/*.md | ≤ 100 行 | 每会话加载 |
| MEMORY.md | ≤ 200 行 | 硬限制 200 行 |
| memory/*.md | 不限 | 按需加载，不计入启动 token |

### 9.6 多人协作场景

```
开发者 A（本地）                    Git 仓库                    开发者 B（本地）
─────────────                   ──────────                   ─────────────
.claude/CLAUDE.md ────────────→ ←─────────────────────────── .claude/CLAUDE.md
.claude/rules/*.md ───────────→ ←─────────────────────────── .claude/rules/*.md
.claude/CLAUDE.local.md ────X   (不提交)                     .claude/CLAUDE.local.md
~/.claude/memory/ ──────────X   (不提交)                     ~/.claude/memory/
```

- **团队共享**：通过 Git 提交 CLAUDE.md 和 rules/*.md
- **个人专属**：CLAUDE.local.md 和 Auto Memory 不提交，各自独立

### 9.7 持续改进闭环

```
Claude 犯错 ─→ 你纠正 ─→ 自动写入 Feedback Memory ─→ 下次不再犯
                                                    │
    如果这个纠正值得全团队知道 ◄─────────────────────┘
                │
                ▼
        手动写入 CLAUDE.md 或 rules/*.md
                │
                ▼
        提交到 Git ─→ 全团队 Claude 共享这个规则
```

**什么时候从 Auto Memory 提升到 CLAUDE.md？**

当一条反馈从"个人偏好"变为"团队规范"时：

- "我不喜欢这个格式" → 留在 Feedback Memory（个人）
- "这个项目禁止这种做法" → 写入 rules/*.md（团队）

---

## 十、实战示例

### 10.1 示例 1：新项目快速上手

```bash
# 第一步：初始化项目结构
mkdir -p .claude/rules

# 第二步：编写 CLAUDE.md
cat > .claude/CLAUDE.md << 'EOF'
# 博客知识库平台

## 技术栈
- 框架：Astro 4.x
- 语言：TypeScript
- 样式：Tailwind CSS 4
- 部署：Vercel

## 约定
- 组件使用 .astro 后缀
- 内容文件放在 src/content/ 下
- 所有图片使用 optimizeImage() 处理
EOF

# 第三步：编写测试规则
cat > .claude/rules/testing.md << 'EOF'
## 测试
- 使用 Vitest 作为测试框架
- 组件测试使用 @astrojs/testing
- 每个新页面必须有基本渲染测试
EOF

# 第四步：提交到 Git
git add .claude/CLAUDE.md .claude/rules/
git commit -m "feat: add Claude Code project configuration"
```

### 10.2 示例 2：Claude 自动学习过程

假设你在项目中与 Claude 的交互过程：

**会话 1**：
```
你：帮我写一个用户注册接口
Claude：好的，这是代码...（用了 try-catch 包裹每个数据库操作）
你：不用每个操作都 try-catch，我们用全局错误中间件处理
Claude：明白了，下次会使用全局错误中间件
（Claude 自动写入 Feedback Memory）
```

**会话 2**（第二天）：
```
你：帮我写一个文章发布接口
Claude：好的，使用全局错误中间件处理错误...
（自动应用了上次学到的规则！）
```

### 10.3 示例 3：手动编辑记忆

你发现 Claude 记住了一条过时的信息：

```bash
# 查看当前记忆
# 在 Claude Code 会话中查看 /memory 或直接读取

cat ~/.claude/projects/my-project/memory/MEMORY.md
```

发现某条记忆已经不适用，直接编辑或删除：

```bash
# 删除过时的记忆文件
rm ~/.claude/projects/my-project/memory/project_old_feature.md

# 从 MEMORY.md 中移除对应索引行
```

---

## 十一、常见问题 FAQ

**Q1：记忆是跨机器的吗？**

不是。Auto Memory 存储在本地机器上（`~/.claude/`），不会随 Git 同步。如果换了机器，需要重新积累或通过 `/memory` 命令导出导入。

**Q2：记忆会泄露敏感信息吗？**

Claude 被指示不要将密钥、密码、token 等敏感信息写入记忆文件。但建议你仍将 `~/.claude/` 目录视为敏感数据，不要上传到公开仓库。

**Q3：Auto Memory 可以关闭吗？**

可以在会话中通过 `/memory` 命令开关 Auto Memory。但关闭后 Claude 就不会记住你的偏好了。

**Q4：记忆文件太多了会影响性能吗？**

不会。MEMORY.md 有 200 行硬限制，启动时只加载索引。主题文件按需加载，不影响启动速度。但 rules 文件过多会影响——每个 rules/*.md 都会全文加载。

**Q5：如何让整个团队共享记忆？**

Auto Memory 是本地化的，不能直接共享。但你可以：
1. 把重要的反馈写入 CLAUDE.md 或 rules/*.md（提交到 Git）
2. 团队约定统一的项目级配置
3. 个人记忆各自维护

---

## 十二、总结

Claude Code 的记忆系统是一套**渐进式、自适应的上下文管理机制**：

| 层次 | 文件 | 谁维护 | 共享范围 |
|------|------|--------|----------|
| 系统规则 | `/etc/claude-code/CLAUDE.md` | 管理员 | 所有用户 |
| 全局指令 | `~/.claude/CLAUDE.md` | 你 | 你所有的项目 |
| 项目指令 | `.claude/CLAUDE.md` | 团队 | 项目团队成员 |
| 模块化规则 | `.claude/rules/*.md` | 团队 | 项目团队成员 |
| 自动记忆 | `~/.claude/projects/.../memory/` | Claude（你审核） | 仅本地机器 |

**企业落地建议**：

1. **起步**：写好 CLAUDE.md，让团队有统一的起点
2. **成长**：让 Auto Memory 自然积累，定期审查
3. **沉淀**：将有价值的个人反馈提升到 rules/*.md，成为团队规范
4. **精简**：持续清理过时记忆，保持系统高效

通过这套系统，Claude Code 从一个"每次都从零开始"的 AI 助手，变成了一个"越用越懂你和你的项目"的智能协作者。
