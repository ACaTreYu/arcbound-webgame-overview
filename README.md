# Arcbound

**The flagship game of ArcBound Interactive LLC** — a real-time multiplayer arena shooter that runs entirely in the browser. Arcbound pairs classic arena-shooter physics and game feel with a modern TypeScript stack: WebGL rendering, WebRTC netcode, and machine-learned bots that train on live gameplay.

**Play it now at [arcboundinteractive.com](https://arcboundinteractive.com)** — no install, no plugin, just a browser tab.

> This is the public overview of a private repository. It documents what the system is and how it works at a concept level; source code, wire formats, and tuning data remain private.

---

## Novel Systems & Methods

The interesting engineering in Arcbound, up front:

### Shared deterministic simulation with prediction & reconciliation
Movement, projectile physics, and collision resolution live in a single `shared/` module compiled into **both** the client and the server. Both sides step the same fixed-timestep simulation (33 Hz), so the client can predict its own movement instantly, tag every input with a sequence number, and — when an authoritative snapshot arrives — rewind to the server's state and replay only the unacknowledged inputs. Remote players are placed on an interpolation timeline driven by a lightweight NTP-style clock-sync handshake rather than packet arrival time, which removes network jitter from what the player sees.

### ARCnet: custom netcode over WebRTC DataChannels
Game traffic runs over unreliable/unordered WebRTC DataChannels (UDP-like semantics in the browser) with **ARCnet**, the studio's own reliability protocol, layered on top: per-channel selective-acknowledgment retransmission, packet batching, adaptive quality tiers, and congestion handling — with WebSocket signaling and fallback. Separate channels carry state snapshots and player inputs so a stall on one never blocks the other. ARCnet is developed as its own library (see **arcnet-overview**) and consumed by both the browser client and the Node server.

### Live reinforcement-learning bots with pure-TypeScript inference
Arcbound's bots are not just scripted. A PPO policy network — trained in PyTorch by the studio's **ABI-PPO** system (see **abi-ppo-overview**) — runs *inside the Node game server* via a hand-written TypeScript forward pass over weights exported to JSON. No native bindings, no Python at runtime, no inference server: the same process that simulates the match evaluates the policy every decision tick. Policies hot-reload without a server restart, and per-player "clone" personas load individual weight sets, aim calibration, and adaptive difficulty profiles.

### A closed training loop running against production
Live matches emit experience rollouts; an automated pipeline polls the server, pulls accumulated experience, trains with ABI-PPO's staged curriculum (movement → combat → strategy, with decoupled actor/critic optimizers and decomposed reward channels), exports fresh JSON weights, deploys them, and hot-reloads the live bots — with pre-deploy sanity gates, automatic backups, and one-command rollback. The bots that players fight tonight learned from the players who fought them last week.

### Hybrid AI: rules + learning
The RL policy sits alongside a substantial rule-based AI subsystem: A*-style pathfinding with map analysis and curated flag routes, geometric weapon-line solving, perception and threat modeling, a playmaking layer that issues squad-level directives (pincers, power plays, disengages), and per-opponent fire-pattern tracking that informs dodge behavior. A utility-scoring decision substrate unifies candidate intents under one evaluation loop. Bots are mode-aware and fill out matches in every game mode, including the run-based modes where they fight alongside players — and a social layer where they host and invite others into runs of their own.

### A roguelite progression layer sharing the arena's simulation
Alongside the arena modes, the same deterministic simulation drives a
run-based dungeon and a wave-survival mode: procedurally generated floors
assembled as open chambers joined by wide mouths, a boss encounter that
generates a fresh approach and arena each time, eight playable archetypes with
their own starter/boss/legendary/ultimate card pools, and a five-rung
difficulty ladder where each rung is unlocked by reaching a floor depth on the
one below. Upgrades compose rather than stack linearly — combos check
prerequisites across card tiers — and every floor is written out in the same
binary map format the arena uses, so a generated dungeon is a real map.
Bots fill any empty party slot and play the run alongside human players.

### Byte-exact map interop with a companion editor
Arcbound reads and writes the classic binary `.map` format, round-trip compatible with the studio's standalone **AC Map Editor** (see **ac-map-editor-overview**). The server itself hosts the editor's publishing endpoints, so maps authored in the editor flow directly into the live rotation — alongside a library of hundreds of community-authored maps.

### Bandwidth-engineered binary protocol
High-frequency state updates use a custom binary snapshot encoding with per-entity presence flags for optional fields — roughly **60–70% smaller than the equivalent JSON** — plus run-length tile compression for map transfer.

### One codebase, three surfaces
The same client builds as the standalone web game, a self-contained interactive tutorial, and a **Discord Activity** (via the Embedded App SDK, with transparent URL-mapping so all traffic traverses Discord's proxy). Playing inside a Discord voice channel is the same game, same servers.

---

## The Game

- **Six-plus arena modes** — Normal (the classic mode family: CTF flag play, Switch control, and combined flag-and-switch maps), Deathmatch, Free-for-All, Domination, Assassin, and Turret Assassin, each implemented as a plug-in win-condition module; modes are declared by map naming convention.
- **Run-based modes** — a solo/co-op dungeon crawl and a wave-survival mode, both on procedurally generated floors, with eight archetypes, boss arenas, and a five-rung difficulty ladder unlocked by depth.
- **A persistent staging arena** — players land in a shared social space (not a menu), fly to zones to queue for matches, party up, or start a run.
- **A pre-game room** — hosting opens a full-screen room with claimable seats, per-seat archetype selection, room chat and live cosmetic changes; the room persists across the match and players return to it afterwards.
- **Leaderboards** — sectioned by difficulty rung and split by game type, with per-rung personal bests and score weighted by the rung played.
- **Progression & cosmetics** — account levels, an in-game currency and shop, saveable cosmetic loadouts, and clans with invites and tags.
- **Interactive tutorial** — a guided, scripted intro sequence built on the real game engine.
- **Map ecosystem** — hundreds of community-authored maps plus studio-authored maps; turrets, switches, warps, bouncy walls, and flag mechanics all fully supported.
- **Accounts & social** — token-based auth, third-party sign-in and email recovery; friends, direct messages, presence and status, ignore lists, ready checks, match voting, and spectator "live look-in."
- **AI showcase matches** — always-on exhibition games where trained bots play each other, doubling as a self-play data source.

## Tech Stack

| Layer | Technology |
|---|---|
| Client | TypeScript, Pixi.js 8 (WebGL), Vite; custom UI layer |
| Server | Node.js, Express, `ws`, `node-datachannel` (WebRTC), SQLite |
| Netcode | ARCnet (in-house reliability protocol) over WebRTC DataChannels |
| Shared sim | Deterministic TypeScript module used by client and server |
| ML training | Python, PyTorch (ABI-PPO), NumPy tooling |
| Testing | Vitest (simulation, AI, and protocol test suites) |

## By the Numbers

**1,440 commits** on `master` · **~175,000 lines of code** — ~143K TypeScript across 296 source files (155 server-side alone) and ~32K Python across 81 pipeline and tooling scripts · six arena modes plus two run-based modes · 339 maps in the library · deployed to a cloud host and live at [arcboundinteractive.com](https://arcboundinteractive.com).

## Related ArcBound Interactive Projects

- **arcnet-overview** — the ARCnet reliability protocol used for Arcbound's netcode
- **abi-ppo-overview** — the ABI-PPO reinforcement-learning training system behind the bots
- **ac-map-editor-overview** — the companion map editor with byte-exact format interop
- **arcbound-unity-overview** — a Unity exploration of the Arcbound universe
- **strangequark-overview** — another ArcBound Interactive title

## Documentation

- [ARCHITECTURE.md](ARCHITECTURE.md) — system design with diagrams: client/server/shared layout, rooms and modes, the netcode stack, the RL loop, and editor integration
- [SETUP.md](SETUP.md) — toolchain overview for the private repository

---

*ArcBound Interactive LLC · [arcboundinteractive.com](https://arcboundinteractive.com)*
