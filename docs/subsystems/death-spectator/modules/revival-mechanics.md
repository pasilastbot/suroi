# Revival Mechanics Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/death-spectator/README.md -->
<!-- @source: server/src/inventory/action.ts, server/src/objects/player.ts -->

## Purpose
Manages the complete revival lifecycle including requirements validation, animation and timing (with perk modifiers), interruption handling, revive range constraints, and successful revive effects for downed players in team modes.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| `server/src/inventory/action.ts` | ReviveAction class, cooldown tracking, completion logic | High |
| `server/src/objects/player.ts` | Downed state, revival tracking (beingRevivedBy), revive method | High |
| `common/src/constants.ts` | GameConstants.player.reviveTime (base timing) | Low |
| `server/src/objects/player.ts` | Perk modification (FieldMedic usage modifier) | Medium |

## Business Rules

### Revival Requirements

Before revive starts, the following must be true:

| Requirement | Details | Check |
|-------------|---------|-------|
| Downed player exists | Target player in downed state (not dead) | `targetPlayer.downed === true` |
| Reviver alive | Reviver not downed or dead | `reviverPlayer.health > 0 && !reviverPlayer.downed` |
| Reviver in range | Distance between players ≤ 16 units | `Vec.dist(reviver.pos, target.pos) ≤ REVIVE_RANGE` |
| Team mode enabled | Game is team game (Duo, Squad, etc.) | `game.isTeamMode === true` |
| Same team | Both players on same team | `reviver.teamId === target.teamId` |
| No ongoing action | Reviver not already reloading/healing | `reviver.action === null` |
| Not already reviving | Another player isn't reviving this target | `targetPlayer.beingRevivedBy === undefined` |
| Reviver not reviving | Reviver isn't already reviving someone else | `reviverPlayer.action !instanceof ReviveAction` |

**Validation sequence:**
```
if (!targetPlayer.downed) return false  // not downed
if (!game.isTeamMode) return false      // not team mode
if (reviver.teamId !== target.teamId) return false  // different teams
if (distance > MAX_REVIVE_RANGE) return false  // too far
if (reviver.action !== null) return false  // reviver busy
if (targetPlayer.beingRevivedBy) return false  // already being revived
if (reviver.downed || !reviver.alive) return false  // reviver invalid state
return true  // OK to start revive
```

### Revival Animation & Timing

**Animation state:**
- Reviver animation set to `AnimationType.Revive` (special gesture)
  - Rendered as kneeling beside downed player
  - Client-side animation loops during revive duration
- Downed player animation unchanged (still in downed pose)
- Both players movement speed set to 0.5× (impaired during revive)
  - Reviver can't run away (commitment)
  - Target can't crawl away (must wait)

**Timing calculation:**
- **Base time:** `GameConstants.player.reviveTime` (default: 4000 ms = 4 seconds)
- **Perk modifier:** Retrieved from reviver's equipped perks
  - `FieldMedic` perk: `usageMod = 0.5` → 50% faster revive (2 seconds)
  - Formula: `baseTime / mapPerkOrDefault(FieldMedic, usageMod, 1.0)`
  - Result: actual revive duration = 4000 ms / 0.5 = 2000 ms
- **No other mods:** Revival time not affected by armor, helmets, skill perks
  - Only `FieldMedic` impacts revive speed

**Timing example:**
```
Without FieldMedic: 4000 ms
With FieldMedic: 4000 ms ÷ 0.5 = 2000 ms (50% faster)
With FieldMedic + other speed perks: Still 2000 ms (speed perks don't apply)
```

### Interruption Handling

Revival can be interrupted by:

| Event | Handler | Effect |
|-------|---------|--------|
| Reviver moves out of range | Range check every tick | `ReviveAction.cancel()` |
| Reviver takes damage | Damage handler | Cancel action immediately |
| Downed player revived prematurely | N/A | Revival completes (no interrupt) |
| Reviver downed/killed | Death handler | `ReviveAction.cancel()` |
| Target killed while down | Death handler | `ReviveAction.cancel()` |
| Network disconnect | Disconnect handler | Abort and cleanup |

**Cancel logic:**
```
cancel() {
  // Revert downed player
  this.target.beingRevivedBy = undefined
  this.target.setDirty()

  // Revert reviver
  this.player.animation = AnimationType.None
  this.player.setDirty()

  // Clear action
  this.player.action = null
}
```

**Implicit behavior:**
- Cancel is silent (no error message shown)
- Both players regain full movement speed
- If reviver takes damage, damage still applies (no damage immunity)
- If downed player dies, both are cleaned up, revive credit NOT awarded

### Revive Range Constraint

- **Max range:** `REVIVE_RANGE = 16 units` (approximately 1 grid cell)
- **Range check:** Every server tick (40 TPS = 25 ms interval)
  - Euclidean distance: `√((reviverX - targetX)² + (reviverY - targetY)²)`
  - If distance exceeds 16 units → revive canceled
- **Why frequent check:** Prevents "rubber-band" revive (player walks away, still revives)
- **Grace period:** None (revive cancels immediately if out of range)

