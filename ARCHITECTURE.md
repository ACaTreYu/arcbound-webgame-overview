# Arcbound — Architecture Overview

A concept-level tour of how Arcbound is put together. The private repository is a monorepo containing the browser client, the game server, a deterministic simulation module shared by both, and the Python training pipeline for the game's reinforcement-learning bots.

## Monorepo Layout

![Monorepo layout](assets/diagram-1-monorepo.svg)

The core idea: **anything that determines gameplay outcomes lives in `shared/`** and is compiled into both bundles. The client and server literally execute the same movement and physics code, so a ship flown by its own client moves exactly as the server's rules say it should.

## Rooms and Game Modes

The server is organized around rooms. Every authenticated player lands in a persistent **staging arena** — a social hub that replaces a lobby screen — and moves from there into matches.

![Rooms and game modes](assets/diagram-2-rooms.svg)

Each mode implements a common hooks contract: the room calls into the active mode once per tick for win-condition checks, and the mode can veto other win paths (for example, kill-race modes disable flag wins). Modes are auto-selected from long-established map naming conventions, and the AI's objective model is mode-aware — bots switch roles when a map has no flags to fight over.

## Netcode Stack

![Netcode stack](assets/diagram-3-netcode.svg)

Key properties, bottom-up:

- **Transport** — WebRTC DataChannels configured for unordered, unreliable delivery give the browser UDP-like semantics; a WebSocket path handles signaling and serves as a fallback transport.
- **Reliability** — ARCnet adds what the game actually needs on top: an ordered, acknowledged channel for match events, a latest-wins channel for snapshots with forward error correction bundled into the same send, selective-ACK retransmission, and packet batching. Inputs and fire events go as compact unreliable packets beside it, so state and input never head-of-line-block each other. The library's adaptive snapshot pacing is off in the game: a snapshot goes out every 33 Hz tick.
- **Encoding** — snapshots use a compact binary format with per-entity flag bits marking which optional fields are present, cutting bandwidth roughly 60–70% versus JSON; tile grids transfer with run-length encoding. The format version is negotiated per connection; the current one sends projectiles by number in fixed point and long-lived bouncing shots as an id between full updates.
- **Time** — each position report carries the time it was true on the sender's clock. The server maps that clock onto its own by the fastest recent delivery and relays each position with its age; a watching client anchors server time to its own clock the same way and draws remote ships behind an adaptive delay sized from measured lateness, instead of inferring timing from jittery packet arrivals.

### Relay authority and remote ships

![Relay authority](assets/diagram-4-prediction.svg)

Both simulations run the identical fixed-timestep code from `shared/`, with frame-time clamping on the client to avoid death spirals. Each client is authoritative for its own ship, as in the classic game: it moves at once with no rewind, and it detects hits against its own true position (the victim decides; the server de-duplicates the report and applies the damage). The server keeps health, scores, rules and the bots, relays every timed position, and in the run-based modes refuses a position that would put a ship inside a wall. Remote ships are interpolated between those real positions on the sender's own timeline, with a short capped extrapolation tail.

## AI Subsystem

Two layers cooperate inside the server process:

![AI subsystem](assets/diagram-5-ai.svg)

The learned policy has separate actor and critic backbones with discrete heads for movement, firing, and aim (aim expressed as target-relative offset bins). Inference is a few small matrix multiplies per decision — cheap enough to run for every bot inside the game server with no external dependencies.

## The RL Loop: Collect → Train → Export → Deploy → Infer

![The RL loop](assets/diagram-6-rlloop.svg)

The loop runs unattended: guardrails (rollout validation, pre-deploy evaluation, closed-loop go-live verification, run-history logging) are always on, and a failed deployment rolls back to the previous known-good policy. Companion tooling renders training metrics, heatmaps, and per-run infographics.

## Map Pipeline & Editor Integration

![Map pipeline and editor integration](assets/diagram-7-maps.svg)

Arcbound preserves full round-trip compatibility with the classic binary map format: maps made two decades ago load unchanged, and maps written by Arcbound's tooling open cleanly in the companion editor (see **ac-map-editor-overview**). The server doubles as the editor's publishing backend, so the authoring-to-live-rotation path is a single upload. Parsed maps also feed the AI layer, which precomputes routes and spatial digests per map.

## Testing

Simulation, protocol, AI, and mode logic carry Vitest suites — collision edge cases, round flow, flag logic, pathfinding, perception, balance rules, and reserved-name handling among them. Determinism in `shared/` makes the physics tests exact rather than tolerance-based.

---

*See [README.md](README.md) for the feature overview and [SETUP.md](SETUP.md) for the toolchain.*
