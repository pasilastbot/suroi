# System Architecture

<!-- @tier: 1 -->
<!-- @see-also: docs/subsystems/objects/, docs/subsystems/packets/, docs/subsystems/game-loop/ -->
<!-- @source: common/src/, server/src/, client/src/ -->
<!-- @updated: 2026-03-04 -->

## Overview

Suroi is a **client-server** multiplayer game. The server runs authoritative game simulation; the client renders the world and sends player inputs. All communication uses a custom binary protocol over WebSockets.

The codebase is organized as a **Bun monorepo** with four workspace packages.

## Monorepo Structure

```
suroi/
├── common/          # Shared code imported by both client and server
│   └── src/
│       ├── constants.ts            # GameConstants, enums (ObjectCategory, Layer, etc.)
│       ├── typings.ts              # Shared TypeScript types
│       ├── defaultInventory.ts     # Default player inventory
│       ├── definitions/            # All game object definitions
│       │   ├── buildings.ts
│       │   ├── bullets.ts
│       │   ├── obstacles.ts
│       │   ├── modes.ts
│       │   └── items/              # guns, melees, throwables, perks, etc.
│       ├── packets/                # Binary network packets
│       │   ├── packet.ts           # Base Packet class, PacketType enum
│       │   ├── packetStream.ts     # PacketStream, Packets array
│       │   └── *.ts                # Individual packet implementations
│       └── utils/                  # Shared utilities
│           ├── suroiByteStream.ts  # Game-specific binary serializer
│           ├── byteStream.ts       # Base byte stream
│           ├── gameObject.ts       # BaseGameObject, CommonObjectMapping
│           ├── objectDefinitions.ts# ObjectDefinitions class, DefinitionType
│           ├── hitbox.ts           # Hitbox types (circle, rect, polygon, group)
│           ├── math.ts             # Angle, Geometry, Collision, etc.
│           ├── vector.ts           # Vec, Vector
│           └── terrain.ts          # Terrain, River, FloorTypes
│
├── server/          # Game server (Bun runtime)
│   └── src/
│       ├── server.ts               # Entry point: cluster setup, HTTP/WebSocket
│       ├── game.ts                 # Game class: 40 TPS tick loop
│       ├── gameManager.ts          # Multi-game management, GameContainer
│       ├── map.ts                  # GameMap: procedural map generation
│       ├── gas.ts                  # Gas zone logic
│       ├── team.ts                 # Custom teams
│       ├── pluginManager.ts        # Event-based plugin system
│       ├── objects/                # Server-side object implementations
│       ├── inventory/              # Inventory, weapons, items
│       ├── data/                   # Static game data (maps, gas stages, loot tables)
│       ├── plugins/                # Built-in server plugins
│       └── utils/                  # Server utilities (config, grid, rate limiting)
│
├── client/          # Browser game client
│   ├── src/
│   │   ├── index.ts                # Entry point: Game.init()
│   │   └── scripts/
│   │       ├── game.ts             # Main Game class: WebSocket, input, PixiJS app
│   │       ├── objects/            # Client-side object rendering (PixiJS)
│   │       ├── managers/           # Camera, sound, particles, input, UI, map, etc.
│   │       ├── console/            # In-game debug console
│   │       └── utils/              # Constants, crosshairs, tweens, translations
│   ├── src/scss/                   # SCSS stylesheets
│   ├── src/translations/           # HJSON i18n files
│   ├── vite/                       # Vite config + custom plugins
│   └── public/img/                 # Game sprites, organized by map variant
│
└── tests/           # Validators and stress tests
    └── src/
        ├── validateDefinitions.ts  # Validates all definition objects
        ├── validateSvgs.ts         # Validates SVG assets
        └── stressTest.ts           # Performance stress test
```

Path alias: `@common/*` maps to `../common/src/*` in both `client` and `server`.

### Key Entry Points (@file)

