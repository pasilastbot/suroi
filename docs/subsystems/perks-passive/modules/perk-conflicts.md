# Perk Stacking & Conflicts Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/perks-passive/README.md -->
<!-- @source: common/src/definitions/items/perks.ts, server/src/objects/player.ts -->

## Purpose
Manages perk mutual exclusivity rules, stacking precedence, application order for overlapping effects, Easter eggs tied to perk combinations, and removal/cleansing of perks through healing items or mode transitions.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| `common/src/definitions/items/perks.ts` | Perk definition structure, categories, conflicts | High |
| `server/src/objects/player.ts` | Perk application, conflict resolution, removal handlers | High |
| `common/src/definitions/items/healingItems.ts` | Perk removal mechanics (cure infection, etc.) | Medium |
| `server/src/game.ts` | Mode-specific perk availability and Easter eggs | Medium |

## Business Rules

### Perk Categories & Mutual Exclusivity

Perks are organized by category, defining which perks conflict:

```
┌────────────────────────────────────────────────────────────┐
│  PerkCategories Enum                                       │
├────────────────┬──────────────────────────────────────────┤
│ Normal (0)     │ All positive/neutral perks (non-exclusive)│
│ Halloween (1)  │ Halloween-specific perks                 │
│ Hunted (2)     │ H.U.N.T.E.D. mode perks                  │
│ Infection (3)  │ Infection mode perks (mutually exclusive)│
└────────────────┴──────────────────────────────────────────┘
```

**Mutual exclusivity rules:**
- **Normal perks:** All can stack (no conflicts)
  - Example: `FieldMedic` + `AdvancedAthletics` both apply
- **Halloween perks:** Max 1 per player
  - If player gets new Halloween perk, old one removed
  - Example: Can't have both `Lycanthropy` and `Bloodthirst`
- **Hunted perks:** Max 1 per player
  - H.U.N.T.E.D. mode-specific
  - Conflict with other categories
