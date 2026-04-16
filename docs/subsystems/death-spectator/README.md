# Death & Spectator System

<!-- @tier: 2 -->
<!-- @parent: docs/architecture.md -->
<!-- @modules: docs/subsystems/death-spectator/modules/ -->
<!-- @source: server/src/objects/player.ts, server/src/game.ts, common/src/packets/killPacket.ts -->

## Purpose

Manages player death lifecycle (down → dead → eliminated), spectator camera control after elimination, kill tracking and killfeed distribution, and game-over determination. This subsystem bridges server-side damage resolution with client-side UI (killfeed, game-over screen) and handles the transition from active play to spectator observation.

## Key Files & Entry Points

| File | Purpose |
|------|---------|
| [server/src/objects/player.ts](../../../../server/src/objects/player.ts) | Player.die(), Player.down(), Player.revive() — death state transitions; Player.spectate() — spectator camera control; Player.sendGameOverPacket() |
| [server/src/objects/deathMarker.ts](../../../../server/src/objects/deathMarker.ts) | DeathMarker entity spawned at death location — marks respawn point for revives; synced to clients |
| [server/src/game.ts](../../../../server/src/game.ts) | Game-over condition evaluation; KillLeader tracking; game-over packet dispatch |
| [common/src/packets/killPacket.ts](../../../../common/src/packets/killPacket.ts) | KillPacket definition & serialization — broadcast to all clients for killfeed |
| [common/src/packets/gameOverPacket.ts](../../../../common/src/packets/gameOverPacket.ts) | GameOverPacket definition — rank, teammate stats, elimination state |
| [client/src/scripts/objects/player.ts](../../../../client/src/scripts/objects/player.ts) | Client Player.updateFromData() — hide models on death, show spectator UI |

## Death Lifecycle

### In-Game State Transitions

```
Player.damage() called with fatal blow
  ↓
health <= 0 && !dead
  ├─ [Team mode + has living teammates + not yet downed]
  │   → Player.down()          [knocked down state]
  │       └→ downed = true
  │       └→ health = 100 (reset for revive window)
  │       └→ send KillPacket(downed=true, killed=false)
  │
  └─ [Solo mode OR no revivable teammates OR already downed]
      → Player.die()           [permanently dead]
          └→ dead = true
          └→ health = 0
          └→ send KillPacket(downed=false, killed=true)
          └→ spawn DeathMarker
          └→ remove from active game.players
```

