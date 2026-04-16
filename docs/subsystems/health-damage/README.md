# Health & Damage System

<!-- @tier: 2 -->
<!-- @parent: ../../architecture.md -->
<!-- @modules: modules/ -->
<!-- @source: server/src/objects/player.ts, server/src/game.ts -->

## Purpose

The Health & Damage subsystem manages player health state, damage application, armor/shield mitigation, and kill detection. It handles the complete pipeline from damage source (bullets, explosions, environment) through armor reduction, shield absorption, health deduction, and kill packet generation. This subsystem is critical to core gameplay balance and stat tracking.

## Key Files & Entry Points

| File | Purpose |
|------|---------|
| `server/src/objects/player.ts` | Player health properties, `damage()` / `piercingDamage()` methods, armor reduction, death/down state, kill credit logic |
| `server/src/objects/gameObject.ts` | `DamageParams` interface, abstract `damage()` method |
| `server/src/objects/bullet.ts` | `DamageRecord` interface, per-bullet damage queuing |
| `server/src/game.ts` | Deferred damage application loop, `DamageRecord` batch processing, kill leader tracking |
| `common/src/packets/killPacket.ts` | `KillPacket` structure, `DamageSources` enum, kill credit serialization |
| `common/src/constants.ts` | `GameConstants.player` — max health, max shield, default modifiers |
| `common/src/definitions/items/armors.ts` | Armor definitions with `damageReduction` values (helmet/vest) |

## Architecture

The damage system follows a **deferred batch processing model** to ensure client-server consistency:

```
┌─────────────────────────────────────────────────────┐
│ PHASE 1: Bullet Collision Detection (per bullet)    │
│ Each bullet checks hitbox collisions, queues         │
│ DamageRecord { object, damage, weapon, source }     │
└─────────────────────────┬───────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────┐
│ PHASE 2: Deferred Damage Application (per tick)     │
│ Game.tick() processes ALL DamageRecords in batch    │
│ Why: All pellets (shotgun) hit before any dies      │
└─────────────────────────┬───────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────┐
│ PHASE 3: Calculation & Reduction                    │
│ Player.damage(params) → armor/perk mods applied     │
│ → Player.piercingDamage(params) → shield/health     │
└─────────────────────────┬───────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────┐
│ PHASE 4: Health ≤ 0 Detection                       │
│ If health ≤ 0: down() [team mode] or die() [final]  │
│ Emits KillPacket to all players                     │
└─────────────────────────┬───────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────┐
│ PHASE 5: Network Sync (next UpdatePacket)           │
│ KillPacket broadcast, health/shield serialized      │
│ Client applies visuals, plays death sound, etc.     │
└─────────────────────────────────────────────────────┘
```

## Data Flow

### Input: Damage Source
```
Gun bullet → Explosion → Melee swing → Grenade → Gas tick → Fall damage
  ↓
DamageRecord enqueued (damage, weapon, source, target, position)
```

### Processing Pipeline
```
DamageRecord → Player.damage() [armor reduction] → Player.piercingDamage() [shield/health]
  ↓
If health ≤ 0:
  ├─ Team mode + teammates alive: down(source, weapon)
  └─ Otherwise: die(source, weapon)
  ↓
KillPacket created + broadcast
```

### Output: Health Serialization
```
UpdatePacket {
  health: normalizedHealth,  // 0-1 (byte encoded)
  maxHealth: number,
  shield: normalizedShield,  // 0-1 (byte encoded)
  maxShield: number
}

KillPacket {
  victimId: number,
  kills: number,
  damageSource: DamageSources enum,
  attackerId?: number,
  creditedId?: number,  // (may differ from attacker in down scenarios)
  weaponUsed?: WeaponDefinition
}
```

## Interfaces & Contracts

