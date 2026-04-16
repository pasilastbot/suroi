# Perk Effects & Application

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/perks-passive/README.md -->
<!-- @source: common/src/definitions/items/perks.ts, server/src/objects/player.ts, server/src/inventory/gunItem.ts, client/src/scripts/managers/perkManager.ts -->

## Purpose

This module documents how **perk stat modifiers** are defined, stored, applied, and synchronized across server and client. Perks grant multiplicative and additive bonuses to player stats (damage, speed, reload, health, etc.). Understanding perk effects is critical for gameplay balance, stat stacking precedence, and troubleshooting modifier bugs.

## Key Files

| File | Purpose | Key Exports | Complexity |
|------|---------|------------|-----------|
| [`common/src/definitions/items/perks.ts`](../../../common/src/definitions/items/perks.ts) | Central perk registry, definition structure, stat modifiers | `Perks`, `PerkIds`, `PerkDefinition`, `BasePerkDefinition`, `PerkData`, `PerkCategories`, `PerkQualities` | High |
| [`server/src/objects/player.ts`](../../../server/src/objects/player.ts) | Server-side perk application, onEquip/onRemove hooks, modifier integration | `Player.perks[]`, `Player.addPerk()`, `Player.removePerk()`, `Player.hasPerk()`, `Player.mapPerk()`, `Player.mapPerkOrDefault()`, `Player.perkUpdateMap` | High |
| [`server/src/inventory/gunItem.ts`](../../../server/src/inventory/gunItem.ts) | Per-tick perk effect application during weapon firing | Perk case handlers in `GunItem.fire()` | Very High |
| [`client/src/scripts/managers/perkManager.ts`](../../../client/src/scripts/managers/perkManager.ts) | Client-side perk tracking, singleton UI manager | `PerkManager.perks[]`, `PerkManager.add()`, `PerkManager.has()`, `PerkManager.remove()`, `PerkManager.map()`, `PerkManager.mapOrDefault()` | Low |
| [`common/src/packets/updatePacket.ts`](../../../common/src/packets/updatePacket.ts) | Perk serialization in network packets | `PlayerData.perks` array encoding | Medium |

## Business Rules

1. **Max perks per player:** 4 simultaneous (see `GameConstants.player.maxPerks`)
2. **FIFO overflow:** Acquiring a 5th perk automatically **drops the oldest one** (first-in-first-out)
3. **Category filtering:** Perks are constrained by game mode:
   - **Normal mode:** `PerkCategories.Normal` perks only
   - **Halloween mode:** `PerkCategories.Halloween` and `PerkCategories.Normal` perks
   - **Infection mode:** `PerkCategories.Infection` perks override normal perks with `infectedEffectIgnore` flag
   - **H.U.N.T.E.D. mode:** `PerkCategories.Hunted` perks
4. **Stat modifier precedence:** 
   - **Multiplicative mods first** (damage, speed, spread, reload, fire rate)
   - **Additive mods after** (health bonus from Engorged +10 HP)
   - **Conditional mods last** (Last Stand damage bonus if health < 30; Second Wind speed if health < 50%)
5. **Perk update intervals:** Some perks update periodically (every 100ms–10s) via `perkUpdateMap`, not per-tick
6. **Perk synchronization:** Only `Player.dirty.perks = true` triggers full perk array reserialize in `UpdatePacket`

## Perk Definition Structure

### Base Interface

```typescript
interface BasePerkDefinition extends ItemDefinition {
    readonly idString: PerkIds           // Unique identifier (e.g., "sabot_rounds")
    readonly category: PerkCategories    // Normal | Halloween | Infection | Hunted
    readonly mechanical?: boolean         // True if self-explanatory, doesn't appear in UI normally
    readonly updateInterval?: number      // Used by perkUpdateMap; frequency in ms
    readonly quality?: PerkQualities      // "positive" | "neutral" | "negative"
    readonly noSwap?: boolean            // Cannot be swapped manually (usually mechanical)
    readonly alwaysAllowSwap?: boolean   // Force swap via PlumpkinShuffle
    readonly plumpkinGambleIgnore?: boolean  // Excluded from PlumpkinGamble rng
    readonly infectedEffectIgnore?: boolean  // Ignores infection mode effects (Hunted perks)
    readonly noDrop?: boolean            // Only appears in special maps/modes
}
```

### Example Perk Definition (Sabot Rounds)

```typescript
{
    idString: PerkIds.SabotRounds,
    name: "Sabot Rounds",
    defType: DefinitionType.Perk,
    category: PerkCategories.Normal,
    
    rangeMod: 1.5,           // Projectile range × 1.5
    speedMod: 1.5,           // Projectile speed × 1.5
    spreadMod: 0.6,          // Spread × 0.6 (tighter grouping)
    damageMod: 0.9,          // Damage × 0.9 (9% reduction)
    tracerLengthMod: 1.2     // Tracer line length × 1.2
}
```

### Stat Modifiers Reference

