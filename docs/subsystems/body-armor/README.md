# Body Armor & Equipment

<!-- @tier: 2 -->
<!-- @parent: docs/architecture.md -->
<!-- @modules: docs/subsystems/body-armor/modules/ -->
<!-- @source: server/src/inventory/inventory.ts -->

## Purpose

Wearable protective equipment that reduces incoming damage. Armor comes in two types—**helmets** and **vests**—with tier-based damage mitigation. Equipment can be upgraded by picking up higher-level armor, and some special armor grants additional perks. Armor is tier-based: only higher-level armor can replace equipped armor; equipping lower/equal tier returns an error message.

## Key Files & Entry Points

| File | Purpose |
|------|---------|
| [common/src/definitions/items/armors.ts](../../../../common/src/definitions/items/armors.ts) | ArmorDefinition, armor registry (Helmet, Vest), level & damage reduction |
| [server/src/inventory/inventory.ts](../../../../server/src/inventory/inventory.ts) | Helmet/vest slot management, armor equipping/dropping logic |
| [server/src/objects/loot.ts](../../../../server/src/objects/loot.ts) | Armor pickup interaction, equipment replacement |
| [server/src/objects/player.ts](../../../../server/src/objects/player.ts) | Damage reduction application, armor state queries |
| [client/src/scripts/objects/player.ts](../../../../client/src/scripts/objects/player.ts) | Client-side armor representation, visual equipment slots |

## Armor Types & Damage Reduction

All armor in the game with tier level, damage reduction, and special properties:

### Helmets

| Armor | Level | Damage Reduction | Perk | Icon |
|-------|-------|------------------|------|------|
| **Basic Helmet** | 1 | 10% | — | — |
| **Regular Helmet** | 2 | 15% | — | — |
| **Tactical Helmet** | 3 | 20% | — | — |
| **NTK-11 Halycon** | 4 | 25% | Thermal Goggles | `helmet_indicator` |

### Vests

| Armor | Level | Damage Reduction | Perk | Special |
|-------|-------|------------------|------|---------|
| **Basic Vest** | 1 | 20% | — | — |
| **Regular Vest** | 2 | 35% | — | — |
| **Tactical Vest** | 3 | 45% | — | — |
| **ERV-3 Core** | 4 | 35% | Experimental Forcefield | Shield recharge perk |
| **Werewolf Fur** | 5 | 20% | — | Special event, hidden in HUD |

**Dev/Special:** *Developr Vest* (Level 99, dev-only): 72% damage reduction

## Armor State

Each player maintains paired armor slots in their inventory:

```typescript
inventory {
  helmet?: ArmorDefinition,    // Current head armor (or undefined)
  vest?: ArmorDefinition,      // Current body armor (or undefined)
}
```

**Armor properties:**
- `level`: Integer tier (1–4, higher = better)
- `damageReduction`: Float 0.0–1.0 (10% = 0.1, 45% = 0.45, etc.)
- `perk`: Optional perk granted when equipped (e.g., ThermalGoggles, ExperimentalForcefield)
- `armorType`: Helmet or Vest (determines slot)
- `emitSound`: Optional special sound on equip (e.g., "werewolf")

## Damage Reduction Formula

When a player takes damage, armor reduces it **only if shield is depleted** (shield ≤ 0):

```
// Pseudo-code
if (shield <= 0) {
  totalReduction = (helmet.damageReduction ?? 0) + (vest.damageReduction ?? 0)
  // Reductions are ADDITIVE, not multiplicative
  
  damageReduction *= (1 - totalReduction)
  
  // Further modified by LastStand perk if active
  damageReduction *= lastStandDamageReceivedMod
}
```

### Example

**100 HP damage incoming. Player has:**
- Tactical Helmet (20% reduction)
- Tactical Vest (45% reduction)
- Total reduction: 20% + 45% = 65%
- Applied damage: 100 × (1 - 0.65) = 100 × 0.35 = **35 HP** taken
- 65 HP blocked by armor

**Note:** Armor only functions when shield <= 0. If shield is active, armor is bypassed entirely.

## Armor Equipping Mechanics

### Pickup Interaction

When a player approaches armor loot:

