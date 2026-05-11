# Blog Anything Knowledge Bases

A personal knowledge base for scattered thoughts, technical notes, and lessons learned. Organized by topic into folders — each directory is its own module.

## Documentation

Read the guides online via GitHub Pages:

| Guide | Description | Link |
|-------|-------------|------|
| Claude Code 记忆系统完全指南 | 从零基础出发，深入剖析记忆系统的架构原理、文件结构、工作流程 | [查看](https://qiuyanlong16.github.io/blog-anything-knowledge-bases/) |
| Harness Engineering 企业实战指南 | 六层架构、渐进式学习路径、多 Agent 协作、企业案例 | [查看](https://qiuyanlong16.github.io/blog-anything-knowledge-bases/harness.html) |

## Structure

Each top-level folder represents a topic area. Add notes freely; no strict formatting required.

```
repo-root/
├── claude-code/       # Claude Code notes and guides
├── harness/           # Harness Engineering notes and guides
└── <topic>/           # New topics added as needed
    └── docs/          # Static pages or demos (optional)
```

## Usage

- Drop markdown files into the relevant topic folder
- Use `docs/` subfolder for deployable static pages (GitHub Pages)
- Commit and push — GitHub Actions handles deployment automatically
