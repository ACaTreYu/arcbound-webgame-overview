# Arcbound — Toolchain & Setup Overview

This page describes the development toolchain for the private Arcbound repository at a general level. It is an overview for readers of the public showcase, not a runnable copy of the private configuration — you can **play the game with zero setup at [arcboundinteractive.com](https://arcboundinteractive.com)**.

## Requirements

| Tool | Purpose |
|---|---|
| **Node.js 20.19+** | Game server runtime and build tooling |
| **npm** | Dependency management and script runner |
| **TypeScript 5.x** | Entire client, server, and shared codebase |
| **Vite 7** | Client dev server and production bundling |
| **Vitest** | Test runner (simulation, AI, protocol suites) |
| **Python 3.10+** | RL training pipeline only (not needed to run the game) |
| **PyTorch + NumPy** | ABI-PPO policy training and analysis tooling |

The **ARCnet** netcode library is consumed as a sibling package (a local file dependency during development), so a working checkout places the two repositories next to each other.

## Build Targets

One codebase produces several artifacts:

- **Server** — compiled with `tsc` from its own `tsconfig`, run directly under Node.
- **Web client** — bundled by Vite into static assets served to the browser.
- **Tutorial** — a separate Vite config builds the standalone interactive tutorial.
- **Discord Activity** — a client build with a base path suited to Discord's embedded-app proxy.

The npm scripts follow the standard shape:

```bash
npm install
npm run dev:server    # TypeScript compiler in watch mode
npm run dev:client    # Vite dev server with hot reload
npm run build         # production build: server + client
npm run start:server  # run the compiled server
npm test              # Vitest suites
```

During development the server and client run side by side; the client dev server proxies to the local game server, and the shared simulation module is compiled into both.

## Runtime Configuration

Deployment-specific values — ports, auth signing secrets, third-party sign-in credentials, mail-delivery settings, and Discord application settings — are supplied through environment variables and are not part of the repository. Persistence uses SQLite via `better-sqlite3`, with databases created on first run; no external database server is required.

## RL Training Pipeline

The Python side lives in `scripts/` and is fully decoupled from serving the game — the live server performs policy inference in pure TypeScript and never runs Python.

A typical cycle, driven by a single pipeline script:

1. Poll the game server for accumulated experience rollouts.
2. Pull rollouts locally once a threshold is reached.
3. Train with ABI-PPO (staged curriculum PPO in PyTorch).
4. Export the checkpoint to JSON weights plus observation-normalizer statistics.
5. Run pre-deploy sanity checks, back up the current policy, deploy, and hot-reload the live bots.

The pipeline supports single-shot and continuous modes, training-aggressiveness profiles, and one-command rollback to the previous policy. Supporting scripts handle rollout validation, policy evaluation, metrics dashboards, and heatmap/infographic generation. See **abi-ppo-overview** for the training system itself.

## Deployment

Production runs on a cloud VPS: the compiled Node server serves the game socket, the WebRTC signaling endpoint, the HTTP API, and the static client build. Deployment is scripted (build, upload, restart, verify) and policy updates deploy independently of code releases via the hot-reload path.

---

*Back to [README.md](README.md) · [ARCHITECTURE.md](ARCHITECTURE.md)*