1. **Interaction Check** → [server/src/objects/loot.ts:236](../../../../server/src/objects/loot.ts#L236)
   - Compare armor level to equipped armor
   - If **higher level** → can pickup (return `true`)
   - If **lower level** → return `InventoryMessages.BetterItemEquipped`
   - If **same level** → return `InventoryMessages.ItemAlreadyEquipped`

2. **Equipping** → [server/src/objects/loot.ts:437](../../../../server/src/objects/loot.ts#L437)
   ```typescript
   // When armor is picked up:
   if (armor.armorType === ArmorType.Helmet) {
     // Drop old helmet if exists
     if (player.inventory.helmet) dropItem(player.inventory.helmet)
     // Equip new helmet
     player.inventory.helmet = definition
   } else { // Vest
     if (player.inventory.vest) dropItem(player.inventory.vest)
     player.inventory.vest = definition
   }
   
   // If armor provides perk, grant it
   if (definition.perk) player.addPerk(definition.perk)
   ```

3. **Sync to Clients** → `player.setDirty()` marks player for update packet

### Dropping Armor

Via inventory drop action → [server/src/inventory/inventory.ts:616](../../../../server/src/inventory/inventory.ts#L616)

```typescript
switch (definition.armorType) {
  case ArmorType.Helmet:
    // Only drop if equipped and level matches
    if (!this.helmet || this.helmet.level !== definition.level) return
    this.helmet = undefined
    break
  case ArmorType.Vest:
    if (!this.vest || this.vest.level !== definition.level) return
    this.vest = undefined
    break
}

// If armor provided perk, remove it
if (definition.perk) player.removePerk(definition.perk)
dropItem(definition) // Create loot in world
```

## Armor with Perks

Special armor pieces provide bonuses:

| Armor | Perk | Effect |
|-------|------|--------|
| **NTK-11 Halycon** | Thermal Goggles | Vision enhancement (scope/building peek) |
| **ERV-3 Core** | Experimental Forcefield | 100 HP shield, recharges after delay |

When armor with perk is equipped:
- Perk is added to player's active perk list
- Perk benefits apply immediately
- Position override feature allows helmet to render at different z-layer

When armor is dropped/unequipped:
- Perk is removed from player
- Player loses perk benefits

## Equipment Slot Overview

Player inventory maintains three equipment slots:

```typescript
inventory {
  helmet?: ArmorDefinition,       // Head protection
  vest?: ArmorDefinition,         // Torso protection
  backpack: BackpackDefinition,   // Inventory capacity (separate subsystem)
}
```

**Slot interactions:**
- Armor is never dropped on death (stored in inventory during respawn)
- Armor can be individually equipped/dropped without affecting other slots
- Armor tier upgrades replace old armor (old armor drops to ground)

## Dependencies

### Depends on:
- [Object Definitions](../object-definitions/) — armor definition registry, lookup by `idString`
- [Inventory](../inventory/) — helmet/vest slot storage, item equipping framework
- [Perks & Passive](../perks-passive/) — perk granting/removal (ThermalGoggles, ExperimentalForcefield)
- [Game Loop](../game-loop/) — integration point for damage application

### Depended on by:
- [Game Loop](../game-loop/) — damage calculations apply armor reduction
- [Game Objects (Server)](../game-objects-server/) — Player.takeDamage() calls armor reduction
- [Networking](../networking/) — armor state serialized in UpdatePacket (helmet/vest definitions)
- [Client Rendering](../client-rendering/) — armor visual on player sprite
- [UI Management](../ui-management/) — armor HUD indicators (level, damage reduction %)
- [Loot System](../airdrop-system/) — armor appears in world as loot, ground spawn

## Data Flow

Armor state flows through the system:

```
Pickup armor loot
  → canInteract() checks armor level vs equipped
  → interact() equips armor (helmet/vest slot)
  → removePerk() from old armor (if had perk)
  → addPerk() to new armor (if has perk)
  → player.setDirty()
  
Player takes damage
  → takeDamage() checks shield > 0?
  → if shield ≤ 0: apply armor reduction
  → reduction = helmet.damageRed + vest.damageRed
  → damage *= (1 - reduction)
  → piercingDamage() applies final damage
  
Drop armor packet
  → dropItem() checks armor definition match
  → removes perk if armor had one
  → creates loot in world
  → inventory.helmet/vest = undefined
  
Respawn / Team Mode
  → fillInventory() may equip random armor
  → Special events: werewolf_fur (Lycanthropy perk)
```

## Client Rendering

Client-side armor representation in [client/src/scripts/objects/player.ts](../../../../client/src/scripts/objects/player.ts):

```typescript
equipment: {
  helmet?: ArmorDefinition,
  vest?: ArmorDefinition,
  backpack: BackpackDefinition,
}
```

**Visual updates:**
- Armor graphics layer rendered on player sprite
- Helmet color: varies by definition
- Vest color: hex color from armor definition (e.g., `color: 0x0d0d0d` = black)
- Position overrides: allows helmet to render at different z-depth (e.g., *NTK-11 Halycon*)

**HUD Indicators:**
- Armor level displayed (1–4)
- Damage reduction % shown
- Map indicators for special armor (if `mapIndicator` property set)

## Known Issues & Gotchas

1. **Armor reduction limited to 65% max** — Only Tactical Helmet (20%) + Tactical Vest (45%) = 65% total in normal gameplay. Stacking reduction is additive, so theoretical cap depends on armor combinations available.

2. **Armor bypassed when shield active** — Players with active shield take full damage to shield, ignoring armor. Must deplete shield first for armor to protect.

3. **Perk armor gives perk immediately** — Equipping *NTK-11 Halycon* grants *Thermal Goggles* right away. Dropping it removes the perk. No intermediate state.

4. **Lower-tier armor cannot be equipped** — Game rejects pickup of same or lower-level armor. UI message: "Better item equipped" / "Item already equipped".

5. **Armor color not meaningful in-world** — Vest visual color is defined but HUD/server don't use it for filtering/identification (only level matters). Safe to ignore color in gameplay logic.

6. **Werewolf Fur hidden from HUD** — Special event armor has `hideInHUD: true`, so players can't see they lost it without looking at inventory.

7. **Armor level comparison only checks level** — Two vests with same level but different colors/perks will be treated as interchangeable. Level is the only upgrade path.

## Related Documents

### Tier 1
- [Architecture Overview](../../architecture.md) — System boundaries and component map
- [Data Model](../../datamodel.md) — Armor enum and armor state schema

### Tier 2
- [Inventory](../inventory/) — Equipment slot management, item equipping framework
- [Perks & Passive](../perks-passive/) — Perk system (armor perks like Thermal Goggles)
- [Object Definitions](../object-definitions/) — Armor registry and lookup
- [Game Loop](../game-loop/) — Damage calculation and armor reduction application
- [Game Objects (Server)](../game-objects-server/) — Player state and damage handling
- [Networking](../networking/) — Armor serialization in update packets
- [Client Rendering](../client-rendering/) — Armor visual on player sprite

### Tier 3
- [Module: Armor Pickup](modules/armor-pickup.md) — Armor interaction and equipping
- [Module: Damage Reduction](modules/damage-reduction.md) — Armor reduction formula and shield interaction