### DamageParams Interface
```typescript
// server/src/objects/gameObject.ts:37-43
interface DamageParams {
    readonly amount: number;
    readonly source?: GameObject | DamageSources.Gas | DamageSources.Obstacle 
                     | DamageSources.BleedOut | DamageSources.FinallyKilled 
                     | DamageSources.Disconnect;
    readonly weaponUsed?: InventoryItem | Explosion | Obstacle;
    readonly position?: Vector;  // Optional: impact position for visuals
}
```

### DamageRecord Interface (Internal)
```typescript
// server/src/objects/bullet.ts:20-25
interface DamageRecord {
    readonly object: Obstacle | Building | Player;
    readonly damage: number;
    readonly weapon: GunItem | Explosion;
    readonly source: GameObject;
    readonly position: Vector;
}
```

### Key Methods

| Method | Signature | Purpose |
|--------|-----------|---------|
| `Player.damage()` | `damage(params: DamageParams): void` | Apply armor reduction → call `piercingDamage()` |
| `Player.piercingDamage()` | `piercingDamage(params: DamageParams): void` | Core damage application (shield/health deduction, death detection) |
| `Player.die()` | `die(params: Omit<DamageParams, "amount">): void` | Enter death state, emit KillPacket, credit killer |
| `Player.down()` | `down(source?, weaponUsed?): void` | Enter down state (team mode), revivable for 10 sec |
| `Player.heal()` | `heal(amount: number): void` | Increase health (healing items) |

## Dependencies

### Depends On
- **Perks & Passive System** (`docs/subsystems/perks-passive/`) — Perk modifiers for health, damage reduction, regen, kill effects
- **Body Armor & Equipment** (`docs/subsystems/body-armor/`) — Armor definitions with `damageReduction` %
- **Inventory System** (`docs/subsystems/inventory/`) — Access equipped armor, weapons for stat tracking
- **Game Loop** (`docs/subsystems/game-loop/`) — Tick processing, timeout management for down state
- **Networking** (`docs/subsystems/networking/`) — `KillPacket` serialization, `UpdatePacket` health encoding
- **Team System** (`docs/subsystems/team-system/`) — Team damage blocking, down state rules
- **Plugin System** — Event hooks (`player_damage`, `player_will_piercing_damaged`, `player_did_piercing_damaged`, `player_will_die`)

### Depended On By
- **Client Rendering** — Displays health/shield bars, plays death animations
- **Stats & Leaderboards** — Uses kill count, damage done/taken, kill leader
- **Game Loop** — Reads dead/downed state to update alive count, process spectators
- **Projectiles & Ballistics** — Generates `DamageRecord`s from bullets/explosions

## Module Index (Tier 3)

For implementation details, see:
- [Damage Pipeline Module](modules/damage-pipeline.md) — Complete damage formula, armor reduction, shield, kill detection, kill credit assignment

## Related Documents

### Tier 1
- [System Architecture](../../architecture.md) — Server-client structure, tick model
- [Data Model](../../datamodel.md) — Player entity, health/shield properties

### Tier 2 Subsystems
- [Body Armor Subsystem](../body-armor/README.md) — Armor definitions, mitigation percentages
- [Perks & Passive System](../perks-passive/README.md) — Perk effects: health mods, damage reductions, kill effects
- [Team System](../team-system/README.md) — Down state rules, team damage prevention, revive mechanics
- [Game Loop](../game-loop/README.md) — Tick processing, damage event loop integration

### Tier 3 Modules
- [Armor Mitigation](../body-armor/modules/armor-mitigation.md) — Armor reduction formula details
- [Perk Effects](../perks-passive/modules/perk-effects.md) — Perk triggers on damage/kill
- [Game Loop — Tick](../game-loop/modules/tick.md) — Damage application phase

## Patterns

### Pattern: Armor Reduction (Shield-Aware)

**When to use:** Calculating incoming damage to player with armor equipped.

**Implementation:**
1. Check if shield > 0
   - **If yes**: Bypass armor entirely, apply to shield first
   - **If no**: Apply armor reduction formula