| Modifier Property | Type | Applied To | Semantics | Examples |
|------------------|------|-----------|-----------|----------|
| **damageMod** | number (multiplicative) | Bullet/explosive damage output | `damage *= mod` | Flechettes: 0.4, Berserker: 1.2, Lycanthropy: 2 |
| **speedMod** | number (multiplicative for movement; projectiles) | Player speed OR projectile speed | `speed *= mod` | SecondWind: 1.4, SabotRounds: 1.5 |
| **spreadMod** | number (multiplicative) | Bullet spread deviation | `spread *= mod` (lower = tighter) | SabotRounds: 0.6, Overclocked: 2 |
| **reloadMod** | number (divisive!) | Gun reload time | `reloadTime /= mod` (larger = faster!) | CombatExpert: 1.25, Butterfingers: 0.75 |
| **fireRateMod** | number (multiply fire delay) | Fire rate / cycle time | `fireDelay *= mod` (larger = slower) | Overclocked: 0.65 |
| **sizeMod** | number (multiplicative) | Player collision radius | `radius *= mod` (affects hit detection) | LowProfile: 0.8, Engorged: 1.05 |
| **rangeMod** | number (multiplicative) | Projectile max distance | `range *= mod` | SabotRounds: 1.5, DemoExpert: 2 |
| **explosionMod** | number (multiplicative) | Explosion damage falloff | `damage *= mod` | LowProfile: 0.5 |
| **healthMod** | number (additive OR multiplicative) | Max health bonus | `maxHealth += mod` (if < 10) or `*= mod` | Engorged: 10 (additive), Lycanthropy: 1.5 (multiplicative) |
| **tracerLengthMod** | number (multiplicative) | Bullet tracer line length | Visual only (rendering) | SabotRounds: 1.2 |

**Key:** Multiplicative mods compound; divisive mods (reload) are inverted in semantics.

## Perk Equip Event (onEquip Logic)

### addPerk() Method — @file server/src/objects/player.ts:2098

**Purpose:** Adds a perk to `player.perks[]`, triggers setup hooks, auto-drops oldest perk if count > 4.

**Call sites:**
- Loot pickup (player collides with `Loot` object containing perk)
- Special game mode logic (Infection mode forces perk addition)
- Easter eggs / perk combinations (Hollow Points + Experimental Forcefield + Thermal Goggles → Overdrive)

**Implementation outline:**

```typescript
addPerk(perk: ReifiableDef<PerkDefinition>): void {
    const perkDef = Perks.reify(perk);
    if (this.perks.includes(perkDef)) return;  // No duplicates
    
    this.perks.push(perkDef);                   // Add to array (FIFO order)
    
    // Auto-drop oldest perk if count exceeds max (4)
    if (this.perks.length > 4) {
        this.perks.shift();                     // Remove first (oldest)
    }
    
    // Set up interval-based updates for perks with updateInterval
    if ("updateInterval" in perkDef) {
        (this.perkUpdateMap ??= new Map<PerkDefinition, number>())
            .set(perkDef, this.game.now);
    }
    
    // ========== PERK-SPECIFIC SETUP (onEquip) ==========
    switch (perkDef.idString) {
        case PerkIds.HollowPoints & ExperimentalForcefield & ThermalGoggles:
            this.addPerk(PerkIds.Overdrive);  // Easter egg trigger
            break;
        case PerkIds.PlumpkinGamble:
            this.removePerk(perk);            // Immediately remove & replace with random Halloween perk
            this.addPerk(pickRandomInArray(allowedHalloweenPerks));
            break;
        case PerkIds.Lycanthropy:
            // Transform player: change skin, drop weapons, equip werewolf fur vest
            [this._perkData["Lycanthropy::old_skin"], this.loadout.skin] = 
                [this.loadout.skin, Skins.fromString("werewolf")];
            this.inventory.dropWeapon(0, true)?.destroy();
            // ... (drop all weapons & armor)
            this.inventory.vest = Armors.fromString("werewolf_fur");
            break;
        case PerkIds.Costumed:
            const { choices } = PerkData[PerkIds.Costumed];
            this.activeDisguise = Obstacles.fromString(
                weightedRandom(Object.keys(choices), Object.values(choices))
            );
            break;
        case PerkIds.PlumpkinBomb:
            this.halloweenThrowableSkin = true;  // Visual effect
            break;
        // ... (more special cases)
    }
    
    this.setDirty();
    this.dirty.perks = true;  // Signal network sync
}
```

**Stat modifiers applied by addPerk:**
- **None directly** — mod values are **read from `perkDef` properties only during usage** (firing, movement, etc.)
- Some perks store **state** (e.g., `this.activeDisguise`, `this.perkUpdateMap`)

## Perk Remove Event (onRemove Logic)

### removePerk() Method — @file server/src/objects/player.ts:2275

**Purpose:** Removes a perk, triggers cleanup hooks, restores prior state.

**Call sites:**
- Player death (may drop perks or clear them)
- Infection mode removal (non-Infection perks removed when infected)
- Perk-specific expiration (Immunity timeout after 15s)
- Weapon swap (ExtendedMags adjusted)

**Implementation outline:**

