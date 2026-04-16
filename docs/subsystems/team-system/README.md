# Team System

<!-- @tier: 2 -->
<!-- @parent: docs/architecture.md -->
<!-- @source: server/src/team.ts -->

## Purpose

Manages player squads in team-based game modes. Groups players into teams with auto-assignment, color management, and team-aware gameplay mechanics. Supports Solo (1 player), Duo (2), and Squad (4) modes with dynamic team joining and lobby team creation.

## Key Files & Entry Points

| File | Purpose |
|------|---------|
| `server/src/team.ts` | `Team`, `CustomTeam`, `CustomTeamPlayer` classes; team lifecycle |
| `server/src/game.ts` | Game instantiates teams per mode; team assignment during player join |
| `server/src/objects/player.ts` | Player holds `_team` reference; `teamID` for custom team lobby |
| `server/src/inventory/action.ts` | `ReviveAction` — teammate revive mechanic |
| `common/src/typings.ts` | `CustomTeamMessage`, `CustomTeamPlayerInfo`, `CustomTeamMessages` enum |
| `common/src/packets/updatePacket.ts` | `teammates` field in UpdatePacket for team data sync |

## Architecture

### Overview

**Two-tier team structure:**

1. **Lobby Teams (`CustomTeam`)** — Pre-game team lobbies where players gather before joining a game
   - Lead by team captain (first player to create or join)
   - Supports up to ~100 players across multiple lobbies
   - Linked to `Game` via `teamID` string
   - Managed by `GameManager` for matchmaking

2. **In-Game Teams (`Team`)** — Game instance teams created per match
   - Size determined by game mode (1/2/4 players)
   - Assigned when player joins via `game.addPlayer()`
   - Tracks kills, team-aware color indexes
   - Support auto-fill and custom team IDs

```
GameManager (lobby layer)
  ↓
CustomTeam ← CustomTeamPlayer (waits for match)
  ↓
Game(TeamMode=Duo)
  ↓
Team[] (max 2 players each)
  ↓
Player (holds _team reference)
```

## Team Classes

### `Team` (In-Game)

Located: `@file server/src/team.ts:6`

Represents a single team within a game instance.

**Constructor:**
```typescript
constructor(id: number, autoFill = true)
```

**Properties:**

| Property | Type | Purpose |
|----------|------|---------|
| `id` | `number` | Unique team ID within game (assigned by `game.nextTeamID`) |
| `players` | `readonly Player[]` | Array of current team members |
| `_indexMapping` | `Map<Player, number>` | Maps player to position index in `players` array |
| `kills` | `number` | Cumulative team kills (incremented when member kills) |
| `autoFill` | `boolean` | Whether the team accepts new players via auto-fill |

**Methods:**

| Method | Purpose |
|--------|---------|
| `addPlayer(player)` | Add member; assign next available color index; mark teammates dirty |
| `removePlayer(player)` | Remove disconnected member; rebuild index mapping; reassign color indexes |
| `setDirty()` | Mark all teammates' dirty flag for packet update |
| `hasLivingPlayers()` | Returns `true` if any member is alive (not dead, not disconnected) |
| `getLivingPlayers()` | Returns array of non-dead, non-disconnected members |
| `getNextAvailableColorIndex()` | Find next unused color index (0–3 typically) |
| `reassignColorIndexes()` | Reset all members' color indexes to sequential order after removal |

**Color Assignment:**
- Assigned at join time via `addPlayer()`, index is gap-filled from removed players
- Used by client to render teammate nameplates with distinct colors
- Limited to ~4 colors; duplicate colors converge if team > 4 (rare at max 4 players)

---

### `CustomTeam` (Lobby)

Located: `@file server/src/team.ts:125`

Represents a pre-game team lobby for matchmaking.

**Constructor:**
```typescript
constructor(readonly gameManager: GameManager)
```
- Generates random 4-character `id` (e.g., `"a3m9"`)
- Starts keep-alive interval (every 60s) to prevent WebSocket timeout

**Properties:**

| Property | Type | Purpose |
|----------|------|---------|
| `id` | `string` | Unique lobby identifier (4 random chars); used to join specific lobby |
| `players` | `CustomTeamPlayer[]` | List of players waiting in lobby |
| `autoFill` | `boolean` | When game starts, automatically fill empty slots from pool |
| `locked` | `boolean` | When true, new players cannot join this lobby |
| `forceStart` | `boolean` | When true, leader can immediately start game without waiting for ready signal |
| `gameID` | `number \| undefined` | ID of game instance assigned after `_startGame()` matches a game |

**Methods:**

| Method | Purpose |
|--------|---------|
| `addPlayer(player)` | Add `CustomTeamPlayer` to lobby; send Join message; publish update |
| `removePlayer(player)` | Remove player; clean up timers if lobby empty |
| `onMessage(player, msg)` | Handle client messages: Settings, Start, KickPlayer, etc. |

**Keep-Alive Interval:**
- Sends `CustomTeamMessages.KeepAlive` every 60 seconds (line 163)
- Prevents proxy/load-balancer from timing out idle connections