**Range visualization:**
```
Reviver
  O
   \
    \ 16 units (max range)
     \
      O
   Downed player
```

### Successful Revive Effects

When revive completes successfully:

**Player state:**
- `targetPlayer.revive()` called
  - Clear downed status: `downed = false`
  - Restore health: `health = MAX_HEALTH × 0.25` (25% health on revive)
  - Clear bleeding status (if applicable)
  - Respawn animation plays (pop-up effect)
- `reviverPlayer.animation = AnimationType.None`
  - Return to normal idle animation

**Network effects:**
- Both players marked dirty (included in next UpdatePacket)
- Revive event logged in game (for stats/achievements)
- Chat message sent: `"[Player A] revived [Player B]"` (if enabled)

**Scoring/Statistics:**
- Reviver receives revive credit (tracked in leaderboard)
- Revived player may have assist/life debt mechanic (varies per game)
- Both players receive team bonus (varies per mode)

## Data Lineage

```
Player initiates revive (interaction with downed teammate)
  ↓
[Validation: range, state, team mode, etc.]
  ↓
Create ReviveAction(reviver, target)
  ↓
Set target.beingRevivedBy = reviver
  ↓
Set reviver.action = ReviveAction
  ↓
Set reviver.animation = AnimationType.Revive
  ↓
[Each tick: decrease duration]
  ↓
[If interrupted: cancel()]
  ↓
  │
  ├─→ Interrupted: target.beingRevivedBy = undefined
  │
  └─→ Completed: target.revive()
       ↓
       target.health = MAX_HEALTH × 0.25
       ↓
       target.downed = false
       ↓
       Mark both players dirty
       ↓
       [Next UpdatePacket: include both players]
       ↓
       Client renders revival complete
```

## Dependencies

- **Internal:**
  - Death & Spectator — Downed state management
  - Action system — Base Action class and action queue
  - Player — Movement, animation, health
- **External:**
  - [Perks & Passive](../../perks-passive/modules/perk-conflicts.md) — FieldMedic perk modifier
  - [Networking](../../networking/README.md) — UpdatePacket player dirty flag
  - [Game Loop](../../game-loop/README.md) — Tick-based action update
  - [Game Objects Server](../../game-objects-server/README.md) — Player lifecycle

## Complex Functions

### ReviveAction Constructor
**Function:** `ReviveAction.constructor(reviver: Player, target: Player)`  
**Source:** `server/src/inventory/action.ts:40–47`  
**Purpose:** Initialize revive action with perk-modified duration

**Implementation:**
```typescript
export class ReviveAction extends Action {
  constructor(reviver: Player, readonly target: Player) {
    super(
      reviver,
      GameConstants.player.reviveTime 
        / reviver.mapPerkOrDefault(
            PerkIds.FieldMedic, 
            ({ usageMod }) => usageMod, 
            1
          )  // type: number
    );
  }
}
```

**Logic breakdown:**
1. Base revive time: `GameConstants.player.reviveTime` (4000 ms)
2. Look up `FieldMedic` perk on reviver
3. Extract `usageMod` from perk definition (0.5 for FieldMedic)
4. Calculate: 4000 / 0.5 = 2000 ms
5. Pass final duration to parent Action class
6. Action timer starts counting down

**Implicit behavior:**
- `mapPerkOrDefault()` returns default (1.0) if perk not found → no speedup
- Division (not multiplication) means lower mod = faster revive
- Duration baked in at action creation (recalc if perk status changes during revive)

**Called by:**
- Player interaction handler — when downed player calls for revive
- Reviver's action queue — `executeAction(new ReviveAction(...))`

### Revive Execution
**Function:** `ReviveAction.execute(): void`  
**Source:** `server/src/inventory/action.ts:53–56`  
**Purpose:** Complete revive when action duration expires

**Implementation:**
```typescript
override execute(): void {
  super.execute();  // base class cleanup (clear action, etc.)
  
  this.target.revive();  // restore target to alive state
  this.player.animation = AnimationType.None;  // clear reviver animation
  this.player.setDirty();  // include in next update packet
}
```

**Logic:**
1. Call parent `execute()` (default action completion)
2. Call `target.revive()` on downed player
   - Restores health, clears downed status
3. Reset reviver animation
4. Mark players dirty

