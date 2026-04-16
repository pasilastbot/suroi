# Airdrop Spawning & Contents

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/airdrop-system/README.md -->
<!-- @source: server/src/game.ts, server/src/objects/parachute.ts, common/src/definitions/obstacles.ts, server/src/data/lootTables.ts -->

## Purpose

This module orchestrates the spawning, flight, descent, and loot distribution of airdrop crates. It manages collision avoidance during spawn placement, animates parachute descent over 8 seconds, triggers crush damage on landing, and distributes 8-item loot pools from curated airdrop loot tables.

## Key Files

| File | Purpose | Complexity |
|------|---------|-----------|
| `server/src/game.ts:summonAirdrop()` | Main spawn function; collision resolution (500-attempt loop); plane + parachute scheduling | High |
| `server/src/objects/parachute.ts` | Parachute entity; performs descent animation; triggers loot crate generation | High |
| `common/src/definitions/obstacles.ts` | Airdrop crate definitions (locked, gold); unlock animation frames | Medium |
| `server/src/data/lootTables.ts` | Loot tables for `airdrop_crate`, `gold_airdrop_crate` | Medium |
| `common/src/constants.ts` | `GameConstants.airdrop` timing & damage constants | Low |

## Business Rules

- **Airdrop spawning is stage-locked** — Only triggered during gas stage 3, 7, or 11 via `gas.summonAirdrop` flag (hardcoded in `gasStages.ts`). No random or interval spawning without game mode override.
- **One spawn per stage** — Each stage triggers exactly one `summonAirdrop()` call; approximately 30 seconds minimum between airdrops (early game ~65s, mid ~190s, late ~320s).
- **Spawn position is gas-center-based** — Airdrop crate is placed at the center of the current gas safe zone (or random point near it). Plane arrives from random compass direction.
- **Collision avoidance is expensive** — Recursive collision checking with up to 500 placement attempts. If all fail, position nudges in 8 compass directions; if still colliding at 500 attempts, placement succeeds anyway (rare edge case).
- **Crush damage is unarmored** — Players hit by landing airdrop take 300 HP of `piercingDamage()`, bypassing all armor and barriers.
- **Locked vs. Gold airdrops** — 95% chance → regular crate, 5% chance → gold crate (triggered by `forceGold` parameter or game mode `summonAirdropsInterval`). Halloween mode spawns pumpkin variant.
- **Height = 0 triggers generation** — Parachute descends for 8 seconds; when internal height < 0, obstacle crate is materialized, crush damage is applied, and parachute is destroyed.
- **Loot spawns on breaking, not on landing** — Airdrop crate has `hasLoot: true` but spawns items only after player destroys it (low health = 150 HP).

## Data Lineage

### Airdrop Spawning Flow

```
Game.summonAirdrop(position: Vector, forceGold: boolean)
  ↓ [Load airdrop crate definition]
  Obstacle.Definitions["airdrop_crate_locked" | "gold_airdrop_crate_locked_force"]
  ↓ [Collision resolution]
  Grid.intersectsHitbox() + existing airdrops + obstacles
  ↓ [Position finalized]
  airdrop: { position, type: ObstacleDefinition }
  ↓ [Scheduled parachute spawn]
  await GameConstants.airdrop.flyTime (30000 ms)
  ↓ [Plane arrives]
  Parachute(game, position, airdrop)
  ↓ [Descent animation]
  Parachute.height: 1 → 0 over GameConstants.airdrop.fallTime (8000 ms)
  ↓ [Landing]
  Map.generateObstacle(airdrop.type, crate_position)
  ↓ [Crush damage + effects]
  DamageSources.Obstacle (300 HP)
  ↓ [Loot table loaded on crate break]
  getLootFromTable(modeName, "airdrop_crate" | "gold_airdrop_crate")
  ↓ [Items dropped]
  Loot[8 items]
```

### Loot Generation

```
Airdrop Crate Definition
  lootTable: "airdrop_crate" | "gold_airdrop_crate"
    ↓
  getLootFromTable(game.modeName, lootTableName)
    ↓ [Lookup loot table by name]
  LootTables[modeName]["airdrop_crate"]
    ↓ [Array of 8 weighted item entries]
  [
    { idString: "airdrop_equipment", count: 1, weight: 1 },        // Slot 1
    { idString: "airdrop_scopes", count: 1, weight: 1 },           // Slot 2
    { idString: "airdrop_healing_items", count: 1, weight: 1 },    // Slot 3
    { idString: "airdrop_skins", count: 1, weight: 1 },            // Slot 4
    { idString: "airdrop_melee", count: 1, weight: 1 },            // Slot 5
    { idString: "ammo", count: 1, weight: 1 },                      // Slot 6
    { idString: "airdrop_guns", count: 1, weight: 1 },             // Slot 7
    { idString: "frag_grenade", count: 3, weight: 2 } OR null      // Slot 8 (50:50 null)
  ]
    ↓
  [Apply per-mode loot overrides]
  LootTables[modeName].airdrop_crate (may override weights/items)
    ↓
  [Weighted random selection from loot pools]
  player.inventory.interact(item) OR loot.count++
```