---

### `CustomTeamPlayer` (Lobby Member)

Located: `@file server/src/team.ts:330`

Represents a player in a `CustomTeam` lobby waiting for game assignment.

**Properties:**

| Property | Type | Purpose |
|----------|------|---------|
| `team` | `CustomTeam` | Reference to parent lobby |
| `ip` | `string` | Player's IP address |
| `name` | `string` | Player display name |
| `skin` | `string` | Selected character skin idString |
| `badge` | `string \| undefined` | Cosmetic badge |
| `nameColor` | `number \| undefined` | Custom name color |
| `ready` | `boolean` | Player's ready state (for start voting) |
| `socket` | `Bun.ServerWebSocket` | WebSocket connection |
| `isLeader` | `boolean` (getter) | True if `id === 0` (first player in array) |
| `id` | `number` (getter) | Index in `team.players` array |

**Methods:**

| Method | Purpose |
|--------|---------|
| `sendMessage(msg)` | Send `CustomTeamMessage` to socket (JSON serialized) |

---

## Game Modes & Team Assignment

### Team Modes Enum

Located: `@file common/src/constants.ts:160`

```typescript
export enum TeamMode {
    Solo = 1,   // 1 player per team
    Duo = 2,    // 2 players per team
    Squad = 4   // 4 players per team
}
```

Game determines team capacity by mode (used as divisor in team size checks).

### Team Assignment Logic

Located: `@file server/src/game.ts:636`

When `game.addPlayer(socket)` is called:

