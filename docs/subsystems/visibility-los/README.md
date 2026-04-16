# Visibility & Line-of-Sight System

<!-- @tier: 2 -->
<!-- @parent: docs/architecture.md -->
<!-- @modules: docs/subsystems/visibility-los/modules/ -->
<!-- @source: server/src/objects/player.ts, server/src/utils/grid.ts, common/src/utils/suroiByteStream.ts -->

## Purpose

Server-side visibility and culling system: determines which game objects each player can see, filters them from the `UpdatePacket` to minimize network bandwidth, and handles special visibility rules for spectators, emotes, explosions, and bullets. The system prevents information leaks (players seeing other players through walls) and optimizes network traffic by only syncing visible state.

## Key Files & Entry Points

| File | Purpose |
|------|---------|
| `server/src/objects/player.ts` | `Player.secondUpdate()` — main visibility culling logic; screen hitbox calculation; visible object filtering |
| `server/src/utils/grid.ts` | `Grid.intersectsHitbox()` — broad-phase spatial query via grid cells; layer-aware filtering |
| `common/src/constants.ts` | `GameConstants.explosionMaxDistSquared` — explosion draw distance; `GameConstants.tps = 40` TPS |
| `common/src/utils/math.ts` | `Collision.lineIntersectsRectTest()` — bullet trajectory vs screen bounds; raycast helpers |
| `common/src/utils/hitbox.ts` | `RectangleHitbox` — screen viewport representation |
| `common/src/packets/updatePacket.ts` | `UpdatePacket` — filtered object data sent to clients; `deletedObjects`, `fullObjectsCache`, `partialObjectsCache` |

## Architecture

### Visibility Pipeline

Each tick, per player:

```
Player.secondUpdate() (called once per player per tick)
  ├─ Calculate screen bounds
  │   ├─ Get zoom from equipped scope or override
  │   ├─ Calculate screen dimension: dim = zoom * 2 + 8
  │   └─ Create RectangleHitbox centered on player position
  │
  ├─ Broad-phase spatial query
  │   ├─ Grid.intersectsHitbox(screenHitbox) → Set<GameObject>
  │   ├─ (Layer-aware via adjacentOrEquivLayer filter)
  │   └─ Returns all objects within screen bounds
  │
  ├─ Track visibility changes
  │   ├─ Compare newVisibleObjects vs this.visibleObjects
  │   ├─ Collect deleted objects → packet.deletedObjects[]
  │   └─ Mark new objects as fullObjects (full serialization)
  │
  ├─ Serialize visible dirty objects
  │   ├─ fullObjects: include all full-dirty + newly visible
  │   └─ partialObjects: include partial-dirty that are visible
  │
  ├─ Special culling (bullets, explosions, emotes)
  │   ├─ Bullets: lineIntersectsRectTest(trajectory, screenBounds)
  │   ├─ Explosions: inside hitbox AND within maxDist
  │   └─ Emotes: if emote.player is visible
  │
  └─ UpdatePacket construction
      └─ Send packet with filtered objects + visibility metadata
```

### View Range Calculation

**Screen Hitbox:**
```typescript
const zoom = player.effectiveScope.zoomLevel;  // e.g., 1.5 for rifle scope
const dim = zoom * 2 + 8;                      // e.g., 11 (in units)
screenHitbox = RectangleHitbox.fromRect(dim, dim, player.position);
```

**Update Frequency:**
- Visibility recalculated every **8+ ticks** (~200ms at 40 TPS)
- Forced refresh on `game.updateObjects` flag (object added/removed, building opened)
- Forced refresh when `player.updateObjects` flag set

**Rationale:** Screen hitbox is small and relatively stable; recalculating every 8 ticks amortizes the cost while maintaining responsive visibility.

## Visibility Culling

### Broad-Phase Spatial Query

```typescript
const newVisibleObjects = game.grid.intersectsHitbox(this.screenHitbox);
```

Returns all game objects whose hitbox or spawn position intersects the player's screen bounds. The grid uses:

- **Cell size:** 32 units (from `Grid.cellSize`)
- **Range calculation:** Round screen bounds to grid cells, collect all objects in those cells
- **Layer filtering:** Only returns objects on adjacent/equivalent layers (prevents seeing through walls)

### Visibility Object Tracking

Players maintain a **persistent set** of visible objects:

```typescript
private visibleObjects: Set<GameObject> = new Set();
```

Each `secondUpdate()`:
1. **Calculate new visible set** via `grid.intersectsHitbox()`
2. **Identify deletions** — objects in old set but not new set
3. **Identify additions** — objects in new set but not old set
4. **Send deletions** — `packet.deletedObjects = [id1, id2, ...]`
5. **Mark additions as full** — newly visible objects get full serialization

