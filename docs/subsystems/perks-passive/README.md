# Perks & Passive System

<!-- @tier: 2 -->
<!-- @parent: docs/architecture.md -->
<!-- @modules: docs/subsystems/perks-passive/modules/ -->
<!-- @source: common/src/definitions/items/perks.ts, server/src/objects/player.ts, client/src/scripts/managers/perkManager.ts -->

## Purpose

**Perks** are passive ability bonuses that players collect during gameplay. Each perk grants unique stat modifications (damage, speed, healing, protection, etc.). Players can carry up to 4 perks simultaneously; acquiring a 5th perk automatically drops the oldest one. Perks are category-specific to the game mode: Normal perks in regular matches, Halloween perks during the Halloween event, Infection perks in Infection mode, and Hunted perks in H.U.N.T.E.D. mode.

## Key Files & Entry Points

| File | Purpose | Exports |
|------|---------|---------|
| [`common/src/definitions/items/perks.ts`](../../../common/src/definitions/items/perks.ts) | PerkDefinition enum, all perk data, categories, qualities | `Perks`, `PerkDefinition`, `PerkIds`, `PerkCategories`, `PerkQualities`, `PerkData` |
| [`server/src/objects/player.ts`](../../../server/src/objects/player.ts) | Player perk storage, application logic, stat modifier integration | `Player.perks[]`, `Player.addPerk()`, `Player.mapPerkOrDefault()`, `Player.hasPerk()` |
| [`client/src/scripts/managers/perkManager.ts`](../../../client/src/scripts/managers/perkManager.ts) | Client-side perk tracking, UI display coordination | `PerkManager` singleton |
| [`common/src/packets/updatePacket.ts`](../../../common/src/packets/updatePacket.ts) | Perk serialization in `PlayerData.perks` | Binary perk array encoding |
| [`server/src/inventory/gunItem.ts`](../../../server/src/inventory/gunItem.ts) | Weapon-specific perk effects (damage, reload, spread, etc.) | Perk-driven modifier application |
| [`server/src/data/maps.ts`](../../../server/src/data/maps.ts) | Loot table filtering by perk category | Perk category selection logic |

## Architecture

```
Game Spawn
  │
  ├─ Perk Definitions (Perks registry)
  │
  ├─ Player.perks[] array (max 4)
  │
  ├─ Perk Application:
  │   ├─ Speed: base × perkMod × adrenalineBonus
  │   ├─ Damage: base × damageMod (via GunItem firing)
  │   ├─ Reload: base ÷ reloadMod (divisive; larger mod = faster)
  │   ├─ Size: base × sizeMod (larger players harder to miss)
  │   └─ Health: base + addition (Engorged: +10 health)
  │
  └─ Client Display:
      └─ PerkManager.perks[] → UI HUD icons
```

## Data Flow

**Perk Acquisition:**
```
Server spawns loot crate (item: Perk)
  ↓
Player moves into collision radius
  ↓
Client sends InputPacket with pickup action
  ↓
Server validates: player.perks.length < 4
  ↓
Server adds perk: player.addPerk(perkDef)
  ↓
If length > 4: Drop earliest perk (FIFO)
  ↓
Server sends UpdatePacket with player.dirty.perks = true
  ↓
Client receives perks[], calls PerkManager.add()
  ↓
Client HUD updates with new perk icon
```

**Stat Modifier Application (per tick):**
```
GunItem.fire() called
  ↓
Loop through player.perks:
  ├─ Flechettes: split bullets, reduce damage
  ├─ SabotRounds: increase range & speed, reduce damage
  ├─ CloseQuartersCombat: bonus damage within 50-unit radius
  ├─ Toploaded: damage bonus at magazine thresholds
  ├─ HollowPoints: bonus damage (Hunted mode)
  ├─ LastStand: bonus damage if health < 30
  ├─ Overclocked: increase spread, fire rate reduction
  └─ ... (apply modifiers to damage, speed, spread, etc.)
  ↓
Projectile created with modified stats
```

## Interfaces & Contracts

