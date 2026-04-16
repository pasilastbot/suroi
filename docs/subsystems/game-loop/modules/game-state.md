# Game Loop — Game State Module

<!-- @tier: 3 -->
<!-- @parent: ../README.md -->
<!-- @source: server/src/game.ts, server/src/gameManager.ts -->

## Purpose
Manages game lifecycle phases (lobby, loading, in-progress, ended), phase transitions, player join/leave mechanics, and game-ending conditions including stats aggregation and team mode state.

## Key Files
| File | Purpose | Complexity |
|------|---------|------------|
| `server/src/game.ts` | Game class lifecycle states and transitions | High |
| `server/src/gameManager.ts` | GameContainer and worker process management | Medium |
| `server/src/team.ts` | Team state and team mode logic | Medium |
| `common/src/definitions/modes.ts` | Gamemode definitions (TDM, BR, etc.) | Low |

## Game Lifecycle Phases

A game progresses through five distinct phases:

```
┌──────────┐  new Game()  ┌────────┐  minPlayersTimeout  ┌──────────┐
│  LOBBY   │ ─────────────▶ │Loading │ ─────────────────▶ │In Progress│
│allowJoin │               │        │                    │ allowJoin │
│= true    │              │        │                    │= false    │
└──────────┘               └────────┘                    └──────────┘
   │                                                        │
   │                                                        │ game.over = true
   │                                                        │ (when aliveCount <= 1)
   │                    New Game                           │
   │                    (or full)                          │
   │                                                        ▼
   │                                                    ┌────────────┐
   └────────────────────────────────────────────────────▶ │ENDED      │
                                                          │ (awaiting │
                                                          │  cleanup) │
                                                          └────────────┘
```

**Phase state variables:**
- `allowJoin: boolean` — whether new players can join
- `over: boolean` — game terminated (winner decided)
- `_started: boolean` — game progressed from loading to in-progress
- `startedTime: number` — timestamp when phase changed to in-progress

## Phase 1: LOBBY (Allow Join)

**Entry condition:** Game instantiated via `new Game()`
**State:**
```typescript
allowJoin = true;      // Players can join
over = false;          // Game not over
_started = false;      // Game not yet started
startedTime = Number.MAX_VALUE;  // Makes this game joinable-first
```

**Duration:** Variable; as soon as `minPlayerCount` reached (usually 2), timer starts
**Config:** @file server/src/utils/config.ts
```typescript
Config.minPlayersNeeded = 2;      // e.g., solos/duos
Config.minPlayerWaitTime = 5000;  // ms before starting with minimum players
```

**Transitions:**
- If min players + min wait time → `Loading` phase
- If max players reached → `Loading` phase (skip wait)
- If game duration max exceeded → game discarded (no one joins)

**Activity:** Players join, pick skins/perks. No game logic running.

## Phase 2: LOADING

**Entry condition:** Min players reached + (wait time or max players)
**State:**
```typescript
allowJoin = true;      // Still joinable (latecomers)
over = false;
_started = false;      // Not yet running game tick
startedTime = Number.MAX_VALUE;
```

**Duration:** ~2–3 seconds
**Purpose:** 
- Server finalizes map generation
- Airdrop schedule computed
- Perks distributed
- Gas stage timeline initialized

**Activity:** 
- Players queued but not yet spawned in game world
- `JoinedPacket` sent; client initializes UI/map

**Transitions:**
- After delay or if players ready → `InProgress` phase

## Phase 3: IN-PROGRESS (Game Running)

**Entry condition:** `minPlayersNeeded` reached + ready delay
**State:**
```typescript
allowJoin = false;     // NO new players (except spectators in team mode)
over = false;
_started = true;       // Game tick running
startedTime = now;     // Timestamp recorded for kill leader decay
```

**Duration:** Variable; depends on player eliminations
**Game logic active:**
- `game.tick()` runs 40 times per second
- Objects move, collide, deal damage
- Gas zone shrinks
- Airdrops drop

**Key condition check:** @file server/src/game.ts:516–540

```typescript
if (this._started && !this.over && this.now - this.startedTime > 5000) {
  // Game running for > 5 seconds; check end condition
  if (this.aliveCount <= 1) {
    // Game over: 1 or fewer alive
    this.announceWinner();
    this.over = true;
  }
}
```

**End conditions:**
- **Solos:** 1 player alive (last man standing)
- **Team modes:** 1 team alive OR all dead
- **Special:** Admin force-end command

## Phase 4: ENDED (Game Over)

**Entry condition:** `this.over = true`
**State:**
```typescript
allowJoin = false;
over = true;
_started = true;
```

**Duration:** ~5–10 seconds
**Activity:**
- `GameOverPacket` sent to all players with stats
- Spectate disabled (match over)
- Loot spawning stops
- Leaderboard computed

**Transitions:**
- Game worker keeps running briefly for stats persistence
- Then respawns new Game instance

## Player Join/Leave Logic

### Join During LOBBY (Phase 1)
- Player can freely join
- Assigned index in `connectedPlayers` set
- Receives `JoinedPacket` with map, objects
- Input processing begins immediately after `joined = true`

### Join During LOADING (Phase 2)
- Player can still join (latecomers)
- Receives partial game state (map, living players already spawned)
- Assigned spawn position (not collision-tested, may overlap)

### Join During IN-PROGRESS (Phase 3)
- **Solos:** NO new players (game already started, unfair)
- **Team modes:** YES, if team has slots (spectators can join)
  - Latecomer joins as spectator initially
  - Can become active if teammate dies before match ends

