# Airdrop System

<!-- @tier: 2 -->
<!-- @parent: docs/architecture.md -->
<!-- @modules: docs/subsystems/airdrop-system/modules/ -->
<!-- @source: server/src/objects/ -->

## Purpose

The airdrop system delivers high-tier loot drops to players at specific game stages. Airdrops spawn via plane animation, descend with parachute visual, and explode into an obstacle crate (locked state) containing curated loot tables. The system includes collision avoidance, crush damage on landing, smoke particle effects, and map pings to alert players.

## Key Files & Entry Points

| File | Purpose | Complexity |
|------|---------|-----------|
| `server/src/game.ts` — `summonAirdrop()` / `lastAirdropTime` | Main airdrop spawn entry point; tracks spawn timing | High |
| `server/src/objects/parachute.ts` | Parachute entity; falls from plane for ~8 seconds; damages on impact | High |
| `server/src/objects/loot.ts` | Loot entity; spawned by broken airdrop crate obstacle | Medium |
| `server/src/objects/obstacle.ts` | Obstacle lifecycle; airdrop_crate_locked spawns locked loot | Medium |
| `server/src/data/gasStages.ts` | Gas stage definitions with `summonAirdrop` flag (stages 3, 7, 11) | Low |
| `server/src/data/lootTables.ts` — `airdrop_crate` / `gold_airdrop_crate` | Loot pool for airdrop crates (8 items each) | Medium |
| `common/src/constants.ts` — `GameConstants.airdrop` | Airdrop timing: flyTime=30s, fallTime=8s, damage=300 | Low |

## Architecture

### Airdrop Lifecycle

```
Game.summonAirdrop(position, forceGold)
  ├─ Resolve collision with existing airdrops + obstacles (up to 500 attempts)
  ├─ Create plane entity at far distance (GameConstants.maxPosition away)
  └─ Schedule parachute spawn after flyTime (30 seconds):
       └─ Plane arrives, spawn Parachute object
       └─ Emit map ping ("airdrop_ping")
       └─ Parachute.update() each frame:
            ├─ Lerp height from 1 → 0 over fallTime (8s)
            ├─ When height < 0:
            │  ├─ Damage players/obstacles in crate hitbox area
            │  ├─ Push nearby loot away from crate
            │  ├─ Spawn "airdrop_smoke_particle" at crate position
            │  ├─ Emit "airdrop_landed" event
            │  └─ Remove airdrop from game.airdrops[]
            └─ Network: send Parachute height for client-side animation
```

### Spawn Triggering

Airdrops are summoned by the gas system during specific stages:

| Stage | Index | State | Duration | summonAirdrop? | Notes |
|-------|-------|-------|----------|---|---------|
| Zone 1 Waiting | 3 | Waiting | 45s | **Yes** | ~1 min 5 sec after game start |
| Zone 3 Waiting | 7 | Waiting | 35s | **Yes** | ~3 min 10 sec after game start |
| Final Zone Waiting | 11 | Waiting | 20s | **Yes** | ~5 min 10 sec after game start |

Each stage triggers exactly one airdrop spawn (no periodic spawning within a stage).

### Position Calculation & Collision Avoidance

1. **Starting position:** Center of current gas safe zone (from `this.gas.position`)
2. **Collision resolution loop:**
   - Attempt to place crate at the starting position
   - If collision detected with:
     - Existing airdrop crates (with 1.25x padding)
     - Indestructible obstacles
     - Building scope hitboxes
   - Then nudge position in one of 8 compass directions (after 500 failed attempts)
3. **Bounds clamping:** Ensure crate stays within map bounds
4. **Plane spawn location:** Vector from final position in random direction (0–2π), distance = `GameConstants.maxPosition` away

**Code reference:** `server/src/game.ts:summonAirdrop()` (lines 1182–1318)

### Loot Tables

**Regular Airdrop (`airdrop_crate`)** — 8 weighted loot slots:

```
[1] airdrop_equipment    (tactical helmet/vest/pack)
[2] airdrop_scopes       (8x scope, rare 16x)
[3] airdrop_healing_items (gauze, medikit, cola, tablets)
[4] airdrop_skins        (cosmetic skins, rare ghillie)
[5] airdrop_melee        (crowbar, hatchet, sickle, rare pan)
[6] ammo                 (standard ammo pool)
[7] airdrop_guns         (mg36, sr25, vss, vector, vepr12, deagle, etc.)
[8] frag_grenade × 3     (or null item, 2:1 weight ratio)
```

**Gold Airdrop (`gold_airdrop_crate`)** — Same structure, but slot [7] replaced with:

```
[7] gold_airdrop_guns    (m1_garand, dp12, acr, pp19, negev, mg5, mk18, l115a1, rare g19)
```

**Airdrop Loot Details:**

- `airdrop_guns`: 14 weapons including assault rifles (mg36, sr25, vss, vector, mosin_nagant, tango_51, stoner_63)
- `gold_airdrop_guns`: 10 weapons including marksman rifles (acr, l115a1) and legendary sidearm (g19, weight 0.0005)
- `airdrop_scopes`: High-tier optics only (8x, 16x)
- `airdrop_healing_items`: Gauze (5-stack), medikit, cola, tablets
- `airdrop_equipment`: Tactical armor only (helmet, vest, pack)

Each slot in the crate spawns exactly one item stack when the crate is destroyed.

## Parachute Entity

### Properties

| Property | Type | Purpose |
|----------|------|---------|
| `height` | number (0–1) | Falling animation; 1 = fully deployed, 0 = landed |
| `endTime` | number | Timestamp when parachute should reach ground |
| `hitbox` | CircleHitbox(r=10) | Collision area for damage calculations |
| `_airdrop` | Airdrop reference | Links to crate definition and position |