2. Formula: `damage *= (1 - (helmet% + vest%))`
3. Call `piercingDamage()` with reduced amount

**Example:** 40 damage, Level 2 Vest (20% reduction), shield > 0:
- Shield absorbs full 40 damage (armor ignored)

**Example:** 40 damage, Level 2 Vest (20% reduction), shield = 0:
- Health loses ~32 damage (40 × 0.8)

### Pattern: Shield Overflow Recursion

**When to use:** Shield breaks mid-damage, remaining damage must hit health.

**Implementation:**
1. Deduct damage from shield: `shield -= amount`
2. If shield < 0 (broke):
   - Capture overflow: `remaining = Math.abs(shield)`
   - Reset shield to 0
   - **Recursively call** `damage({ ...params, amount: remaining })`
   - This re-applies armor reduction to overflow (intentional)

**Example:** Shield 20 HP, 50 damage, vest 20% reduction:
- Shield absorbs 20 → overflow 30
- Recursive call: 30 × 0.8 = 24 health damage
- Total health loss: 24 HP (not 30)

### Pattern: Deferred Damage Application

**When to use:** Batch damage from multiple bullets (shotgun, etc.) in same tick.

**Implementation:**
1. During bullet update: queue `DamageRecord[]` (don't apply yet)
2. After all bullets updated: iterate `DamageRecord[]`
3. Call `object.damage()` for each record
4. This ensures **shotgun pellets all land before target dies**

**Effect:**
- Client and server agree: all 8 pellets hit before death
- Without deferral: server kills target mid-pellets, client still rendering hits

### Pattern: Kill Credit Assignment (Downer vs Killer)

**When to use:** Determining who gets kill credit in team mode.

**Implementation:**
1. If death source is **player**:
   - Check if victim was downed by a teammate
   - **If yes**: that teammate gets credit (if they're on killer's team)
   - **If no**: killer gets credit
2. If death source is **environment** (gas, timeout, etc.):
   - Credit goes to player who downed the victim
   - If no downer: no credit

**Example:** Team A (Alice + Bob) vs Team B (Charlie)
- Bob shoots Alice, Alice downed
- Gas kills Alice
- **Alice's killer**: Bob (down credit) not Charlie (gas damage)

## Configuration

| Setting | Effect | Source |
|---------|--------|--------|
| `GameConstants.player.defaultHealth` | Max health per player | `common/src/constants.ts` |
| `GameConstants.player.maxShield` | Max shield (0 by default, perk-based) | `common/src/constants.ts` |
| `ArmorDefinition.damageReduction` | % damage blocked per armor | `common/src/definitions/items/armors.ts` |
| `PerkDefinition.damageReceivedMod` | Damage multiplier (e.g., LastStand perk) | `common/src/definitions/items/perks.ts` |
| `PerkDefinition.shieldRegenRate` | Shield regen per tick (e.g., ExperimentalForcefield) | Perk definitions |

## Known Issues & Gotchas

1. **Double Armor Reduction on Shield Break**: When shield breaks and remaining damage is applied recursively, armor is re-applied a second time. This makes shield + armor weaker than expected.

2. **Armor Only Works Without Shield**: As long as shield > 0, armor is completely bypassed. This asymmetry is intentional but can confuse players.

3. **Team Damage Completely Blocked**: In team modes, friendly fire equals zero damage (hard-blocked). There's no "reduced friendly fire" option.

4. **Late Kill Credit in Down State**: Kill credit is assigned when victim dies, not when they're downed. In team mode, if a downed teammate dies from timeout 10 seconds later, the down player's killer *still* gets credit.

5. **Bleed/Poison Not Fully Implemented**: Status effects are tracked via `lastDamagedBy` but there's no persistent DoT (damage-over-time) system. Damage is instantaneous.

6. **Invulnerability Frame Timing**: Player's `invulnerable` flag doesn't auto-expire; it's managed externally by other systems.

