# AgentWorld: Greenfield Architecture Stub
## A Game-Native Visibility and Interaction Layer for Multi-Agent Systems

---

## The Core Thesis

Current agent observability is stuck in two inadequate paradigms: terminal text (IndyDevDan/TMUX — powerful but inaccessible) and workflow boxes-and-arrows (Dify/n8n/LangGraph — oversimplified and static). Neither will displace Office-class productivity tools because neither creates a *space you inhabit*.

Games solved this decades ago. An ARPG like Zelda gives you: real-time entity tracking, spatial awareness of many autonomous actors, inventory/capability management, visible state transitions, interaction at a distance, and a sense of *place*. The insight is that agents aren't workflow nodes — they're characters with state, capabilities, goals, and relationships. The human isn't a workflow designer — they're a player in a world.

**Design north star:** If you can't explain the system state to a non-technical person by pointing at the screen and saying "see that agent? it's carrying your document to the review agent, and that sparkle means the API tool just fired" — the visualization has failed.

---

## Architecture Layers

```
┌─────────────────────────────────────────────────┐
│  RENDERING LAYER (Phaser 3 / PixiJS)            │
│  Tilemaps, sprites, particles, camera, HUD      │
├─────────────────────────────────────────────────┤
│  INTERACTION LAYER (React shell + game bridge)   │
│  Panels, inspectors, message composer, timeline  │
├─────────────────────────────────────────────────┤
│  WORLD STATE (Event-sourced entity store)        │
│  Agents, artifacts, tools, rooms, messages       │
├─────────────────────────────────────────────────┤
│  EVENT BUS (WebSocket + local event emitter)     │
│  Normalized events from all adapters             │
├─────────────────────────────────────────────────┤
│  ADAPTER LAYER (Pluggable bridges)               │
│  MCP ↔ EventBus | TMUX ↔ EventBus | OpenClaw ↔  │
│  n8n ↔ EventBus | Claude Code ↔ EventBus | ...  │
└─────────────────────────────────────────────────┘
```

---

## Core Abstractions

### 1. Entity (base)

Everything in the world is an Entity. Entities have position, visual representation, and state.

```typescript
interface Entity {
  id: string;
  type: 'agent' | 'artifact' | 'tool' | 'portal' | 'zone';
  position: { x: number; y: number; room: string };
  sprite: SpriteConfig;       // visual representation
  state: Record<string, any>; // type-specific state
  metadata: Record<string, any>;
  createdAt: number;
  updatedAt: number;
}
```

### 2. Agent (the playing characters)

