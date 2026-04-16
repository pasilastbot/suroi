# Armor Mitigation

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/body-armor/README.md -->
<!-- @source: common/src/definitions/items/armors.ts | server/src/objects/player.ts | common/src/definitions/items/perks.ts -->

## Purpose

This module documents how body armor (helmets and vests) reduces incoming damage,
including tier structure, damage reduction formulas, perk interactions, and armor
state synchronization to clients.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| `common/src/definitions/items/armors.ts` | Armor definition schema (`ArmorDefinition`, `ArmorType` enum, concrete armor data) | Medium |
| `server/src/objects/player.ts:2626-2720` | Damage calculation pipeline (`damage()`, `piercingDamage()`, armor reduction logic) | High |
| `server/src/inventory/inventory.ts:50-70` | Armor slot storage (`inventory.helmet`, `inventory.vest`) | Low |
| `common/src/definitions/items/perks.ts:371-390, 641-655` | Armor-related perks (`LastStand`, `ExperimentalForcefield`, `ThermalGoggles`) | Medium |
| `common/src/utils/objectsSerializations.ts:55-56, 253-254, 269-270` | Client serialization schema (helmet/vest in `FullData<Player>`) | Low |

---

## Armor Types & Tier Structure

Suroi features two armor categories: **helmets** and **vests**, each with discrete
tier levels.

### Helmet Tiers

All helmets are defined in `common/src/definitions/items/armors.ts` with `armorType: ArmorType.Helmet`.

| ID String | Name | Level | Damage Reduction | Perk | Notes |
|-----------|------|-------|------------------|------|-------|
| `basic_helmet` | Basic Helmet | 1 | 10% | — | Entry-level |
| `regular_helmet` | Regular Helmet | 2 | 15% | — | Common drop |
| `tactical_helmet` | Tactical Helmet | 3 | 20% | — | High-tier |
| `power_helmet` | NTK-11 Halycon | 4 | 25% | `ThermalGoggles` | Exotic (map-indicated) |

### Vest Tiers

All vests are defined with `armorType: ArmorType.Vest` and include a hex color for
rendering.

| ID String | Name | Level | Damage Reduction | Perk | Color | Notes |
|-----------|------|-------|------------------|------|-------|-------|
| `basic_vest` | Basic Vest | 1 | 20% | — | `0xc8c8c6` (white) | Entry-level |
| `regular_vest` | Regular Vest | 2 | 35% | — | `0x404d2e` (green) | Common drop |
| `tactical_vest` | Tactical Vest | 3 | 45% | — | `0x0d0d0d` (black) | High-tier |
| `power_vest` | ERV-3 Core | 4 | 35% | `ExperimentalForcefield` | `0xffffff` (white) | Exotic; provides shield |
| `werewolf_fur` | Werewolf Fur | 5 | 20% | — | `0x4d4d4d` (gray) | Event item (no drop, hidden) |
| `developr_vest` | Developr Vest | 99 | 72% | — | `0x2f0000` (red) | Dev item (no drop) |

**Upgrade Restrictions:**
- Vest damage reduction does **not** always scale with level (e.g., `power_vest` is
  level 4 but provides only 35% reduction, same as `regular_vest`).
  This is intentional to balance the `ExperimentalForcefield` perk value.
- Same rule applies to `werewolf_fur` (level 5, 20% reduction) — balancing the perk
  eco-system.

---

## Damage Reduction Formula

The damage reduction calculation is **additive** and applies only when the player's
shield is depleted.

### Formula

```
Final Damage = Raw Damage × (1 - armor_reduction) × perk_multiplier

Where:
  armor_reduction = (helmet.damageReduction ?? 0) + (vest.damageReduction ?? 0)
  perk_multiplier = LastStand bonus (if applicable) or 1.0
```

### Implementation

Source: `server/src/objects/player.ts:2626-2652`