### PerkDefinition
```typescript
interface PerkDefinition extends ItemDefinition {
    readonly defType: DefinitionType.Perk
    readonly category: PerkCategories        // Normal, Halloween, Infection, Hunted
    readonly quality?: PerkQualities         // Positive, Neutral, Negative
    readonly mechanical?: boolean            // True if triggers special logic
    readonly updateInterval?: number         // ms: for periodic effects (Bloodthirst, etc.)
    readonly noDrop?: boolean                // Cannot be dropped normally
    readonly alwaysAllowSwap?: boolean       // Can swap despite noDrop
    readonly plumpkinGambleIgnore?: boolean  // Excluded from Plumpkin Gamble
    readonly infectedEffectIgnore?: boolean  // Unaffected by Infected perk
    
    // Stat modifiers (perk-specific):
    readonly speedMod?: number                // Multiplicative (1.2 = 20% faster)
    readonly damageMod?: number               // Multiplicative for damage
    readonly reloadMod?: number               // Divisive (1.3 = 30% faster reload)
    readonly rangeMod?: number                // Multiplicative for projectile range
    readonly spreadMod?: number               // Multiplicative for bullet spread
    readonly sizeMod?: number                 // Multiplicative for player hitbox
    readonly explosionMod?: number            // Multiplicative for grenade/explosion knockback
    readonly usageMod?: number                // Divisive for healing item speed
    readonly healthMod?: number               // Multiplicative or additive (depends on value)
    readonly fireRateMod?: number             // Multiplicative for fire rate
    // ... ~30+ perk-specific fields (see perks.ts for full list)
}
```

### Player Perk Interface
```typescript
class Player extends BaseGameObject {
    readonly perks: PerkDefinition[] = [];  // Max 4 perks
    
    addPerk(perk: ReifiableDef<PerkDefinition>): void
        // Adds perk, drops oldest if > 4
    
    hasPerk(perk: ReifiableDef<PerkDefinition>): boolean
        // O(1) lookup
    
    mapPerkOrDefault<U>(
        perk: ReifiableDef<PerkDefinition>,
        mapper: (data: PerkDefinition) => U,
        defaultValue: U
    ): U
        // Apply function if perk present, else return default
}
```

### PerkManager (Client)
```typescript
class PerkManagerClass {
    perks: PerkDefinition[] = [];
    
    add(perk: ReifiableDef<PerkDefinition>): void
        // Add perk to UI display list
    
    has(perk: ReifiableDef<PerkDefinition>): boolean
        // Check if player has perk
    
    remove(perk: ReifiableDef<PerkDefinition>): void
        // Remove from UI (when dropped)
}

export const PerkManager = new PerkManagerClass();
```

## Perk Catalog by Category

### Normal Perks (Regular Game Mode)

| # | Perk | Effect | Key Stats | Quality |
|---|------|--------|-----------|---------|
| 1 | **Second Wind** | Speed boost when health low | speedMod: 1.4, cutoff: 50% health | Positive |
| 2 | **Fléchettes** | Bullets split into 3 projectiles (reduced damage) | split: 3, damageMod: 0.4, deviation: 0.7° | Neutral |
| 3 | **Sabot Rounds** | Higher range/speed, reduced damage & spread | rangeMod: 1.5, speedMod: 1.5, damageMod: 0.9, spreadMod: 0.6 | Neutral |
| 4 | **Extended Mags** | Increased magazine capacity | Per-weapon definition (varies by gun) | Positive |
| 5 | **Demo Expert** | Grenade area boosts, restocks grenades periodically | rangeMod: 2, updateInterval: 10s, restoreAmount: 25% capacity | Positive |
| 6 | **Advanced Athletics** | Move faster in water & smoke | waterSpeedMod: 1.3×, smokeSpeedMod: 1.3× | Positive |
| 7 | **Toploaded** | Damage bonus at magazine thresholds | thresholds: [20% ammo: 1.25×], [49% ammo: 1.1×] | Positive |
| 8 | **Infinite Ammo** | Never run out (reloads from air) | airdropCallerLimit: 3 | Positive |
| 9 | **Field Medic** | Healing items work 50% faster | usageMod: 1.5 (divisive) | Positive |
| 10 | **Berserker** | Permanent speed & damage boost | speedMod: 1.2, damageMod: 1.2 | Positive |
| 11 | **Close Quarters Combat** | Damage & reload bonus within 50 units | cutoff: 50 units, damageMod: 1.2, reloadMod: 1.3 | Positive |
| 12 | **Low Profile** | Smaller hitbox, less knockback from explosions | sizeMod: 0.8, explosionMod: 0.5 | Positive |
| 13 | **Combat Expert** | Faster reload time | reloadMod: 1.25 | Positive |
| 14 | **Precision Recycling** | Get ammo back on accurate shots | hitReq: 2 hits, accThreshold: 50%, refund: 2 bullets | Positive |
| 15 | **Loot Baron** | Find more items | lootBonus: 1 (doubles item spawn) | Positive |
| 16 | **Overclocked** | Increased fire rate (massive spread penalty) | fireRateMod: 0.65, spreadMod: 2.0 | Neutral |