### Object Serialization Strategy

| Category | Condition | Serialization | Purpose |
|----------|-----------|----------------|---------|
| Full-dirty | Object changed state AND visible | `ObjectSerializations[category].serializeFull()` | Complete state for new/reset objects |
| Partial-dirty | Object changed AND visible AND not full-dirty | `ObjectSerializations[category].serializePartial()` | Delta update for existing objects |
| Not visible | Any dirty state | *skipped* | Information leak prevention |
| Deleted visible | Was visible, now not | `deletedObjects[]` | Client cleanup |

### Layer Visibility

Objects on **adjacent or equivalent layers** are visible:

```typescript
// From grid.intersectsHitbox()
if (includeAll || (object.layer !== undefined && adjacentOrEquivLayer(object, layer))) {
    objects.add(object);
}
```

Layer adjacency matrix:
```
Basement   (layer -2) →  can see objects on: basement, basement transitions
Ground     (layer  0) →  can see objects on: ground, ground transitions
Upstairs   (layer  2) →  can see objects on: upstairs, upstairs transitions
Transitions (layer ±1) →  can see adjacent layers (basement ↔ ground ↔ upstairs)
```

Precise adjacency rules defined by `adjacentOrEquivLayer(object, playerLayer)` utility.

## Special Visibility Rules

### Bullets

Bullets use **trajectory culling**, not grid-based:

```typescript
for (const bullet of game.newBullets) {
    if (!Collision.lineIntersectsRectTest(
        bullet.initialPosition,
        bullet.finalPosition,
        this.screenHitbox.min,
        this.screenHitbox.max
    )) continue;  // Skip if trajectory doesn't intersect screen bounds
    packet.bullets.push(bullet);
}
```

**Known issue:** Uses bullet's initial→final path, not continuous trajectory as bullet moves. For fast-projectile weapons (radio, firework launcher), may miss bullets if player moves quickly. Low impact due to ~0.3–0.8s projectile flight time.

### Explosions

Explosions require **two conditions**:

```typescript
for (const explosion of game.explosions) {
    if (
        !this.screenHitbox.isPointInside(explosion.position)
        || Geometry.distanceSquared(explosion.position, this.position)
            > GameConstants.explosionMaxDistSquared
    ) continue;
    packet.explosions.push(explosion);
}
```

1. Explosion center must be inside **screen hitbox** (viewport)
2. Distance from player to explosion ≤ **`GameConstants.explosionMaxDistSquared`** (draw distance limit)

**Rationale:** Even though explosion is in viewport, very distant explosions are not rendered (performance). Prevents syncing invisible explosions.

### Emotes

Emotes are only visible if their **originating player is visible**:

```typescript
for (const emote of game.emotes) {
    if (!this.visibleObjects.has(emote.player)) continue;
    packet.emotes.push(emote);
}
```

No independent emote distance check; emotes follow owner visibility.

### New Players

Broadcast of newly-joined players differs for **first packet vs subsequent**:

```typescript
const newPlayers = this._firstPacket
    ? game.grid.pool.getCategory(ObjectCategory.Player)  // All players on first join
    : game.newPlayers;                                    // Only new arrivals afterward
```

On first `UpdatePacket`, client learns about all players in the game (for killfeed, etc.). Subsequent updates only announce new joins.

**Team mode special case:**
```typescript
if (!this.game.isTeamMode || teamID !== player.teamID) continue;
fullObjects.add(newPlayer);  // Teammates added to full-sync set
```

Teammates are always visible (no spatial culling) and appear in viewport even if far away.

## Spectator Mode

Spectators see the **spectated player's view**, not their own:

```typescript
const player = this.spectating ?? this;  // Use spectated player if in spectate mode
```

Affects:
- **Visible objects:** Calculated from spectated player's screen position
- **Player data:** Teammate list, health, inventory from spectated player
- **Zoom level:** Spectated player's equipped scope

**Spectate mode transition:**
- When spectate mode starts, `this.startedSpectating = true`
- Next `secondUpdate()` sets `forceInclude = true`, forcing all optional fields to send
- Prevents partial updates from causing inconsistent spectator view

## UpdatePacket Filtering

### Packet Structure