```typescript
override damage(params: DamageParams): void {
    if (this.invulnerable) return;
    const { source, weaponUsed } = params;
    let { amount } = params;

    if (amount < 0) return this.heal(-amount);

    this.game.pluginManager.emit("player_damage", {
        amount,
        player: this,
        source,
        weaponUsed
    });

    if (this.shield <= 0) {
        // Reductions are merged additively
        amount *= (1 - (
            (this.inventory.helmet?.damageReduction ?? 0) + 
            (this.inventory.vest?.damageReduction ?? 0)
        )) * this.mapPerkOrDefault(PerkIds.LastStand, 
            ({ damageReceivedMod }) => damageReceivedMod, 1);

        amount = this._clampDamageAmount(amount);
    }

    this.piercingDamage({ amount, source, weaponUsed });
}
```

### Example Calculations

**Case 1: Basic Helmet + Regular Vest**
```
Raw damage:     50
Armor:          0.10 (helmet) + 0.35 (vest) = 0.45 total
Calculation:    50 × (1 - 0.45) = 50 × 0.55 = 27.5 damage taken
```

**Case 2: Tactical Helmet + Tactical Vest + LastStand Perk (health ≤ 30)**
```
Raw damage:     50
Armor:          0.20 (helmet) + 0.45 (vest) = 0.65 total
Perk multiplier: 0.8 (LastStand)
Calculation:    50 × (1 - 0.65) × 0.8 = 50 × 0.35 × 0.8 = 14 damage taken
Reduction:      86% total (65% armor + 20% perk)
```

**Case 3: No Armor**
```
Raw damage:     50
Armor:          0 (no helmet) + 0 (no vest) = 0 total
Calculation:    50 × (1 - 0) = 50 damage taken
```

### Key Rules

1. **Reduction only applies when shield = 0.** If shield > 0, damage reduces shield
   first without armor modification; only residual damage applies armor reduction.
2. **Reductions are additive, not multiplicative.** Cap is theoretically 100% if
   armor reduction reaches 1.0 (not possible with current definition pool).
3. **Perk multiplier applies after armor reduction.** LastStand multiplies the
   already-reduced damage value, not the armor factor itself.
4. **Negative damage heals.** If `amount < 0`, `damage()` immediately returns
   `heal(-amount)` without armor logic.

---

## Armor Durability

**There is no durability system in Suroi.** Armor does not degrade with use or damage.

An equipped helmet or vest remains functional until:
1. The player manually unequips it (swaps to different armor or drops it)
2. The player dies (armor auto-drops via `die()`)
3. The player is downed (armor auto-drops via `downed` state)

Armor properties (e.g., `damageReduction`, perk) are immutable and do not change
during a player's lifetime.

---

## Helmet Mechanics

### Armor Definition Schema

```typescript
type ArmorDefinition = ItemDefinition & {
    readonly defType: DefinitionType.Armor
    readonly level: number
    readonly damageReduction: number
    readonly perk?: ReferenceTo<PerkDefinition>           // Optional perk
    readonly positionOverride?: number                     // Rendering position override
    readonly positionOverrideDowned?: number              // Position when downed
    readonly emitSound?: string                           // Sound played on equip
    armorType: ArmorType.Helmet
}
```

### Helmet-Specific Properties

| Property | Purpose | Examples |
|----------|---------|----------|
| `perk` | Grants a passive perk when equipped | `power_helmet` → `ThermalGoggles` |
| `positionOverride` | Z-layer offset for helmet sprite rendering | `power_helmet`: 0 (forces layer 0) |
| `positionOverrideDowned` | Z-layer when player is downed | Ensures helmet renders above downed body |
| `emitSound` | Sound effect played when helmet is equipped | `werewolf_suit` → `"werewolf"` (not helmet) |

### Helmet Rendering

**Client-side rendering** (PixiJS):
- Helmet sprite positioned relative to player sprite
- `positionOverride` controls Z-order (depth); normally positive for "on head"
- If `positionOverride` is `0`, it forces the helmet to render at a fixed layer
- When player is downed, `positionOverrideDowned` adjust position (if defined)

Source: `server/src/objects/player.ts:3673` (serialized to client)

---

## Vest Mechanics

### Vest-Specific Properties

| Property | Purpose | Examples |
|----------|---------|----------|
| `color` | Hex color for rendering the vest sprite | `tactical_vest`: `0x0d0d0d` (black) |
| `worldImage` | Optional custom sprite for world (dropped) loot | `power_vest`: `"power_vest_world"` |
| `perk` | Grants a passive perk when equipped | `power_vest` → `ExperimentalForcefield` |