### Halloween Perks (Halloween Event Only)

| # | Perk | Effect | Key Notes | Quality |
|---|------|--------|-----------|---------|
| 1 | **Plumpkin Gamble** | Roll dice for random effects | Mechanical. Cannot stack variants. `noDrop: true` | Neutral |
| 2 | **Plumpkin Shuffle** | Shuffle inventory & perks | Mechanical. Affects only Halloween perks. `noDrop: true` | Neutral |
| 3 | **Lycanthropy** | Werewolf form (30% speed, 50% health, 2× damage) | speedMod: 1.3, healthMod: 1.5, damageMod: 2, regenRate: 1 | Positive |
| 4 | **Bloodthirst** | Kill spree: speed bursts, life drain | Takes 1 health/tick, restores 25 on heal/adrenaline | Neutral |
| 5 | **Plumpkin Bomb** | Grenades deal 20% more damage | damageMod: 1.2 (grenade-only) | Positive |
| 6 | **Shrouded** | Ambient invisibility mechanic | updateInterval: 100ms | Positive |
| 7 | **Eternal Magnetism** | Loot magnetically pulls toward player | radius: 20 units, depletion: 0.05, lootPush: 0.0005 | Positive |
| 8 | **Last Stand** | Damage boost when low health | healthReq: 30, damageMod: 1.267, damageReceivedMod: 0.8 | Positive |
| 9 | **Plumpkin's Blessing** | Better loot drops via quality threshold | qualityValue: 0.21 (filters loot table) | Positive |
| 10 | **Experimental Treatment** | No adrenaline decay (balanced by reduced health) | adrenDecay: 0, healthMod: 0.8 | Neutral |
| 11 | **Engorged** | Large health bonus but bigger hitbox | healthMod: 10 (additive), sizeMod: 1.05, killsLimit: 10 | Neutral |
| 12 | **Baby Plumpkin Pie** | Periodic spooky effects | updateInterval: 10s | Neutral |
| 13 | **Costumed** | Disguise as a random obstacle | Choices weighted (oak_tree: 1.0, airdrop: 0.1, plumpkin: 0.01). `alwaysAllowSwap: true` | Neutral |
| 14 | **Torn Pockets** | Continuously drop items | updateInterval: 2s, dropCount: 2 | Negative |
| 15 | **Claustrophobic** | Movement penalty in buildings | speedMod: 0.75 (when indoors) | Negative |
| 16 | **Laced Stimulants** | Healing damages you | healDmgRate: 0.5, lowerHpLimit: 5 (min health when healed) | Negative |
| 17 | **Rotten Plumpkin** | Periodic vomiting (health & adrenaline loss) | updateInterval: 10s, adrenLoss: 5%, healthLoss: 5 (absolute) | Negative |
| 18 | **Priority Target** | Glowing indicator above your head | Broadcasts your location to all players | Negative |
| 19 | **Butterfingers** | Slower reload (75% reload time) | reloadMod: 0.75 | Negative |
| 20 | **Overweight** | Larger hitbox (30% bigger) | sizeMod: 1.3 | Negative |
| 21 | **Aching Knees** | Random slowdown effects | updateInterval: 10s | Negative |

### Infection Mode Perks

| # | Perk | Effect | Key Stats |
|---|------|--------|-----------|
| 1 | **Infected** | Spreads infection to nearby players (anti-perk) | updateInterval: 30s, `noDrop: true` |
| 2 | **Necrosis** | Constant damage + spreads infection | dps: 0.78, infectionRadius: 20, infectionUnits: 5, `hideInHUD: true` |
| 3 | **Immunity** | Temporary infection protection | duration: 15s, `noDrop: true` |

### H.U.N.T.E.D. Mode Perks

| # | Perk | Effect | Key Notes |
|---|------|--------|-----------|
| 1 | **Hollow Points** | Rifle rounds highlight targets when shot | damageMod: 1.1, soundMod: 75, highlightDuration: 5s |
| 2 | **Experimental Forcefield** | Shield barrier that regenerates | shieldRegenRate: 1, shieldRespawnTime: 20s, particle effects |
| 3 | **Thermal Goggles** | Detect other players within radius | detectionRadius: 100 units |
| 4 | **Overdrive** | Activated ability: 25% speed & heal bonus (0-kill cooldown) | speedMod: 1.25, speedBoostDuration: 10s, cooldown: 12s |