## Airdrop Entity Structure

### Game.airdrops Array

```typescript
type Airdrop = {
    position: Vector          // Final crate position (after collision resolution)
    type: ObstacleDefinition  // Airdrop obstacle definition (from Obstacles registry)
}
```

**Stored in:** `Game.airdrops: Airdrop[]` (@file server/src/game.ts:146)

**Lifecycle:**
1. Created by `summonAirdrop()` as plain object
2. Referenced by scheduled `Parachute` instance
3. Removed when parachute lands (`this.game.airdrops.splice(index, 1)`)

### Parachute Entity

```typescript
class Parachute extends BaseGameObject.derive(ObjectCategory.Parachute) {
    private _height: number             // 1 (deployed) → 0 (landed)
    endTime: number                     // Date.now() + GameConstants.airdrop.fallTime (8000 ms)
    hitbox: CircleHitbox(r=10)          // Collision area for crush damage
    _airdrop: Airdrop                   // Reference to crate definition + position
    
    fullAllocBytes = 8                  // Position (X, Y)
    partialAllocBytes = 4               // Height (lerped 0–1)
}
```

**Serialized fields:** `{ height, full: { position } }` (@file server/src/objects/parachute.ts:93)

**Network budget:** Full update = 8 bytes, partial update = 4 bytes (per packet)

### Locked Crate Definition

**idString:** `airdrop_crate_locked` (@file common/src/definitions/obstacles.ts:2386)

```typescript
{
    idString: "airdrop_crate_locked",
    name: "Airdrop",
    material: "metal_light",
    health: 10000,                      // Indestructible during parachute phase
    indestructible: true,               // Cannot be damaged until replaced
    reflectBullets: true,
    hitbox: RectangleHitbox.fromRect(8.7, 8.7),
    spawnHitbox: RectangleHitbox.fromRect(10, 10),
    hideOnMap: true,                    // Don't show on minimap during descent
    isActivatable: true,                // Players can interact
    sound: { name: "airdrop_unlock", maxRange: 64 },
    replaceWith: {
        idString: { 
            "airdrop_crate": 0.95,      // Regular crate (95%)
            "gold_airdrop_crate": 0.05  // Gold crate (5%)
        },
        delay: 800                      // Time until locked → unlocked transition
    },
    airdrop: {
        unlockFrame: "airdrop_crate_unlocking",     // Animation frame during unlock
        particle: "airdrop_particle",                // Particle effect during unlock
        particleVariations: 2                        // Variation index 0–1
    }
}
```

**Gold variant:** `airdrop_crate_locked_force` (forces gold crate, used when `forceGold = true`)

### Unlocked Crate Definition

**idString:** `airdrop_crate` (@file common/src/definitions/obstacles.ts:2489)

Spawned when locked crate is activated/destroyed:

```typescript
{
    idString: "airdrop_crate",
    name: "Airdrop Crate",
    material: "crate",
    health: 150,                        // Destroyable by players (low HP)
    scale: { spawnMin: 1, spawnMax: 1, destroy: 0.5 },
    hitbox: RectangleHitbox.fromRect(8.7, 8.7),
    spawnHitbox: RectangleHitbox.fromRect(10, 10),
    hideOnMap: true,
    hasLoot: true,                      // Contains loot table entry
    lootTable: "airdrop_crate"          // Loot pool name
}
```

**Gold variant:** `gold_airdrop_crate` (health: 170, higher tier loot)

## Airdrop Spawning Rules

### Stage-Based Triggering

Airdrop spawning is driven by the gas system and is **not random**. Spawns occur at fixed stages:

| Stage Index | Stage Name | Elapsed Time | Triggered? | Notes |
|---|---|---|---|---|
| 0 | Zone 0 Waiting | 0 sec | No | Game starting |
| 1 | Zone 0 → 1 | ~10 sec | No | Gas shrinking |
| 2 | Zone 1 Waiting | ~45 sec | No | Waiting for players |
| **3** | **Zone 1 → 2** | **~65 sec** | **YES** | **First airdrop** |
| 4 | Zone 2 Waiting | ~100 sec | No |
| 5 | Zone 2 → 3 | ~145 sec | No |
| 6 | Zone 3 Waiting | ~190 sec | No |
| **7** | **Zone 3 → 4** | **~190 sec** | **YES** | **Second airdrop** |
| 8 | Zone 4 Waiting | ~240 sec | No |
| 9 | Zone 4 → 5 | ~290 sec | No |
| 10 | Zone 5 Waiting | ~320 sec | No |
| **11** | **Zone 5 → Final** | **~320 sec** | **YES** | **Third airdrop** |
| 12+ | Final Zone | ~360 sec+ | No | No more airdrops |

