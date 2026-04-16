# Status Effects & Conditions — Effect Application Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/status-effects/README.md -->
<!-- @source: server/src/objects/player.ts | server/src/objects/gameObject.ts -->

## Purpose

Implements persistent status conditions (bleed, infection, speed modifiers) with per-tick damage/healing, visual indicators, and duration management.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| `server/src/objects/player.ts` | Status effect application, infection tracking, perk-based effects | High |
| `server/src/objects/gameObject.ts` | Base damage infrastructure; damage source types | Medium |
| `common/src/definitions/items/perks.ts` | Perk definitions with effect properties | Medium |
| `common/src/constants.ts` | Game constants (bleed-out DPM, tick rates) | Low |

## Business Rules

- **Bleed-Out (Downed State):** When `player.downed = true`, applies continuous damage at rate `GameConstants.player.bleedOutDPMs` (damage per minute, scaled by `dt`)
  - Source: `DamageSources.BleedOut` (special source type)
  - No visual indicator required (player is already downed in UI)
  
- **Infection:** Accumulates via `player.infection` property (0–100 scale implied)
  - Increments by `dt / 1000` each tick when `[PerkIds.Infected]` is active
  - Broadcasts to nearby players via `Necrosis` perk raycasting (circular AOE)
  - Infection spreads only if target lacks `Immunity` perk and shares layer with source
  
- **Speed Modifiers (Effect Speed):** Temporary multiplier on movement speed via `player.effectSpeedMultiplier`
  - Default: 1.0 (normal speed)
  - Example: Vaccinator slow = 0.8× speed for duration
  - Cleared/reset by clearing `effectSpeedTimeout`
  
- **Per-Tick Damage (Perk-Based):** Perks with `updateInterval` (e.g., 1000 ms) trigger per-tick logic:
  - `Bloodthirst`: Applies `perk.healthLoss` damage per tick
  - `Necrosis`: Applies `perk.dps` health loss + spreads infection
  - `RottenPlumpkin`: Applies health/adrenaline loss + visual effects
  - Custom perks can add arbitrary per-tick behavior
  
- **Duration Control:** Status effects controlled via:
  - Automatic timeout mechanism (`effectSpeedTimeout: NodeJS.Timeout`)
  - Perk disabling (remove from `player.perks` set)
  - Removal conditions (player death, team mode changes, etc.)

## Data Lineage

```
Player Update Loop (every game tick, dt seconds)
│
├─ Infection accumulation
│  ├─ Check: player.hasPerk(PerkIds.Infected)
│  ├─ Increment: player.infection += dt / 1000
│  └─ No upper limit (unclamped)
│
├─ Bleed-out check
│  ├─ Condition: player.downed && !player.beingRevivedBy
│  ├─ Damage: GameConstants.player.bleedOutDPMs * dt
│  ├─ Source: DamageSources.BleedOut
│  └─ Update player health via piercingDamage()
│
├─ Smoke/Synced Particle effects (statusEffect-like)
│  ├─ Iteration: for (syncedParticle of definition)
│  ├─ Health depletion: depletion.health * dt
│  ├─ Adrenaline depletion: depletion.adrenaline * dt
│  └─ Scope snap-in (if lifetime not exceeded)
│
├─ Perk update loop
│  ├─ Condition: perkUpdateMap entries with 1000+ ms elapsed
│  ├─ Per-perk logic:
│  │  ├─ Bloodthirst: healthLoss damage tick
│  │  ├─ Necrosis: dps health + infection spread
│  │  ├─ RottenPlumpkin: healthLoss + adrenLoss + emote
│  │  └─ [Others]: Custom behavior
│  └─ Reschedule next update at now + updateInterval
│
└─ Speed effect cancellation (if timeout expired)
   └─ Reset: effectSpeedMultiplier = 1.0
```

## Complex Functions

### Per-tick Infection Increment — @file server/src/objects/player.ts:1400–1410

**Code:**
```typescript
if (this.hasPerk(PerkIds.Infected)) {
    this.infection += dt / 1000;  // dt in milliseconds; result in seconds
}
```

**Implicit Behavior:**
- Runs every game tick (40 TPS = dt ≈ 25 ms per tick)
- Per-tick increment: 0.025 seconds → uncapped accumulation
- Infection affects HUD display and enabling of Necrosis perk passives
- No visible indicator in gameplay (visual shown via HUD client-side)

### Bleed-Out Damage Application — @file server/src/objects/player.ts:1435–1440

**Code:**
```typescript
if (this.downed && !this.beingRevivedBy) {
    this.piercingDamage({
        amount: GameConstants.player.bleedOutDPMs * dt,
        source: DamageSources.BleedOut
    });
}
```

**Implicit Behavior:**
- Only applies when downed (incapacitated state)
- Stops if teammate initiates revive action (`beingRevivedBy` set)
- Uses `piercingDamage()` (bypasses armor/special damage reduction)
- DPM converted to delta time (constant damage rate regardless of frame rate)
- Death triggers when health ≤ 0; death.source = `BleedOut`

