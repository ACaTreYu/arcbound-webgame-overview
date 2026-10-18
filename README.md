# Arcbound

**The flagship game of ArcBound Interactive LLC** — a real-time multiplayer arena shooter that runs entirely in the browser. Arcbound pairs classic arena-shooter physics and game feel with a modern TypeScript stack: WebGL rendering, WebRTC netcode, and machine-learned bots that train on live gameplay.

**Play it now at [arcboundinteractive.com](https://arcboundinteractive.com)** — no install, no plugin, just a browser tab.

> This is the public overview of a private repository. It documents what the system is and how it works at a concept level; source code, wire formats, and tuning data remain private.

---

## Novel Systems & Methods

The interesting engineering in Arcbound, up front:

### Shared deterministic simulation with classic relay authority
Movement, projectile physics, and collision resolution live in a single `shared/` module compiled into **both** the client and the server, and both step the same fixed-timestep simulation (33 Hz). Authority follows the classic game Arcbound recreates: each client flies its own ship with that shared physics and no server correction, and detects hits against its own true position — the victim decides, the server de-duplicates the report and applies the damage. The server owns everything else: health, scores, rules, rooms and the bots. It relays each ship's position to the other players, and every position report carries the time it was true on the sender's clock; the server maps that clock onto its own by the fastest recent delivery and forwards each position with its age, so another screen places it where it belongs on the timeline rather than when it happened to arrive, and draws it behind an interpolation delay sized from measured lateness.

### ARCnet: custom netcode over WebRTC DataChannels
Game traffic runs over unreliable/unordered WebRTC DataChannels (UDP-like semantics in the browser) with **ARCnet**, the studio's own reliability protocol, layered on top: four channels with their own delivery rules, piggybacked selective acknowledgements, sequence-tagged forward error correction and packet batching. Match events travel ordered and acknowledged, state snapshots on a latest-wins channel with their FEC parity bundled into the same send, and inputs and fire events as compact unreliable codec packets beside it, so a stall on one never blocks another. Signaling and account traffic use a WebSocket, which also carries the game where the WebRTC upgrade does not connect. ARCnet is developed as its own library (see **arcnet-overview**) and consumed by both the browser client and the Node server; its adaptive snapshot pacing is not switched on in the game, which sends a snapshot every 33 Hz tick.

### Live reinforcement-learning bots with pure-TypeScript inference
Arcbound's bots are not just scripted. A PPO policy network — trained in PyTorch by the studio's **ABI-PPO** system (see **abi-ppo-overview**) — runs *inside the Node game server* via a hand-written TypeScript forward pass over weights exported to JSON. No native bindings, no Python at runtime, no inference server: the same process that simulates the match evaluates the policy every decision tick. Policies hot-reload without a server restart, and per-player "clone" personas load individual weight sets, aim calibration, and adaptive difficulty profiles.

### A closed training loop running against production
Live matches emit experience rollouts; an automated pipeline polls the server, pulls accumulated experience, trains with ABI-PPO's staged curriculum (movement → combat → strategy, with decoupled actor/critic optimizers and decomposed reward channels), exports fresh JSON weights, deploys them, and hot-reloads the live bots — with pre-deploy sanity gates, automatic backups, and one-command rollback. No model ships on its loss curve: each candidate first plays the live policy head-to-head in a headless match harness, sides swapped every match, against verdict bars set before the run.

### Hybrid AI: rules + learning
The rules decide and the RL policy holds bounded authority over a few action heads, extended only when a controlled trial shows the net beats the rule it would replace. Around it sits a substantial rule-based AI subsystem: A*-style pathfinding with map analysis and curated flag routes, geometric weapon-line solving, perception and threat modeling, a playmaking layer that issues squad-level directives (pincers, power plays, disengages), and per-opponent fire-pattern tracking that informs dodge behavior. A utility-scoring decision substrate, Orch-OR, is the production rule layer: it unifies candidate intents under one evaluation loop and adds a squad positioning doctrine measured on human replays. Bots are mode-aware and fill out matches in every game mode, including the run-based modes where they fight alongside players — and a social layer where they host and invite others into runs of their own.

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
High-frequency state updates use a custom binary snapshot encoding with per-entity presence flags for optional fields — roughly **60–70% smaller than the equivalent JSON** — plus run-length tile compression for map transfer. The snapshot format is negotiated per connection, so old and new clients coexist and either side can deploy first. The current version identifies projectiles by number, stores their positions in fixed point, and sends a long-lived bouncing shot in full only while it is new, after a bounce and periodically, otherwise as its id: about **36% smaller snapshots** on projectile-heavy rooms, and **~23 KB → ~2 KB** per snapshot on a room with a thousand bouncing shots in the air.

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

**2,272 commits** on `master` · **~240,000 lines of code** — ~190K TypeScript across 379 source files (176 server-side) and ~50K Python across 136 pipeline and tooling scripts, plus ~70K lines of tests in 416 test files · six arena modes plus two run-based modes · a library of hundreds of maps · deployed to a cloud host and live at [arcboundinteractive.com](https://arcboundinteractive.com). *(October 2026.)*

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