1. **If team mode is enabled** (`this.isTeamMode`):
   - Check if socket provides `teamID` (custom lobby) and `autoFill` flag
   - If `teamID` specified:
     - Attempt to join that team
     - If team is invalid (doesn't exist, full, all dead) → create new team, add to `customTeams` map
   - If no `teamID` (solo join):
     - Find vacant teams (in autoFill, has space, has living players)
     - Pick random vacant team; if none, create new team

2. **Assign player to team:**
   - Call `team.addPlayer(player)` → assigns color index
   - Set `player._team = team`
   - Set `player.teamID = team.id`

3. **If solo mode** (`!this.isTeamMode`):
   - No team assignment; each player is independent

**Code snippet:**
```typescript
if (this.isTeamMode) {
    const { teamID, autoFill } = socket?.data ?? {};
    if (teamID) {
        team = this.customTeams.get(teamID);
        // Validate team exists, has living players, has space
        if (!team || ... || team.players.length >= (this.teamMode as number)) {
            this.teams.add(team = new Team(this.nextTeamID));
            this.customTeams.set(teamID, team);
        }
    } else {
        // Auto-fill logic
        const vacantTeams = this.teams.valueArray.filter(...);
        team = vacantTeams.length ? pickRandomInArray(vacantTeams) : new Team(this.nextTeamID);
    }
}
```

---

## Winning Conditions

Located: `@file server/src/game.ts:519`

Game ends when:

**Solo Mode:**
- Only 1 player alive (`this.aliveCount <= 1`)

**Team Modes (Duo/Squad):**
- All alive players belong to same team
- AND alive count ≤ team mode number (e.g., Duo: ≤ 2 players, Squad: ≤ 4)
- AND game has been running ≥ 5 seconds

**Logic:**
```typescript
const isWon = this.isTeamMode
    ? this.aliveCount <= (this.teamMode as number) 
      && new Set([...this.livingPlayers].map(p => p.teamID)).size <= 1
    : this.aliveCount <= 1;
```

All surviving players on winning team emit victory emote and receive `GameOverPacket` with `won: true`.

---

## Team-Based Gameplay Mechanics

### 1. Color Indexing (Client Rendering)

- Each team member assigned sequential color index (0, 1, 2, 3)
- Client renders teammate nameplates with distinct colors
- Color array size: 4 colors; only enforced when team size ≤ 4
- On player removal, indexes recomputed to fill gaps (O(n) operation in `removePlayer`)

### 2. Revive Mechanic (Squad-Specific)

Located: `@file server/src/inventory/action.ts:40`

Only available in Duo/Squad modes via `ReviveAction`:

- Downed teammate (not fully dead) can be revived
- Reviver stands over downed player, triggers revive action
- Action time: `GameConstants.player.reviveTime` (typically 5–10s)
- Modified by `FieldMedic` perk (speed multiplier)
- On execution: `target.revive()` restores health; reviver marked dirty

**Revive Rules:**
- Only alive teammates can revive
- Cannot revive already-dead or disconnected players
- Cannot revive yourself
- Reviver has speed penalty (0.5× movement) during revive

### 3. Teammate Visibility

Located: `@file server/src/game.ts:428`

Teammates are **always visible** in UpdatePacket update loop:
```typescript
// Objects visible to player if:
const canSee = !collides 
    && (!this.isTeamMode || object.teamID !== source.teamID || object.id === source.id)
    //                       ^^^^^ Team members excluded from culling; always sent
```

Every teammate receives full UpdatePacket including other teammates' position/health/status.

### 4. Team Spawn Positioning

Located: `@file server/src/game.ts:673`

In team modes, new players spawn near existing teammates:
- If team has living players, spawn within ~20 unit radius of random teammate
- Solo players spawn randomly on map

```typescript
const teamPosition = this.isTeamMode && team
    ? pickRandomInArray(team.getLivingPlayers())?.position
    : undefined;

// If teamPosition exists, constrain spawn to circle around teammate
const getPosition = this.isTeamMode && teamPosition
    ? () => randomPointInsideCircle(teamPosition, 20, 10)
    : undefined;
```

---

## Team Data Networking

### UpdatePacket Teammates Field

Located: `@file common/src/packets/updatePacket.ts:30`

Player receives teammate data in `UpdatePacket.allies`:

```typescript
interface UpdatePacket {
    teammates?: Array<{
        id: number           // Player ID
        name: string
        health: number
        maxHealth: number
        adrenaline: number
        status: Bitfield     // Animations, death state
        colorIndex: number   // Team color (0–3)
    }>
}
```

**Sync Condition:**
- Sent if player's `dirty.teammates === true`
- Set to true when team composition changes (join/leave/revive/death)

### Custom Team Messages

Located: `@file common/src/typings.ts:33`

Pre-game lobby messages (CustomTeam ↔ CustomTeamPlayer):

| Message Type | Direction | Payload |
|--------------|-----------|---------|
| `Join` | Server → Client | `teamID`, `isLeader`, `autoFill`, `locked`, `forceStart` |
| `Update` | Server → Client | `players: CustomTeamPlayerInfo[]`, `ready`, `forceStart` |
| `Settings` | Both | `autoFill?`, `locked?`, `forceStart?` (leader-only) |
| `KickPlayer` | Client → Server | `playerId` (leader-only) |
| `Start` | Client → Server | (ready toggle or force-start) |
| `Started` | Server → Client | (game matched and assigned) |
| `KeepAlive` | Server → Client | (empty; every 60s) |

---

## Dependencies

### Depends On

- **[Game Loop](../game-loop/)** — Team created per game instance; game loop drives team spawn positioning, win condition checks
- **[Game Objects (Server)](../game-objects-server/)** — Player class holds `_team` reference; team manages Player array
- **[Networking](../networking/)** — UpdatePacket serializes teammate data; CustomTeamMessage WebSocket frame
- **[Inventory](../inventory/)** — ReviveAction revives downed teammates

### Depended On By

- **[Game Objects (Server)](../game-objects-server/)** — Player visibility, spawn, revive tied to team
- **[Networking](../networking/)** — Teammates field in UpdatePacket
- **[Client Rendering](../client-rendering/)** — Teammate colors, names, health bars, ping indicators

---

## Known Issues & Gotchas

### 1. Permanent Team Assignment

Once a player joins a game and is assigned to a team, **team cannot change mid-game**. Player must disconnect and rejoin to switch teams. This is by design (prevents griefing/balance issues in team modes).

### 2. Color Index Management is O(n)

- `removePlayer()` rebuilds entire `_indexMapping` to compact gaps (lines 35–95)
- For teams of size ≤ 4, this is negligible
- If team size ever exceeds 4 (not currently possible by design), this becomes a concern

### 3. Revive Blocked Without Teammate

- If entire team is dead except one player, that player **cannot revive anyone** (no downed teammates remain)
- In duo/squad, a player is only "downed" (not fully dead) if a teammate is alive when they hit 0 HP
- Once all teammates dead, player is marked fully dead and cannot be revived

### 4. Keep-Alive Interval Not Cancellable Gracefully

- `CustomTeam.keepAliveInterval` is set at construction
- Cleared only when last player leaves (`removePlayer` sees empty array at line 173)
- If connection drops before cleanup, interval may persist briefly in memory
- Not a critical issue (intervals cleaned on server restart), but could accumulate in long-running processes

### 5. Team Mode Detection Enum Comparison

- `this.isTeamMode = this.teamMode > TeamMode.Solo` (line 243)
- Relies on enum numeric values (1, 2, 4)
- If new modes added with values ≤ 1, this check breaks
- Should use explicit enum check if new modes introduced

---

## Related Documents

### Tier 1
- [Architecture Overview](../../architecture.md) — Game initialization, GameManager
- [Data Model](../../datamodel.md) — Team/Player entities

### Tier 2 — Subsystems
- [Game Loop](../game-loop/) — Team spawn, win condition, team visibility culling
- [Game Objects (Server)](../game-objects-server/) — Player class, team membership
- [Networking](../networking/) — UpdatePacket, CustomTeamMessage protocol
- [Inventory](../inventory/) — ReviveAction mechanics
- [Client Rendering](../client-rendering/) — Teammate color rendering, name plates

### Tier 3 — Modules
- [Game Loop — Tick](../game-loop/modules/tick.md) — Win condition logic per tick
- [Game Objects (Server) — Player](../game-objects-server/modules/player.md) — `_team` property, team-aware mechanics

### Patterns
- [Team System Patterns](patterns.md) — Team color assignment, team spawn patterns
