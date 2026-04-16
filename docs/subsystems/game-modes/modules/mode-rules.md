# Mode-Specific Rules Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/game-modes/README.md -->
<!-- @source: common/src/definitions/modes.ts, server/src/game.ts -->

## Purpose
Documents mode-specific gameplay rules and mechanics for Halloween, Infection, Fall (gravity), and other seasonal/special modes, including bunker unlocking, costume costumes, chasing mechanics, stage transitions, and mode-specific perk rules.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| `common/src/definitions/modes.ts` | ModeDefinition interface with mode-specific configuration fields | High |
| `server/src/game.ts` | Game mode initialization, stage progression, mode-specific logic | High |
| `common/src/definitions/items/perks.ts` | Mode-specific perk categories (Halloween, Infection, Hunted) | Medium |
| `common/src/definitions/obstacles.ts` | Bunker doors, unlock conditions, perk on destroy (Infected) | Medium |
| `server/src/data/gameModes.ts` (if exists) | Mode definitions and configuration data | High |

## Business Rules

### Halloween Mode

**Overview:** Seasonal mode with bunker progression, costume cosmetics, and special Halloween perks.

**Progression system:**
- **Stages:** Game progresses through unlockable stages (0 → 1 → 2 → 3)
  - Stage 0: Game start, bunker doors locked
  - Stage 1: After gas stage 1 shrink, bunker doors unlock
  - Stage 2: After gas stage 2 shrink, special costumes available
  - Stage 3: Final shrink, ultimate perks available
- **Unlock mechanism:** `mode.unlockStage` field in ModeDefinition
  - When `game.stage >= mode.unlockStage`, non-locked doors auto-unlock
  - No player interaction required (doors silently unlock)
- **Bunker access:** Locked doors (`door.locked = true`) block entry until stage reached
  - Prevents early game rushing (balanced progression)
  - Doors checked in `Obstacle.onInteract()` before allowing opening

**Costume system:**
- **Costume perks:** Special perks awarded in Halloween mode
  - Perks like `Costumed` modify player appearance
  - Applied when loot dropped or obstacle destroyed
  - Player wears costume (visual model change) for remainder of game
