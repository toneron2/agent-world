# AgentWorld

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE) [![Release](https://img.shields.io/github/v/release/toneron2/agent-world)](https://github.com/toneron2/agent-world/releases)

**Spatial visualization for multi-agent AI systems.** It works. It was the wrong answer. This
README is the postmortem, which is the only part still worth reading.

https://youtu.be/wxGq3uq7hfs?is=kwMKqe64EKYzWcTN

![AgentWorld Overview](docs/images/hero.png)

---

## The idea

I watched Moonshot AI debut K2.5. Somewhere in it was a short, dark clip of agent swarms
actually interacting, visually, in motion. Not a log file. Not a trace viewer. A picture of a
system thinking.

My immediate reaction was "I can do better than that."

I'd also just worked through IndyDevDan's
[The One Agent to RULE them ALL](https://youtu.be/p0mrXfwAbCg), which puts real-time
observability next to the orchestrator and agent lifecycle management as one of three pillars
of multi-agent work. That framing is correct. If five agents are running across three
workspaces, you should be able to point at the one that is stuck.

So: make the agents visible. And if you are making them visible anyway, why settle for boxes
and arrows?

## The wrong turn

Make it Zelda.

Different characters in different roles. Rooms you walk between. Artifacts you pick up, carry,
and hand off. NPCs transacting in a little economy of sprites doing your work.

I built that. 7 role sprites, 5 animation states, 19 event types, 10 Bevy plugins compiled to
WASM, a SQLite bridge, a Preact HUD, 8 procedural Web Audio sounds, three themed rooms with
ambient particles, portal transitions with particle bursts, and a 27-section manual with 24
annotated screenshots.

It worked. It added nothing.

## Why

The question I actually had was "which agent is stuck, and on what." AgentWorld answered it by
making you watch a sprite walk across a room. The scrolling text list in the right-hand panel
was the only part anyone ever read. Everything else was an expensive way of not reading it.

The numbers are the honest summary:

| | |
|---|---|
| Source | ~5,800 lines (4,400 Rust, 1,400 TypeScript) |
| Build artifacts on disk | **17 GB** |
| Part of the UI that answered the question | the text list |

A game engine, a WASM toolchain, a pinned Rust version and a WebGL2 context, to render
information that `tail -f` renders for free.

The metaphor is the actual mistake, not the implementation. A game world is built to reward a
player who has hours to explore it. Observability is built for an operator who has four
seconds and one question. I copied the architecture of the first thing while needing the
second, and the copy was worse at being a game than Zelda and worse at being an instrument
than a log file.

Then there is the unglamorous part. To see your agents you install Rust 1.89, add a WASM
target, install Trunk, install Bun, start two servers and open a URL with a query parameter.
Nobody does that to check on a job.

## What replaced it

One HTML page. A connection dot, a live activity strip, and a searchable event stream showing
agent name, tool, event type and timestamp. No sprites, no rooms, no sound, no build step. It
answers the question on sight, and it is boring.

That is what the 17 GB bought: **observability is a reading problem, not a rendering problem.**
The interesting-looking version of the answer was not the useful one.

The Moonshot clip was beautiful. It was also a rendering of a system rather than an instrument
for operating one. Those are different products, and I built the first while believing it was
the second. The whole thing went up in five days, 27 February to 3 March 2026, which is the
part that should have been the warning: nothing that answers a real operational question is
that easy to finish.

---

## What is actually in here

If you want the code, it builds and runs. Demo mode needs no event source.

```
agent-world/
├── crates/core/     # Provider-agnostic types + event-sourced state, zero framework deps
├── crates/game/     # Bevy 0.18 rendering, 10 plugins, compiled to WASM via WebGL2
├── bridge/          # SQLite -> WorldEvent translator (Bun + WebSocket)
└── frontend/src/    # Preact HUD overlay, 31 KB built
```

World state is a pure projection of the event stream, so replay and time travel come free.
That part was a good idea and is the piece worth stealing.

```bash
rustup target add wasm32-unknown-unknown && cargo install trunk
cd frontend && bun install && cd ..
trunk serve --address 0.0.0.0 --port 8080     # demo mode, 5 scripted agents, offline
```

Live mode wants `bun run bridge/server.ts --replay` alongside it, then
`http://localhost:8080/?ws=ws://localhost:9090/ws`. The bridge reads any SQLite table shaped
like `(hook_event_type, event_category, team_name, agent_name, agent_type, payload, summary,
timestamp)`. For anything else, emit the 19 `WorldEvent` types from
[`crates/core/src/events.rs`](crates/core/src/events.rs) over a WebSocket.

The full 27-section manual is [`docs/manual.html`](docs/manual.html), self-contained, dark
themed, 24 screenshots. It is a genuinely thorough document about a thing that should not
exist.

### Screenshots

| | |
|---|---|
| ![Sprites](docs/images/agentworld-sprite-animations.png) | Role sprites, status rings, thought bubbles |
| ![Inventory](docs/images/agentworld-inventory.png) | Inspector panel with carried artifacts |
| ![Portals](docs/images/agentworld-portal-particles.png) | Portal transitions between rooms |
| ![Shell](docs/images/agentworld-react-shell.png) | Roster, world view, event log, minimap |

---

## Status

Archived experiment. v0.1.0 is the only release and there will not be another. No issues, no
roadmap, no maintenance. It stays public because the postmortem is more useful than the code
and deleting it would delete the lesson too.

MIT. Copyright 2026 TODOMODO.IO AGENCY LLC. Take any of it.