```typescript
removePerk(perk: ReifiableDef<PerkDefinition>): void {
    const perkDef = Perks.reify(perk);
    if (!this.perks.includes(perkDef)) return;  // Safety check
    
    removeFrom(this.perks, perkDef);            // Remove from array
    
    // Clean up interval-based update tracking
    if ("updateInterval" in perkDef && this.perkUpdateMap !== undefined) {
        this.perkUpdateMap.delete(perkDef);
        if (this.perkUpdateMap.size === 0) {
            this.perkUpdateMap = undefined;
        }
    }
    
    // ========== PERK-SPECIFIC CLEANUP (onRemove) ==========
    switch (perkDef.idString) {
        case PerkIds.Lycanthropy:
            // Restore original skin, unlock inventory slots
            this.inventory.vest = undefined;
            this.loadout.skin = Skins.fromStringSafe(...) ?? Skins.fromString("hazel_jumpsuit");
            this.inventory.unlockAllSlots();
            break;
        case PerkIds.ExtendedMags:
            // Trim excess ammo (ammo > base capacity)
            for (const weapon of this.inventory.weapons) {
                const extra = weapon.ammo - weapon.definition.capacity;
                if (extra > 0) {
                    weapon.ammo = weapon.definition.capacity;
                    this.inventory.giveItem(weapon.definition.ammoType, extra);
                }
            }
            break;
        case PerkIds.PlumpkinBomb:
            this.halloweenThrowableSkin = false;
            break;
        case PerkIds.Costumed:
            this.activeDisguise = undefined;
            this.emitLowHealthParticles = false;
            break;
        case PerkIds.Infected:
            // Remove Necrosis, grant Immunity (15s duration)
            this.removePerk(PerkIds.Necrosis);
            const perkToRemove = this.perks.find(/* remove non-Infection perks */);
            if (perkToRemove) this.removePerk(perkToRemove);
            this.infection = 0;
            this.addPerk(PerkIds.Immunity);
            this.immunityTimeout = this.game.addTimeout(
                () => this.removePerk(PerkIds.Immunity), 
                15000
            );
            break;
        case PerkIds.PrecisionRecycling:
            this.bulletTargetHitCount = 0;  // Reset accuracy tracker
            this.targetHitCountExpiration?.kill();
            break;
        case PerkIds.EternalMagnetism:
            this.hasMagneticField = false;
            break;
        case PerkIds.Overdrive:
            this.overdriveTimeout?.kill();
            this.overdriveKills = 0;
            break;
        // ... (more special cases)
    }
    
    this.updateAndApplyModifiers();  // Recalculate effective stats
    this.dirty.perks = true;         // Signal network sync
}
```

## Stat Modifiers — Actual Multipliers & Formulas

### Movement Speed

**Formula (per tick, 40 TPS):**
```
finalSpeed = baseSpeed 
    × floorType.speedMultiplier
    × recoilMult
    × mapPerkOrDefault(AdvancedAthletics, waterMod × smokeMod, 1)
    × mapPerkOrDefault(Claustrophobic, isInsideBldg ? speedMod : 1, 1)
    × adrenSpeedMod
    × (action?.speedMultiplier ?? 1)
    × (downed ? 0.5 : activeItem.speedMult ?? 1)
    × (beingRevived ? 0.5 : 1)
    × effectSpeedMultiplier
    × modifiers.baseSpeed
```

**Perk modifiers:**
- **Second Wind** (Normal): `speedMod: 1.4` (40% faster) — **only if health < 50%**
- **AdvancedAthletics** (Normal): `waterSpeedMod: 1.3` (negate water slow), `smokeSpeedMod: 1.3`
- **Claustrophobic** (Halloween): `speedMod: 0.75` (25% slower) — **only inside buildings**
- **Lycanthropy** (Halloween): `speedMod: 1.3` (30% faster)
- **Bloodthirst** (Halloween): `speedMod: 1.5` (50% faster) + temporary boost for 2s after kill
- **Overdrive** (H.U.N.T.E.D.): `speedMod: 1.25` (25% faster) + boost for 10s after kill
- **AchingKnees** (Halloween): `reversedMovement = true` (input reversed, not speed change)

### Damage Output

**Formula: Applied during `GunItem.fire()` for each perk in `owner.perks`**

```typescript
modifiers.damage *= perk.damageMod  // Multiplicative stacking

// Example: Flechettes (0.4) + CloseQuartersCombat (1.2) + Toploaded (1.1 @ magazine threshold)
// = base × 0.4 × 1.2 × 1.1 = base × 0.528 (47.2% of normal)
```

**Unconditional damage mods:**
- **Flechettes**: `0.4` (60% reduction) — only non-explosive, non-special weapons
- **SabotRounds**: `0.9` (10% reduction) — only standard bullets
- **Berserker**: `1.2` (20% bonus)
- **HollowPoints** (Hunted): `1.1` (10% bonus) — only shotguns/slug rounds
- **PlumpkinBomb** (Halloween): `1.2` (20% bonus) — only grenades
- **Lycanthropy** (Halloween): `2.0` (100% bonus) — **this perk is overpowered intentionally**