- **Infection perks:** Max 1 per player, overrides all others
  - `Infected` perk is forced on infected players
  - `Immunity` is exclusive (can't have `Infected` + `Immunity`)

**Implicit rules:**
- Player obtains perk from loot, backpack, or obstacle (perk on destroy)
- If new perk conflicts with existing:
  - Existing perk **removed** (not stacked)
  - New perk **applied**
  - Player notified (visual/audio feedback)
- If player has 2 conflicting perks (glitch):
  - Dev logs warning
  - Removal order: older perk removed first

### Perk Application Order

When applying multiple perks, order matters (for overlapping stat modifiers):

```
1. Negative/Debuff perks first  (e.g., Infected applies negative mods)
   - MOD: Move speed -20%
   - MOD: Max HP -25%

2. Positive perks next            (e.g., FieldMedic, Berserker)
   - MOD: Revive speed +100%
   - MOD: Melee damage +30%

3. Utility/Conditional perks last (e.g., AdvancedAthletics, LootBaron)
   - MOD: Stamina regen +25%
   - MOD: Loot drop rate +10%
```

**Formula precedence:**
- Multiplicative mods (×) applied before additive mods (+)
  - `hpMultiplier = 0.75` (from Infected)
  - `hpAdditive = +50` (from other source)
  - Final HP = baseHP × 0.75 + 50
- Per-stat category: all mods for that stat batched
  - Example: movement speed mods
    - Base speed = 100
    - Mod 1: ×1.2 (AdvancedAthletics) → 120
    - Mod 2: ×0.8 (Infected) → 96
    - Mod 3: +10 (timed buff) → 106

### Perk Conflicts & Overrides

**Conflicting perk pairs:**

| Perk A | Perk B | Winner | Condition |
|--------|--------|--------|-----------|
| `Infected` | `Immunity` | Immunity | Applied last (higher precedence) |
| `Infected` | Normal perk | Infected | Infected overrides (game mode rule) |
| `Lycanthropy` | `Bloodthirst` | First obtained | Mutually exclusive (Halloween) |
| `LastStand` | `Infected` | Infected | Infected overrides in Infection mode |

**Resolution algorithm:**
```
if (newPerk.category === PerkCategories.Infection) {
  // Infection perk: remove all other perks
  removeAllPerks()
  applyPerk(newPerk)
} else if (newPerk.category === PerkCategories.Halloween) {
  // Halloween: remove other Halloween perks
  const existingHalloween = player.perks.find(p => p.category === PerkCategories.Halloween)
  if (existingHalloween) {
    removePerk(existingHalloween)
  }
  applyPerk(newPerk)
} else if (newPerk.category === PerkCategories.Hunted) {
  // Hunted: remove other Hunted perks
  const existingHunted = player.perks.find(p => p.category === PerkCategories.Hunted)
  if (existingHunted) {
    removePerk(existingHunted)
  }
  applyPerk(newPerk)
} else {
  // Normal: no conflict, just add
  applyPerk(newPerk)
}
```

### Perk Stat Modifications

Each perk applies a set of stat mods:

| Perk | Effect | Stat Change | Type |
|------|--------|-------------|------|
| `FieldMedic` | Faster revives | reviveTime ×0.5 | Multiplicative |
| `AdvancedAthletics` | Faster movement | moveSpeed ×1.2 | Multiplicative |
| `Berserker` | Melee damage boost | meleeDamage +30 | Additive |
| `Infected` | Status debuff | moveSpeed ×0.8, maxHP ×0.75 | Mixed |
| `Immunity` | Cures infection | Removes `Infected` perk | Removal |
| `CloseQuartersCombat` | Melee range extension | meleeRange +2 units | Additive |
| `ExtendedMags` | Magazine size | magazineSize +50% | Multiplicative |

### Special Perk Behaviors

**Mechanical vs. Passive:**
- `mechanical: true` — Perk has active behavior or cooldown
  - Example: `Overclocked` (resets throwable cooldown)
  - Requires server-side event handling
- `mechanical: false` — Perk is passive stat modifier
  - Automatically applied via `mapPerkOrDefault()` calls

**Perk Swap Restrictions:**
- `noSwap: true` — Can't intentionally swap this perk
  - Example: `Infected` (forced by game mode)
- `alwaysAllowSwap: true` — Can swap even if locked
  - Example: `Costumed` (swappable in Halloween)
- Default: Can swap if not in explicit no-swap list

**Mode Exclusions:**
- `plumpkinGambleIgnore: true` — Perk excluded from PlumpkinGamble results
  - PlumpkinGamble (random perk effect) skips certain perks
- `infectedEffectIgnore: true` — Perk unaffected by Infected mode debuff
  - Example: `EternalMagnetism` (not weakened by Infected)

### Easter Eggs & Combinations

**Notable combinations:**
- **`Berserker` + `CloseQuartersCombat`** → Melee dominator build
  - Damage: +30 (Berserker) + base
  - Range: +2 units (CloseQuartersCombat)
  - Combined effect: Most dangerous close-range player
- **`Infected` + `Lycanthropy`** → Super-infected (lore Easter egg)
  - Plays special announcer voice line: "Double-infected!"
  - Damage modifier bonus: +20% extra
  - Not balanced; gimmick/fun only
- **`PlumpkinGamble` + `PlumpkinShuffle`** → Perk roulette (Easter egg)
  - Every 30 seconds, random perk swap
  - Triggers special voiceover
  - Chaotic gameplay: fun for meme builds

### Perk Removal & Cleansing

**Removal triggers:**

| Trigger | Effect | Description |
|---------|--------|-------------|
| Perk conflict | Auto-remove | New perk replaces old |
| Healing item | Use Antidote | Remove Infection perk |
| Mode transition | Auto-remove | Perks invalid in new mode |
| Player death | Remove some | Infection removed, others persist |
| Plugin event | Custom | Custom perk removal logic |

**Healing item removal:**
- `healingItem.removePerk: PerkIds.Infected` — Antidote removes infection
- Consume item → scan player.perks → find matching perk → remove
- Only one perk removed per item (can't cure multiple perks)

**Mode transition (mode change between games):**
- Player switches from Infection mode → Normal mode
- All Infection perks removed automatically
- Normal perks **persist** (carry over)
- Player notified: "Perks cleared for new mode"

**Death handling:**
- `Infected` perk **removed** on death (cleaned up, not reincarnated)
- Halloween perks **persist** (ghost can have perk)
- Normal perks **persist** (no clear/cleansing)

## Data Lineage

```
Player obtains perk from loot/backpack
  ↓
[Check perk availability in current mode]
  ↓
[Check for conflicts with existing perks]
  ↓
[Remove conflicting perks (if any)]
  ↓
Apply new perk: player.perks.add(perkDef)
  ↓
Apply stat modifications: player.recalculateStats()
  ↓
Mark player dirty
  ↓
[Next UpdatePacket includes new perk state]
  ↓
Client receives UpdatePacket.playerData.perks
  ↓
Client renders perk UI indicator (icon + name)
  ↓
[Perk effects active for duration of match]
```

## Dependencies

- **Internal:**
  - Perks & Passive — Perk definition registry
  - Player object — Perk state tracking
  - Health & Damage — Stat modification application
- **External:**
  - [Object Definitions](../../object-definitions/README.md) — Perks registry O(1) lookup
  - [Game Modes](../../game-modes/README.md) — Mode-specific perk availability
  - [Networking](../../networking/README.md) — UpdatePacket.playerData.perks serialization
  - [Health & Damage](../../health-damage/README.md) — Stat recalculation after perk change

## Complex Functions

### Conflict Resolution
**Function:** `addPerk(player: Player, perkDef: PerkDefinition): void`  
**Source:** `server/src/objects/player.ts`  
**Purpose:** Add perk to player, removing conflicts

**Logic:**
```
1. Check perk available in current mode (filter by category)
2. If player already has this perk: return (no duplicate)

3. Determine conflicts based on category:
   - INFECTION: Remove ALL existing perks
   - HALLOWEEN: Remove other Halloween perks
   - HUNTED: Remove other Hunted perks
   - NORMAL: No inherent conflicts

4. For each conflicting perk:
   removePerk(player, conflictingPerk)

5. Add new perk:
   player.perks.add(perkDef)

6. Recalculate stats:
   player.recalculateStats()

7. Mark player dirty (trigger UpdatePacket)

8. Emit event: 'perk_added' (for plugins)
```

**Implicit behavior:**
- Conflict check happens BEFORE removal (prevents race condition)
- Old perk removed silently (no animation or sound)
- Stats recalculated atomically (no intermediate invalid state)
- Player may receive UI notification (tooltip, chat message)

**Called by:**
- Loot collection — when picking up perk item
- Backpack equip — when equipping backpack with perk
- Obstacle destruction — when breaking obstacle with perk on destroy
- Mode trigger — when mode applies forced perk (e.g., Infection mode)

### Stat Recalculation
**Function:** `recalculateStats(player: Player): void`  
**Source:** `server/src/objects/player.ts`  
**Purpose:** Recompute all player stats from base + perk mods

**Logic:**
```
// Reset to base stats
let stats = JSON.parse(JSON.stringify(BASE_PLAYER_STATS))

// Apply each perk's mods in order
for (const perk of player.perks) {
  // Negative perks first
  if (perk.quality === PerkQualities.Negative) {
    for (const mod of perk.statMods) {
      applyMod(stats, mod)
    }
  }
}

for (const perk of player.perks) {
  // Positive perks
  if (perk.quality === PerkQualities.Positive) {
    for (const mod of perk.statMods) {
      applyMod(stats, mod)
    }
  }
}

// Assignment
player.moveSpeed = stats.moveSpeed
player.maxHealth = stats.maxHealth
player.reviveTime = stats.reviveTime
// ... etc for all stats
```

**Implicit behavior:**
- Order: negative → positive → utility
- Multiplicative mods batched separately from additive
- No clamping (can result in 0 or negative stats if modifiers extreme)
- Called after EVERY perk change (expensive, should be optimized)

**Called by:**
- `addPerk()` — after adding new perk
- `removePerk()` — after removing perk
- Mode selection — when spawning in mode with default perks
- Plugin event — plugins might call to force recompute

### Perk Removal
**Function:** `removePerk(player: Player, perkDef: PerkDefinition): void`  
**Source:** `server/src/objects/player.ts`  
**Purpose:** Remove perk and recompute stats

**Logic:**
```
const index = player.perks.findIndex(p => p.idString === perkDef.idString)
if (index === -1) return  // perk not found

// Remove from collection
player.perks.splice(index, 1)

// Undo perk-specific behaviors (if any)
if (perkDef.idString === "infected") {
  player.setAnimation(AnimationType.None)  // stop infection animation
  player.recoveryTime = 0
}

// Recompute stats
player.recalculateStats()

// Mark dirty
player.setDirty()

// Emit event for plugins
game.pluginManager.emit('perk_removed', { player, perk: perkDef })
```

**Implicit behavior:**
- Removal is permanent (not temporary debuff)
- Stats recalc happens immediately (no delay)
- Perk-specific cleanup (e.g., removal of Infected visual state)
- No notification sent to player (silent removal)

**Called by:**
- `addPerk()` — during conflict resolution
- Healing item use — when antidote cure infection
- Mode transition — when mode change clears invalid perks
- Death handler — when player dies (Infection removed)

## Configuration

| Setting | Location | Effect | Default |
|---------|----------|--------|---------|
| `PerkCategories` | enum | Perk conflict categories | N/A (enum) |
| `PerkQualities` | enum | Perk effect type (positive/negative/neutral) | N/A (enum) |
| `noSwap` | perk definition | Can't swap this perk | false |
| `alwaysAllowSwap` | perk definition | Can swap even if locked | false |
| `mechanical` | perk definition | Active behavior (not passive stat mod) | false |
| `infectedEffectIgnore` | perk definition | Unaffected by Infected debuff | false |

## Edge Cases & Gotchas

### Gotcha: Negative Perk Stacking
- **Issue:** Multiple negative perks stack, resulting in unplayable character
  - Player has `Infected` (move speed ×0.8) + `Claustrophobic` (move speed ×0.7)
  - Final move speed = base × 0.8 × 0.7 = 56% of normal
  - Character moves very slowly; unintended difficulty spike
- **Prevention:** Design negative perk combinations to be playable; hard-cap final stat values

### Gotcha: Easter Egg Stat Bonus Missing
- **Issue:** `Infected` + `Lycanthropy` combo should grant +20% damage
  - Code checks for perk presence but stat bonus not applied
  - Bonus intended by design but forgotten in recalc
- **Prevention:** Document Easter egg logic explicitly; add test case for combo

### Gotcha: Perk Persistence Across Respawn
- **Issue:** Player dies with `FieldMedic` perk, respawns, still has it
  - Design intention: some perks should clear on death
  - `FieldMedic` should NOT persist (utility perk, not tied to player state)
  - Allows infinite respawn revivals (unbalanced)
- **Prevention:** Define `clearOnDeath: true` flag in perk definitions; check on respawn

### Gotcha: Healing Item Removes Wrong Perk
- **Issue:** Antidote item has `removePerk: PerkIds.Infected`
  - Player has both `Infected` + `Immunity` (should be impossible; conflict prevents this)
  - Antidote removes `Infection`; but `Immunity` now invalid without `Infected`
  - State becomes inconsistent
- **Prevention:** Validate conflict resolution to prevent invalid combinations

### Gotcha: Mode Change Perk Teleport
- **Issue:** Player in Infection mode has 5 perks
  - Mode changes to Normal (Infection perks cleared)
  - UI still shows 5 perk icons, only 1 or 2 are valid
  - Visual mismatch; client confusion
- **Prevention:** Clear perk UI on mode transition; reload from UpdatePacket

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Perks & Passive subsystem overview
- **Tier 1:** [../../architecture.md](../../architecture.md) — System architecture
- **Patterns:** [../patterns.md](../patterns.md) — Perk application patterns
- **Related modules:**
  - [Game Modes](../../game-modes/modules/mode-rules.md) — Mode-specific perk availability
  - [Health & Damage](../../health-damage/README.md) — Stat modification and recalculation
  - [Game Objects Server](../../game-objects-server/README.md) — Player object lifecycle