**Code reference:** `server/src/data/gasStages.ts` contains `summonAirdrop: true` flag for stages 3, 7, 11

**Triggering in game loop:** (@file server/src/game.ts:328)
```typescript
if (this.mode.summonAirdropsInterval !== undefined 
    && (this.now - this.lastAirdropTime) >= this.mode.summonAirdropsInterval) {
    this.summonAirdrop(...)
    this.lastAirdropTime = this.now
}
```

### Location Selection

**Primary logic:** (@file server/src/game.ts:328–331)
```typescript
position = this.map.getRandomPosition(
    new CircleHitbox(15),           // Spawn area radius = 15 units
    {
        maxAttempts: 500,
        spawnMode: MapObjectSpawnMode.GrassAndSand,
        collides: pos => 
            Geometry.distanceSquared(pos, this.gas.currentPosition) 
            >= this.gas.newRadius ** 2   // Must be outside current gas
    }
) ?? this.gas.newPosition               // Fallback: gas zone center
```

**Constraints:**
- **Inside safe zone?** Airdrop position must be ≥ current gas safe radius away from gas center (i.e., in the zone about to be gassed)
- **Terrain type?** Only spawn on grass or sand (block trees, water, buildings)
- **Max attempts?** Try up to 500 random positions; fallback to gas center if all fail

**Result:** Airdrop lands in the shrinking zone, incentivizing players to fight near the airdrop location.

### Frequency

**Regular gameplay:** ~30 seconds between airdrops (3 total per 6-minute game)
- Stage 3: ~1 min 5 sec
- Stage 7: ~3 min 10 sec
- Stage 11: ~5 min 10 sec

**Mode-dependent:** `ModeDefinition.summonAirdropsInterval` can override (default: `30e3` ms) (@file common/src/definitions/modes.ts:78)

**Plane spawn location:** (@file server/src/game.ts:1307–1310)
```typescript
const direction = randomRotation()                    // 0–2π radians
const planePos = Vec.add(
    position,
    Vec.fromPolar(direction, -GameConstants.maxPosition)  // Distance ~2048 units
)
```

## Contents Table

### Airdrop Crate Loot Slots

Each airdrop crate contains **exactly 8 item stacks**, one per slot. Slots are filled by weighted random selection from loot pools.

#### Regular Airdrop (`airdrop_crate`)

| Slot | Loot Pool | Weight | Items | Purpose |
|---|---|---|---|---|
| 1 | `airdrop_equipment` | 1 | Tactical helmet (3.5k HP), tactical vest (60 armor), tactical pack (1500 cap) | Early-game armor upgrade |
| 2 | `airdrop_scopes` | 1 | 8× scope (0.85 weight), rare 16× (0.15 weight) | High-tier optics |
| 3 | `airdrop_healing_items` | 1 | Gauze stack (5×), medikit, cola, tablets | Sustain items |
| 4 | `airdrop_skins` | 1 | Cosmetic skins (various), rare ghillie | Cosmetics |
| 5 | `airdrop_melee` | 1 | Crowbar, hatchet, sickle, rare pan (0.05 weight) | Melee weapons |
| 6 | `ammo` | 1 | Bullets, shells, 7.62 rounds, 12g buckshot | Ammunition |
| 7 | `airdrop_guns` | 1 | MG36, SR-25, VSS, Vector, Vepr-12, Deagle, M16A4, FAMAS, AK-47, Mosin Nagant, Tango 51, Stoner 63, M4, Intervention | High-tier weapons (11 assault/marksman, 3 sniper) |
| 8 | `frag_grenade` × 3 OR null | 2:1 | Frag grenade (3 count) or nothing | Throwable or slot skip |

**Code reference:** `server/src/data/lootTables.ts` — `airdrop_crate` (key lookup in `LootTables[modeName]`)

**Total items per crate:** 8 stacks (assuming all slots roll)

#### Gold Airdrop (`gold_airdrop_crate`)

Same as regular **except Slot 7** replaces `airdrop_guns` with:

| Slot | Loot Pool | Weight | Items | Purpose |
|---|---|---|---|---|
| 7 | `gold_airdrop_guns` | 1 | M1 Garand, DP-12, ACR, PP-19, Negev, MG5, MK-18, L115A1, rare G19 (0.0005 weight) | Legendary weapons: marksman + exotic |

**Gold weapons are tier-2 guns** (higher TTK, better handling) + 1 legendary sidearm (G19 with 0.05% spawn chance).

**Code reference:** Defined in `LootTables[modeName].gold_airdrop_crate`

### Loot Pool Definitions