Agents are autonomous entities with goals, capabilities (tools), and inventory (artifacts they're working on). They move, they act, they communicate.

```typescript
interface Agent extends Entity {
  type: 'agent';
  name: string;
  role: string;              // "researcher", "coder", "reviewer", etc.
  provider: string;          // "claude", "gpt", "openclaw", "local"
  status: 'idle' | 'thinking' | 'acting' | 'waiting' | 'error' | 'paused';
  equippedTools: string[];   // tool entity IDs currently available
  inventory: string[];       // artifact entity IDs currently held
  currentTask: TaskState | null;
  health: number;            // metaphor for error rate / reliability
  energy: number;            // metaphor for token budget / rate limit remaining
  thought: string | null;    // current chain-of-thought (speech bubble)
}

interface TaskState {
  id: string;
  description: string;
  progress: number;          // 0-1
  subtasks: TaskState[];
  assignedBy: string;        // entity ID of assigner (human or agent)
}
```

### 3. Artifact (the gold coins / items)

Artifacts are work products — documents, code files, data, images, plans. They're visible, collectible, transferable.

```typescript
interface Artifact extends Entity {
  type: 'artifact';
  artifactKind: 'document' | 'code' | 'data' | 'image' | 'plan' | 'message-bundle';
  name: string;
  contentRef: string;        // URI to actual content
  owner: string | null;      // entity ID of current holder
  quality: number;           // visual indicator: dull → gleaming
  size: 'small' | 'medium' | 'large';  // visual scale
  history: ArtifactEvent[];  // provenance chain
}
```

### 4. Tool (the weapons / abilities)

Tools are capabilities agents can invoke. When an agent uses a tool, it's visually analogous to a weapon swing or spell cast — there's an animation, an effect, and a result.

```typescript
interface Tool extends Entity {
  type: 'tool';
  toolKind: 'mcp' | 'api' | 'shell' | 'browser' | 'file' | 'custom';
  name: string;
  description: string;
  provider: string;          // which MCP server or service
  cooldown: number;          // rate limit as cooldown timer
  power: number;             // visual intensity of effect
  range: 'self' | 'target' | 'area';  // does it affect the agent, a target, or a zone
  equipped_by: string[];     // which agents have this tool
}
```

### 5. Room / Zone (the map)

The world is spatial. Rooms represent logical groupings — a project workspace, a review area, a staging zone, an archive. Agents move between rooms.

```typescript
interface Room {
  id: string;
  name: string;
  tilemap: string;           // reference to tilemap asset
  bounds: { width: number; height: number };
  portals: Portal[];         // connections to other rooms
  ambientEffects: string[];  // background particles, lighting
  purpose: string;           // "workspace", "review", "deploy", "archive"
}

interface Portal extends Entity {
  type: 'portal';
  targetRoom: string;
  targetPosition: { x: number; y: number };
}
```

### 6. Message (the visible communication)

Messages between entities are *visible*. They're projectiles, speech bubbles, particle trails — not hidden log lines.

```typescript
interface AgentMessage {
  id: string;
  from: string;              // entity ID
  to: string | string[];     // entity ID(s), or 'broadcast'
  channel: 'direct' | 'broadcast' | 'tool-call' | 'tool-result' | 'human';
  content: string;
  contentPreview: string;    // truncated for speech bubble
  timestamp: number;
  visualStyle: 'projectile' | 'bubble' | 'beam' | 'ripple' | 'scroll';
}
```

---

## Event System (the nervous system)

Everything flows through events. The game world is a *projection* of the event stream — this means you get replay, time-travel debugging, and recording for free.

```typescript
type WorldEvent =
  | { type: 'agent:spawn'; agent: Agent }
  | { type: 'agent:move'; agentId: string; to: Position }
  | { type: 'agent:status-change'; agentId: string; status: AgentStatus; reason?: string }
  | { type: 'agent:think'; agentId: string; thought: string }
  | { type: 'agent:equip-tool'; agentId: string; toolId: string }
  | { type: 'agent:use-tool'; agentId: string; toolId: string; target?: string; params?: any }
  | { type: 'agent:tool-result'; agentId: string; toolId: string; result: any; success: boolean }
  | { type: 'agent:pick-up'; agentId: string; artifactId: string }
  | { type: 'agent:drop'; agentId: string; artifactId: string }
  | { type: 'agent:transfer'; fromId: string; toId: string; artifactId: string }
  | { type: 'artifact:create'; artifact: Artifact }
  | { type: 'artifact:transform'; artifactId: string; changes: Partial<Artifact> }
  | { type: 'artifact:quality-change'; artifactId: string; quality: number }
  | { type: 'message:send'; message: AgentMessage }
  | { type: 'message:deliver'; messageId: string }
  | { type: 'room:enter'; agentId: string; roomId: string }
  | { type: 'room:exit'; agentId: string; roomId: string }
  | { type: 'human:command'; targetId: string; command: string }
  | { type: 'human:inject-message'; targetId: string; content: string }
  | { type: 'governance:decision'; rule: string; result: 'allow' | 'deny'; reason: string }
  | { type: 'error:agent'; agentId: string; error: string };
```

### Event Store

```typescript
class EventStore {
  private events: WorldEvent[] = [];
  private listeners: Map<string, Set<(event: WorldEvent) => void>> = new Map();

  emit(event: WorldEvent): void { /* append + notify */ }
  subscribe(type: string, handler: (event: WorldEvent) => void): () => void;
  replay(from?: number, to?: number): WorldEvent[];
  snapshot(): WorldState;     // current projected state
}
```

---

## Adapter Layer (bridges to real systems)

Each adapter translates real-world agent system events into WorldEvents. This is the critical abstraction — the game world doesn't care where agents come from.

### Adapter Interface

```typescript
interface AgentSystemAdapter {
  name: string;
  connect(config: AdapterConfig): Promise<void>;
  disconnect(): Promise<void>;
  onEvent(handler: (event: WorldEvent) => void): void;

  // Bidirectional: human can send commands back through the adapter
  sendCommand(agentId: string, command: string): Promise<void>;
  sendMessage(agentId: string, message: string): Promise<void>;
}
```

### Priority Adapters to Build First

```
1. MCP Adapter
   - Connects to MCP servers via stdio/SSE
   - Maps tool_call / tool_result to agent:use-tool / agent:tool-result
   - Maps resource access to artifact:create / agent:pick-up
   - Your 7 MCP servers with ~205 tools become equippable items

2. TMUX/Terminal Adapter
   - Watches tmux panes for output patterns
   - Maps Claude Code sessions to agent entities
   - Captures shell commands as tool use
   - This bridges the IndyDevDan observability layer

3. Process/Container Adapter
   - Monitors running agent processes (Docker, systemd)
   - Maps process lifecycle to agent:spawn / agent status
   - CPU/memory maps to agent energy/health metaphors

4. OpenClaw Adapter (future)
   - Bridges OpenClaw's messaging protocol
   - Maps OpenClaw skills to tools
   - Maps persistent memory to agent inventory

5. n8n/Workflow Adapter
   - Monitors n8n webhook triggers and workflow executions
   - Maps workflow nodes to tool use sequences
   - Your Geekom A6 n8n orchestration becomes visible
```

---

## Rendering: Why Phaser 3

The rendering layer needs to be a *real game engine*, not a React animation library pretending to be one. Phaser 3 is the right choice for the greenfield because:

- Mature 2D engine, huge ecosystem, Zelda-style games are its sweet spot
- Runs in browser (no native install barrier)
- Tilemaps, sprite sheets, particle systems, cameras, physics all built in
- Can embed in a React shell (Phaser handles the canvas, React handles the chrome)
- Active community with extensive top-down RPG examples and assets
- Exports to desktop via Electron when you want a standalone app later

### Visual Language

| System Concept          | Game Metaphor           | Visual Treatment                    |
|-------------------------|-------------------------|-------------------------------------|
| Agent                   | Player character / NPC  | Animated sprite with role-based skin|
| Agent thinking          | Speech bubble           | Floating text + thinking particles  |
| Agent idle              | Standing animation      | Gentle idle loop                    |
| Agent working           | Walking / action anim   | Movement toward target + particles  |
| Agent error             | Damage flash            | Red flash + shake + ❌ particle     |
| Tool use                | Weapon swing / spell    | Directional animation + SFX        |
| Tool result (success)   | Hit effect              | Sparkle / impact particles          |
| Tool result (failure)   | Miss / fizzle           | Poof / smoke particles              |
| Artifact                | Collectible item        | Glowing sprite, quality = brightness|
| Artifact transfer       | Item toss               | Projectile arc between agents       |
| Message (direct)        | Arrow / beam            | Particle trail, sender→receiver     |
| Message (broadcast)     | Area pulse              | Expanding ring from sender          |
| Room                    | Dungeon room / area     | Tilemap with themed decoration      |
| Portal between rooms    | Door / staircase        | Animated portal sprite              |
| Governance gate         | Locked door / barrier   | Barrier that flashes on deny        |
| Human command           | Player action           | Click target + command radial menu  |
| Rate limit              | Cooldown timer          | Circular cooldown overlay on tool   |
| Token budget            | Energy bar              | Depleting bar above agent           |
| Error rate              | Health bar              | Standard HP bar                     |

### Camera System

- **Free cam**: Pan/zoom around the world freely
- **Follow cam**: Lock onto a specific agent and follow their journey
- **Overview cam**: Zoom out to see all rooms and agent positions (minimap)
- **Theater cam**: Automated camera that follows the action (for demos/recordings)

---

## React Shell (the HUD and chrome)

The game canvas lives inside a React application that provides:

```
┌──────────────────────────────────────────────────────────┐
│ [Toolbar: room selector, camera mode, speed, record]     │
├──────────────┬───────────────────────┬───────────────────┤
│              │                       │                   │
│  Agent       │                       │  Inspector        │
│  Roster      │    GAME CANVAS        │  Panel            │
│  (sidebar)   │    (Phaser 3)         │  (detail view     │
│              │                       │   of selected     │
│  - status    │                       │   entity)         │
│  - tasks     │                       │                   │
│  - quick     │                       │  - full state     │
│    actions   │                       │  - event history  │
│              │                       │  - raw logs       │
│              │                       │  - actions        │
│              │                       │                   │
├──────────────┴───────────────────────┴───────────────────┤
│ [Event Timeline - scrubbable, filterable, replayable]    │
│ ●━━━━━━━━━●━━━━━━●━━━━━━━━━━━━━━●━━━━━━━━━━━━━━━━━●━━━▶│
└──────────────────────────────────────────────────────────┘
```

### Key Panels

1. **Agent Roster** — Left sidebar, shows all agents with status icons, click to follow
2. **Inspector** — Right panel, shows full detail of whatever entity is selected in the game world
3. **Message Composer** — Pop-up when you want to inject a message to an agent (human → agent)
4. **Command Radial** — Right-click an agent in-game, get a radial menu: pause, resume, reassign, inspect, message
5. **Event Timeline** — Bottom scrubber, every event is a dot, scrub to replay, filter by type
6. **Minimap** — Corner overlay showing all rooms and agent positions

---

## Project Structure (the actual greenfield)

```
agent-world/
├── package.json
├── tsconfig.json
├── vite.config.ts                    # Vite for dev server + build
│
├── src/
│   ├── main.tsx                      # React entry point
│   ├── App.tsx                       # Shell layout (panels + canvas mount)
│   │
│   ├── core/                         # Framework-agnostic core
│   │   ├── types.ts                  # All interfaces from above
│   │   ├── event-store.ts            # Event sourcing engine
│   │   ├── world-state.ts            # Projected state from events
│   │   ├── entity-manager.ts         # CRUD + query for entities
│   │   └── clock.ts                  # Simulation clock (pause, speed, replay)
│   │
│   ├── adapters/                     # Bridges to real agent systems
│   │   ├── adapter-interface.ts      # Base adapter contract
│   │   ├── mock-adapter.ts           # Fake agents for development/demo
│   │   ├── mcp-adapter.ts            # MCP server bridge
│   │   ├── tmux-adapter.ts           # TMUX pane watcher
│   │   ├── process-adapter.ts        # System process monitor
│   │   └── websocket-adapter.ts      # Generic WebSocket bridge
│   │
│   ├── game/                         # Phaser 3 game layer
│   │   ├── config.ts                 # Phaser game configuration
│   │   ├── scenes/
│   │   │   ├── BootScene.ts          # Asset loading
│   │   │   ├── WorldScene.ts         # Main game scene
│   │   │   └── MinimapScene.ts       # Minimap overlay
│   │   ├── entities/
│   │   │   ├── AgentSprite.ts        # Agent visual + animations
│   │   │   ├── ArtifactSprite.ts     # Artifact visual
│   │   │   ├── ToolEffect.ts         # Tool use visual effects
│   │   │   ├── MessageProjectile.ts  # Message visualization
│   │   │   └── PortalSprite.ts       # Room transition visual
│   │   ├── systems/
│   │   │   ├── MovementSystem.ts     # Agent pathfinding + movement
│   │   │   ├── ParticleSystem.ts     # Effects and ambient particles
│   │   │   ├── CameraSystem.ts       # Camera modes + controls
│   │   │   ├── InteractionSystem.ts  # Click/hover/select handlers
│   │   │   └── SpeechBubbleSystem.ts # Thought/message bubbles
│   │   ├── maps/
│   │   │   ├── room-templates.ts     # Procedural room generation
│   │   │   └── tileset-config.ts     # Tileset definitions
│   │   └── assets/                   # Sprites, tilemaps, audio
│   │       ├── sprites/
│   │       ├── tilemaps/
│   │       ├── particles/
│   │       └── audio/
│   │
│   ├── ui/                           # React UI components
│   │   ├── Shell.tsx                 # Main layout container
│   │   ├── AgentRoster.tsx           # Left sidebar agent list
│   │   ├── InspectorPanel.tsx        # Right panel entity detail
│   │   ├── EventTimeline.tsx         # Bottom scrubber
│   │   ├── CommandRadial.tsx         # Right-click radial menu
│   │   ├── MessageComposer.tsx       # Human → agent message input
│   │   ├── Minimap.tsx               # React wrapper for minimap scene
│   │   ├── Toolbar.tsx               # Top bar controls
│   │   └── GameCanvas.tsx            # Phaser ↔ React bridge component
│   │
│   ├── hooks/                        # React hooks
│   │   ├── useWorldState.ts          # Subscribe to projected state
│   │   ├── useSelectedEntity.ts      # Currently selected entity
│   │   ├── useEventStream.ts         # Live event feed
│   │   └── useGameBridge.ts          # React ↔ Phaser communication
│   │
│   └── utils/
│       ├── sprite-mapper.ts          # Map agent roles to sprite configs
│       ├── layout-engine.ts          # Auto-arrange agents in rooms
│       └── event-filter.ts           # Timeline filtering logic
│
├── server/                           # Local bridge server (Node)
│   ├── index.ts                      # WebSocket server entry
│   ├── adapter-registry.ts           # Manages active adapters
│   ├── mcp-bridge.ts                 # MCP stdio/SSE → WebSocket
│   ├── tmux-bridge.ts                # tmux capture-pane → events
│   └── process-bridge.ts             # ps/docker → events
│
├── assets/                           # Raw art assets / Aseprite files
│   └── README.md
│
└── docs/
    ├── ARCHITECTURE.md               # This document
    ├── EVENT-CATALOG.md              # All event types documented
    ├── ADAPTER-GUIDE.md              # How to write a new adapter
    └── VISUAL-LANGUAGE.md            # Sprite/effect design guide
```

---

## Build Sequence (what to do in what order)

### Phase 0: Skeleton + Mock World (week 1-2)
Get the core loop running with fake data.

1. Scaffold Vite + React + TypeScript project
2. Implement `EventStore` and `WorldState` (core/*)
3. Create `MockAdapter` that emits a stream of fake events (agents spawning, moving, using tools, exchanging artifacts)
4. Stand up Phaser 3 inside React (`GameCanvas.tsx`)
5. Create `WorldScene` with a simple tilemap (one room, grid floor)
6. Create `AgentSprite` — colored circle with name label, animates between positions
7. Wire EventStore → Phaser scene (events move sprites)
8. **Milestone: Fake agents visibly moving and acting in a game world**

### Phase 1: Visual Language (week 2-3)
Make it look and feel like a game.

1. Replace circles with actual sprites (use free top-down RPG assets initially — LPC, Kenney, or similar)
2. Implement `ToolEffect` — particle burst when agent uses a tool
3. Implement `MessageProjectile` — visible arc/beam between agents
4. Implement `ArtifactSprite` — glowing items agents carry
5. Add speech bubbles for agent thoughts
6. Add status indicators (idle animation, thinking particles, error flash)
7. Implement basic camera controls (pan, zoom, follow)
8. **Milestone: Visually legible multi-agent activity that a non-technical person can follow**

### Phase 2: React Chrome (week 3-4)
Add the HUD and interaction layer.

1. Agent Roster sidebar
2. Inspector panel (click an entity, see its full state)
3. Event Timeline (scrubber at bottom)
4. Command Radial (right-click agent → pause, message, inspect)
5. Message Composer (type a message, inject into an agent)
6. **Milestone: Full interactive shell — watch, inspect, and interact with mock agents**

### Phase 3: First Real Adapter (week 4-6)
Connect to actual agent systems.

1. Build the WebSocket bridge server
2. Implement MCP adapter (your existing 7 servers become visible)
   - Tool calls → agent:use-tool events with ToolEffect visuals
   - Tool results → sparkle or fizzle effects
   - Each MCP server becomes a visible "armory" in a room
3. Test with a real Claude Code session or MCP interaction
4. **Milestone: Real agent activity visible in the game world**

### Phase 4: Multi-Room + TMUX (week 6-8)
Scale the world.

1. Implement room system — multiple tilemaps, portals between them
2. Auto-generate rooms from project/workspace structure
3. TMUX adapter — map panes to agents/rooms
4. Process adapter — running containers/services as ambient entities
5. **Milestone: Your actual lab environment represented as a navigable game world**

### Phase 5: Governance Visualization (week 8+)
This is where BROAD's architecture becomes visible.

1. Governance gates as barriers/checkpoints between zones
2. Allow/deny decisions visualized as gates opening/closing
3. Policy rules as visible "enchantments" on zones
4. Decision audit trail in the inspector
5. Skill layers as visible equipment loadouts on agents
6. **Milestone: Governance isn't a log file, it's a visible force in the world**

---

## Technology Choices: Summary

| Concern                | Choice              | Rationale                                  |
|------------------------|----------------------|--------------------------------------------|
| Game rendering         | Phaser 3             | Mature 2D engine, Zelda-style sweet spot   |
| UI chrome              | React + TypeScript   | Already in your stack, good ecosystem      |
| Build tooling          | Vite                 | Fast, modern, great TS support             |
| State management       | Custom event sourcing | Replay/time-travel is essential, not optional |
| Bridge server          | Node.js + ws         | Lightweight, same language as frontend     |
| MCP bridge             | @modelcontextprotocol/sdk | Native MCP client library            |
| Styling                | Tailwind             | Fast iteration on React panels             |
| Sprites (initial)      | Kenney / LPC assets  | Free, high quality, top-down RPG style     |
| Sprites (later)        | Custom via Aseprite   | Once visual language is proven             |
| Desktop packaging      | Electron (future)    | When you want standalone app               |

---

## Key Design Decisions to Make Early

1. **Isometric vs top-down?** Pure top-down (Zelda 1/Link to the Past) is simpler to implement and read. Isometric (Zelda: Link's Awakening remake) is prettier but harder. Recommend starting top-down.

2. **Procedural vs authored rooms?** Start procedural (auto-generate from project structure), evolve to letting users customize layouts.

3. **One world or multiple?** A single world per machine/lab initially. Multi-user shared worlds (team TMUX equivalent) is a Phase 5+ feature.

4. **Asset style?** 16-bit pixel art is fastest to produce, reads well at any zoom, and matches the Zelda aesthetic. Don't go 3D — it's a trap for this use case.

5. **Sound?** Yes, eventually. Ambient sounds per room, subtle audio cues for tool use, message arrival, errors. Sound is surprisingly important for background awareness — you can "hear" your agents working while focused elsewhere. Low priority but high impact.

---

## What This Becomes

Short term: A visually rich, game-like observability dashboard for your lab's agent systems. You watch your agents work like watching NPCs in a game.

Medium term: An interaction layer where you direct agents, reassign tasks, and pass messages through a spatial interface rather than a terminal. The virtual office you described.

Long term: The default interface for multi-agent systems. When Altman says "his agents working with your agents," this is how you *see* it happening. Agent-to-agent commerce, negotiation, and collaboration rendered as visible interaction in a shared world. The governance layer (BROAD) becomes the physics engine — the rules that determine what agents can and can't do in this world.

The paradigm shift: **You don't manage agents in a text editor. You manage them in a world.**