**Conditional damage mods:**
- **CloseQuartersCombat**: `damageMod: 1.2` **IF enemy within 50-unit radius**; also `reloadMod: 1.3` (reload 30% faster)
- **LastStand** (Halloween): `damageMod: 1.267`, `damageReceivedMod: 0.8` **IF health < 30 HP** (27% more out, 20% less in)
- **Toploaded** (Normal): 
  - Magazine 0–20% empty: `damageMod: 1.25` (25% bonus)
  - Magazine 20–49% empty: `damageMod: 1.1` (10% bonus)
  - Magazine 50%+ empty: no bonus
  - **Thresholds array from definition:** `[[0.2, 1.25], [0.49, 1.1]]`

### Reload Time

**Semantics:** `reloadTime /= mod` (larger mod = faster reload)

```
actualReloadTime = baseReloadTime / (mod1 × mod2 × ...)
```

**Reload modifiers:**
- **CombatExpert** (Normal): `reloadMod: 1.25` (25% faster reload)
- **CloseQuartersCombat**: `reloadMod: 1.3` (30% faster) — **only in range (50 units)**
- **Butterfingers** (Halloween): `reloadMod: 0.75` (25% **slower** reload) — NEGATIVE Easter egg
- **FieldMedic** (Normal): `usageMod: 1.5` (50% faster healing, not reload)

### Spread (Accuracy)

```
actualSpread = baseSpread × (mod1 × mod2 × ...)
```

**Spread modifiers (lower = tighter):**
- **SabotRounds**: `spreadMod: 0.6` (40% tighter spread)
- **Overclocked**: `spreadMod: 2` (200% wider; this perk increases chaos)

### Fire Rate

```
fireDelay *= fireRateMod  (larger mod = slower fire)
```

**Fire rate modifiers:**
- **Overclocked**: `fireRateMod: 0.65` (35% faster fire rate)

### Player Size

```
playerRadius *= sizeMod
```

**Size modifiers:**
- **LowProfile**: `sizeMod: 0.8` (20% smaller) — harder to hit
- **Overweight** (Halloween): `sizeMod: 1.3` (30% larger) — NEGATIVE perk
- **Engorged** (Halloween): `sizeMod: 1.05` (5% larger) — side effect

### Health

**Additive:** `maxHealth += mod` (if mod < 10)
**Multiplicative:** `maxHealth *= mod` (if mod >= 10)

**Health modifiers:**
- **Engorged** (Halloween): `healthMod: 10` (additive; +10 max HP) — also `sizeMod: 1.05`
- **ExperimentalTreatment** (Halloween): `healthMod: 0.8` (multiplicative; 20% **reduction**) — also locks adrenaline at 100
- **Lycanthropy**: `healthMod: 1.5`, `regenRate: 1` (+1 HP/s regen)

### Range & Projectile Speed

- **SabotRounds**: `rangeMod: 1.5` (bullets travel 50% farther), `speedMod: 1.5` (50% faster)
- **DemoExpert** (Normal): `rangeMod: 2` (grenades go 2× distance), `restoreAmount: 0.25` (restore 25% capacity periodically)

## Special Abilities

### Thermal Goggles (H.U.N.T.E.D. mode)

**Server-side (periodic updates):**
```typescript
if (player.hasPerk(PerkIds.ThermalGoggles)) {
    const detectionRadius = 100;
    // Find all players within 100 units in game.grid
    // Send detection data to client
}
```

**Client-side:**
- Highlight discovered enemies with special visual effect (thermal vision overlay)
- Audio cue when new player detected

### Experimental Forcefield (H.U.N.T.E.D. mode)

**Server-side:**
```typescript
if (!hasBubble && player.hasPerk(PerkIds.ExperimentalForcefield)) {
    player.shield = 100;  // Regen 100 shield per respawn
}

// OnRemove:
player.hadShield = true;
player.shield = 0;  // Destroy shield on removal
```

**Properties:**
- `shieldRegenRate: 1` (shield regenerates 1 HP/s while perk held)
- `shieldRespawnTime: 20e3` (respawn with shield; 20s cooldown)
- Sound effects: `shield_obtained`, `shield_destroyed`, `glass_hit_1/2`
- Particle effect: `window_particle`

### Thermal Goggles + Hollow Points + Experimental Forcefield = Overdrive

**Easter egg trigger (in addPerk):**
```typescript
if (hasPerk(HollowPoints) && hasPerk(ExperimentalForcefield) && hasPerk(ThermalGoggles)) {
    addPerk(PerkIds.Overdrive);
}
```

**Overdrive properties:**
- Automatically granted when all three H.U.N.T.E.D. perks collected
- `speedMod: 1.25`, `sizeMod: 1.05`, `healBonus: 25`, `adrenalineBonus: 25`
- Activates after `requiredKills: 0` (actually 1, but coded as 0)
- Boost lasts `speedBoostDuration: 10e3` (10 seconds)
- Cooldown: `cooldown: 12e3` (12 seconds)
- Visual: `charged_particle` effect
- Audio: `overdrive` sound
- **Cannot be manually removed** (`noDrop: true`, `plumpkinGambleIgnore: true`)

### Shrouded (Halloween)

**Purpose:** Cloak effect (visual invisibility)

**Mechanics:**
```typescript
updateInterval: 100,  // Updates every 100ms
```

**Server-side:** Cloaking visuals, footstep suppression, reduced detection range
**Client-side:** Render transparency, particles