#### `airdrop_equipment`
```
[Tactical helmet, tactical vest, tactical pack]
Weight: uniform 1:1:1
Count: 1 per slot
```

#### `airdrop_scopes`
```
[8× scope (weight: 0.85), 16× scope (weight: 0.15)]
Rare 16× occurs 15% of the time
```

#### `airdrop_healing_items`
```
[
    { gauze, count: 5, weight: 0.25 },
    { medikit, count: 1, weight: 0.25 },
    { cola, count: 1, weight: 0.25 },
    { tablets, count: 1, weight: 0.25 }
]
Uniform distribution
```

#### `airdrop_melee`
```
[
    { crowbar, count: 1, weight: 1 },
    { hatchet, count: 1, weight: 1 },
    { sickle, count: 1, weight: 1 },
    { pan, count: 1, weight: 0.05 }  ← Rare (5% chance)
]
```

#### `airdrop_guns` (regular)
```
[
    mg36, sr25, vss, vector, vepr12, deagle (exotic sidearm),
    m16a4, famas, ak47 (assault rifles),
    mosin_nagant, tango_51 (bolt sniper),
    stoner_63, m4, intervention (assault → sniper crossover)
]
Count: 1 per selection
Weight: uniform unless specified
```

#### `gold_airdrop_guns` (premium)
```
[
    m1_garand, dp12, acr, pp19, negev, mg5, mk18, l115a1 (tier-2 weapons),
    g19 (weight: 0.0005)  ← Legendary sidearm, ultra-rare
]
Count: 1 per selection
```

### Rarity & Distribution

| Rarity | Examples | Frequency |
|---|---|---|
| **Guaranteed** | Slot 1–7 (armor, scopes, healing, melee, ammo, guns) | 100% per airdrop (if slot rolls) |
| **Common** | Regular weapons (mg36, sr25, vector) | ~12–14% each within `airdrop_guns` |
| **Uncommon** | Tactical armor, 8× scope, healing stacks | ~25–33% per slot |
| **Rare** | Pan (melee), 16× scope, gold guns | 5–15% per item within pool |
| **Ultra-rare** | G19 (gold galand sidearm) | 0.05% (1 in 2000 gold airdrops) |
| **Slot skip** | Slot 8 frag_grenade = null | 33% (2:1 weights) |

### Overflow Handling

**No overflow in airdrops.** Each loot stack is spawned as a separate `Loot` entity at crate position with identical count. If inventory full:

```typescript
Loot.interact(player) → 
    if (player.inventory.isFull()) {
        loot.count stays in world      // Stays on ground
        loot.highlight()               // Visual indicator
    } else {
        inventory.interact(loot)       // Takes count
        if (loot.count === 0) remove loot
    }
```

**Network sync:** Loot counts not replicated in airdrops (loot spawns post-break, standard loot pickup rules apply).

## Flight Path Animation

### Plane Phase (flyTime = 30s)

**Duration:** 30 seconds (@file common/src/constants.ts:87)

**Plane movement:** Not animated server-side. Plane is a simple object in `Game.planes[]`:

```typescript
Game.planes.push({
    position: planePos,         // Far distance (~2048 units away)
    direction: randomDirection  // 0–2π radians toward airdrop target
})
```

**Network sync:** Plane position sent each tick to all players (@file common/src/packets/updatePacket.ts).

**Client-side rendering:** Client animates plane sprite moving across sky toward airdrop.

**Purpose:** Visual warning that airdrop is incoming; allows players to rotate toward airdrop location.

### Parachute Phase (fallTime = 8s)

**Duration:** 8 seconds (@file common/src/constants.ts:87)

**Height interpolation:** (@file server/src/objects/parachute.ts:79–81)

```typescript
const elapsed = this.endTime - this.game.now
this._height = Numeric.lerp(0, 1, elapsed / GameConstants.airdrop.fallTime)
```

**Descent curve:** Linear lerp from `height = 1` (deployed) to `height = 0` (landed).
- `height = 1.0` → Parachute fully deployed (t = 0s)
- `height = 0.5` → Halfway down (t = 4s)
- `height = 0.0` → Landing (t = 8s)
- `height < 0` → Trigger landing sequence

**Visual state transitions** (client-side interpretation):

| Height | State | Visual Effect |
|---|---|---|
| > 0.8 | Deployed | Parachute fully open, slow descent |
| 0.5–0.8 | Descending | Parachute partially closing |
| 0.1–0.5 | Fast descent | Parachute nearly closed |
| ≤ 0 | Landed | Crate materialized, smoke particle spawned |

**Network budget:** Height sent as partial update (4 bytes per tick) to all players for client-side animation.

## Landing Mechanics

### Landing Detection

**Trigger:** (@file server/src/objects/parachute.ts:28–29)

```typescript
if (this._height < 0) {
    // Landing sequence
}
```

**Timing:** Occurs 8 seconds after parachute creation.