| Role | File |
|------|------|
| Server entry | @file server/src/server.ts |
| Game tick loop | @file server/src/game.ts |
| Client entry | @file client/src/index.ts |
| Main Game (client) | @file client/src/scripts/game.ts |
| Definitions index | @file common/src/definitions/index.ts |
| Packet definitions | @file common/src/packets/packetStream.ts |

## Component Map

```
┌─────────────────────────────────────────────────────────────────────┐
│                           BROWSER (Client)                          │
│                                                                     │
│  ┌──────────┐  ┌──────────────┐  ┌──────────────────────────────┐ │
│  │ Svelte 5 │  │  PixiJS v8   │  │   Input / Managers           │ │
│  │   (UI)   │  │  (Renderer)  │  │  (camera, sound, particles)  │ │
│  └────┬─────┘  └──────┬───────┘  └──────────────┬───────────────┘ │
│       └───────────────┴──────────────────────────┘                 │
│                              │                                      │
│                     ┌────────┴─────────┐                           │
│                     │   game.ts (Game) │                           │
│                     │ WebSocket client  │                           │
│                     └────────┬─────────┘                           │
└──────────────────────────────┼──────────────────────────────────────┘
                               │ WebSocket (binary frames)
                               │ PacketStream (SuroiByteStream)
┌──────────────────────────────┼──────────────────────────────────────┐
│                     ┌────────┴─────────┐                           │
│                     │   server.ts      │   SERVER (Bun + cluster)  │
│                     │ HTTP + WebSocket │                            │
│                     └────────┬─────────┘                           │
│                              │                                      │
│                     ┌────────┴─────────┐                           │
│                     │  GameManager     │                           │
│                     │  (GameContainer) │                           │
│                     └────────┬─────────┘                           │
│                              │ spawns workers                      │
│              ┌───────────────┼───────────────┐                     │
│      ┌───────┴───────┐ ┌─────┴─────┐ ┌───────┴───────┐           │
│      │  Game #1      │ │  Game #2  │ │  Game #N      │           │
│      │  40 TPS loop  │ │  ...      │ │  ...          │           │
│      └───────────────┘ └───────────┘ └───────────────┘           │
└─────────────────────────────────────────────────────────────────────┘
```

## Client-Server Communication

All game data is exchanged over a single persistent WebSocket connection using **binary frames**. The custom `SuroiByteStream` format (backed by `ArrayBuffer`) packs data densely, minimizing bandwidth.

### Connection Lifecycle

```
Client                                          Server
  │                                               │
  │── HTTP GET /api/serverInfo ──────────────────>│  (find available game)
  │<─ { address, protocolVersion } ───────────────│
  │                                               │
  │── WebSocket upgrade ─────────────────────────>│
  │── JoinPacket (name, skin, team, etc.) ───────>│
  │<─ JoinedPacket (player ID, map data) ─────────│
  │                                               │
  │   [game loop — every frame or tick]            │
  │── InputPacket (movement, actions) ───────────>│  (client → server)
  │<─ UpdatePacket (objects, state, killfeed) ────│  (server → client, 40 TPS)
  │                                               │
  │── SpectatePacket (after death) ──────────────>│
  │<─ GameOverPacket (final stats) ───────────────│
```

**Direction key:**
- **Client → Server:** `JoinPacket`, `InputPacket`, `SpectatePacket`, `ReportPacket`
- **Server → Client:** `JoinedPacket`, `MapPacket`, `UpdatePacket`, `KillPacket`, `GameOverPacket`, `PickupPacket`

## Server Architecture

### Multi-Process Model

The server uses Node's `cluster` module. The **primary process** runs `GameManager` and handles HTTP/WebSocket routing. Each active game runs in a **worker process** via `GameContainer`.

```
Primary Process (server.ts)
├── GameManager
│   ├── GameContainer #1 (worker)  → Game instance, 40 TPS
│   ├── GameContainer #2 (worker)  → Game instance, 40 TPS
│   └── ...
└── HTTP/WebSocket server
    ├── /api/serverInfo     → list available games
    ├── /api/getGame        → join a specific game
    └── /play/:id           → WebSocket upgrade → player connection
```

### Game Tick Loop (40 TPS)

