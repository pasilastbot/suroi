# Team Mechanics

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/team-system/README.md -->
<!-- @source: server/src/team.ts, server/src/objects/player.ts, common/src/packets/updatePacket.ts -->

## Purpose

This module implements the core team mechanics for squad-based gameplay. It defines the `Team` class that manages team membership, synchronizes teammate state, applies damage immunity between teammates, coordinates team elimination, handles team spectation, and manages revival mechanics for downed teammates.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| `server/src/team.ts` | Team class, team formation, player roster, color index management | Medium |
| `server/src/objects/player.ts` | Team field, damage immunity checks, downed state, revival, team spectation logic | High |
| `common/src/packets/updatePacket.ts` | TeammateData serialization in UpdatePacket, team ID and teammate list encoding | Medium |
| `server/src/game.ts` | Team mode detection, team win condition (last team standing), team elimination tracking | Medium |

## Team Structure

### The Team Class — @file server/src/team.ts:7

```typescript
export class Team {
    readonly id: number;
    private readonly _players: Player[] = [];
    get players(): readonly Player[] { return this._players; }
    readonly _indexMapping = new Map<Player, number>();
    kills = 0;
    readonly autoFill: boolean;
}
```

**Properties:**
- `id`: Unique numeric team identifier (0, 1, 2, ... or custom UUID for custom lobbies)
- `_players`: Array of players currently in the team (dynamic roster)
- `_indexMapping`: Map tracking each player's index in the roster (used for color assignment)
- `kills`: Shared team kill count (tracked separately from individual player kills)
- `autoFill`: Boolean flag indicating if the team accepts additional players via matchmaking

**Key Methods:**
- `addPlayer(player)` — Assigns the next available color index, adds player to roster, marks team dirty
- `removePlayer(player)` — Removes player, rebuilds index mapping to keep roster compact
- `setDirty()` — Marks all teammates' `dirty.teammates` flag for UpdatePacket serialization
- `hasLivingPlayers()` — Returns true if any teammate is alive and not disconnected
- `getLivingPlayers()` — Returns array of all alive teammates (excluding downed, dead, disconnected)

**Implementation Detail: Index Mapping**

The `_indexMapping` keeps a tight coupling between array indices and player positions. When a player is removed:

1. Remove from `_players` array
2. Skip index mapping updates for indices before the removed player
3. Decrement all higher indices
4. Reassign color indices to maintain sequential 0, 1, 2, 3, ...

This ensures O(1) lookup but requires careful management when players leave.

## Team Types & Configuration

### Team Sizes

Team size is configurable via GameMode and enforced at the Game level. The system supports:

| Type | Size | Example |
|------|------|---------|
| **Solo** | 1 | FFA, deathmatch (no team) |
| **Duo** | 2 | 2v2, 2 squads of 2 |
| **Squad** | 3–4 | 3v3 or 4v4 |
| **Larger Teams** | 5+ | Custom modes (mega-squad) |

The `TeamMode` enum in `@common/constants.ts` controls this:
```typescript
export const enum TeamMode {
    Solo = 0,    // No teams
    Duo = 2,
    Squad = 4,
    // Game enforces: aliveCount <= (teamMode as number) for win condition
}
```

## Team Formation & Matching

### Pre-Game Join (Custom Lobbies)

Custom teams are created via `CustomTeam` in `server/src/team.ts` for pre-game team formation:
- Players join a lobby and form teams before the match starts
- Team leader (`CustomTeamPlayer.isLeader`) can lock the team, enable auto-fill, or force start
- Team settings sent in real-time via `CustomTeamMessages` (Join, Settings, Update, Started)

### In-Game Assignment (Matchmaking)

When a match starts via `Game.addPlayer()`:
- Server calls `Team(teamID)` constructor to create a new team
- Players are assigned to teams sequentially or via custom lobby assignment
- Each player receives a `colorIndex` (0, 1, 2, 3, ...) for UI identifiers

### Late Joins

If a team is not full and has `autoFill: true`:
- New players can join an existing team up to the team size limit
- The new player is assigned the next available `colorIndex`
- All teammates receive a `dirty.teammates` update

## Teammate Damage Rules

### Friendly Fire Disabled