### Collision Detection

**Crate hitbox query:** (@file server/src/objects/parachute.ts:32)

```typescript
const crate = this.game.map.generateObstacle(this._airdrop.type, this.position)
```

**Objects checked for crush damage:** (@file server/src/objects/parachute.ts:37–70)

```typescript
for (const object of this.game.grid.intersectsHitbox(crate.hitbox, crate.layer)) {
    if (object.hitbox?.collidesWith(crate.hitbox)) {
        case object.isPlayer:
            object.piercingDamage({ amount: 300, source: DamageSources.Obstacle })
        case object.isObstacle:
            object.damage({ amount: Infinity, source: crate })
        case object.isBuilding && object.scopeHitbox?.collidesWith(crate.hitbox):
            object.damageCeiling(Infinity)
    }
}
```

### Final Position

**Determined at spawn time** by collision resolution loop (@file server/src/game.ts:1182–1300):

1. Start position: `map.getRandomPosition()` or gas center
2. Collision checks: Iterate up to 500 times
3. Final position: Clamped to map bounds
4. Parachute spawns at this position, descends vertically (no horizontal drift)
5. Crate appears at same position when height < 0

**No drift or horizontal movement** — airdrop falls straight down.

### Crate Opening Mechanics

**Locked → Unlocked transition:** (@file common/src/definitions/obstacles.ts:2404–2417)

```typescript
replaceWith: {
    idString: { "airdrop_crate": 0.95, "gold_airdrop_crate": 0.05 },
    delay: 800  // 0.8 seconds after landing
}
```

**Unlock animation:**
- Locked crate displays frame `"airdrop_crate_unlocking"` for 800 ms
- Spawns particle effect `"airdrop_particle"` (2 variations)
- After 800 ms, locked crate replaced with unlocked crate

**Player interaction:** (@file server/src/objects/obstacle.ts)

Once unlocked, players can:
1. **Destroy** — Deal 150+ HP damage, crate breaks, loot spawns
2. **Interact** — If `isActivatable`, trigger open animation (visual only, loot still requires break)

**Loot spawning:** (@file server/src/objects/obstacle.ts:124–127)

```typescript
if (definition.hasLoot) {
    this.loot = getLootFromTable(this.game.modeName, definition.lootTable ?? definition.idString)
}
// Later, when health ≤ 0:
for (const item of this.loot) {
    this.game.addLoot(item.idString, crate.position, layer, { count: item.count })
}
```

## Loot Distribution

### Multiple Items Per Airdrop

**Slots:** Airdrop crate distributes 8 loot items, one per slot.

**Per-slot generation:** (@file server/src/data/lootTables.ts)

Each slot references a loot pool name (string key), which is resolved at break time:

```typescript
const lootTable = LootTables[game.modeName][definition.lootTable]
const items = lootTable.map(slot =>
    pickRandomInArray(slot.options, true)  // Weighted random selection
)
```

**Weighted selection:** Each slot pool has items with `weight` values; higher weight = more likely.

Example: `airdrop_melee` pool:
```
[
    { idString: "crowbar", weight: 1 },
    { idString: "hatchet", weight: 1 },
    { idString: "sickle", weight: 1 },
    { idString: "pan", weight: 0.05 }
]
```

Probability:
- Pan: 0.05 / (1 + 1 + 1 + 0.05) = **1.6%**
- Crowbar/Hatchet/Sickle: 1.0 / 3.05 = **32.8% each**

### Stacking Rules

**Single-item stacks:** Most loot (weapons, armor, scopes) have count = 1.

**Multi-item stacks:**
- Gauze: count = 5 (@file server/src/data/lootTables.ts)
- Frag grenade: count = 3
- Ammo: varies by ammunition type (usually 30–60 rounds)

**Inventory placement:**

```typescript
player.inventory.interact(item) →
    if (item.count > 1) {
        slot = inventory.findStack(item.idString)
        if (slot.free) slot.count += item.count
        else item.count -= cap, stay in world
    } else {
        slot = inventory.findEmpty()
        slot.item = item, count = 1
        remove from world if fully taken
    }
```

### Overflow Handling

**No server-side overflow.** Each item spawns independently at crate position.

**Player-side inventory full?**

```
Loot.interact(player) →
    canAdd = player.inventory.add(loot)
    if (!canAdd) {
        loot.count stays > 0
        loot.highlight() // Visual indicator
    } else {
        Loot.remove()
    }
```

**Items on ground:** Multiple loot entities around crate can coexist. Players must decide what to pick up given limited inventory space.

**Network sync:** Loot positions/counts sent via `UpdatePacket` standard loot serialization (not airdrop-specific).

## Airdrop Signals

### Visual Indicators

#### Plane Sprite

**Appearance:** Animated plane sprite flying across sky at high altitude.