### Serialization

Sent to clients via `UpdatePacket`:
- **Full data:** position (X, Y)
- **Partial data:** height (lerped 0–1 over 8 seconds)

Client-side rendering animates parachute descent based on height value.

### Network Budget

- **fullAllocBytes:** 8 bytes (position)
- **partialAllocBytes:** 4 bytes (height)

## Crate Obstacle Definition

When `Parachute.height < 0`, game spawns locked obstacle via `game.map.generateObstacle()`:

```
Obstacle Definition:
├─ idString: "airdrop_crate_locked" or "airdrop_crate_locked_force" (gold)
├─ hasLoot: true
├─ lootTable: "airdrop_crate" or "gold_airdrop_crate"
├─ spawnWithLoot: false (loot only drops after player breaks crate)
├─ indestructible: false (players can destroy to access loot)
└─ locked: true (initially locked, requires key/interaction)
```

When loot is demanded (crate broken), `Obstacle.damage()` calls:
```typescript
this.loot = getLootFromTable(this.game.modeName, definition.lootTable);
for (const item of this.loot) {
    this.game.addLoot(item.idString, crate.position, layer, { count: item.count });
}
```

## Crush Damage & Effects

When parachute lands, checks all objects within crate hitbox:

| Target | Damage Type | Amount |
|--------|-----------|--------|
| Player | `piercingDamage()` (bypasses armor) | 300 HP |
| Obstacle | Instant destruction | Infinity |
| Building ceiling | `damageCeiling()` (structure damage) | Infinity |

After damage: nearby loot is pushed away from crate using `loot.push(angle, velocity)`.

Particle effect `"airdrop_smoke_particle"` spawned at crate position.

## Pickup Mechanics

Airdrop loot (once broken from crate) follows standard pickup rules:

**Interact range:** CircleHitbox radius = 3 × player.sizeMod (typically 3 units)

**Interaction logic:**
```
if distance(player.position, loot.position) ≤ 3
  AND loot.canInteract(player) returns true
    → player.inventory.interact(loot)
    → if inventory full: loot remains, highlights on ground
    → else: loot.count decreases or removed if fully picked up
```

## Dependencies

### Depends on:
- [Server Data Systems](../server-data/) — Gas stages (`gasStages.ts`) with `summonAirdrop` flag
- [Game Objects (Server)](../game-objects-server/) — `Parachute` entity, `Obstacle.damage()` loot spawning
- [Game Loop](../game-loop/) — `game.summonAirdrop()` invoked during stage transitions
- [Networking](../networking/) — `UpdatePacket`, parachute height serialization
- [Spatial Grid](../spatial-grid/) — Collision detection for spawn position, crush damage queries
- [Loot Tables](../server-data/) — `airdrop_crate` and `gold_airdrop_crate` loot definitions
- [Inventory](../inventory/) — Loot item pickup and inventory space checks

### Depended on by:
- [Game Loop](../game-loop/) — Calls `summonAirdrop()` at stage 3, 7, 11
- [Networking](../networking/) — Sends parachute updates to clients
- [Plugin System](../plugins/) — Emits `airdrop_will_summon`, `airdrop_landed` events

## Known Issues & Gotchas

1. **Deterministic spawn timing locked to gas stages** — Airdrops always spawn at the same game time (stage 3, 7, 11); cannot be randomized or scheduled dynamically without mode override.

2. **Collision avoidance exhaustive after 500 attempts** — If map has many airdrops already or tight obstacles, airdrop may spawn inside building/obstacle after 500 failed nudges.

3. **Parachute height sent every frame** — Even small lerp changes cause `setPartialDirty()`, increasing network traffic (4 bytes per update).

4. **Crush damage applies before loot spawn** — If player stands in crate landing spot, they take full 300 damage before looting; no invulnerability window.

5. **Gold airdrop mode-locked to game mode** — `forceGold` parameter only checks `this.modeName === "halloween"` for pumpkin override; hard-coded mode dependency.

6. **No audio cue for airdrop spawn** — Only visual effects (smoke) and map ping; players must see ping or hear plane engine (not implemented).

7. **Loot table queries non-deterministic** — `getLootFromTable()` uses weighted random selection; same crate may drop different items on replay (unless seeded).

8. **Parachute fallTime hardcoded** — 8 seconds is not configurable; cannot adjust descent speed per game mode.

## Team System Integration

Airdrops are **not team-aware**:
- No team allocation or fair distribution
- Any player can access the same crate loot
- No "blue loot" vs "red loot" mode variant

## Related Documents

### Tier 1
- [docs/architecture.md](../../architecture.md) — System overview and tech stack
- [docs/datamodel.md](../../datamodel.md) — Loot entity and obstacle definitions

### Tier 2
- [Server Data Systems](../server-data/) — Gas stages, loot tables
- [Game Loop](../game-loop/) — Stage progression and airdrop spawn scheduling
- [Game Objects (Server)](../game-objects-server/) — Obstacle and parachute entity lifecycle
- [Spatial Grid](../spatial-grid/) — Collision detection and object queries
- [Networking](../networking/) — Packet serialization for parachute animation
- [Plugin System](../plugins/) — Event system for airdrop hooks
- [Inventory](../inventory/) — Loot pickup mechanics

### Related Files
- `server/src/objects/parachute.ts:10` — Parachute class definition
- `server/src/game.ts:1182` — `summonAirdrop()` method
- `server/src/data/gasStages.ts:53–122` — Gas stage definitions with airdrop flags
- `server/src/data/lootTables.ts:453–489` — `airdrop_crate` and `gold_airdrop_crate` definitions
- `common/src/constants.ts:87–89` — Airdrop timing constants