Each `Game` instance runs its own `setInterval` at `GameConstants.tps = 40` ticks per second (25 ms/tick). Each tick:

1. Process all pending `InputPacket`s from connected players
2. Update physics (movement, collisions using `Grid` spatial index)
3. Update bullets, explosions, projectiles
4. Update gas zone
5. Resolve player actions (reload, use item, interact, etc.)
6. Serialize and broadcast `UpdatePacket` to all players

### Spatial Grid

`server/src/utils/grid.ts` implements a spatial hash grid (`GridSize = 32` units) for fast broad-phase collision detection.

## Client Architecture

### Rendering Pipeline

```
WebSocket message
    ↓
PacketStream.deserialize() → UpdatePacket data
    ↓
Game.onMessage() → dispatches to object handlers
    ↓
GameObject.updateFromData() / GameObject.updateFull() [PixiJS objects]
    ↓
PixiJS Application.ticker → renders frame
```

### Object Lifecycle (Client)

Client game objects extend `Container` (PixiJS). They are created, updated, and destroyed in sync with server `UpdatePacket` data. Each category has its own class in `client/src/scripts/objects/`.

### Managers

The client game is managed by a set of singleton manager classes in `client/src/scripts/managers/`:

| Manager | Responsibility |
|---------|----------------|
| `CameraManager` | Viewport pan/zoom |
| `InputManager` | Keyboard/mouse/touch input |
| `SoundManager` | Audio playback via `@pixi/sound` |
| `ParticleManager` | Particle effects |
| `MapManager` | Minimap rendering |
| `UIManager` | Svelte-backed HUD updates |
| `GasManager` | Gas zone rendering |
| `EmoteWheelManager` | Emote selection UI |
| `PerkManager` | Active perk display |
| `ScreenRecordManager` | Screen recording |

## Shared Code (`common`)

The `common` package is the single source of truth for:
- All **game object definitions** (`common/src/definitions/`)
- **Packet schemas** and serialization (`common/src/packets/`)
- **Object base classes** and category mapping (`common/src/utils/gameObject.ts`)
- **Math, hitboxes, vectors** (`common/src/utils/`)
- **Game constants** (`common/src/constants.ts`)

Changes to `common` often affect both client and server. Changes to serialization format always require a `GameConstants.protocolVersion` bump.

## Documentation Index

See [content-plan.md](content-plan.md) for the full documentation index and status.

## Subsystem References

Navigate to Tier 2 for subsystem details:

- [Definitions](subsystems/definitions/) — Game object definitions (guns, obstacles, buildings, etc.)
- [Packets](subsystems/packets/) — Binary protocol and serialization
- [Object Model](subsystems/objects/) — Shared object hierarchy and per-category mapping
- [Game Loop](subsystems/game-loop/) — Server tick loop, physics, state management
- [Inventory](subsystems/inventory/) — Player inventory, weapons, items
- [Map Generation](subsystems/map/) — Procedural map generation, loot tables
- [Gas System](subsystems/gas/) — Battle royale gas zone mechanics
- [Plugin System](subsystems/plugins/) — Event-based server plugin architecture
- [Rendering](subsystems/rendering/) — PixiJS client rendering pipeline
- [Input](subsystems/input/) — Client input handling
- [UI & Translations](subsystems/ui/) — HUD (jQuery), Svelte (news/changelog), i18n
- [Game Console](subsystems/console/) — In-game debug console, commands, cvars
- [Validation](subsystems/validation/) — Definition and SVG validators

## Related Documents

- **Tier 1:** [description.md](description.md) — What Suroi is and who it's for
- **Tier 1:** [datamodel.md](datamodel.md) — Game entities and their relationships
- **Tier 1:** [protocol.md](protocol.md) — Network protocol details
- **Tier 1:** [development.md](development.md) — Setup and development guide

## Known Issues / Tech Debt

- **Client UI stack:** Mix of jQuery (HUD), Svelte (news/changelog), and vanilla DOM. No unified UI framework.
- **Game loop:** Worker processes communicate via `process.send`; no shared memory for game state.