### Vest Rendering

- Vest color is applied as a tint/color overlay to the base sprite in the client
- If `worldImage` is defined, the dropped loot uses a custom sprite instead of
  the player's visual

### Vest-Inventory Interaction

Vests **do not provide inventory capacity.** The `Inventory` class treats helmets
and vests as **separate slots**, not as containers.

```typescript
// From server/src/inventory/inventory.ts
export class Inventory {
    helmet?: ArmorDefinition;
    vest?: ArmorDefinition;
    backpack: BackpackDefinition = Loots.fromString("bag");
    // ...
}
```

Backpack capacity (weight/inventory space) is controlled solely by the
`backpack` property, not vest type.

---

## Shield / Forcefield Mechanics

The **Experimental Forcefield** is a shield system granted by the `ExperimentalForcefield`
perk, exclusively attached to the `power_vest`.

### Forcefield Properties

**Perk Definition** (source: `common/src/definitions/items/perks.ts:641-655`):

```typescript
{
    idString: PerkIds.ExperimentalForcefield,
    name: "Experimental Forcefield",
    defType: DefinitionType.Perk,
    category: PerkCategories.Hunted,
    noDrop: true,
    shieldRegenRate: 1,              // HP/tick
    shieldRespawnTime: 20e3,         // 20 seconds
    shieldObtainSound: "shield_obtained",
    shieldDestroySound: "shield_destroyed",
    shieldHitSound: "glass",         // "_hit_1/2" suffix added by client
    shieldParticle: "window_particle",
    infectedEffectIgnore: true
}
```

### Shield Behavior

1. **Shield absorbs damage first.** When `player.shield > 0`, incoming damage
   reduces shield before armor calculates:
   ```typescript
   if (this.shield <= 0) {
       // Apply armor reduction HERE
       amount *= (1 - totalArmorReduction) * perkMult;
   } else {
       // Shield absorbs damage without armor modification
       this.shield -= amount;        // Full damage to shield
       if (remainingDamage > 0) {
           this.damage(...);         // Recursive call for residual
       }
   }
   ```

2. **Shield regenerates over time.** The perk's `shieldRegenRate: 1` means shield
   recovers 1 HP per game tick. Max shield is capped at `GameConstants.player.maxShield`.

3. **Shield respawns on break.** Once shield reaches 0, it enters a `shieldRespawnTime`
   (20 seconds) cooldown before regeneration resumes.

4. **Sound effects on use.** Client plays sounds on shield hit, obtain, and destroy
   events.

5. **Particle effects.** A visual "window_particle" effect displays on shield impacts.

### Color Coding

On `power_vest`, the perk indicator is `mapIndicator: "vest_indicator"` — clients
render a special icon or color to indicate the forcefield bonus.

---

## Damage Bypass / Armor Penetration

**There is no armor penetration mechanic in the current implementation.**