### EternalMagnetism (Halloween)

**Purpose:** Magnetic field attracts nearby loot

**Server-side:**
```typescript
if (player.hasPerk(PerkIds.EternalMagnetism)) {
    const { radius, lootPush } = PerkData[PerkIds.EternalMagnetism];
    // Query game.grid for loot within radius (20 units)
    // Apply force: loot.velocity += direction × lootPush
}
```

**Properties:**
- `radius: 20` (magnetism range in units)
- `depletion: 0.05` (energy cost per tick, affects duration?)
- `lootPush: 0.0005` (magnitude of force applied to loot)
- `minHealth: 5` (minimum player health to maintain perk)

### Bloodthirst (Halloween)

**Mechanics:**
```typescript
updateInterval: 1e3,           // Check every 1000ms
speedBoostDuration: 2000,       // Boost lasts 2 seconds
healthLoss: 1,                  // Passive health drain (1 HP/s?)
healBonus: 25,                  // +25 HP on kill
adrenalineBonus: 25            // +25 adrenaline on kill
speedMod: 1.5                  // Always apply 50% speed boost
```

**Behavior:** High-speed aggressive perk that drains health but heals on kills.

### Costumed (Halloween)

**Randomized disguise as an obstacle (tree, crate, barrel, rock, pumpkin, etc.)**

**Properties:**
- `choices: { oak_tree: 1, rock: 1, barrel: 0.75, airdrop_crate: 0.1, plumpkin: 0.01, ... }`
- Weights determine random selection probability
- `noSwap: false, alwaysAllowSwap: true` (can be swapped manually despite being a gimmick)

**Server-side:**
```typescript
this.activeDisguise = Obstacles.fromString(
    weightedRandom(Object.keys(choices), Object.values(choices))
);
```

**Client-side:** Render player as the selected obstacle model.

## Interaction with Damage System

### Damage Calculation Pipeline

**@file server/src/inventory/gunItem.ts, `fire()` method (~line 213)**

1. **Iterate perk array:** `for (const perk of owner.perks)`
2. **Apply perk-specific modifiers:**
   ```typescript
   switch (perk.idString) {
       case PerkIds.Flechettes:
           modifiers.damage *= perk.damageMod;      // 0.4×
           doSplinterGrouping = true;               // Change projectile behavior
           break;
       case PerkIds.SabotRounds:
           modifiers.range *= perk.rangeMod;        // 1.5×
           modifiers.speed *= perk.speedMod;        // 1.5×
           modifiers.damage *= perk.damageMod;      // 0.9×
           spread *= perk.spreadMod;                // 0.6× (tighter)
           modifiers.tracer.length *= perk.tracerLengthMod;  // 1.2×
           break;
       case PerkIds.CloseQuartersCombat:
           if (nearbyEnemyInRange(50)) {
               modifiers.damage *= perk.damageMod;  // 1.2×
               owner.reloadMod = perk.reloadMod;    // 1.3 (faster)
           } else {
               owner.reloadMod = 1;  // Reset if no enemies nearby
           }
           break;
       case PerkIds.Toploaded:
           const ratio = 1 - (this.ammo / capacity);  // % of mag empty
           for (const [cutoff, mod] of perk.thresholds) {  // [[0.2, 1.25], [0.49, 1.1]]
               if (ratio <= cutoff) {
                   modifiers.damage *= mod;
                   break;
               }
           }
           break;
       case PerkIds.HollowPoints:
           if (shotgun && !slugRound) {
               modifiers.damage *= perk.damageMod;  // 1.1×
           }
           break;
       case PerkIds.LastStand:
           if (owner.health < perk.healthReq) {   // 30 HP threshold
               modifiers.damage *= perk.damageMod;  // 1.267×
           }
           break;
       case PerkIds.Overclocked:
           spread *= perk.spreadMod;  // 2× (wider, more chaotic)
           break;
   }
   ```
3. **Create projectiles:** `spawn(position, spread, shotFX)` with modified `modifiers` object
4. **Projectile damage applied on hit:** `Bullet.damage *= modifiers.damage`

### Modifier Stacking Order (CRITICAL)

**Multiplicative modifiers stack additively in exponents:**
```
finalDamage = baseDamage × mod1 × mod2 × mod3 × ...

Example:
baseDamage = 20
Flechettes:           20 × 0.4   = 8
CloseQuarters:        8 × 1.2    = 9.6
Toploaded (1.25x):    9.6 × 1.25 = 12
FINAL:                12 (vs base 20)
```

**Order:** All multiplicative mods apply in iteration order (order of `owner.perks` array).

## Interaction with Inventory

### Perk Slots & Armor Integration

**Perks are separate from armor inventory slots:**
- Armor (helmet/vest/backpack) stored in `Player.inventory.helmet/vest/backpack`
- Perks stored in `Player.perks[]` (independent array)
- No slot conflict or swap mechanism (not armor slots)

### ExtendedMags Special Case

**On equip:**
```typescript
case PerkIds.ExtendedMags:
    // Define per-weapon capacity in weapon definitions
    // (no server code change needed; weapon definition has extendedCapacity property)
    break;
```