**Duration:** 30 seconds (flyTime).

**Direction:** Toward airdrop target.

**Client-side:** Rendered by client based on `Game.planes` array in `UpdatePacket`.

#### Parachute Sprite

**Appearance:** Colorful parachute sprite with ropes, descending vertically.

**Height animation:** Client-side lerp based on `Parachute._height` (0–1).

**Duration:** 8 seconds (fallTime).

**Client-side:** Rendered by PixiJS animation, height controls sprite scale/position.

#### Map Ping

**Type:** `MapPings.fromString("airdrop_ping")` (@file server/src/game.ts:1315)

**Appearance:** Distinctive icon on minimap / world map.

**Duration:** Indefinite (persists until crate is looted/destroyed or game ends).

**Broadcast:** All players receive ping in `UpdatePacket.mapPings[]`.

```typescript
this.mapPings.push({
    definition: MapPings.fromString<MapPing>("airdrop_ping"),
    position: airdrop.position
})
```

### Audio & Particle Signals

#### Unlock Sound

**Trigger:** When locked crate transitions to unlocked (800 ms after landing).

**Sound:** `"airdrop_unlock"` audio clip (@file common/src/definitions/obstacles.ts:2398)

**Range:** 64 units
**Falloff:** 0.3 (sound weakens quickly with distance)

**Code reference:** (@file server/src/objects/obstacle.ts)

#### Particle Effects

**Landing smoke:** `"airdrop_smoke_particle"` spawned at crate position (@file server/src/objects/parachute.ts:39)

```typescript
this.game.addSyncedParticles("airdrop_smoke_particle", crate.position, crate.layer)
```

**Unlock particles:** `"airdrop_particle"` with variations (0–1) per crate definition (@file common/src/definitions/obstacles.ts:2414–2415)

```typescript
{ 
    particle: "airdrop_particle",
    particleVariations: 2
}
```

**Halloween variant:** `"pumpkin_airdrop_metal_particle"` with 3 variations, 4 particles per effect.

**Duration:** Particles decay server-side and client watches for `SyncedParticle` updates.

## Pickup Interaction

### Standard Loot Interaction

**Proximity check:** (@file server/src/objects/loot.ts or player.ts)

```typescript
if (distance(player, loot) ≤ 3 * player.sizeMod) {
    if (loot.canInteract(player)) {
        player.inventory.interact(loot)
    }
}
```

**Interaction range:** Default 3 units (scaled by player size modifier).

**Items are generic loot entities** — airdrop loot has no special "airdrop" tag, so pickup follows standard rules.

### Crate Destruction

**Health:** 150 HP (regular), 170 HP (gold) (@file common/src/definitions/obstacles.ts:2507, 2525)

**Destruction triggers loot spawn:**

```typescript
if (crate.health ≤ 0) {
    for (const item of this.loot) {
        game.addLoot(item.idString, crate.position, layer, { count: item.count })
    }
}
```

**Crate removal:** Obstacle is deleted after health ≤ 0 (standard obstacle lifespan).

**Loot persistence:** Items remain until picked up or looted by another player.

## Network Synchronization

### Airdrop Position & State Updates

#### Plane Updates

**Sent each tick in `UpdatePacket`:**

```typescript
// Pseudocode from UpdatePacket
for (const plane of this.game.planes) {
    stream.writeVec(plane.position)   // X, Y (network-compressed)
    stream.writeRotation(plane.direction)  // Facing direction
}
```

**Frequency:** Every game tick (40 Hz = 25 ms per update).

**Broadcast:** All connected players receive plane positions.

#### Parachute Updates

**Full data (once):**
```typescript
stream.writeVec(parachute.position)    // Position (X, Y)
```

**Partial data (every tick):**
```typescript
stream.writeUint8(parachute._height * 255)  // Height (0–1) as byte
```

**Allocation:** `fullAllocBytes = 8`, `partialAllocBytes = 4` (@file server/src/objects/parachute.ts:11–12)

**Frequency:** Full update sent on creation; partial sent every tick thereafter.

**Broadcast:** All connected players.

#### Crate Updates

**Crate is standard `Obstacle` entity:**
```typescript
stream.writeVec(crate.position)       // Position
stream.writeRotation(crate.rotation)  // Rotation (0 for airdrop)
stream.writeUint8(crate.scale)        // Scale (1.0 for airdrop)
stream.writeUint16(crate.health)      // Current HP
```

**Frequency:** Partial updates only if dirty (health changed).

**Broadcast:** All players who can see the crate (within fov/viewport).

### Contents Serialization

**Loot table is NOT serialized.** Loot is generated server-side at break time.

