# Blog Anything Knowledge Bases

A personal knowledge base for scattered thoughts, technical notes, and lessons learned. Organized by topic into folders — each directory is its own module.

## Documentation

Read the guides online via GitHub Pages:

| Guide | Description | Link |
|-------|-------------|------|
| Claude Code Memory System | Architecture, file structure, workflow, and enterprise best practices | [View](https://qiuyanlong16.github.io/blog-anything-knowledge-bases/) |
| Harness Engineering | Six-layer architecture, progressive learning path, multi-agent orchestration | [View](https://qiuyanlong16.github.io/blog-anything-knowledge-bases/harness.html) |
| QClaw Architecture Analysis | ByClaw cross-platform desktop AI Agent — architecture, kernel switching, model routing | [View](https://qiuyanlong16.github.io/blog-anything-knowledge-bases/qclaw-architecture.html) |
| ByClaw Research & Tech Selection | Framework comparison, security design, upgrade strategy, implementation plan | [View](ByClaw/RESEARCH.md) |

## Structure

Each top-level folder represents a topic area. Add notes freely; no strict formatting required.

```
repo-root/
├── claude-code/       # Claude Code notes and guides
├── harness/           # Harness Engineering notes and guides
├── ByClaw/            # Cross-platform desktop AI Agent — research & tech selection
└── <topic>/           # New topics added as needed
    └── docs/          # Static pages or demos (optional)
```

## Usage

- Drop markdown files into the relevant topic folder
- Use `docs/` subfolder for deployable static pages (GitHub Pages)
- Commit and push — GitHub Actions handles deployment automatically