**Core Logic** — @file server/src/objects/player.ts:2662

```typescript
piercingDamage(params: DamageParams): void {
    const { source, weaponUsed } = params;
    let { amount } = params;
    if (
        this.invulnerable
        || (
            this.game.isTeamMode
            && source instanceof Player
            && source.teamID === this.teamID          // ← Team check
            && source.id !== this.id                  // ← Self-damage exception
            && !this.disconnected
            && amount > 0                              // ← Positive damage only
        )
    ) return;  // Early exit: no damage applied
    // ... rest of damage logic
}
```

**Rules:**
1. If attacker (`source`) is a `Player` and in same team (`source.teamID === this.teamID`)
2. AND target is not the attacker themselves (`source.id !== this.id`)
3. AND target is still connected
4. AND damage is positive (healing applies normally)
→ **Damage is blocked entirely** (early return)

**Self-Damage Exception:**
- A player CAN damage themselves (e.g., splash damage from own grenade)
- Useful for balancing grenade throws and explosions

### Shared Kill Credit

When a player dies after being downed by a teammate (down → revival window → death):

**Kill Credit Logic** — @file server/src/objects/player.ts:3025

```typescript
if (downedBy && downedBy.teamID === source.teamID) {
    packet.creditedId = downedBy.id;
    packet.kills = ++downedBy.kills;  // ← Credit to downer
} else {
    packet.kills = ++source.kills;     // ← Credit to killer
}
```

**Rules:**
- If `downedBy` exists (teammate who downed the target) AND is on same team as killer
  → Credit goes to the player who downed
- Otherwise → Credit goes to the finishing player
- Killstreak and weapon swaps always credit the killer (not the downer)

## Team Chat & Symbols

### Color Indices for Identification

Each teammate has a `colorIndex` (0–3 for squads) that represents their position. This enables:
- **HUD Role Indicators:** "Teammate 1", "Teammate 2", etc.
- **Team Color Coding:** Each teammate displays in a unique color on the minimap and directly
- **Kill Notifications:** "Teammate 2 eliminated" vs "Enemy XYZ eliminated"

The color is assigned in `Team.addPlayer()`:
```typescript
getNextAvailableColorIndex(): number {
    const existingIndexes = this.players.map(player => player.colorIndex);
    let newIndex = 0;
    while (existingIndexes.includes(newIndex)) {
        newIndex++;
    }
    return newIndex;
}
```

### Teammate Indicators

Teammates are rendered with distinct visual markers:
- **Health Bar Position:** Center of teammate entity (minimap/HUD)
- **Name & Color:** Rendered above teammate in-game with team color
- **Downed State Overlay:** Visual feedback when a teammate is incapacitated

## Team Win Condition

### Last Team Standing

A team wins when **all other teams are eliminated**.

**Win Logic** — @file server/src/game.ts:521

```typescript
if (
    this.isTeamMode
        ? this.aliveCount <= (this.teamMode as number) && new Set([...this.livingPlayers].map(p => p.teamID)).size <= 1
        : this.aliveCount <= 1
) {
    // Team mode: ≤ [teamSize] players alive AND all from 1 team
    // Solo mode: ≤ 1 player alive
}
```

**Win Conditions (Team Mode):**
- Alive count ≤ team size (e.g., ≤ 4 for squads)
- All remaining living players are on the same team
- Game started at least 5 seconds ago (prevent instant wins)

**Team Elimination Logic** — @file server/src/objects/player.ts:3255

```typescript
teamWipe(): void {
    let team: Team | undefined;
    let players: readonly Player[] | undefined;
    if ((players = (team = this._team)?.players)?.every(p => p.dead || p.disconnected || p.downed)) {
        // If ALL teammates are dead, disconnected, OR downed → wipe the team
        for (const player of players) {
            if (player === this) continue;
            player.health = 0;
            player.die({ source: DamageSources.FinallyKilled });
        }
        this.game.teams.delete(team!);
    }
}
```

**Team Elimination Trigger:**
When a player dies, check if all teammates are now:
- `dead` ✓
- `disconnected` ✓
- `downed` (incapacitated, can still revive) ✓

If all three conditions apply, the entire team is eliminated (downed players are forced to die).

## Team Spectate Mode

