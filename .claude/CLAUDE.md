# Project Context

This repo is a **personal knowledge base** — a place for scattered notes, technical write-ups, quick experiments, and lessons learned. Think of it as a digital notebook organized by topic.

## How It Works

- **Each top-level folder = one topic/module.** Examples: `claude-code/`, `kubernetes/`, `ai-patterns/`, etc.
- New topics are created as new top-level folders whenever needed. No permission or discussion required.
- Notes are written in Markdown. No strict formatting rules — just make them readable.
- Static demo pages (HTML, mini sites) go inside a `docs/` subfolder under the relevant topic.
- GitHub Pages can be enabled to serve `*/docs/` folders.

## How to Maintain This Repo

When adding content:
1. Check if a relevant topic folder already exists. If yes, add the file there. If no, create a new folder.
2. Keep filenames descriptive (e.g., `claude-code-memory-system-guide.md`, not `note1.md`).
3. Commit and push — this repo is personal, no PR process needed.

When creating static pages:
1. Put all files under `<topic>/docs/`.
2. The entry point should be `index.html`.
3. These can be deployed via GitHub Pages (branch `main`, folder `docs/` or via action).

## Style

- Notes should be **practical and actionable** — not academic papers.
- Code examples should be complete and runnable when possible.
- Chinese is the primary language for notes, but English is fine for READMEs, code, or guides intended for a wider audience.