## Stat Modifier Application

Perks apply stat modifiers through **multiplicative** and **divisive** operations:

### Speed Modifiers
```typescript
finalSpeed = baseSpeed
    × (floor.speedMultiplier)        // Water slows to 60%
    × perkMod.speedMod               // E.g., Berserker: 1.2
    × perkMod.waterSpeedMod          // E.g., Advanced Athletics: 1.3 in water
    × (isIndoors ? 0.75 : 1)         // Claustrophobic in buildings
    × adrenBonus                      // Adrenaline: logarithmic curve, max ~1.15
    × activeItemMod                   // Weapon+item modifiers
```

### Damage Modifiers
**GunItem.ts applies per-perk:**
- **Flechettes** (split mode): `damage × 0.4`
- **Sabot Rounds**: `damage × 0.9`
- **Berserker**: `damage × 1.2`
- **Close Quarters Combat** (range < 50): `damage × 1.2`
- **LastStand** (if health < 30): `damage × 1.267`
- **HollowPoints** (Hunted): `damage × 1.1`
- **Plumpkin Bomb** (grenades): `damage × 1.2`

### Reload Modifiers
**Divisive** (larger = faster reload):
- **Combat Expert**: `base ÷ 1.25`
- **Close Quarters Combat**: `base ÷ 1.3`
- **Butterfingers**: `base ÷ 0.75` (slower, multiplicative inverse)

### Magazine Capacity
- **Extended Mags**: Per-weapon definition (e.g., M16: capacity 30 → 45)
- **Toploaded**: Damage bonus at magazine thresholds based on ammo ÷ capacity percentage

## Perk Collection Mechanics

**Spawn & Discovery:**
1. Perks spawn as ground loot (same as weapons/items)
2. Category filter applied: Normal perks in regular, Halloween in Halloween event, etc.
3. Player walks into collision radius (~collision.toCircle())
4. Client sends InputPacket.itemPickupAction

**Server Processing:**
```typescript
if (player.perks.length < 4) {
    player.addPerk(perkDef)
} else {
    // Drop oldest perk (FIFO)
    removeFrom(player.perks, player.perks[0])
    player.addPerk(perkDef)
}
player.dirty.perks = true
```

**Client Synchronization:**
```typescript
// UpdatePacket deserialization
if (playerData.perks) {
    PerkManager.perks = playerData.perks  // Replace array
    UIManager.updatePerkHUD()             // Refresh display
}
```

## Dependencies

### Depends On
- [Object Definitions](../object-definitions/) — Perks registry (O(1) ID lookup via `Perks.reify()`)
- [Game Objects (Server)](../game-objects-server/) — Player class, perk storage array
- [Inventory](../inventory/) — GunItem perk effect application
- [Game Loop](../game-loop/) — Per-tick stat calculation

### Depended On By
- [Client Managers](../client-managers/) — PerkManager singleton (UI display)
- [Networking](../networking/) — UpdatePacket serialization (PlayerData.perks)
- [UI Management](../ui-management/) — Perk HUD (icons, tooltips, descriptions)
- [Game Loop](../game-loop/) — Stat recalculation when perks added/removed

## Known Issues & Gotchas