### Automatic Teammate Spectation

When a player is eliminated (health ≤ 0), they can spectate allies:

**Spectate Priority** — @file server/src/objects/player.ts:2463

```typescript
case SpectateActions.BeginSpectating: {
    if (this.game.isTeamMode && this._team?.hasLivingPlayers()) {
        // Find closest alive teammate
        toSpectate = this._team.getLivingPlayers()
            .reduce((a, b) => 
                Geometry.distanceSquared(a.position, this.position) < 
                Geometry.distanceSquared(b.position, this.position) ? a : b
            );
    } else if (this.killedBy !== undefined && !this.killedBy.dead) {
        // Fallback: spectate the player who killed you
        toSpectate = this.killedBy;
    } else if (spectatablePlayers.length > 1) {
        // Fallback: random spectatable player
        toSpectate = pickRandomInArray(spectatablePlayers);
    }
}
```

**Spectate Behavior:**
1. **First choice:** Closest living teammate (if team mode and team has survivors)
2. **Second choice:** The player who killed you (revenge spectate)
3. **Third choice:** Random alive player
4. **Cycling:** Players can cycle through prev/next alive players with SpectateActions

## Team Downing & Revival

### Downed State

A player enters "downed" state when health hits zero **but is not eliminated**:

**Downed Trigger** — @file server/src/objects/player.ts:3279

```typescript
down(source?: GameObject | DamageSources, weaponUsed?: DamageParams["weaponUsed"]): void {
    // ...
    this.downed = true;
    this.downedBy = {
        player: source instanceof Player ? source : undefined,
        item: weaponUsed instanceof InventoryItemBase ? weaponUsed : undefined
    };
    this.health = 100;  // ← Reset to incapacitation health
    this.adrenaline = this.minAdrenaline;
    this._team?.setDirty();
}
```

**Properties:**
- `downed: boolean` — Incapacitated state
- `downedBy: { player: Player, item: InventoryItem }` — Who downed the player (for credit)
- `health: 100` — Respawn at a fixed health pool while downed

### Revival Mechanics

Only teammates can revive a downed ally. Revival is an action with duration and movement restrictions.

**Revival Action** — @file server/src/inventory/action.ts

```typescript
class ReviveAction extends Action {
    readonly duration: number;  // e.g., 8 seconds
    // Called repeatedly while reviving
    onEquip() { /* start revive timer */ }
    // Triggered when revive completes
    onEnd() { player.revive(); }
    // Triggered if interrupted
    cancel() { player.beingRevivedBy = undefined; }
}
```

**Revival Flow:**
1. Teammate enters ReviveAction (8-second duration)
2. Reviving teammate's movement speed reduced 50% (`beingRevivedBy ? 0.5 : 1`)
3. If reviving teammate takes damage or moves away → action cancelled
4. If timer completes → `player.revive()` called

**Revive Method** — @file server/src/objects/player.ts:3322

```typescript
revive(): void {
    this.downed = false;
    this.beingRevivedBy = undefined;
    this.downedBy = undefined;
    this.health = [some value];  // Reset health from incapacitation pool
    // ... restore normal state
}
```

### Revival Timing & Kill Credit

- **Revival Window:** Downed player can be revived for up to ~30 seconds (game-defined)
- **Time-Out:** If not revived in time, downed player automatically dies
- **Kill Credit:** If downed player dies after time-out → credit goes to player who downed (stored in `downedBy`)
- **Interrupt Kill:** If downed player is damaged while downed → killed by the attacker (not the original downer)

## Serialization

### TeammateData in UpdatePacket

Teammates are sent to the client in each UpdatePacket to display health, position, and status.

**TeammateData Structure** — @file common/src/packets/updatePacket.ts:342

```typescript
data.teammates = strm.readArray(() => ({
    id: strm.readObjectId(),
    position: strm.readPosition(),
    normalizedHealth: strm.readFloat(0, 1, 1),
    downed: (flags & 2) !== 0,          // Bit 1
    disconnected: (flags & 1) !== 0,    // Bit 0
    colorIndex: strm.readUint8()        // Role identifier (0–3)
}));
```