**Implicit behavior:**
- `revive()` is idempotent (safe to call multiple times)
- Reviver animation cleared even if revive fails (shouldn't happen)
- No distance check (assumes action completed during valid duration)

**Called by:**
- Action system tick handler — when action duration reaches 0

### Revive Cancellation
**Function:** `ReviveAction.cancel(): void`  
**Source:** `server/src/inventory/action.ts:61–62`  
**Purpose:** Abort revive due to interruption

**Implementation:**
```typescript
override cancel(): void {
  super.cancel();  // base class cleanup
  
  this.target.beingRevivedBy = undefined;  // clear downed player's reviver
  this.target.setDirty();  // include in next update
  
  this.player.animation = AnimationType.None;  // clear reviver animation
  this.player.setDirty();
}
```

**Logic:**
1. Call parent `cancel()` (base action clear)
2. Clear downed player's `beingRevivedBy` reference
3. Mark downed player dirty
4. Clear reviver animation
5. Mark reviver dirty

**Implicit behavior:**
- Cancellation is silent (no error message)
- Both players added to next UpdatePacket (dirty check)
- No rollback of health/state (downed player remains downed)
- Movement speed returns to normal (no longer impaired)

**Called by:**
- Range check — if reviver walks out of range
- Reviver damage handler — if reviver takes damage
- Death handler — if reviver or target dies
- Interruption handler — any other interruption

### Player Revive Method
**Function:** `Player.revive(): void`  
**Source:** `server/src/objects/player.ts:3317–3330`  
**Purpose:** Restore downed player to playable state

**Implementation (conceptual):**
```typescript
revive(): void {
  // Validate state
  if (!this.downed) return;

  // Restore state
  this.downed = false;
  this.health = GameConstants.player.maxHealth * 0.25;  // 25% health
  
  // Clear status effects
  this.bleeding = false;
  this.beingRevivedBy = undefined;
  
  // Clear impairment
  this.animation = AnimationType.None;
  
  // Mark dirty
  this.setDirty();
  
  // Emit event (for plugins)
  this.game.pluginManager.emit('player_revived', { 
    player: this, 
    revivedBy: this.beingRevivedBy 
  });
}
```

**Logic:**
1. Check downed (prevent double-revive)
2. Set health to 25% of max (low health, but playable)
3. Clear status conditions (bleeding, etc.)
4. Clear revival reference
5. Reset animation
6. Mark dirty for UpdatePacket
7. Fire event for plugins/hooks

**Implicit behavior:**
- Health hardcoded to 25% (not configurable)
- Bleeding removed (player starts fresh)
- No other stat modifications (armor, etc. unchanged)
- Async event (plugins run, but don't block)

**Called by:**
- `ReviveAction.execute()` — on successful revive completion
- Manual revive admin command — for admin testing

## Configuration

| Setting | Location | Effect | Default |
|---------|----------|--------|---------|
| `reviveTime` | `GameConstants.player` | Base revive duration (ms) | 4000 |
| `MAX_REVIVE_RANGE` | constants or game.ts | Max distance to revive | 16 units |
| `REVIVE_HEALTH_PCT` | constants | % health restored on revive | 0.25 (25%) |
| `FieldMedic.usageMod` | perk definition | Revive speed multiplier | 0.5 (50% faster) |

## Edge Cases & Gotchas

### Gotcha: Revive Range Check Timing
- **Issue:** Network latency causes reviver to appear in range on client, but out of range on server
  - Client starts animation
  - Server rejects revive due to range
  - Visual/server mismatch
  - Client remains playing animation; server cancels
- **Prevention:** Client-side range validation before sending request; server double-check with grace period (e.g., ±2 units)

### Gotcha: Revive During Knockback
- **Issue:** Reviver knocked back while reviving
  - Range check should cancel revive
  - But knockback applies AFTER range check (same tick)
  - Revive continues despite distance increasing
- **Prevention:** Check distance AFTER applying knockback; or use continuous range monitoring (every frame, not tick-based)

### Gotcha: FieldMedic Perk Lost Mid-Revive
- **Issue:** Reviver has FieldMedic, starts revive (2 sec duration)
  - 1 second in, reviver dies and respawns without FieldMedic
  - Revive still uses 2-sec duration (baked at action creation)
  - Inconsistent with reviver's current perk state
- **Prevention:** Recalculate duration on each tick (expensive); or document this edge case behavior

### Gotcha: Simultaneous Revives
- **Issue:** Two teammates try to revive same downed player simultaneously
  - Both ReviveAction objects created
  - Both pass validation
  - Both run execute() on completion
  - `target.revive()` called twice (harmless, idempotent)
  - Both reviver animations cleared
- **Prevention:** Check `targetPlayer.beingRevivedBy` BEFORE creating action; reject if already reviving

### Gotcha: Reviver Self-Down During Revive
- **Issue:** Reviver gets downed while reviving (e.g., grenade damage)
  - Reviver state: `downed = true`
  - ReviveAction still active
  - Target still marked `beingRevivedBy = reviver`
  - Downed reviver can't move/act; target trapped waiting
- **Prevention:** In death handler, detect if player is reviver; cancel their ReviveAction immediately

### Gotcha: Team Switch Mid-Revive
- **Issue:** Player switches team (via admin command or glitch)
  - Revive action ongoing
  - Reviver's teamId changed
  - Server checks range/state; cancels revive OR allows it
- **Prevention:** Disable team switching during active revive; or auto-cancel revive if team changes

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Death & Spectator subsystem overview
- **Tier 1:** [../../architecture.md](../../architecture.md) — System architecture
- **Patterns:** [../patterns.md](../patterns.md) — Downed state and recovery patterns
- **Related modules:**
  - [Perks & Passive](../../perks-passive/modules/perk-conflicts.md) — FieldMedic perk modifier
  - [Game Loop](../../game-loop/README.md) — Action tick-based execution
  - [Game Objects Server](../../game-objects-server/README.md) — Player state management
