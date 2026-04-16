# Weapon Customization Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/equipment-scope/README.md -->
<!-- @source: common/src/definitions/items/guns.ts, client/src/scripts/ui.ts -->

## Purpose
Manages slot-based weapon attachment customization, variant selection (single/dual), loadout persistence, and client-side preview of equipped firepower configurations.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| `common/src/definitions/items/guns.ts` | Gun definition structure with attachment slot metadata | High |
| `client/src/scripts/ui.ts` | Loadout UI rendering and customization state management | High |
| `client/src/scripts/managers/uiManager.ts` | Inventory display, equipment level indicators, HUD updates | Medium |
| `common/src/defaultInventory.ts` | Default loadout template for new players | Low |

## Business Rules

### Attachment Slots
- **Barrel slot:** One barrel variant per gun (silencer, muzzle brake, extended barrel)
  - Affects: bullet spread, fire rate, damage, muzzle velocity
  - Mutually exclusive with other barrels
- **Magazine slot:** Increases clip size and/or reload speed
  - Affects: ammo per magazine, reload duration
  - Stacks with perk modifiers (e.g., Extended Mags perk)
- **Scope slot:** Replaces default scope; affects zoom level and ADS time
  - Affects: magnification, accuracy degradation, iron sights fallback
  - Can be removed for iron sights (default scope)
- **Stock/Grip slot:** Affects recoil control and handling
  - Affects: recoil recovery, aim stability, weapon swap speed
- **Ammunition slot:** Configures ammo type (hollow-points, armor-piercing, flechette rounds)
  - Affects: damage falloff, armor penetration, ricochet behavior
  - Loaded via relevant perk definitions (e.g., `SabotRounds`, `Flechettes`)

### Variant Selection
- **Single vs. Dual:** Guns may support dual variant (two-weapon configuration)
  - Dual variant tracked via `dualVariant` reference in gun definition
  - Single variant field (`singleVariant`) points back to base gun
  - Dual weapons consume 2 primary slots in inventory but act as single entity
  - Damage per bullet: same as single, ROF: doubled
  - Loadout state: both variants share attachment configuration
- **Variant Switching:** Can switch between single/dual at any time (with cost)
  - Cost: action cooldown, requires free inventory space
  - Network: dual/single status sent in player update packet

### Loadout Persistence
- **Client storage:** Loadout cached in browser localStorage
  - Key: `suroi_loadout_<userId>` (if authed) or session-based
  - Persists across game sessions until cleared
- **Server validation:** Loadout validated on spawn:
  - All items exist and have correct definition type
  - No duplicate primary slots (2 primary guns not allowed)
  - Attachment chains are valid (all slots filled from available items)
- **Default fallback:** If loadout invalid, revert to `DEFAULT_INVENTORY` (@file `common/src/defaultInventory.ts`)

### Equipment Level Preview
- **Visual indicator:** Colored badge shows max equipment level for mode
  - Yellow-ish: max level (e.g., 3 by default, configurable per mode)
  - Tier indicators: loot rarity mapped to equipment level
- **Filtering:** UI may hide equipment above max level in store/loadout UI
  - Controlled by `mode.maxEquipmentLevel` in mode definition
  - Applies to backpacks, armor, armor plating, helmets, scopes

## Data Lineage

```
Player Loadout (UI State)
  ↓
[Validation: All items & references exist]
  ↓
Inventory Allocation [Equipment slots]
  ↓
[Per-slot processing: resolve attachments]
  ↓
Gun Instance Created (with final stats)
  ↓
UpdatePacket.playerData.weapon [sent to clients]
  ↓
Client Renders Gun in HUD / Hands
```

### Data Fields in Gun Definition

| Field | Type | Purpose | Example |
|-------|------|---------|---------|
| `idString` | string | Unique gun ID | `"ak47"` |
| `itemType` | ItemType.Gun | Object type | ItemType.Gun |
| `slots` | string[] | List of slot names this gun supports | `["barrel", "magazine", "scope"]` |
| `dualVariant?` | ReferenceTo<GunDefinition> | Link to dual version | `"ak47_dual"` |
| `singleVariant` | ReferenceTo<GunDefinition> | Link to base/single version | `"ak47"` |
| `baseAmmo` | ReferenceTo<AmmoDefinition> | Default ammo type | `"rifle_ammo"` |
| `statsOffset` | { [...] } | Modification to base stats for variant | `{ firingDelay: -2 }` |

## Dependencies

- **Internal:** 
  - Equipment & Scope — Scope definitions
  - Inventory — Inventory slot management
  - UI Management — Loadout UI components
- **External:** 
  - [Object Definitions](../../object-definitions/README.md) — Gun registry O(1) lookup via `Guns.tryGet(idString)`
  - [Serialization System](../../serialization-system/README.md) — Packet encoding of selected variants and attachments
  - [Networking](../../networking/README.md) — Player data structure in UpdatePacket

## Complex Functions

### Variant Resolution
**Function:** `resolveLoadoutVariants(loadout: Loadout): Loadout`  
**Source:** `client/src/scripts/ui.ts` (implied pattern)  
**Purpose:** Validate single/dual variant selection and resolve attachment chains