**Key Condition for Down** ([server/src/objects/player.ts](../../../../server/src/objects/player.ts#L2760)):
```typescript
if (
  game.isTeamMode &&
  this._team?.players.some(p => !p.dead && !p.downed && !p.disconnected && p !== this) &&
  !this.downed &&
  source !== DamageSources.Disconnect
) {
  this.down(source, weaponUsed);
} else {
  this.die(params);
}
```

### Down State (Revivable)

**Duration:** 30 seconds before auto-bleedout → death

**Revival Window:**
- Teammate must initiate revive interaction within range (~3 units, on same layer)
- Teammate must be living, not knocked down, not reviving someone else
- `revive()` restores `downed = false`, sets `health = 30`

**Auto-Bleed Logic:**
If downed player not revived within window, damage source becomes `DamageSources.BleedOut` and player dies. Kill credit goes to `downedBy.player` if still alive.

**Bleed Timeout** ([server/src/objects/player.ts](../../../../server/src/objects/player.ts)):
```typescript
private autoBleedTimeout?: Timeout;
// Auto-bleed timer not explicitly shown in snippet but referenced in down flow
```

### Death State (Permanent)

`die()` method ([server/src/objects/player.ts](../../../../server/src/objects/player.ts#L2969)):

```typescript
die(params: Omit<DamageParams, "amount">): void {
  if (this.health > 0 || this.dead) return;  // Guard: already dead or alive

  this.health = 0;
  this.dead = true;
  const wasDowned = this.downed;  // Track if killed while knocked
  this.downed = false;
  this.lastDamagedBy = undefined;
  this.canDespawn = false;  // Corpse persists, not despawned
  this._team?.setDirty();   // Update team state (one fewer alive)

  // Emit event for plugins
  this.game.pluginManager.emit("player_will_die", {
    player: this,
    ...params
  });

  // Create KillPacket for killfeed
  const packet = KillPacket.create();
  packet.victimId = this.id;
  packet.downed = wasDowned;
  packet.killed = true;

  // ... kill credit logic & weapon swap logic ...
}
```

## DeathMarker Entity

**Purpose:** Visual marker at death location; enables respawn interaction targeting; synced to all clients

**Lifetime:** Spawned at player death, marked as "new" for 100ms (client-side visual effect), persists on map until game ends

**Data Structure** ([server/src/objects/deathMarker.ts](../../../../server/src/objects/deathMarker.ts)):

```typescript
export class DeathMarker extends BaseGameObject.derive(ObjectCategory.DeathMarker) {
  readonly player: Player;           // Reference to deceased player
  isNew = true;                      // True for 100ms after spawn (client effect)
  readonly hitbox: RectangleHitbox;  // 5×5 rect at death position
  ...
}
```

**Network Allocation:**
- `fullAllocBytes = 1` (minimal footprint)
- `partialAllocBytes = 12` (changed position or isNew flag)
- Synced in UpdatePacket like other game objects

**Client Rendering:** Visible gravestone/skull icon at position

## Down vs. Kill Credit

Kill credit logic determines who receives the elimination:

**For Player-to-Player Kills:**

```typescript
// Check if downed player was downed by someone
const downedBy = this.downedBy?.player;

if (downedBy && downedBy.teamID === source.teamID) {
  // Killer is on same team as who downed the victim
  packet.creditedId = downedBy.id;      // Credit the downer
  packet.kills = ++downedBy.kills;      // Increment downer's kill count
} else {
  // Killer downed victim OR solo kill
  packet.kills = ++source.kills;        // Killer gets credit
}
```

**For Environmental Kills (Gas, Obstacle, BleedOut, Disconnect):**

```typescript
if (source === DamageSources.Gas || source === DamageSources.Obstacle || ...) {
  if (downedBy !== undefined) {
    packet.creditedId = downedBy.id;
    if (downedBy !== this) packet.kills = ++downedBy.kills;
  }
}
```

## Spectator Mode

**Entry Trigger:** Player becomes dead (`this.dead === true`)

**Spectate Method** ([server/src/objects/player.ts](../../../../server/src/objects/player.ts#L2451)):

```typescript
spectate(packet: SpectateData): void {
  if (!this.dead) return;  // Only dead players can spectate

  // Rate limit: max 1 spectate action every 200ms
  if (game.now - this.lastSpectateActionTime < 200) return;
  this.lastSpectateActionTime = game.now;

  let toSpectate: Player | undefined;

  switch (packet.spectateAction) {
    case SpectateActions.BeginSpectating: {
      // Prefer closest teammate (team mode)
      if (this.game.isTeamMode && this._team?.hasLivingPlayers()) {
        toSpectate = this._team.getLivingPlayers()
          .reduce((a, b) =>
            Geometry.distanceSquared(a.position, this.position) <
            Geometry.distanceSquared(b.position, this.position) ? a : b
          );
      }
      // Fall back to killer
      else if (this.killedBy !== undefined && !this.killedBy.dead) {
        toSpectate = this.killedBy;
      }
      // Fall back to random living player
      else if (spectatablePlayers.length > 1) {
        toSpectate = pickRandomInArray(spectatablePlayers);
      }
      break;
    }
    case SpectateActions.SpectatePrevious:
      // Cycle to previous player in spectatablePlayers array
      toSpectate = spectatablePlayers[
        Numeric.absMod(spectatablePlayers.indexOf(this.spectating) - 1, spectatablePlayers.length)
      ];
      break;
    case SpectateActions.SpectateNext:
      // Cycle to next player
      toSpectate = spectatablePlayers[
        Numeric.absMod(spectatablePlayers.indexOf(this.spectating) + 1, spectatablePlayers.length)
      ];
      break;
    case SpectateActions.SpectateSpecific:
      toSpectate = spectatablePlayers.find(player => player.id === packet.playerID);
      break;
    case SpectateActions.SpectateKillLeader:
      toSpectate = game.killLeader;
      break;
    case SpectateActions.Report:
      // Report current spectate target for violation
      this.reportedPlayerIDs.set(this.spectating.id, true);
      break;
  }

  this.spectating = toSpectate;  // Update who we're watching
}
```

**Spectatableplayers:** Subset of all players currently alive (not dead, not disconnected)

**Client-Side Spectator UI:**
- Free camera navigation (follows spectated player or manual pan/zoom)
- Player rotation and highlight
- Cannot directly control spectated player
- Killfeed remains visible and updates in real-time
- Minimap shows full visibility (no fog of war for spectators in some modes)

## Kill Packet Structure

**Broadcast on:** Every down() and die() call; sent to ALL connected players

**KillData Definition** ([common/src/packets/killPacket.ts](../../../../common/src/packets/killPacket.ts)):

```typescript
export interface KillData {
  readonly type: PacketType.Kill
  readonly victimId: number
  readonly attackerId?: number        // Attacker player ID (undefined for environmental)
  readonly creditedId?: number          // Actual credit receiver (may differ from attacker)
  readonly kills?: number              // Total kill count of credited player
  readonly damageSource: DamageSources  // Gun|Melee|Throwable|Explosion|Gas|Obstacle|BleedOut|FinallyKilled|Disconnect
  readonly weaponUsed?: GunDefinition | MeleeDefinition | ThrowableDefinition | ExplosionDefinition | ObstacleDefinition
  readonly killstreak?: number          // Kills with current weapon (Gun/Melee/Throwable only)
  readonly downed: boolean              // True if player knocked down, false if killed
  readonly killed: boolean              // True if player killed, false if knocked down
}
```

**Serialization:** Binary encoding via SuroiByteStream for efficient transmission

**Client Killfeed Display:** Renders as:
```
[Killer Avatar] [Killer Name] [Weapon Icon] [Victim Name]
```

Additional details:
- **Headshot:** Trophy icon overlay
- **Elimination (last team):** Special color/text
- **Environmental death:** Gray text, no killer avatar
- **Suicide:** Self arrow, gray text

**Killfeed Lifetime:** ~5 seconds per entry, fades then removes from display

## Game-Over Determination

**Win Condition Evaluation** ([server/src/game.ts](../../../../server/src/game.ts#L517)):

```typescript
if (
  this._started &&
  !this.over &&
  (Config.minTeamsToStart ?? 2) > 1 &&
  (
    this.isTeamMode
      ? this.aliveCount <= (this.teamMode as number) &&
        new Set([...this.livingPlayers].map(p => p.teamID)).size <= 1
      : this.aliveCount <= 1
  ) &&
  this.now - this.startedTime > 5000  // Minimum 5 seconds elapsed
) {
  // Declare winners
  for (const player of this.livingPlayers) {
    player.sendGameOverPacket(true);  // won = true
    this.pluginManager.emit("player_did_win", player);
  }

  this.pluginManager.emit("game_end", this);
  this.setGameData({ allowJoin: false, over: true });

  // Disconnect all after 1 second
  this.addTimeout(() => {
    for (const player of this.connectedPlayers) {
      player.disconnect("Game ended");
    }
    this._stopped = true;
  }, 1000);
}
```

**Solo Mode:** `aliveCount <= 1` → last player standing wins

**Team Modes:** `aliveCount <= teamMode AND only 1 unique teamID` → winning team all receive game-over

### GameOverPacket

**Sent to:** Eliminated player when game ends; contains final stats

**GameOverData Structure** ([common/src/packets/gameOverPacket.ts](../../../../common/src/packets/gameOverPacket.ts)):

```typescript
export interface GameOverData {
  readonly type: PacketType.GameOver
  readonly rank: number                // Final placement (1 = winner, higher = worse)
  readonly teammates: TeammateGameOverData[]
}

export interface TeammateGameOverData {
  readonly playerID: number
  readonly kills: number
  readonly damageDone: number           // Total damage dealt
  readonly damageTaken: number          // Total damage taken
  readonly timeAlive: number            // Seconds survived
  readonly alive: boolean               // Whether alive at game end
}
```

**Rank Calculation:** Based on aliveCount when eliminated:
- Last player alive: rank 1
- Each earlier elimination: rank +1
- Team modes: all on winning team rank 1

## Kill Leader Tracking

[server/src/game.ts](../../../../server/src/game.ts#L168):

```typescript
killLeader: Player | undefined;                    // Current kill leader
killLeaderDirty = false;                          // Needs update flag

updateKillLeader(player: Player): void {
  const killLeader = this.killLeader;
  if (!killLeader || player.kills > killLeader.kills) {
    this.killLeader = player;
    this.killLeaderDirty = true;  // Triggers broadcast to all clients
  }
}
```

**Updated on:** Every player kill (in die() method)

**Used for:** Spectator target selection (SpectateActions.SpectateKillLeader); UI crown icon

## Data Flow

```
Damage dealt to player
  ↓
[health <= 0?]
  ├─ [Revivable (down state)]
  │   → Player.down()
  │   → KillPacket(downed=true, killed=false)
  │   → Broadcast to all clients → Killfeed shows "[Attacker] knocked down [Victim]"
  │   → 30-second revive window opens
  │
  └─ [Not revivable or auto-bleed expired]
      → Player.die()
      → KillPacket(downed=false, killed=true)
      → DeathMarker spawn at position
      → Remove player.dead = true
      → Update killLeader if applicable
      → Check game win condition
      ├─ [Game ongoing]
      │   → Player enters spectator mode
      │   → Broadcast KillPacket → Killfeed shows death
      │
      └─ [Game over (last player/team)]
          → SendGameOverPacket to all connected players
          → Disconnect all clients after 1 second
```

## Dependencies

### Depends on:
- [Game Objects (Server)](../game-objects-server/) — BaseGameObject, GameObject hierarchy; player state mutations
- [Damage System](../damage-system/) — DamageParams, damage resolution logic (if separate subsystem)
- [Networking](../networking/) — KillPacket, GameOverPacket serialization & dispatch
- [Game Loop](../game-loop/) — tick cadence for damage processing, game-over checks
- [Team System](../team-system/) — team membership checks, living player queries
- [Inventory](../inventory/) — weapon definitions in kill packets, equipment stats
- [Plugin System](../plugins/) — "player_will_die", "player_did_win", "game_end" events

### Depended on by:
- [Game Loop](../game-loop/) — win condition must be checked each tick
- [Networking](../networking/) — KillPacket/GameOverPacket must be transmitted to clients
- [UI Management](../ui-management/) — killfeed rendering, game-over screen display
- [Client Rendering](../client-rendering/) — player model visibility (hide on death)
- [Input Management](../input-management/) — spectate action input handling

## Configuration & Constants

**Death-Related GameConstants** ([common/src/constants.ts](../../../../common/src/constants.ts)):
- `GameConstants.player.defaultHealth` — initial health (~100)
- `GameConstants.player.startAdrenaline` — revival health (~30)
- Game tick rate (40 TPS = 25ms tick) affects down timer precision

**Revive Window:** Hardcoded to 30 seconds (verify in down() auto-bleed timeout setup)

**Spectate Rate Limit:** 200ms between actions (verified in spectate() method)

**Minimum Game Duration:** 5 seconds before game-over condition can trigger

## Known Issues & Gotchas

1. **Down Timer Not Visible:** Auto-bleed timeout is set but not exposed in UI (no countdown timer shows to downed player)

2. **DeathMarker Position May Be Off-Grid:** No snapping enforcement; position inherits from player.position which may not align to grid cells

3. **Killfeed Entry Lifetime Hardcoded:** Must search client-side `UIManager` class to find 5-second timeout; not configurable

4. **Spectator Camera Unbounded:** Free camera can fly beyond map boundaries (no hard bounds on spectator pan/zoom)

5. **Multiple Eliminations Same Tick (Undefined Behavior):** If two damage sources eliminate a player in same tick (e.g., gas + bullet), last one processed gets kill credit; no tie-breaking logic

6. **Revival Doesn't Break on Damage:** Reviver receiving damage does NOT cancel the revive action; must be explicitly triggered to stop

7. **Kill Streak Counter Resets:** Not explicitly cleared on revive or between matches; may carry over (verify in code)

8. **Spectating Killed Player:** If you spectate killer and killer dies, spectate target must be manually switched; no auto-switch to next best target

9. **Report System Silent:** Reporting a player has no confirmation feedback to reporter; reports are silently logged

## Related Documents

### Tier 1
- [System Architecture](../../architecture.md) — game lifecycle, tick-based updates, win conditions
- [Data Model](../../datamodel.md) — Player entity, team structure, game state

### Tier 2
- [Game Loop](../game-loop/) — tick evaluation, game-over checks (depends on this subsystem)
- [Networking](../networking/) — packet definitions, serialization, client dispatch
- [Game Objects (Server)](../game-objects-server/) — Player, DeathMarker entities, damage flow
- [Team System](../team-system/) — team membership, living player queries
- [Inventory](../inventory/) — weapon definitions, kill streaks
- [UI Management](../ui-management/) — killfeed rendering, game-over screen
- [Client Rendering](../client-rendering/) — player visibility state (hide on death)
- [Input Management](../input-management/) — spectate action processing
- [Plugin System](../plugins/) — death/win/game-end events

### Tier 3
- (None yet) — Module-level docs for kill credit logic, spectator state machine, game-over condition evaluation
