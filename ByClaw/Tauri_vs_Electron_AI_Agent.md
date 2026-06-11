# Tauri vs Electron (2026 Guide)

**For Building a Local AI Agent (OpenClaw-style, e.g. ByClaw / QClaw)**  
**Target Platforms: macOS / Windows 11 / Ubuntu**

> **Note:** This is a framework-layer comparison. For this project's full architecture decision, see [RESEARCH.md](RESEARCH.md).  
> **Current project conclusion:** Electron — due to deep OpenClaw customization, proven ecosystem, and full TypeScript stack.

## Quick Comparison Table (Shell Layer Only)

| Criteria | **Tauri 2.x** | **Electron** | Winner |
|----------|---------------|--------------|--------|
| Shell Bundle Size | 3–15 MB | 80–250+ MB | **Tauri** |
| Shell Idle Memory | 30–80 MB | 150–400+ MB | **Tauri** |
| Startup Time | 0.3–0.8 s | 1.5–4 s | **Tauri** |
| Runtime Performance | Strong (Rust backend) | Good, heavier Chromium | **Tauri** |
| Development Speed | Steeper (Rust + Web) | Very fast (JS/TS) | **Electron** |
| UI Consistency | Variable (system WebView) | Excellent (bundled Chromium) | **Electron** |
| Security Defaults | Strong (capability-based) | Larger surface; hardenable | **Tauri** |
| Ecosystem Maturity | Growing fast | Extremely mature | **Electron** |
| OpenClaw Deep Integration | IPC only; Rust rewrite for custom logic | Direct `require()`, same TS codebase | **Electron** |

## Real-World Bundle Size (This Project)

Shell size alone is misleading. Including OpenClaw + dependencies:

| Component | Approx. Size |
|-----------|--------------|
| Desktop shell (Electron) | ~120 MB |
| Desktop shell (Tauri) | ~5–15 MB |
| OpenClaw + node_modules | ~50–150 MB |
| Hermes Python env (macOS/Ubuntu) | ~100–300 MB |
| **Total (Electron, no Hermes)** | **~180–300 MB** |
| **Total (Electron + Hermes)** | **~280–600 MB** |

Tauri reduces shell overhead, but **does not eliminate** OpenClaw/Hermes runtime size.

## When to Choose Tauri

Choose Tauri if you want:

- Minimal shell footprint and lower idle RAM/CPU
- Strong default security (capabilities model)
- Rust backend for new native features (sandbox, schedulers, OS integration)
- You accept **IPC-only** integration with OpenClaw (no direct `require()`)

Best fit when OpenClaw is a **black-box sidecar** (spawned process / HTTP / WebSocket), not deeply customized in-process.

## When to Choose Electron

Choose Electron if you prioritize:

- Maximum development speed (especially when following QClaw/LobsterAI/ClawX patterns)
- Deep OpenClaw customization: Hooks, Channels, plugins, telemetry, guardian — all in TypeScript
- Full web-team ownership (no Rust/Go backend engineer required)
- Consistent UI across macOS, Windows, and Linux
- Proven path: **all known OpenClaw desktop products use Electron**

## Project-Specific Recommendation

For **ByClaw** (deep OpenClaw 二次开发 + Hermes dual-engine + Vue 3 frontend):

**Electron remains the recommended choice.**

### Why Electron Wins for This Project

1. QClaw and all comparable products validated Electron for OpenClaw desktopization
2. Custom logic (模型路由, 多 Channel, 埋点, 网关守护) stays in one TypeScript codebase
3. OpenClaw upgrades are isolated to the adapter layer
4. Hermes WSL2 deployment and kernel switching are simpler from Node main process

### When Tauri Would Make Sense

Revisit Tauri if priorities shift to:

- Shell size/RAM as top priority **and** you're willing to treat OpenClaw as an external process only
- Adding substantial Rust-native features (tool sandbox, OS hooks) **alongside** a thin web UI
- Hiring or onboarding Rust capability

### Possible Hybrid (Not "Pure Tauri")

| Pattern | Description | Trade-off |
|---------|-------------|-----------|
| **Tauri + Node sidecar** | Tauri shell; OpenClaw runs as separate Node process | Smaller shell; still ships Node runtime; complex IPC |
| **Electron (current plan)** | OpenClaw in main process via adapter | Larger shell; simplest deep integration |

## Decision Checklist

**Choose Tauri if 4+ are true:**

- [ ] Package size matters more than dev speed
- [ ] Low resource usage is critical (always-on background agent)
- [ ] Security defaults outweigh integration depth
- [ ] OpenClaw is used as external API only (no in-process `require()`)
- [ ] Team can invest in Rust for backend/sidecar logic

**Choose Electron if 3+ are true:**

- [ ] Need to ship in 2–4 weeks with existing web team
- [ ] Deep OpenClaw customization (Hooks, plugins, Channels)
- [ ] Following QClaw/LobsterAI/ClawX architecture
- [ ] Dual-engine (OpenClaw + Hermes) with unified TypeScript orchestration
- [ ] UI consistency across three desktop platforms

## Recommended Implementation Path (Electron — Current Plan)

1. Use OpenClaw as **npm dependency** (version-pinned), not a fork
2. Electron 37+ as desktop shell
3. Vue 3 + TypeScript frontend (pnpm monorepo: `shell` + `web`)
4. `OpenClawAdapter` for customization without modifying OpenClaw source
5. Hermes as stdio ACP subprocess; Windows via WSL2 on demand

---

**Prepared:** June 2026  
**Scope:** macOS / Windows 11 / Ubuntu — Local AI Agent Desktop  
**Related:** [RESEARCH.md](RESEARCH.md) · [QClaw-Architecture-Analysis.md](QClaw-Architecture-Analysis.md)