**Logic:**
1. Check if selected gun has `dualVariant` reference
2. If dual selected AND gun does NOT support dual → downgrade to single
3. If single selected AND gun supports dual → upgrade to dual (if user preference)
4. Validate all attachments exist and are valid for gun definition
5. Return resolved loadout or fall back to previous valid state

**Implicit behavior:**
- Does NOT persist to server; UI-only validation
- If dual variant unavailable → silently downgrades to single (no error shown)
- Attachment validation happens before variant resolution to catch mismatches

**Called by:**
- `updateLoadoutUI()` — on user interaction (dropdown change)
- `saveLoadout()` — before caching to localStorage
- `spawnPlayer()` — before sending selected loadout to server

### Stats Application
**Function:** `applyAttachmentModifiers(gun: GunDefinition, attachments: Attachment[]): FinalGunStats`  
**Source:** `common/src/definitions/items/guns.ts` or `server/src/objects/player.ts`  
**Purpose:** Compose final gun stats by layering attachment modifications

**Logic:**
```
baseStats = gun.stats
for each attachment in attachments:
  if attachment is barrel:
    baseStats.firingDelay += attachment.statsOffset.firingDelay
    baseStats.recoil *= attachment.statsOffset.recoil
  if attachment is magazine:
    baseStats.magSize += attachment.statsOffset.magSize
    baseStats.reloadTime -= attachment.statsOffset.reloadTime

return baseStats
```

**Implicit behavior:**
- Order matters: barrel → magazine → scope → stock/grip → ammo
- Some modifications are additive (+/-), some are multiplicative (×/÷)
- Does NOT apply perk modifiers (those happen in `applyPerkModifiers()`)
- Recursive: dual guns call this twice (once per gun copy)

**Called by:**
- `onEquipItem()` — when gun equipped to player
- `previewAttachmentStats()` — in UI tooltip on hover
- Packet serialization — to encode final stats in UpdatePacket

### loadout Validation
**Function:** `validateLoadout(loadout: Loadout): { valid: boolean; errors: string[] }`  
**Source:** `client/src/scripts/ui.ts` or `server/src/game.ts`  
**Purpose:** Check loadout can be equipped before spawn

**Validation checks:**
1. All guns exist in Guns registry
2. All attachments exist in respective definition registries
3. No loadout with 2 primary guns (max 1 active primary)
4. All attachment slots filled OR explicitly empty
5. Scope slot has scope definition or null (iron sights)
6. Equipment level ≤ mode.maxEquipmentLevel

**Errors if:**
- `guns.tryGet(id)` returns undefined → "Unknown gun"
- Attachment from undefined registry → "Unknown attachment"
- Dual variant requested but gun.dualVariant is undefined → "Gun doesn't support dual"
- Primary slot × 2 defined → "Multiple primary weapons not allowed"

**Called by:**
- `saveLoadout()` — before caching
- `spawnPlayer()` on server — before giving player items

## Configuration

| Setting | File | Effect | Default |
|---------|------|--------|---------|
| `maxEquipmentLevel` | `mode: ModeDefinition` | Highest equipment tier visible in UI | `3` |
| `defaultScope` | `mode: ModeDefinition` | Scope equipped on spawn if no loadout | `DEFAULT_SCOPE` |
| `dualVariant` | `gun: GunDefinition` | Whether dual version exists | undefined (single only) |
| `slots` | `gun: GunDefinition` | Which attachment slots gun supports | Varies per gun |

## Edge Cases & Gotchas

### Gotcha: Dual Variant Equipment Level
- **Issue:** Dual variant may have higher equipment level than single
  - User selects dual variant in loadout
  - Mode max level is 2 (lower than dual's level 3)
  - Loadout validation fails on spawn; falls back to DEFAULT_INVENTORY
- **Prevention:** Mode definitions should clamp max equipment level AFTER UI renders, not before validation

### Gotcha: Attachment Cascading
- **Issue:** Removing barrel removes attached silencer; silencer persists in UI state
- **Prevention:** When attachment slot cleared, clear all dependent attachments (cascade delete)

### Gotcha: Scope Default Fallback
- **Issue:** If selected scope doesn't exist in current mode definition, gun uses iron sights
  - Code: `gun.scope = gun.defaultScope ?? ironSights`
  - UI doesn't warn user scope was replaced
- **Prevention:** Scope selector in UI should only show scopes valid for current mode

### Gotcha: Persistent Loadout vs. Mode Change
- **Issue:** User saves loadout in "Normal" mode with FieldMedic perk, switches to "Halloween" mode
  - Halloween perks exclusive; FieldMedic not in Halloween mode definitions
  - Loadout still cached locally with invalid perk reference
  - Server rejects and falls back on spawn
- **Prevention:** Clear client loadout cache when mode changes, or server-side whitelist perks per mode

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Equipment & Scope subsystem overview
- **Tier 1:** [../../architecture.md](../../architecture.md) — System architecture
- **Patterns:** [../patterns.md](../patterns.md) — Attachment application patterns
- **Related modules:** 
  - [Perks & Passive](../../perks-passive/README.md) — Perk modifiers that affect gun stats
  - [Serialization System](../../serialization-system/modules/packet-encoding.md) — How loadouts encoded in UpdatePacket