### 1. **Perk Effect Order Undefined**
- When multiple perks modify the same stat (e.g., two `speedMod` perks), application order is not documented.
- **Impact:** Two different perk orderings might yield different final stats.
- **File:** [`server/src/inventory/gunItem.ts`](../../../server/src/inventory/gunItem.ts#L213-L290) — Loop iterates `player.perks` in FIFO order.
- **Workaround:** Rely on FIFO order (oldest perk applies first), but this is implementation-dependent.

### 2. **Duplicate Perks Allowed**
- Players can collect the same perk twice, stacking effects.
- **Impact:** Two Berserkers = 1.44× damage (1.2 × 1.2), vs. intended single boost.
- **File:** [`client/src/scripts/managers/perkManager.ts`](../../../client/src/scripts/managers/perkManager.ts#L9-L12) — `add()` method checks `if (this.perks.includes(perkDef)) return` (prevents UI duplication but doesn't prevent server-side stacking).
- **Real issue:** Server-side [`server/src/objects/player.ts:addPerk()`](../../../server/src/objects/player.ts#L2099) also allows duplicates.

### 3. **Holiday Perk Mechanics Fragile**
- Halloween perks hardcoded with `noDrop: true` and special filters.
- **Impact:** Cannot drop or reroll Halloween perks manually; Plumpkin Gamble is the only dynamic reroll.
- **File:** [`server/src/data/maps.ts:1333`](../../../server/src/data/maps.ts#L1333) — Filters Infection perks when `game.modeName !== "infection"` and vice versa.

### 4. **Perks Affecting All Weapons**
- Stat modifiers apply globally (e.g., Overclocked speeds up ALL guns, not selective).
- **Impact:** No per-weapon perk selectivity; "shotgun-only" or "smg-only" perks cannot be implemented.
- **File:** [`server/src/inventory/gunItem.ts:213-290`](../../../server/src/inventory/gunItem.ts#L213-L290) — Loop applies modifier to `modifiers` object directly.

### 5. **HUD Never Shows Actual Stat Values**
- Perk descriptions are static (e.g., "Speed +20%") and don't reflect current player stats.
- **Impact:** Player sees "+20%" but doesn't know final speed after adrenaline/items/perks stack.
- **File:** [`client/src/scripts/managers/perkManager.ts`](../../../client/src/scripts/managers/perkManager.ts) — Only stores `perks[]` array; no stat calculation mirroring.

### 6. **Rarity Distribution Hardcoded in Definitions**
- Perks don't declare rarity explicitly; must infer from code comments or special flags (`plumpkinGambleIgnore`, `noDrop`).
- **Impact:** No dynamic weighting or game-state-driven rarity adjustments.
- **File:** [`common/src/definitions/items/perks.ts`](../../../common/src/definitions/items/perks.ts) — All perks equally likely in loot tables unless explicitly filtered.

### 7. **Perk "Quality" Not Used in Gameplay**
- `PerkQualities` enum (Positive, Neutral, Negative) is defined but not enforced.
- **Impact:** Halloween event can roll negative perks (Butterfingers, Overweight) as frequently as positive ones.
- **File:** [`common/src/definitions/items/perks.ts:35-40`](../../../common/src/definitions/items/perks.ts#L35-L40) — Defined but never consulted in `LootTables` or perk selection logic.

### 8. **Thermal Goggles & Hollow Points Interaction Unclear**
- Both highlight players but mechanics differ (radius detection vs. shot-triggered).
- **Impact:** Overlapping visual indicators; tooltip may confuse players.
- **File:** [`server/src/objects/player.ts:1620-1660`](../../../server/src/objects/player.ts#L1620-L1660) — Separate "alreadyHighlighted" tracking for each perk.

### 9. **Infection Perk Spreading Not Balanced**
- `Necrosis` continuously spreads infection, but `Immunity` duration is fixed (15s).
- **Impact:** Infection spreads faster than protection decays; end-game becomes infection-dominant.
- **File:** [`common/src/definitions/items/perks.ts:586-605`](../../../common/src/definitions/items/perks.ts#L586-L605) — `infectionUnits: 5` per tick vs. `Immunity duration: 15s` (15 ticks at 40 TPS).

### 10. **Plumpkin Gamble Escapes Documentation**
- Special comment: "krr krr krr *buzzer* aw dang it!" with no actual spec.
- **Impact:** Behavior is pure code mystery; reroll logic undocumented.
- **File:** [`common/src/definitions/items/perks.ts:230-261`](../../../common/src/definitions/items/perks.ts#L230-L261) — Defined as `mechanical: true` but implementation is elsewhere.

## Related Documents

### Tier 1
- [Data Model](../../datamodel.md#perkdefinition) — PerkDefinition schema, PerkIds enum, PerkCategories enum
- [Architecture Overview](../../architecture.md) — System-wide stat modifier flow

### Tier 2
- [Object Definitions](../object-definitions/) — Perks registry via `ObjectDefinitions<PerkDefinition>`
- [Game Objects (Server)](../game-objects-server/) — Player class, perk storage & application
- [Inventory](../inventory/) — GunItem perk effect switch statement
- [Game Loop](../game-loop/) — Per-tick stat recalculation
- [UI Management](../ui-management/) — Perk HUD icons & tooltips
- [Networking](../networking/) — UpdatePacket perk serialization

### Tier 3
- [Inventory — GunItem Module](../inventory/modules/gun-item.md) — Weapon-specific perk logic
- [Game Loop — Tick Module](../game-loop/modules/tick.md) — Per-tick player stat recalculation

### Patterns (TBD)
- Perk effect composition patterns (chaining modifiers)
- Holiday event perk filtering (mode-based selection)