- All weapons and sources apply raw damage uniformly
- Armor reduction is a global player property, not negotiable per-source
- Special damage types (e.g., gas, blur, team mode self-damage) bypass armor
  only because they call `piercingDamage()` directly (see [Damage Event Logging](#damage-event-logging))

Future armor penetration would require:
1. A `penetration` field in `WeaponDefinition`
2. Conditional logic in `damage()` to reduce armor factor by penetration %
3. Tooltip updates to explain penetration to players

---

## Armor Swap

### Equipping Armor

When a player picks up armor, it replaces the currently equipped armor of the
same type.

Source: `server/src/inventory/inventory.ts` and `server/src/objects/player.ts:940, 2146-2147`

```typescript
// In inventory.ts, armor is stored as a simple reference:
helmet?: ArmorDefinition;
vest?: ArmorDefinition;

// Player equips armor via:
this.inventory.helmet = armorDefinition;
this.inventory.vest = armorDefinition;

// Auto-equips highest-tier armor on spawn:
this.inventory.helmet = max
    ? Array.from(Armors)
        .filter(({ armorType }) => armorType === ArmorType.Helmet)
        .sort(({ level: a }, { level: b }) => b - a)[0]
    : pickRandomInArray(Armors.definitions.filter(...));
```

### Dropping Armor

When a player drops armor, it becomes a world loot object.

Source: `server/src/objects/player.ts:2146-2147, 2194-2197`

```typescript
// Manual drop (via inventory UI):
inventory.dropItem(inventory.helmet);    // Becomes loot at player position
inventory.dropItem(inventory.vest);

// Auto-drop on death:
if (inventory.helmet) inventory.dropItem(inventory.helmet);
if (inventory.vest) inventory.dropItem(inventory.vest);
```

### Multiple Helmet / Vest Restriction

A player can have **one and only one** helmet and **one and only one** vest equipped
at any time. Equipping a second helmet automatically unequips (potentially drops) the first.

---

## Damage Event Logging

Armor damage is tracked via **plugin events** and player statistics.

### Plugin Events

The damage pipeline emits the following events (source: `server/src/objects/player.ts`):

| Event | When | Data | Cancelable |
|-------|------|------|-----------|
| `player_damage` | Before armor reduction | `{ amount, player, source, weaponUsed }` | No |
| `player_will_piercing_damaged` | Before pierce (after armor) | `{ amount, player, source, weaponUsed }` | Yes |
| `player_did_piercing_damaged` | After pierce (health updated) | `{ amount, player, source, weaponUsed }` | No |

### Player Statistics

Damage taken/dealt is tracked independently of armor:

```typescript
// In Player class (server/src/objects/player.ts)
damageDone = 0;       // Cumulative damage this player dealt to others
damageTaken = 0;      // Cumulative damage this player received

// In piercingDamage():
if (amount > 0) {
    this.damageTaken += amount;        // Track all incoming damage
    if (sourceIsPlayer && source !== this) {
        source.damageDone += amount;   // Only count player-to-player
    }
}
```

### Weapon Statistics

If the damage source is a weapon (gun, melee, throwable), weapon-specific stats
are updated:

```typescript
const canTrackStats = weaponUsed instanceof InventoryItemBase;
if (canTrackStats && !this.dead) {
    weaponUsed.stats.damage += amount;
    // Check for kill bonuses, on-damage perks, etc.
}
```

---

## Client Sync

### Dirty Flagging

When armor is equipped or unequipped, the player object is marked dirty:

Source: `server/src/objects/player.ts:940, 989, 1547, 2142, 3132`

```typescript
this.dirty.items = true;   // Marks inventory (helmet, vest, scope) as changed
```

### Serialization

On the next game update, the server serializes helmet and vest **in the full data**,
not partial deltas:

Source: `common/src/utils/objectsSerializations.ts:253-254, 269-270`

```typescript
readonly helmet?: ArmorDefinition    // In FullData<ObjectCategory.Player>
readonly vest?: ArmorDefinition

// During serialization:
const hasHelmet = helmet !== undefined;
const hasVest = vest !== undefined;

// Armors.writeToStream(stream, armor) sends single-byte index into Armors[] registry
```

### Transmission Format

- Each armor is transmitted as a **single byte index** into the `Armors` registry
  (via `ObjectDefinitions.writeToStream()`), not the full definition
- Client reconstructs the `ArmorDefinition` by looking up the index
- This is space-efficient: 1 byte per armor vs. multi-byte serialized definition

### When Helmets/Vests are Sent

Helmet and vest **always transmit in FullData** (initial spawn data), and also on
**partial updates when `dirty.items = true`**.

This ensures:
- New players joining the game learn current armor state
- Armor changes are synced in real-time during gameplay
- Armor color changes propagate to all clients for rendering

---

## Known Gotchas, Edge Cases & Tech Debt

### 1. Double Damage Reduction (Shield + Armor)

When a player has both shield and armor:
- Shield absorbs **100% of damage** at full value (no armor reduction)
- Only after shield breaks does armor apply

This is intentional but non-obvious. Example:
```
Raw damage: 50
Shield: 30, Armor: (Helmet 0.2 + Vest 0.4)

Step 1: Shield absorbs 50 → Shield becomes 0, residual 20 damage
Step 2: Residual 20 hits armor → 20 × (1 - 0.6) = 8 damage to health
```

**Implication:** A player with 30 shield + 70 health takes 50 base damage and ends
at 28 health (not 42 as naive calculation would suggest).

### 2. Vest Level ≠ Damage Reduction

Vests do **not** follow a strict tier → damage reduction progression:

- `regular_vest`: level 2, 35% reduction
- `tactical_vest`: level 3, 45% reduction
- `power_vest`: level 4, **35% reduction** (intentional, because perk is valuable)

Clients may assume higher level = more reduction. Server-side code is correct,
but UI/tooltips should clarify this.

### 3. Armor Perk Mutually Exclusive Slots

Helmets and vests can grant perks, but only one of each type per player:
- Only one helmet → only one helmet perk at a time
- Only one vest → only one vest perk at a time

If a player swaps from `power_helmet` (ThermalGoggles) to `tactical_helmet`,
`ThermalGoggles` is immediately lost.

Future work: Warn player when swapping away perked armor.

### 4. Armor Color / Vest Rendering

Vest color is **hardcoded in definition**, not player-configurable:

```typescript
{
    idString: "regular_vest",
    name: "Regular Vest",
    color: 0x404d2e        // Fixed green tint
}
```

Skins do **not** change vest color; all players with `regular_vest` see the same green.

### 5. No Armor Degradation / Durability

Armor is **pristine forever** until dropped. No "damaged armor" state exists.

This simplifies code but may feel unrealistic compared to other BR games. Future
work could add:
- Durability bars
- Armor breaks permanently after X hits
- Temporary armor weakening below 50% health

### 6. Helmet Position Override Edge Case

The `positionOverride` field forces a helmet to render at a specific Z-layer:

```typescript
{
    idString: "power_helmet",
    positionOverride: 0,         // Forces layer 0
    positionOverrideDowned: 0,   // Also when downed
}
```

If `positionOverride: 0` and another object is also at layer 0, client rendering
order may be ambiguous. Ensure client's renderer stable-sorts by object ID if
rendering at the same explicit layer.

### 7. LastStand Perk Health Threshold Not Validated

The `LastStand` perk definition has:

```typescript
{
    idString: PerkIds.LastStand,
    healthReq: 30,                  // Damage reduction actives < 30 health
    damageReceivedMod: 0.8,         // 20% reduction
}
```

But the code applies the multiplier via `mapPerkOrDefault()` without confirming
health threshold. Verify this is calculated correctly server-side.

---

## Related Documentation

### Tier 2 (Subsystem)
- [📋 Body Armor Subsystem](../README.md) — Overview of helmets, vests, shields, perks
- [📋 Perks & Passives](../../perks-passive/README.md) — Perk mechanics, including `LastStand`, `ExperimentalForcefield`, `ThermalGoggles`

### Tier 1 (Architecture)
- [📋 System Architecture](../../../architecture.md) — Suroi system design, player object model
- [📋 Data Model](../../../datamodel.md) — Core game entities, including `Player` and `Inventory`
- [📋 Game Loop](../../../subsystems/game-loop/README.md) — How damage is applied each tick

### Tier 3 (Related Modules)
- [📋 Inventory Management](../../inventory/modules/inventory-management.md) — How helmets/vests are stored and swapped
- [📋 Damage Application](../../game-loop/modules/damage-handling.md) — Broader damage pipeline
- [📋 Perk Application](../../perks-passive/modules/perk-mechanics.md) — How perks modify damage reduction

### Related Specs
- Look for `specs/features/armor.md` or `specs/features/shields.md` if features are actively being modified

---

## Cross-Tier References

**Depends on:**
- Tier 2: [Body Armor](../README.md) — Subsystem architecture
- Tier 1: [Data Model](../../../datamodel.md) — `ArmorDefinition` schema
- Tier 1: [Game Loop](../../../subsystems/game-loop/README.md) — When damage is applied

**Referenced by:**
- Tier 3: Inventory modules (armor equipping/dropping)
- Tier 3: Damage modules (damage calculation)
- Tier 3: Perk modules (perk application during damage)