```typescript
interface UpdatePacket {
    // Visibility metadata
    deletedObjects: number[]              // IDs of objects no longer visible
    fullObjectsCache: Set<BaseGameObject> // Objects needing full serialization
    partialObjectsCache: BaseGameObject[] // Objects needing partial serialization
    
    // Filtered object arrays
    bullets: Bullet[]                     // Visible new bullets
    explosions: Explosion[]               // Visible explosions
    emotes: Emote[]                       // Emotes from visible players
    
    // Player-specific data
    playerData: PlayerData                // Your health, position, inventory, etc.
    
    // Broadcast data (same for all players)
    newPlayers: PlayerJoinInfo[]          // New player announcements
    deletedPlayers: number[]              // Disconnected player IDs
    gas: GasState | undefined            // Gas state if changed
    kill Leader: KillLeaderInfo           // Current kill leader
    // ... etc
}
```

**Serialization flow:**
```
fullObjectsCache → ObjectSerializations[category].serializeFull()
partialObjectsCache → ObjectSerializations[category].serializePartial()
→ PacketStream packs into binary ArrayBuffer
→ WebSocket send to client
```

## Performance Characteristics

**Time complexity per player per tick:**
- Grid cell lookup: O(n) neighbors in cells (typically 5–50 objects)
- Visibility change detection: O(n) set diff (insert/delete operations)
- Bullet culling: O(m) where m = new bullets (typically 0–20)
- Explosion culling: O(p) where p = active explosions (typically 0–10)
- **Total:** O(n + m + p) ≈ O(100) per player per tick

**Optimization:**
- Visibility updated only every 8 ticks (25ms × 8 = 200ms minimum latency for discovering visibility changes)
- Spatial grid (32-unit cells) reduces search from O(n²) to O(n)
- Only visible objects serialized (reduces UpdatePacket size)
- Spectators use *virtual* camera view (no additional grid queries)

**Memory:**
- Per-player: `visibleObjects` Set, `screenHitbox` Rect (~1 KB per player)
- Packet caches: temporary Sets reused across ticks

## Known Issues & Gotchas

1. **Visibility refresh latency:** Objects entering view may have up to 8 ticks (200ms) latency before first sync. Rapid position changes can cause choppy appearance.

2. **Bullet trajectory assumption:** Bullet culling uses initial→final position, not continuous path. Fast-moving players may miss bullets that traverse the screen but didn't intersect the *static* screen hitbox.

3. **Explosion distance in viewport:** Explosions inside the screen hitbox are always sent, regardless of `explosionMaxDistSquared` distance. Inconsistent with the two-condition check (off-screen explosions require distance check, on-screen don't).

4. **Layer transitions:** Layer adjacency for stair/door transitions may not be perfectly permissive; players on basement stairs may not see ground-level objects directly above.

5. **Zoom-based view range:** Very high zoom scopes (e.g., 2.0+) create large screen hitboxes, increasing grid query cost. No max-zoom limit imposed.

6. **Spectator-to-player swap:** When spectator returns to playing, there's a 1-frame flicker as visibility recalculates around their actual position (not the spectated player's).

7. **No occlusion culling:** Walls, buildings, obstacles do NOT block LOS (line-of-sight). Visibility is purely based on distance and zoom viewport; players behind walls are still synced if within the rectangular screen bounds.

## Dependencies

- **Depends on:**
  - [Spatial Grid](../spatial-grid/) — `Grid.intersectsHitbox()` broad-phase queries
  - [Collision & Hitbox](../collision-hitbox/) — `RectangleHitbox`, `Collision.lineIntersectsRectTest()`, distance calculations
  - [Core Math & Physics](../core-math-physics/) — `Geometry.distanceSquared()`, angle math
  - [Serialization System](../serialization-system/) — `ObjectSerializations[category]` serialize methods
  - [Server-Side Game Objects](../game-objects-server/) — object definitions, `BaseGameObject` interface
  - [Networking](../networking/) — `UpdatePacket`, `PacketStream` wire protocol

- **Depended on by:**
  - [Game Loop](../game-loop/) — `Player.secondUpdate()` called each tick (step 15)
  - [Networking](../networking/) — determines which objects are in each `UpdatePacket`

## Module Index (Tier 3)

For implementation details, see:
- *No modules documented yet; `Player.secondUpdate()` is monolithic.*

## Related Documents

- **Tier 1:** [System Architecture](../../architecture.md) — network filtering overview
- **Tier 2:** [Spatial Grid](../spatial-grid/) — grid queries for visibility
- **Tier 2:** [Networking](../networking/) — UpdatePacket structure and wire protocol
- **Tier 2:** [Collision & Hitbox](../collision-hitbox/) — hitbox and raycast algorithms
- **Tier 2:** [Game Loop](../game-loop/) — where visibility check runs (step 15)
- **Tier 2:** [Camera Management](../camera-management/) — client-side camera zoom
- **Tier 2:** [Server-Side Game Objects](../game-objects-server/) — object lifecycle and dirty flags