**Code:** @file server/src/gameManager.ts; Worker message filtering

### Leave at Any Time
- Player marked `disconnected = true`
- Removed from `connectedPlayers` set
- In team mode, affects team survival (team can extinct if all leave)
- Triggers checks for new end condition

## Player State Transitions

```
┌────────────┐  Input.Spawn  ┌───────┐  take_damage    ┌────────┐
│  Spawned   │ ─────────────▶ │ Alive │ ─────────────▶ │Downed  │
│new player  │                │active │                │        │
└────────────┘                └───────┘ ◀──────────────└────────┘
                                │        revive_action
                                │
                                │ take_damage (HP=0)
                                │
                                ▼
                              ┌───────┐
                              │ Dead  │
                              │killed │
                              └───────┘
```

**Player dead transition logic:** @file server/src/objects/player.ts:damage()
- Health reduced to <= 0
- Dead flag set (`this.dead = true`)
- Killer stored for kill feed
- Spectate eligibility set (can watch teammates or spectate)
- Loot dropped from inventory

## Team Mode State

In **team modes** (TDM, Domination):

### Team Creation
- Players assigned team ID on join (or pre-assigned via squad)
- Team object created if doesn't exist: `new Team(id, game, ...)`
- Teams persist until match end (individual players can join/leave)

**Code:** @file server/src/team.ts

### Team-Specific State
```typescript
Team {
  id: number,
  game: Game,
  players: Set<Player>,    // Current players on team
  alive: boolean,          // At least one player alive
  kills: number,           // Aggregate stats
  deaths: number,
  // ...
}
```

### Team Win Condition
- Only 1 team has `alive = true` (all other teams eliminated)
- OR match timer expires (highest kill count wins)

### Spectate in Team Mode
- Deceased player can spectate live teammates
- Spectate camera follows teammate's POV
- Once all teammates dead, falls back to Observer spectate
- Receives limited state (teammate position, visible objects only)

## Game Data Synchronization (GameManager ↔ Worker)

Game runs in Node.js Worker process; GameManager in main process needs to know state:

```
Worker (game.ts)              Main Process (gameManager.ts)
       │                              │
       init Game                      │
       tick() loop          ────────────────────────
       setGameData()                  │
       send message                   │
                                      │ on("message")
                                      ├─ Update _data.aliveCount
                                      ├─ Update _data.allowJoin
                                      └─ Notify waiting connections
       │
       more ticks...
```

**Message types:** @file server/src/gameManager.ts:14–32
```typescript
export enum WorkerMessages {
  UpdateTeamMode,
  UpdateMap,
  UpdateMapOptions,
  NewGame   // Trigger new game instance creation
}
```

**Data sync:** `game.setGameData()` sends `Partial<GameData>`:
```typescript
interface GameData {
  aliveCount: number;   // for matchmaking queue sorting
  allowJoin: boolean;
  over: boolean;
  startedTime: number;
}
```

## Stats Aggregation

When game ends, stats are computed:

```typescript
// Per player
player.kills
player.deaths
player.damageDealt
player.damageTaken
player.distanceTraveled

// Per team (team mode)
team.kills
team.deaths
team.place      // 1st, 2nd, 3rd
```

**Serialized in:** `GameOverPacket` sent to all players
**Used by:** Client UI leaderboard, player profile stats

## Complex Functions

### `game.tick()` — @file server/src/game.ts:313+
**Purpose:** Main game tick orchestrator
**Complexity:** ~200 lines
**Precondition:** `_started === true` and `!over`
**Implicit behavior:**
- Runs game logic (object updates, damage)
- Checks end condition after tick
- Broadcasts packets to players
- Returns early if game not started yet

### `game.setGameData()` — @file server/src/game.ts
**Purpose:** Send state update to GameManager
**Parameters:** `Partial<GameData>` — only changed fields
**Implicit behavior:**
- Serializes data to JSON
- Sends via worker.send() to parent process
- Parent process updates GameContainer._data

### `gameManager.handleNewGame()` — @file server/src/gameManager.ts
**Purpose:** Allocate new Game worker and track it
**Complexity:** ~20 lines
**Returns:** Promise<GameContainer> resolved when game ready
**Implicit behavior:**
- Forks worker process
- Sets up message listener
- Tracks game in `games` array
- Resolves waiting client connections once game enters LOBBY phase

## Configuration & Tuning

Game phase durations: @file server/src/utils/config.ts
```typescript
Config.minPlayersNeeded = 2;
Config.minPlayerWaitTime = 5000;    // ms before starting with min players
Config.gameMaxDuration = 600000;    // 10 minutes (discard if no one joins)
Config.endGameAwaitDuration = 10000; // 10 seconds after game over
```

End condition: @file server/src/game.ts:524
```typescript
if (this.aliveCount <= 1 && this.now - this.startedTime > 5000) {
  // At least 5 seconds in-game before declaring winner
}
```

## Related Documents
- **Tier 2:** [Game Loop](../README.md) — subsystem overview
- **Tier 2:** [Team System](../../team-system/README.md) — team state details
- **Tier 2:** [Game Modes](../../game-modes/README.md) — mode-specific rules
- **Tier 3:** [Player Update](player-update.md) — per-tick player logic
- **Tier 3:** [Object Update](object-update.md) — object lifecycle
- **Tier 3:** [Network Tick](network-tick.md) — packet broadcasting
- **Patterns:** [../patterns.md](../patterns.md) — subsystem patterns