**On remove:**
```typescript
case PerkIds.ExtendedMags:
    for (const weapon of inventory.weapons) {
        const def = weapon.definition;
        const extra = weapon.ammo - def.capacity;
        if (extra > 0) {
            weapon.ammo = def.capacity;  // Trim to base capacity
            inventory.giveItem(def.ammoType, extra);  // Return excess ammo
        }
    }
    break;
```

### Lycanthropy Special Case

**On equip:**
```typescript
case PerkIds.Lycanthropy:
    // Change skin to werewolf
    this.loadout.skin = Skins.fromString("werewolf");
    // Drop all weapons & ammo
    inventory.dropWeapon(0, true)?.destroy();
    inventory.dropWeapon(1, true)?.destroy();
    inventory.dropWeapon(2, true)?.destroy();
    // Drop armor (helm/vest only; keep backpack)
    if (inventory.helmet) inventory.dropItem(inventory.helmet);
    if (inventory.vest) inventory.dropItem(inventory.vest);
    // Equip werewolf fur vest (special armor)
    inventory.vest = Armors.fromString("werewolf_fur");
    break;
```

**On remove:**
```typescript
case PerkIds.Lycanthropy:
    // Restore original skin from saved `_perkData` cache
    this.loadout.skin = Skins.fromStringSafe(this._perkData["Lycanthropy::old_skin"]) ?? /* fallback */;
    inventory.vest = undefined;  // Remove werewolf fur
    inventory.unlockAllSlots();  // Restore locked slots
    break;
```

## Client Display

### Synchronization

**Network packet (`UpdatePacket`):**
- Client receives `PlayerData.perks: PerkDefinition[]` (array of perk definitions)
- Only sent when `Player.dirty.perks = true`
- Full array serialized (not delta-compressed)

### PerkManager (Client-side)

**@file client/src/scripts/managers/perkManager.ts**

```typescript
class PerkManagerClass {
    perks: PerkDefinition[] = [];
    
    add(perk: ReifiableDef<PerkDefinition>): void {
        const perkDef = Perks.reify(perk);
        if (this.perks.includes(perkDef)) return;
        this.perks.push(perkDef);
    }
    
    has(perk: ReifiableDef<PerkDefinition>): boolean {
        return this.perks.includes(Perks.reify(perk));
    }
    
    remove(perk: ReifiableDef<PerkDefinition>): void {
        const perkDef = Perks.reify(perk);
        if (!this.perks.includes(perkDef)) return;
        removeFrom(this.perks, perkDef);
    }
    
    map<U>(perk: PerkId, mapper: (data: PerkDef) => U): U | undefined {
        const def = Perks.reify(perk);
        if (this.perks.includes(def)) {
            return mapper(def);
        }
    }
}

export const PerkManager = new PerkManagerClass();  // Singleton
```

### HUD Display