**Serialization Order:**
1. Array length (variable-length encoded)
2. For each teammate:
   - Flags byte: `(downed ? 2 : 0) + (disconnected ? 1 : 0)`
   - Object ID (2 bytes)
   - Position (4 bytes: x, y each 2 bytes)
   - Normalized health (1 byte: 0–255 mapped to 0–1)
   - Color index (1 byte: 0–3)

**Total per teammate:** ~9 bytes

### Team ID in UpdatePacket

Each player's team ID is sent separately for UI purposes:

```typescript
if (hasTeamID) {
    strm.writeUint8(teamID);  // 1 byte: 0–255 team IDs
}
```

### Dirty Flags for Team Updates

Team data is only sent when the team roster changes (join, leave, death, revive):

- `player.dirty.teammates = true` → next UpdatePacket includes teammates array
- `team.setDirty()` → marks all teammates for update in next tick
- Update only occurs if players changed (join, leave, downed, revived, died)

## Client Display

### Teammate List (HUD)

The client renders a list of teammates with:
- **Player Name** (colored by team)
- **Health Bar** (percentage of max health)
- **Status Icons:**
  - ✓ Alive
  - ↓ Downed (incapacitated)
  - ✗ Disconnected
- **Color Index** (for easy reference: "Teammate 1", "Teammate 2", etc.)

Data comes from the `teammates` array in UpdatePacket.

### Minimap Markers

Each teammate is shown on the minimap with:
- **Position indicator** (circle/dot)
- **Team color** (distinct from enemies)
- **Health bar** (beneath marker)
- **Downed state** (different opacity or icon)

### Position Indicators In-Game

Teammates are rendered with:
- **Name Label** (above player)
- **Team Color Highlight** (outline or team-colored shadow)
- **Health Bar** (below name)

### Death Counter

The HUD displays alive teammate count: "Allies: 2 / 4"

## Known Gotchas & Edge Cases

### 1. Downed All Means Eliminated

If all teammates are downed, **they are all automatically eliminated**. There's no scenario where a team can be revived from a wipe. The `teamWipe()` method forces all downed teammates to die:

```typescript
// If all teammates downed → die with "FinallyKilled" source
if (players.every(p => p.dead || p.disconnected || p.downed)) {
    // Eliminate everyone
}
```

### 2. Shared Kill Credit is Contextual

Kill credit only goes to the player who downed IF:
- The downer is still alive (not dead)
- The downer is on the same team as the finisher

Otherwise, the finisher always gets credit:

```typescript
if (downedBy && downedBy.teamID === source.teamID) {
    // Credit to downer
} else {
    // Credit to finisher
}
```

### 3. Self-Damage Applies Even in Teams

A player can damage themselves (splash damage, grenade throws). This allows grenade balance but players cannot eliminate their own team by self-damage due to `source.id !== this.id` check.

### 4. Disconnection ≠ Death

A disconnected player is excluded from "hasLivingPlayers()" and counts as eliminated for win conditions. However:
- Disconnected teammates still appear in the teamnates list (marked as disconnected)
- Disconnected teammates can be revived if they reconnect (rare edge case)
- If all teammates disconnect, the team is immediately wiped

### 5. Revival Interruption

Revival can be interrupted by:
- Reviving teammate takes damage
- Reviving teammate moves away (action cancelled)
- Downed player takes damage (not interrupted, but kills the downed)
- Network disconnect

If interrupted, the action restarts from zero. No partial progress is saved.

### 6. Late-Game Team Elimination

The game checks win condition every tick. As soon as:
- Alive count ≤ team size (e.g., 4)
- All survivors are on same team

→ Game immediately ends. No explicit "team elimination" event; the system relies on the win condition check.

## Related Documents

- **Tier 2:** [Team System](../README.md) — Team roster management, custom lobbies, team formation
- **Tier 2:** [Game Loop](../../game-loop/) — Game tick cycle, when team state is updated
- **Tier 2:** [Networking](../../networking/) — UpdatePacket structure, how teammate data is sent to clients
- **Tier 3:** [UpdatePacket Module](../../networking/modules/updatepacket.md) — Teammate data serialization details
- **Tier 1:** [Architecture](../../../../architecture.md) — System overview and team mode configuration
- **Tier 1:** [Data Model](../../../../datamodel.md) — Team data structures and relationships