- **Exclusive perks:** Halloween perks category
  - Only available in Halloween mode
  - Mutually exclusive (can't have 2 Halloween perks)
  - Examples: `Lycanthropy`, `Bloodthirst`, `PlumpkinBomb`, `Claustrophobic`
- **Perk conflicts:** New Halloween perk removes old Halloween perk
  - `addPerk()` detects category conflict
  - Old perk replaced automatically

**Environment:**
- **Sprite override:** Halloween spritesheet with themed decorations
  - Trees look spooky, terrain colors adjusted
  - `mode.spriteSheets = ["halloween", "shared"]` (load halloween first)
- **Canvas filters:** Brightness/saturation reduced
  - `mode.canvasFilters = { brightness: 0.8, saturation: 0.7 }`
  - UI/overlay has Halloween tint
- **Particle effects:** Falling leaves or fog
  - `mode.particleEffects = { frames: "leaf", delay: 100, gravity: true }`

**Special mechanics:**
- **Plumpkin items:** Halloweening-themed loot that triggers perk effects
  - PlumpkinGamble: get random perk
  - PlumpkinShuffle: swap perks periodically
  - PlumpkinBomb: explosive effect (lore/flavor)
- **Menu music:** Special Halloween music plays in lobby
  - `mode.replaceMenuMusic = true` → loads `menu_music_halloween.mp3`

### Infection Mode

**Overview:** Competitive asymmetric mode where infected players spread infection, can't use guns, and must chase down others.

**Infection mechanics:**
- **Infected state:** Player has `Infected` perk applied
  - Infected perk forces: move speed ×0.8, max HP ×0.75, no primary guns
  - Can only use melee weapons and healing items
  - Can be revived (if teammate) but remains infected
- **Spread mechanism:** Melee damage from infected player spreads infection
  - On hit: apply `Infected` perk to target
  - Target becomes infected (green tint applied)
  - Target's gun hidden; can't switch to primary weapons
  - Conversion is permanent (until cured)
- **Cure:** Antidote healing item removes infection
  - Use item → `Infected` perk removed
  - Player regains gun access
  - Can be re-infected by infected teammate
- **Immunity perk:** `Immunity` perk blocks infection
  - Rare drop in Infection mode
  - `Infected` perk cannot be applied if `Immunity` present
  - Mutual exclusivity: can't have both
- **Mode override:** Infection perk cannot be swapped out
  - `noSwap: true` in Infected perk definition
  - Even if player wants to drop infection, can't (forced)

**Chaos mechanics:**
- **Infection spreads:** More players infected → fewer uninfected
  - Early game: 1 infected vs. many uninfected
  - Mid-game: infection spreads exponentially
  - Late game: few remaining uninfected hunted
- **Visual indicators:** Infected players glow red/green
  - Client shows infected status on HUD
  - Allows uninfected to coordinate escapes
- **Chat notifications:** "Player X became infected!"
  - Alerts team; may trigger panic or coordinated response

**Perk rules:**
- **Infection perks only:** Mode-specific perks
  - `Infected`, `Necrosis`, `Immunity` in Infection category
  - Normal perks NOT available (filtered on loot)
  - Halloween perks disabled
- **Perk sources:** Obstacles and special crates spawn infection perks
  - Destroyed obstacle: apply perk from definition (`applyPerkOnDestroy`)
  - Special crates: guarantee Immunity or Necrosis

### Fall Mode (Gravity-Enabled)

**Overview:** Experimental/seasonal mode with gravity, falling damage, and verticality.

**Gravity mechanics:**
- **Falling damage:** Players take damage when landing from height
  - Damage = fall distance × coefficient (e.g., 1 damage per 2 units dropped)
  - Minimum threshold: 8 units to trigger damage
  - Caps at max damage (e.g., 50 HP max from fall)
- **Knockback interaction:** Knockback launches player upward; gravity pulls down
  - Knockback + Fall = can use knockback for height/escape
  - Damage on landing increased if knocked from higher
- **Platform design:** Map may have verticality (cliffs, structures)
  - Buildings have rooftops
  - Falling off rooftop triggers fall damage
  - Strategic high-ground spots

**Mode flags:**
- `overrideUpstairsFunctionality: true` — stairs act differently in Fall mode
  - Some stairs may be disabled or behave as ramps
  - Falling from stairs still triggers damage
- Particle effects: Rain or falling debris
  - `particleEffects = { frames: "rain", gravity: true }`

### Hunted Mode (H.U.N.T.E.D.)

**Overview:** Competitive asymmetric mode with hunters vs. hunted, bunker progression, and stage-specific unlocks.

**Progression:**
- **Team assignment:** Players spawn as hunters or hunted
  - Hunters: well-equipped, can leave bunker
  - Hunted: worse equipment, locked in bunker until stage unlock
- **Stage unlocking:** Gates/doors unlock at stages
  - `mode.unlockStage` field in ModeDefinition
  - Stage 1: Bunker door opens → hunted can escape
  - Stage 2: Secondary exits unlock
  - Stage 3: Final areas accessible
- **Bunker mechanics:** Special obstacle property for bunker
  - Doors: `operationStyle: "bunker"`
  - These doors linked to `mode.unlockStage` progression
  - Players WITH `APKey` item can unlock early
  - Players can also be "key" (temporary unlock as infected?)

**Perk rules:**
- **Hunted perks category:** Mode-specific perks
  - Mutually exclusive (one Hunted perk per player)
  - Different from Infection/Halloween
  - Examples: speed boosts, stealth, tracking awareness

### Fall Mode (Gravity System)

**Vertical gameplay:**
- **Falling damage calculation:** `damage = (fallHeight - THRESHOLD) / DIVISOR`
  - THRESHOLD: 8 units before damage (~1 player height)
  - DIVISOR: 2 units per damage point (8 units drop = 0 dmg, 10 units = 1 dmg)
- **Terminal velocity:** Players have max fall speed
  - Prevents infinite acceleration (unrealistic physics)
  - Capped at ~50 units/sec downward
- **Jump physics:** Players can jump with limited height
  - Jump height: ~4–8 units (height of wall)
  - Knockback can catapult player much higher
  - Knockback + Fall mode = high tactics

## Data Lineage

```
Game initialized with mode name (e.g., "halloween")
  ↓
Mode definition loaded: Modes[modeName]
  ↓
[Extract mode properties: stages, perks, rules, etc.]
  ↓
Game spawn phase:
  - Apply mode-specific environment (spritesheet, filters)
  - Unlock doors per unlockStage
  - Filter loot table (Halloween: only Halloween perks; Infection: only Infection perks)
  ↓
Game loop each tick:
  - Check stage progression (gas stage → game stage)
  - If stage >= unlockStage: auto-unlock applicable doors
  - Apply mode-specific physics (Fall mode: gravity)
  - Update player state (Infection: spread perk if melee hit)
  ↓
[Game continues with mode rules active until victory]
```

## Dependencies

- **Internal:**
  - Game Modes — Mode registry and definitions
  - Perks & Passive — Mode-specific perk categories
  - Buildings & Props — Bunker doors and unlock conditions
- **External:**
  - [Object Definitions](../../object-definitions/README.md) — Modes registry O(1) lookup
  - [Gas System](../../gas/README.md) — Stage progression triggers
  - [Perks & Passive](../../perks-passive/modules/perk-conflicts.md) — Mode-specific perk availability
  - [Game Loop](../../game-loop/README.md) — Tick-based stage updates
  - [Networking](../../networking/README.md) — ModeDefinition in game init packet

## Complex Functions

### Stage Progression Handler
**Function:** `updateGameStage(game: Game): void`  
**Source:** `server/src/game.ts` (implied pattern)  
**Purpose:** Advance game stage based on gas shrink progression

**Logic:**
```
// Determine next stage based on gas stages
const currentGasStage = game.gas.getCurrentStage()
const newGameStage = min(currentGasStage, game.mode.unlockStage ?? Infinity)

if (newGameStage > game.stage) {
  game.stage = newGameStage
  
  // Auto-unlock doors for this stage
  for (const obstacle of game.obstacles) {
    if (obstacle.isDoor && obstacle.definition.unlockableWithStage) {
      if (obstacle.door.unlockStage <= game.stage && obstacle.door.locked) {
        obstacle.door.locked = false
        obstacle.setDirty()
      }
    }
  }
  
  emit('stage_advanced', { game, newStage: game.stage })
}
```

**Implicit behavior:**
- Stage advances automatically (tied to gas progression)
- Doors unlock silently (no animation or notification)
- All applicable doors unlock simultaneously
- Events may trigger special behaviors (plugins can listen)

**Called by:**
- Game tick handler — after updating gas state
- Frequency: ~once per 10–20 seconds (gas stage changes infrequently)

### Infection Spread
**Function:** `spreadInfection(infected: Player, damaged: Player): void`  
**Source:** `server/src/objects/player.ts` (implied, called from damage handler)  
**Purpose:** Apply Infected perk to damaged player if attacker is infected

**Logic:**
```
if (!game.mode.idString === "infection") return  // only in Infection mode

if (infected.perks.find(p => p.idString === "infected") && !infected.downed) {
  if (damaged.perks.find(p => p.idString === "immunity")) {
    // Immunity blocks infection
    return
  }
  
  // Apply infection
  game.addPerk(damaged, Perks.get("infected"))
  
  // Notify
  game.chat.broadcast(`${damaged.name} became infected!`)
}
```

**Implicit behavior:**
- Only applies in Infection mode
- Immunity perk blocks spread (no override)
- Spread happens on melee hit (gun shots don't spread)
- Notification visible to all players
- No cooldown (each hit can infect)

**Called by:**
- Damage handler — when infected player hits with melee weapon
- Condition: attacker infected AND victim doesn't have immunity

### Mode Transition
**Function:** `transitionMode(oldMode: ModeName, newMode: ModeName, game: Game): void`  
**Source:** `server/src/game.ts` (implied pattern)  
**Purpose:** Clear mode-specific state when game mode changes

**Logic:**
```
// Clear mode-specific perks for all players
for (const player of game.players) {
  // Remove perks from old mode
  player.perks = player.perks.filter(p => {
    const perkValid = isPerkValidInMode(p.category, newMode)
    if (!perkValid) {
      player.removePerk(p)
    }
    return perkValid
  })
  
  // Recalculate stats
  player.recalculateStats()
  
  // Reset animations (esp. Infection visual)
  player.animation = AnimationType.None
}

// Update UI/environment
game.mode = Modes[newMode]
game.spriteSheet = loadSpriteSheet(game.mode.spriteSheets)
game.handleCanvasFilters(game.mode.canvasFilters)
game.stage = 0  // reset stage
```

**Implicit behavior:**
- Old mode perks removed (one-way conversion)
- Stats recalculated for all players
- Stage reset to 0 (new mode starts fresh)
- Local cache cleared on clients
- No gradual fadeout (immediate visual change)

**Called by:**
- Game rotation/schedule — when map/mode changes between games
- Frequency: Once per game (at game end)

## Configuration

| Setting | File | Effect | Examples |
|---------|------|--------|----------|
| `unlockStage` | mode definition | Stage number for bunker unlock | 1, 2, 3 |
| `forcedGoldAirdropStage` | mode definition | Stage for golden airdrop drop | 2 |
| `maxEquipmentLevel` | mode definition | Highest equipment tier available | 2 or 3 |
| `overrideUpstairsFunctionality` | mode definition | Disable stairs physics | true/false |
| `particleEffects` | mode definition | Ambient particles (rain, leaves) | { frames, delay, gravity } |
| `canvasFilters` | mode definition | Screen brightness/saturation | { brightness, saturation } |
| `spriteSheets` | mode definition | Loaded sprite sheets | ["halloween", "shared"] |

## Edge Cases & Gotchas

### Gotcha: Bunker Door Unlock Race
- **Issue:** Stage advances; game tries to unlock all bunker doors
  - Door unlock loop iterates all obstacles
  - Obstacle from new stage not yet spawned
  - Unlock check passes (obstacle doesn't exist)
  - But obstacle spawns 1 tick later, still locked
- **Prevention:** Deferred unlock (mark for unlock, process after obstacle spawn); or scan after spawn phase

### Gotcha: Infection Perk Swap Attempt
- **Issue:** Infected player tries to drop Infected perk via UI
  - `Infected.noSwap = true` prevents swap
  - But check happens client-side (can be bypassed)
  - Server receives "drop perk" request
  - Server double-checks, prevents drop
  - Player frustrated (unclear why can't drop)
- **Prevention:** Clear error message: "This perk can't be dropped in this mode"

### Gotcha: Mode Perk Category Mismatch
- **Issue:** Player in Halloween mode has Infection perk (cross-mode glitch)
  - Mode transitions; game tries to filter perks
  - Infection perk wasn't supposed to spawn in Halloween
  - Filter removes it
  - Player loses perk unexpectedly
- **Prevention:** Validate loot tables per mode; prevent perk cross-contamination during initialization

### Gotcha: Stage Unlock Timing
- **Issue:** Gas stage advances; bunker door unlock queued
  - Player opens door before unlock processed
  - Door.locked still true; interaction fails
  - 1 tick later, door unlocks automatically
  - Timing confusion for player
- **Prevention:** Auto-unlock BEFORE player interactions; process unlock at START of tick, not end

### Gotcha: Fall Mode Falling Through Obstacles
- **Issue:** Gravity enabled; player falls through solid objects
  - Physics engine bug: object collision not updated for falling
  - Player clips through obstacle
  - Game assumes player is falling; applies fall damage
  - Damage appears to come from nowhere
- **Prevention:** Grid-based collision check for falling entities; prevent clipping

### Gotcha: Multi-Mode Costume Persistence
- **Issue:** Player gets Costume perk in Halloween mode
  - Mode rotates; next game is Normal mode
  - Costume perk removed (invalid in Normal)
  - Client still renders costume sprite (cache not cleared)
  - Visual bug: Normal mode player w/ Halloween costume
- **Prevention:** Clear sprite cache on mode transition; force re-render all players

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Game Modes subsystem overview
- **Tier 1:** [../../architecture.md](../../architecture.md) — System architecture
- **Patterns:** [../patterns.md](../patterns.md) — Mode configuration and progression patterns
- **Related modules:**
  - [Gas System](../../gas/README.md) — Stage progression and shrinking mechanics
  - [Perks & Passive](../../perks-passive/modules/perk-conflicts.md) — Mode-specific perk availability
  - [Buildings & Props](../../buildings-props/modules/door-mechanics.md) — Bunker unlock mechanics
  - [Game Loop](../../game-loop/README.md) — Tick-based game progression