**Example (Bleed-out rate = 60 DPM):**
- Per-tick at 40 TPS: 60 / 60000 ms * 25 ms = 0.025 damage/tick
- Time to death from 100 HP: 100 / 0.025 = 4000 ticks ÷ 40 TPS = 100 seconds

### Necrosis Infection Spread — @file server/src/objects/player.ts:1590–1610

**Code:**
```typescript
case PerkIds.Necrosis: {
    // Health bleed
    if (this.health > perk.minHealth) {
        this.health = Numeric.max(this.health - perk.dps, perk.minHealth);
    }
    
    // Infection spread
    const detectionHitbox = new CircleHitbox(perk.infectionRadius, this.position);
    for (const player of this.game.grid.intersectsHitbox(detectionHitbox)) {
        if (
            !player.isPlayer
            || !player.hitbox.collidesWith(detectionHitbox)
            || player.hasPerk(PerkIds.Immunity)
            || !adjacentOrEqualLayer(this.layer, player.layer)
            || player.dead
        ) continue;
        player.infection += perk.infectionUnits;
    }
    break;
}
```

**Implicit Behavior:**
- Health decreases per tick: `dps` (damage per second) not DPM
- Health clamped to `minHealth` (prevents negative health while infected)
- Circular AOE check with hitbox collision (dual validation)
- Infection spread blocked by: `Immunity` perk, death, layer mismatch
- Spread direction: Always unidirectional (carrier → susceptible)
- Update interval: Controlled by perk definition (`updateInterval`)

**Example (Necrosis perk):**
- DPS: 5 per second = continuous health reduction
- Infection radius: 300 mm (30-tile radius)
- Infection units: 8 per tick (at 40 TPS = 200 units/sec)

### Smoke/Synced Particle Depletion — @file server/src/objects/player.ts:1454–1475

**Code:**
```typescript
for (const syncedParticle of syncedParticles) {
    const def = syncedParticle.definition;
    const depletion = def.depletePerMs;
    
    if (depletion?.health) {
        this.piercingDamage({
            amount: depletion.health * dt,
            source: DamageSources.Gas  // Misleading; should be custom source
        });
    }
    if (depletion?.adrenaline) {
        this.adrenaline -= depletion.adrenaline * dt;
    }
}
```

**Implicit Behavior:**
- Applies damage/depletion from active synced particles (smoke clouds, gas, etc.)
- Depletion is per-millisecond, scaled by delta time
- Health damage bypasses armor (uses `piercingDamage`)
- Adrenaline depletion is unmitigated (direct subtraction)
- Particle lifetime controls duration (auto-expire after `_lifetime` ms)

## Dependencies

**Internal:**
- [Perks & Passives](../../perks-passive/) — Perk definitions and properties
- [Health & Damage](../../health-damage/) — piercingDamage() method, damage source types
- [Spatial Grid](../../spatial-grid/) — Infection spread raycasting via grid.intersectsHitbox()
- [Hitbox System](../../collision-hitbox/) — Collision checks for infection spread and scope snap-in

**External:**
- Game Constants: `GameConstants.player.bleedOutDPMs`, tick rate
- Perk Definition Schema (`@common/definitions/items/perks`)
- Synced Particle Definition Schema (`@common/definitions/syncedParticles`)

## Configuration

| Setting | Effect | Type | Path |
|---------|--------|------|------|
| `GameConstants.player.bleedOutDPMs` | Bleed-out damage rate (damage/minute) | number | `common/src/constants.ts` |
| `PerkDefinition.updateInterval` | Milliseconds between per-tick effect triggers | number | Per-perk |
| `PerkDefinition.healthLoss` | Damage per tick (e.g., Bloodthirst) | number | Per-perk |
| `PerkDefinition.dps` | Damage per second (e.g., Necrosis) | number | Per-perk |
| `PerkDefinition.infectionRadius` | Infection spread AOE radius (mm) | number | Necrosis perk |
| `PerkDefinition.infectionUnits` | Infection amount per spread tick | number | Necrosis perk |
| `SyncedParticleDefinition.depletePerMs` | Health/adrenaline depletion rate | object | Per-particle |

## Known Issues & Gotchas

- **Infection Unclamped:** `player.infection` accumulates indefinitely (no max value); high infection visually may break HUD
- **BleedOut Source Misleading:** Smoke particle depletion uses `DamageSources.Gas` even though it's not actual gas (tech debt)
- **Perk Update Interval Race:** If multiple perks have same `updateInterval`, they all trigger simultaneously; no staggering for performance
- **Necrosis + Immunity Edge Case:** Immunity blocks spread but not self-damage (intentional: prevents Immunity from negating self-damage perk)

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Status Effects & Conditions subsystem
- **Tier 1:** [../../../architecture.md](../../../architecture.md) — System architecture
- **Perks:** [../../perks-passive/README.md](../../perks-passive/README.md) — Perk system overview
- **Damage:** [../../health-damage/README.md](../../health-damage/README.md) — Damage processing
- **Grid:** [../../spatial-grid/README.md](../../spatial-grid/README.md) — Spatial queries for spread
- **Particles:** [../../particle-system/README.md](../../particle-system/README.md) — Synced particle effects