**Expected client implementation (not fully shown in codebase):**
- Render perk icons in top-left or dedicated perk UI area
- Icons ordered by acquisition (FIFO visual order)
- Color-code by quality: positive (green), neutral (gray), negative (red)
- Hover tooltip showing perk name & effect description
- Update incrementally (add/remove individual perk icons, don't redraw entire HUD)

### Particle Effects

**Perk-specific visuals (server broadcasts, client renders):**
- **Shrouded:** Cloaking transparency, particles
- **PlumpkinBomb:** Explosive particle effect on grenade throws
- **EternalMagnetism:** `spriteScale: 1.5` particle visual
- **Overdrive:** `charged_particle` effect on activation
- **Costumed:** Obstacle model replaces player model

### Audio Cues

- **ExperimentalForcefield:** `shield_obtained`, `shield_destroyed`, `glass_hit_1/2`
- **Overdrive:** `overdrive` activation sound
- **ThermalGoggles:** Detection ping sound (likely)
- **HollowPoints:** `soundMod: 75` (audio parameter for shot detection by hunted targets)

## Perk Combinations & Stacking Rules

### Easter Eggs

#### Hollow Points + Experimental Forcefield + Thermal Goggles → Overdrive

**Trigger location:** @file server/src/objects/player.ts:2111

```typescript
if (this.hasPerk(PerkIds.HollowPoints) && 
    this.hasPerk(PerkIds.ExperimentalForcefield) && 
    this.hasPerk(PerkIds.ThermalGoggles)) {
    this.addPerk(PerkIds.Overdrive);  // Auto-grant
}
```

**Result:** Gain Overdrive perk (H.U.N.T.E.D. mode combo bonus).

#### Combat Expert + Butterfingers → Explosion

**Trigger location:** @file server/src/objects/player.ts:2116

```typescript
if (this.hasPerk(PerkIds.CombatExpert) && this.hasPerk(PerkIds.Butterfingers)) {
    this.removePerk(PerkIds.CombatExpert);
    this.removePerk(PerkIds.Butterfingers);
    this.game.addExplosion("corrupted_explosion", this.position, this, this.layer);
}
```

**Result:** Both perks removed, player takes explosion damage (punishment for unfortunate combo).

### Damage Multiplier Stacking

**All multiplicative perks compound (see Damage Calculation Pipeline above).**

Example combo (Halloween):
```
Base damage: 20
Lycanthropy:  20 × 2.0 = 40
LastStand:    40 × 1.267 = 50.68 (if health < 30)
FINAL:        50.68 damage per shot
```

### Speed Modifier Stacking

**Multiplicative scaling:**
```
Base speed: 0.06
Second Wind:     0.06 × 1.4 = 0.084  (if health < 50%)
AdvancedAthletics: 0.084 × 1.3 = 0.1092  (if in water)
Adrenaline boost: 0.1092 × 1.15 = 0.12558  (at max adrenaline)
FINAL (theoretical max):  ~2.1× base speed
```

### Infection Mode Constraints

**When player infected (has `Infected` perk):**

```typescript
case PerkIds.Infected:
    const allowedPerks = Perks.definitions.filter(
        perk => perk.category !== PerkCategories.Infection && 
               !perk.infectedEffectIgnore
    );
    const perkToRemove = this.perks.find(p => /* not Infection, not infectedEffectIgnore */);
    if (perkToRemove) this.removePerk(perkToRemove);
    
    this.addPerk(pickRandomInArray(allowedPerks));  // Replace with random permitted perk
    break;
```

**Effect:** Non-Infection perks with `infectedEffectIgnore: false` are automatically replaced.
**Perks with `infectedEffectIgnore: true` are immune** (e.g., every H.U.N.T.E.D. perk has this flag).

### PlumpkinGamble Randomization

**Excluded perks:**
- `plumpkinGambleIgnore: true` (some Halloween perks, Infected, Overdrive)
- `category === Infection` (cannot gamble to Infection perks)

**Result:** Replaces with random valid Halloween perk.

### Perk Quality (Visual/UI Only)

**Property:** `quality: PerkQualities`

Values: `"positive"` | `"neutral"` | `"negative"`

**Used for:**
- HUD color coding (green/yellow/red)
- Sorting/filtering in perk lists
- Wiki categorization (not game logic)

## Known Gotchas & Implementation Quirks

### 1. Multiplicative vs Additive Confusion

**Gotcha:** Some mods are **multiplicative** (`×`), others **additive** (`+`).

| Modifier | Type | Semantics |
|----------|------|-----------|
| `damageMod: 1.2` | Multiplicative | `damage *= 1.2` (20% **increase**) |
| `healthMod: 10` | **Additive** | `maxHealth += 10` (flat +10 HP) |
| `healthMod: 1.5` | Multiplicative | `maxHealth *= 1.5` (50% **increase**) |
| `reloadMod: 1.25` | **Divisive** | `reloadTime /= 1.25` (faster = larger mod) |

**Solution:** Check bounds: if `mod < 10`, treat as additive; else multiplicative.

### 2. Reload Mod Inverse Semantics

**Gotcha:** `reloadMod` is **inverted**: larger values = **faster** reload.

```
reloadTime = baseReloadTime / reloadMod

reloadMod: 1.25  → reloadTime = 100 / 1.25 = 80 ms (FASTER)
reloadMod: 0.75  → reloadTime = 100 / 0.75 = 133 ms (SLOWER)
```

**Examples:**
- **CombatExpert:** `reloadMod: 1.25` → **25% faster** reload
- **Butterfingers:** `reloadMod: 0.75` → **25% slower** reload (punishment perk)

### 3. Perk Update Intervals (Not Per-Tick)

**Gotcha:** Perks with `updateInterval` **do NOT update every game tick** (40 TPS = 25 ms per tick).

**Instead:**
```typescript
if ("updateInterval" in perkDef) {
    this.perkUpdateMap.set(perkDef, lastUpdateTime);
    // Check: (game.now - lastUpdateTime) >= updateInterval
}
```

**Examples:**
- `updateInterval: 100` — runs ~every 100ms (every 4 ticks)
- `updateInterval: 1e3` — runs ~every 1000ms (every 40 ticks)
- `updateInterval: 10e3` — runs every 10 seconds

**Impact:** `DemoExpert`, `PrecisionRecycling`, `Necrosis`, `BabyPlumpkinPie` and others don't activate every frame—performance optimization.

### 4. Perk-Specific State Storage

**Pattern:** Complex perks store state in `Player._perkData` object:

```typescript
{
    "Lycanthropy::old_skin": "hazel_jumpsuit",  // Cached original skin
    "Costumed::activeDisguise": "oak_tree",     // Current costume
    // ...
}
```

**Why:** Some perks need to remember pre-perk state (original skin, etc.). Direct properties are simpler but used when only one instance needed per player.

### 5. Perk Array Order Matters (FIFO Oldest-Drop)

**Gotcha:** `this.perks` is **ordered by acquisition time** (FIFO).

```
Acquired: [SecondWind, Flechettes, SabotRounds, Berserker]
New 5th perk acquired → SecondWind dropped (index 0)
Result: [Flechettes, SabotRounds, Berserker, NewPerk]
```

**Impact:**
- Order affects iteration (and thus modifier stacking order if sensitive to sequence)
- UI should display in FIFO order for clarity
- Dropping oldest is *intuitive* but not explained in-game

### 6. Conditional Perk Mods Not Stored in Definition

**Gotcha:** Perks like `SecondWind` and `CloseQuartersCombat` have **conditional modifiers** that are **not applied directly**.

Instead, the **client/server check the condition at use time:**

```typescript
// SecondWind speed only applies if health < 50%
if (this.hasPerk(PerkIds.SecondWind) && !(this.health / this.maxHealth < 0.5)) {
    this.modifiers.baseSpeed = 1;  // Disable the mod if condition fails
}

// CloseQuartersCombat damage only applies if enemy within 50 units
if ([...grid.intersectsHitbox(circleHitbox)].some(isEnemyWithin50)) {
    modifiers.damage *= perk.damageMod;
}
```

**Solution:** Read condition checks in `player.ts` update loop and `gunItem.ts` firing loop; definition alone is insufficient.

### 7. Easter Egg Interactions Must Be Checked in Order

**Gotcha:** `addPerk()` checks perk combinations **in sequence**. Order of enum union matters:

```typescript
// HollowPoints + ExperimentalForcefield + ThermalGoggles → AUTO-ADD Overdrive
if (this.hasPerk(A) && this.hasPerk(B) && this.hasPerk(C)) {
    this.addPerk(D);
}

// CombatExpert + Butterfingers → EXPLOSION + both removed
if (this.hasPerk(A) && this.hasPerk(B)) {
    this.removePerk(A);
    this.removePerk(B);
    this.game.addExplosion(...);
}
```

**Impact:** If you have all three Hunted perks, Overdrive is automatically added (good). If you accidentally get CombatExpert + Butterfingers, both are destroyed (bad).

### 8. Infection Mode Perk Replacement Logic

**Gotcha:** When `Infected` perk acquired:

```typescript
const perkToRemove = this.perks.find(p => 
    p.category !== PerkCategories.Infection && 
    !p.infectedEffectIgnore
);
```

**This removes ONE non-Infection perk**, then adds a random allowed perk.

**Edge case:** If player has all Hunted perks (all `infectedEffectIgnore: true`), `perkToRemove` is `undefined` and **no perk is replaced** → player keeps Hunted perks + gains Infected.

### 9. ExtendedMags Ammo Trim on Removal

**Gotcha:** If player removes `ExtendedMags` while holding excess ammo, ammo > base capacity is **dropped into inventory**, not deleted.

```typescript
case PerkIds.ExtendedMags:
    for (const weapon of inventory.weapons) {
        const extra = weapon.ammo - weapon.definition.capacity;
        if (extra > 0) {
            weapon.ammo = weapon.definition.capacity;
            inventory.giveItem(weapon.definition.ammoType, extra);  // Return to inventory
        }
    }
```

**Impact:** Safe perk swap; no ammo loss.

### 10. Costume (Costumed Perk) Quality Weights

**Gotcha:** Costume selection uses weighted random:

```typescript
const choices = {
    oak_tree: 1,              // Common
    airdrop_crate: 0.1,       // Rare
    plumpkin: 0.01            // Ultra-rare
};
```

**Lower weight = rarer disguise.** Ultra-rare costumes (plumpkin) appear 1% of the time.

### 11. Overdrive Easter Egg Cannot Be Manually Dropped

**Properties:**
```typescript
{
    noDrop: true,
    plumpkinGambleIgnore: true,
    infectedEffectIgnore: true
}
```

**Gotcha:** If you have Overdrive (from three-perk combo):
- Cannot be dropped via pickup menu (`noDrop: true`)
- Cannot be obtained via `PlumpkinGamble` (ignored)
- Cannot be replaced by Infection mode (immune)

**It's "sticky"** and must be explicitly removed via special game logic.

## Cross-Tier References

### Tier 1 (Architecture & High-Level)

- [docs/architecture.md](../../../architecture.md) — System components, network packet structure
- [docs/datamodel.md](../../../datamodel.md) — Player entity, inventory schema, perks array definition

### Tier 2 (Subsystem Overview)

- [docs/subsystems/perks-passive/README.md](../README.md) — Perk system overview, acquisition flow, category filtering
- [docs/subsystems/perks-passive/patterns.md](patterns.md) — Perk definition patterns, stat modifier conventions
- [docs/subsystems/networking/README.md](../../networking/README.md) — `UpdatePacket` serialization, delta encoding
- [docs/subsystems/game-loop/README.md](../../game-loop/README.md) — Game tick structure (40 TPS), per-tick updates
- [docs/subsystems/inventory/README.md](../../inventory/README.md) — Inventory slots, armor integration, item swap logic

### Tier 3 (Module Details)

- [docs/subsystems/game-loop/modules/tick.md](../../game-loop/modules/tick.md) — Per-tick update cycle, perk interval checks
- [docs/subsystems/inventory/modules/gunItem.md](../../inventory/modules/gunItem.md) — Weapon firing, perk modifier application
- [docs/subsystems/networking/modules/updatePacket.md](../../networking/modules/updatePacket.md) — PlayerData fields, perk array encoding/decoding

### Skills & Tools

- **Debugging modifier calculations:** Use `server/src/utils/debug.ts` perk logging
- **Testing perk combinations:** Jest test suite in `tests/src/perks.test.ts` (if exists)