**Why?** Prevents cheating (clients can't predict loot contents before crate breaks) and reduces bandwidth (8 items × 3-4 bytes per item = 24–32 bytes saved per airdrop).

**Loot spawning:**
1. Server determines loot slots when crate health ≤ 0
2. Server calls `game.addLoot(item.idString, position, ...)`
3. Loot entities created, added to `ObjectPool`
4. Loot entities serialized normally in next `UpdatePacket`

**Timeline:**
- t=0: Airdrop plane appears (sent to clients)
- t=30s: Parachute spawns (sent to clients)
- t=38s: Parachute lands, crate appears (sent to clients)
- t=38.8s: Crate unlocks, unlock animation plays (crate definition already sent)
- t=??: Player breaks crate (damage synced)
- t=??: Loot entities spawned (sent as part of UpdatePacket `objects[]` array)

**Server-side authoritative:** Server controls all loot generation; clients cannot override or predict.

## Map Integration

### Airdrop Landing Zones

**No hardcoded "landing zone" terrain.** Airdrop can land anywhere in the map that:

1. **Passes collision checks** — Not inside obstacles, buildings, or existing airdrops
2. **Is inside the gas safe zone** — Must be ≥ `gas.newRadius` away from gas center
3. **Is on grass or sand** — Terrain type `MapObjectSpawnMode.GrassAndSand`

**Map variations:**
- Fixed maps (`dev_map`, `debug_map`, etc.) can have hardcoded airdrop spawn points if desired (not currently used)
- Generated maps (random spawn) use the algorithm above

### Restricted Areas

**Airdrops will NOT spawn:**
- Inside water (@file server/src/game.ts:331 — `spawnMode: GrassAndSand`)
- Inside indestructible obstacles (collision resolution prevents)
- Inside buildings with `scopeHitbox` (collision resolution prevents)
- Outside map bounds (clamped)

**Edge case:** If gas center is inside a mountain/building (e.g., late-game mountain), airdrop may fail to find valid position after 500 attempts and default to gas center (fallback: `?? this.gas.newPosition`).

### Map Data Interaction

**Gas stage integration:** (@file server/src/data/gasStages.ts)

Each stage has `summonAirdrop: boolean` flag. Game reads this flag each tick:

```typescript
if (stage.summonAirdrop && (this.now - this.lastAirdropTime) >= this.mode.summonAirdropsInterval) {
    this.summonAirdrop(...)
    this.lastAirdropTime = this.now
}
```

**Gas current position:** (`this.gas.currentPosition`, `this.gas.newPosition`, `this.gas.newRadius`)

Used to determine spawn location (outside current radius, inside next).

**Map dimensions:** Airdrop position clamped to `[0, mapWidth] × [0, mapHeight]`.

## Known Gotchas

### 1. Landing Outside Map Bounds

**Symptom:** Airdrop crate appears at edge of map or in water.

**Root cause:** Collision resolution nudges airdrop position 8 directions after 500 failed attempts. If all nudges are out-of-bounds, final clamp may place crate in ocean or outside playable area.

**Prevention:** Check stage boundaries; ensure gas zone is sufficiently far from map edges.

**Impact:** Rare; usually airdrops land inside map safely.

### 2. Overlapping Airdrop Spawns

**Symptom:** Two airdrops appear at nearly identical positions within 1–2 seconds.

**Root cause:** Collision resolution checks existing `Game.airdrops[]` hitboxes. If second airdrop tries to spawn while first is still parachuting (not yet in `Game.airdrops[]` officially), collision detection fails.

**Prevention:** Staging system ensures 30 seconds minimum between airdrops (unlikely in practice within single game).

**Impact:** Visual clutter; loot stacking on same crate, potential pickup confusion.

### 3. Loot Overflow (Too Many Items)

**Symptom:** Not possible server-side. Items are spawned as individual `Loot` entities.

**Client-side note:** If all 8 airdrop slots are filled AND player inventory is full, items remain on ground indefinitely until picked up.

**Prevention:** Design loot tables to ensure high-value items (guns, armor). Less critical items (gauze) stack and take one slot each.

### 4. Network Lag During Descent

**Symptom:** Client parachute height doesn't interpolate smoothly; jumps from high to ground.

**Root cause:** Parachute height is sent as partial update (4 bytes). If network lag, height lerp becomes non-linear on client.

**Prevention:** Client-side Parachute class should interpolate missing frames locally based on elapsed time.

**Impact:** Visual jitter; no gameplay impact (position is authoritative).

### 5. Collision Resolution Loops (CPU Overhead)

**Symptom:** Game tick takes > 25 ms when spawning airdrop during congested stage.

**Root cause:** `summonAirdrop()` runs 500 collision checks per spawn, each checking grid intersections, obstacle hitboxes, building scopes.

**Prevention:** Optimize for late-game stages (fewer airdrops spawn after stage 11); consider early-exit heuristics (e.g., stop after finding valid position in first 50 attempts).

**Impact:** Minimal; 500 iterations × sparse grid = typically < 5 ms overhead.

### 6. Gold Airdrop Trigger

**Symptom:** All airdrops are regular (95% weighting); gold airdrops never spawn.

**Root cause:** Airdrops only spawn with `forceGold = false` by default. To override, game mode must pass `forceGold = true` explicitly.

**Prevention:** Check `Game.summonAirdrop()` call site. Ensure `forceGold` logic is implemented if desired (e.g., "gold airdrop every Nth stage").

**Impact:** Game mode-dependent; not a bug unless mode intends random gold airdrops.

### 7. Parachute Height Reaches Negative Values

**Symptom:** Code checks `if (this._height < 0)` to trigger landing. What if height glitches to -1.5?

**Root cause:** Linear lerp can exceed bounds if `endTime - now` becomes very negative.

**Safety:** Clamp height: `this._height = Numeric.clamp(lerped, 0, 1)` (@file server/src/objects/parachute.ts).

**Impact:** None; landing sequence triggers once, then parachute is deleted.

### 8. Loot Tables Not Found

**Symptom:** Airdrop breaks but loot is undefined (null error).

**Root cause:** `LootTables[modeName]["airdrop_crate"]` doesn't exist for a custom game mode.

**Prevention:** Ensure all game modes have fallback loot tables. Check `lootHelpers.ts` for missing mode coverage.

**Impact:** High; crash if loot table is undefined.

### 9. Obstacle Definition `replaceWith.delay` Timing

**Symptom:** Locked crate unlocks immediately (not 800 ms after landing).

**Root cause:** Obstacle replacement happens in `obstacle.update()` on next tick if `replaceWithTimer` expires. If timer not set, replacement is instant.

**Prevention:** Ensure `Obstacle.constructor()` initializes `replaceWithTimer` from definition.

**Impact:** Visual polish only; loot is still available to break after unlock.

## Complex Functions

### `Game.summonAirdrop(position: Vector, forceGold?: boolean)` — @file server/src/game.ts:1182

**Purpose:** Spawn an airdrop at specified position with full collision resolution.

**Implicit behavior:**
- Mutates `position` parameter in-place (nudges hitbox center with `Vec.equals()` and `resolveCollision()`)
- Adds airdrop to `Game.airdrops[]` array
- Schedules parachute spawn after `flyTime` (30s)
- Emits `"airdrop_will_summon"` plugin event (can cancel with return true)
- Emits `"airdrop_landed"` plugin event when parachute lands

**Gotcha:** Position can move up to 8× (crate width × 1.25) × 500 iterations = very far from starting position if heavily collided.

### `Parachute.update()` — @file server/src/objects/parachute.ts:44

**Purpose:** Animate parachute descent and trigger landing when height ≤ 0.

**Implicit behavior:**
- Calls `game.map.generateObstacle()` (creates physical crate obstacle)
- Applies crush damage to all colliding players/obstacles/buildings
- Pushes nearby loot away from crate
- Spawns `airdrop_smoke_particle` particle effect
- Emits `"airdrop_landed"` plugin event
- Removes self from game

**Gotcha:** If `generateObstacle()` fails (returns null), landing succeeds but no crate is created. Plane exists but no loot ever appears.

### `getLootFromTable(modeName: string, tableName: string): ItemData[]` — @file server/src/utils/lootHelpers.ts (inferred)

**Purpose:** Resolve a named loot table to an array of 8 items with counts.

**Implicit behavior:**
- Looks up `LootTables[modeName][tableName]`
- For each slot, performs weighted random selection
- Returns array of `[{ idString, count }, ...]` (8 items)

**Gotcha:** If table has fewer slots, returns fewer items. If table is undefined, throws error.

## Related Documents

### Tier 1
- [Architecture Overview](docs/architecture.md) — System-wide component map
- [Data Model](docs/datamodel.md) — Entity relationships and schemas

### Tier 2
- [Airdrop System](../README.md) — Subsystem overview, lifecycle, loot tables
- [Game Loop](../game-loop/) — Tick-based spawning and update cycle
- [Spatial Grid](../spatial-grid/) — Collision detection used in spawn placement
- [Game Objects (Server)](../game-objects-server/) — `Parachute` and `Obstacle` class hierarchies
- [Server Data](../server-data/) — Game modes, gas stages, loot table definitions
- [Networking](../networking/) — `UpdatePacket` serialization of airdrops/parachutes
- [Plugin System](../plugin-system/) — `airdrop_will_summon` and `airdrop_landed` events

### Tier 3
- (this document) Spawning & Contents — Airdrop spawn logic, loot pools, collision resolution

### Cross-subsystem
- Perks: `PlumpkinBlessing`, `LootBaron` modify airdrop loot (see Inventory subsystem)
- Halloween mode: Pumpkin variant airdrops (see Game Modes subsystem)
- Visual effects: Particle definitions in Particle System subsystem
