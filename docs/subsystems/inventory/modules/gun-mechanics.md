# Gun Mechanics & Firing System

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/inventory/README.md -->
<!-- @source: server/src/inventory/gunItem.ts, common/src/definitions/items/guns.ts, common/src/utils/objectDefinitions.ts -->

## Purpose

Implements comprehensive server-side gun firing mechanics: weapon definition loading, fire mode dispatch (Single/Burst/Auto), spread calculation with first-shot accuracy, bullet spawning with wall-clip detection, perk modifier application, dual-gun barrel alternation, cyclic fire delays, ammo management, reload sequencing, and native `setTimeout` scheduling for burst/auto-fire phases that exceed the 40 TPS game tick rate.

---

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| `server/src/inventory/gunItem.ts` | `GunItem` class — all firing state, fire-delay enforcement, bullet spawning, perk application, reload triggering | High |
| `common/src/definitions/items/guns.ts` | `GunDefinition` type — static gun configuration: stats, ammo, fire modes, spread, recoil, ballistics | Medium |
| `server/src/inventory/action.ts` | `ReloadAction` class — executes reload animation, applies ammo modifiers, handles full vs. partial reload logic | Medium |
| `server/src/inventory/inventoryItem.ts` | `InventoryItemBase` — base class with `_bufferAttack`, item lifecycle, modifier stacking system | Medium |
| `common/src/utils/objectDefinitions.ts` | `DefinitionType.Gun`, `InventoryItemDefinition` — type system for guns as registry-managed definitions | Low |

---

## Gun Definition Structure

### Base Gun Definition Architecture

Every gun in Suroi is defined as a `GunDefinition` — a static object automatically registered into the `Guns` registry at startup. Definitions use a **mixin-based union type** pattern to express optional features without inheritance:

```
GunDefinition
├── Required fields (all guns have these)
│   ├── ammoType: "9mm" | "762mm" | "50cal" | etc.
│   ├── capacity: number (magazine size)
│   ├── reloadTime: number
│   ├── fireDelay: number (ms)
│   ├── fireMode: FireMode (Single | Burst | Auto)
│   ├── recoilMultiplier, recoilDuration
│   ├── shotSpread, moveSpread (accuracy)
│   ├── length: number (barrel length)
│   └── ballistics: BaseBulletDefinition
│
├── Optional mixins (only if applicable)
│   ├── ReloadOnEmptyMixin — fullReloadTime (if reloadFullOnEmpty: true)
│   ├── BurstFireMixin — burstProperties (if fireMode === Burst)
│   ├── DualDefMixin — isDual, leftRightOffset (for dual-wield variants)
│   ├── Cycling (DP-12) — cycle.delay, cycle.shotsRequired
│   └── Advanced ballistics — bulletCount, jitterRadius, consistentPatterning, fsaReset
│
└── Visual/Audio fields (not consumed by GunItem, but loaded in definition)
    ├── image: { position, angle, zIndex }
    ├── fists: { animationDuration, leftZIndex, rightZIndex }
    ├── casingParticles: array
    ├── gasParticles: smoke effect config
    └── ballistics.tracer, lastShotFX, etc.
```

### Core Fields Used During Firing

| Field | Type | Purpose | Example |
|-------|------|---------|---------|
| `ammoType` | `string` | Ammo pool identifier in inventory | `"9mm"`, `"762mm"`, `"50cal"` |
| `capacity` | `number` | Normal magazine capacity (rounds) | G19: `15`, AK-74: `30`, M1895: `7` |
| `extendedCapacity` | `number?` | Magazine size with `ExtendedMags` perk | G19: `24`, AK-74: `45` |
| `fireDelay` | `number` | Milliseconds between shots (minimum); for Burst, this is the **intra-burst** delay | G19: `110`, CZ-75A: `60`, M16: `52.5` |
| `reloadTime` | `number` | Seconds to reload (scaled by `owner.reloadMod`) | G19: `1.5`, M1895: `2.1`, PKP: `7.3` |
| `fireMode` | `FireMode` | `Single` (one shot/click), `Burst` (fixed shots), `Auto` (continuous) | — |
| `recoilMultiplier` | `number` | Speed penalty applied to player during recoil | G19: `0.8`, M1895: `0.75`, Deagle: `0.65` |
| `recoilDuration` | `number` | How long (ms) the recoil speed penalty lasts | G19: `90`, M1895: `135`, AWM: `400` |
| `shotSpread` | `number` | Max spread (degrees) **half-angle** when stationary | G19: `4°`, CZ-75A: `8°`, MP5k: `4°`, M1895: `2°` |
| `moveSpread` | `number` | Max spread (degrees) half-angle when moving | G19: `8°`, CZ-75A: `14°`, MP5k: `8°`, M1895: `5°` |
| `fsaReset` | `number?` (ms) | Time after last shot that spread resets to 0 ("first-shot accuracy") | undefined (no reset), RSh-12: `800` |
| `jitterRadius` | `number?` | Random position jitter (game units) applied to each bullet spawn | KSG-12: `0.75`, MP7: undefined |
| `consistentPatterning` | `boolean?` | Shotgun-style deterministic cubic pattern instead of random jitter | ~~KSG-12~~ (uses jitter), DP-12 (true) |
| `bulletCount` | `number?` | Projectiles per trigger pull (default 1) | KSG-12: `8`, Double Barrel: `2` |
| `length` | `number` | Barrel length (game units); controls spawn distance from player | G19: `4.8`, M1895: `5.35`, RSh-12: `6.6` |
| `isDual` | `boolean?` | Whether this is a dual-wield variant (enables alternating barrel firing) | — |
| `leftRightOffset` | `number` | Perpendicular offset (to right/left) for dual-gun spawn positions | Dual G19: `1.3`, Dual Deagle: `1.4` |
| `cycle` | `{ delay, shotsRequired }?` | Special cycle mode (DP-12): override fireDelay every N shots | DP-12: `delay: 200, shotsRequired: 2` |

---

## Gun Variants — Tier Upgrades & Configurations

### Single vs. Dual

**Single variants** are the standard gun configurations. **Dual variants** exist as separate definitions (`isDual: true`) with alternate spawn positions and typically different fire stats. When a player picks up a `Gun`, the game auto-detects the dual form from the `dualVariant` registry reference.

#### Example: G19 (Glock 19) Progression

```javascript
// Single variant (base)
{
    idString: "g19",
    name: "G19",
    tier: Tier.D,
    capacity: 15,
    extendedCapacity: 24,
    fireDelay: 110,
    fireMode: FireMode.Single,
    shotSpread: 4,
    moveSpread: 8,
    ballistics: { damage: 13, speed: 0.22, range: 120 }
}

// Dual variant (separate definition, isDual: true)
{
    idString: "g19_dual",  // registered as dualVariant in single
    isDual: true,
    singleVariant: "g19",
    leftRightOffset: 1.3,
    fireDelay: 75,          // 35% faster
    shotSpread: 5,          // slightly worse accuracy
    moveSpread: 10,
    capacity: 30,
    extendedCapacity: 48,
    ballistics: { damage: 13 }  // damage unchanged
}
```

**Stats comparison:**
- **Duals fire faster** (fireDelay reduced by ~30–50%)
- **Duals have worse accuracy** (spreads ~25% worse)
- **Magazine sizes doubled** (30 → 60 with extended mags)
- **Damage per bullet stays the same** (but double the bullets in flight)

#### Example: Rifle Progression (5.56mm / 7.62mm variants)

The game uses a **single definition tree**. For example, the AK-74 family:

| idString | Name | Ammo | Capacity | Fire Delay | Fire Mode | Tier | Notes |
|----------|------|------|----------|-----------|-----------|------|-------|
| `ak74` | AK-74 | 7.62mm | 30 | 52.5 ms | Auto | C | Baseline rifle |
| `ak74_dual` | Dual AK-74 | 7.62mm | 60 | 26.25 ms | Auto | B | ~2× faster, dual offset |
| `famas` | FAMAS | 5.56mm | 25 | 62.5 ms | Burst | B | 3 rounds/burst, 250 ms cooldown |
| `hk416` | HK416 | 5.56mm | 30 | 57 ms | Auto | B | Higher damage than AK |
| `m16` | M16 | 5.56mm | 30 | 52.5 ms | Burst | C | 3 rounds/burst, 260 ms cooldown |

**Patterns:**
- **Auto rifles:** Fast fire rate (52–70 ms), medium spread, lower damage per shot
- **Burst rifles:** Slower trigger (50–62 ms intra-burst), tight spread, medium damage
- **Sniper rifles (bolt-action):** Slowest (200–600 ms), 1–5 round capacity, highest damage per shot
- **Shotguns:** Medium spread, multiple pellets (8–12 per shot), close-range effective

---

## Ammo System

### Ammo Pool Architecture

Ammunition is **pooled by type** in the player's inventory:

```
player.inventory.items = {
    "9mm": 120,           // Shared pool for all 9mm guns
    "7.62mm": 60,
    "12g": 40,
    "50cal": 10
}
```

When a gun fires, `--ammo` decrements the **instance's magazine** (`this.ammo`). When the magazine is empty, `reload()` transfers from the pool into the magazine:

```
Before reload:  inventory["9mm"] = 120, gun.ammo = 0
Reload starts:  → (reload animation plays for 1.5s)
Reload ends:    → inventory["9mm"] = 105, gun.ammo = 15 (capacity)
                   or gun.ammo = 24 if ExtendedMags perk active
```

### Ammo Consumption Rules

| Scenario | Behavior |
|----------|----------|
| **Normal shot** | `--ammo` (decrement magazine) |
| **infiniteAmmo: true** (e.g., Airstrike) | No ammo decrement; no inventory check |
| **Reload starts** | Checks `inventory.hasItem(ammoType)` before starting action |
| **InfiniteAmmo perk active** | Bypasses inventory ammo check during `reload()` |
| **ExtendedMags perk active** | Reload targets `extendedCapacity` instead of `capacity` |
| **ammo < 0 after decrement** | Clamped to 0; auto-reload triggered via `game.addTimeout()` |

### Magazine Sizing

Magazines have two sizes: **normal** and **extended** (with ExtendedMags perk):

- **Normal capacity** — always used unless perk active
- **Extended capacity** — used if player has `PerkIds.ExtendedMags`

Example (AR-15):
- Normal: 30 rounds
- Extended: 45 rounds
- Storage: `definition.capacity: 30`, `definition.extendedCapacity: 45`

---

## Fire Modes

Suroi implements three distinct fire modes, each with different behavior and timing:

### FireMode.Single (0)

**Fires one projectile per attack input.**

```
Player Input:  Attack button press
    ↓
GunItem.useItem()  → _bufferAttack(fireDelay * fireRateMod, ...)
    ↓
if (fireDelay elapsed):  _useItemNoDelayCheck(skipAttackCheck=true)
    → Spawn 1 bullet (or bulletCount bullets if multi-projectile)
    → --ammo
    → NO auto-timeout scheduled
    ↓
Player must release and re-press to fire again
```

**Fire rate calculation:**
```
Time between shots = fireDelay * owner.fireRateMod

Examples:
  G19:       110 ms × 1.0 = 110 ms  = ~9.1 shots/sec
  Deagle:    200 ms × 1.0 = 200 ms  = 5 shots/sec
  M1895:     375 ms × 1.0 = 375 ms  = 2.7 shots/sec
```

### FireMode.Burst (1)

**Fires a fixed number of shots (`shotsPerBurst`), then waits before the next burst.**

```
Player Input: Attack button (held or tapped)
    ↓
GunItem.useItem()  → _bufferAttack(...)
    ↓
Shot 1 fires:  _consecutiveShots = 1, fireDelay = definition.fireDelay
Shot 2 fires:  _consecutiveShots = 2, auto-timeout reschedules at (fireDelay * fireRateMod)
Shot 3 fires:  _consecutiveShots = 3, hits limit
    ↓
Burst complete:  fireDelay = definition.burstProperties.burstCooldown
                 _burstTimeout = setTimeout(next burst, burstCooldown)
                 _consecutiveShots = 0
    ↓
Wait burstCooldown ms (e.g., 250 ms for MP5k)
    ↓
Burst repeats or stops based on player input
```

**Example: MP5k Burst**

```javascript
{
    idString: "mp5k",
    fireMode: FireMode.Burst,
    fireDelay: 62,  // intra-burst delay (ms)
    burstProperties: {
        shotsPerBurst: 3,
        burstCooldown: 250  // delay between bursts
    }
}
```

**Timing:**
| Shot | Time | Action |
|------|------|--------|
| 1 | 0 ms | Fire, `_consecutiveShots = 1` |
| 2 | +62 ms | Fire, `_consecutiveShots = 2` |
| 3 | +124 ms | Fire, `_consecutiveShots = 3` (burst complete) |
| (cooldown) | +124 ms to +374 ms | Wait 250 ms for next burst |
| Next burst | +374 ms | Fire shot 1 of next burst, _altFire toggles |

**Total RPS (rounds per second) for MP5k:**
```
3 shots / (2×fireDelay + burstCooldown) = 3 / (2×62 + 250) = 3 / 374 ≈ 8.0 RPS
```

### FireMode.Auto (2)

**Fires continuously while attack button is held.**

```
Player Input: Attack button (held)
    ↓
First shot:  _useItemNoDelayCheck()  → spawn bullet, --ammo
    ↓
Auto-timeout scheduled:  _autoFireTimeout = setTimeout(_useItemNoDelayCheck, fireDelay * fireRateMod)
    ↓
    ... repeat until:
    → Attack button released
    → ammo <= 0 (auto-reload triggered instead)
    → Player switches weapons
    → Player dies/goes down
```

**Example: CZ-75A Auto**

```javascript
{
    idString: "cz75a",
    fireMode: FireMode.Auto,
    fireDelay: 60,  // ms between each shot
}
```

**RPS:**
```
1 shot / (fireDelay ms) = 1 / 60 = 16.7 shots/sec
```

---

## Firing Mechanics

### Shot Spread Calculation

Spread represents bullet trajectory deviation from center. It's calculated **per shot** and affects the rotation offset applied to each bullet:

```typescript
// From GunItem._useItemNoDelayCheck()
const { moveSpread, shotSpread, fsaReset } = definition;

// FSA check: reset spread to 0 if enough time has passed since last shot
let spread = owner.game.now - this.lastUse >= (fsaReset ?? Infinity)
    ? 0  // First-shot accuracy: perfect aim
    : Angle.degreesToRadians((owner.isMoving ? moveSpread : shotSpread) / 2);

// Final angle offset applied to bullet rotation
const bulletRotation = owner.rotation + HALF_PI + spread;

// spread is saved for the lifetime of this shot
this.lastUse = owner.game.now;
```

**Spread formula:**
```
spread_radians = Angle.degreesToRadians(spread_degrees / 2)
                          ↑ divided by 2 because spread is "half-angle"
bullet_angle = player_angle + 90° + randomOffset(-spread, +spread)
```

**Examples:**

| Gun | Standing Spread | Stationariness | Moving Spread | Difference |
|-----|-----------------|---|---|---|
| G19 | 4° | Stationary → spread = 2° offset | 8° | 2× worse when moving |
| M1895 (sniper) | 2° | Very accurate | 5° | 2.5× worse when moving |
| CZ-75A (spray) | 8° | Inaccurate | 14° | 1.75× worse when moving |
| Shotgun (KSG) | 12° | Very spread | 20° | Pellets scatter heavily when moving |

### First-Shot Accuracy (FSA)

**FSA reset time** (`definition.fsaReset`) is the delay after which spread resets to 0:

```javascript
// Example: RSh-12 (sniper rifle)
{
    idString: "rsh12",
    fireDelay: 600,      // Very slow (1 shot per 0.6 sec)
    fsaReset: 800,       // Reset spread 800ms after the last shot
    shotSpread: 4,
    moveSpread: 6        // Not used with snipe; shot takes 600ms
}
```

**Behavior:**
```
Time 0 ms:    Player fires → spread = 4° / 2 = 2° offset
Time 100 ms:  Player fires again → spread = 4° / 2 = 2° (no decay yet)
Time 800+ ms:  Player fires → spread = 0° (FSA resets!)
              Next shot is perfectly accurate (no spread)
```

**Guns without FSA** (e.g., G19) never reset unless stopped for a very long time — each shot compounds spread.

### Bullet Spawning & Wall-Clip Prevention

Before spawning a bullet, the game checks if the barrel is obstructed:

```typescript
// Line trace from gun origin to barrel tip
const startPosition = Vec.add(ownerPos, Vec.rotate(Vec(0, offset), owner.rotation));
let position = Vec.add(
    ownerPos,
    Vec.scale(Vec.rotate(Vec(definition.length, offset), owner.rotation), owner.sizeMod)
);

// Check spatial grid for walls/buildings/obstacles
for (const object of owner.game.grid.intersectsHitbox(...)) {
    const intersection = object.hitbox.intersectsLine(startPosition, position);
    if (intersection !== null) {
        // Clipped to intersection minus retraction
        position = Vec.sub(intersection.point, Vec.rotate(Vec(0.2 + jitter, 0), owner.rotation));
    }
}

// Bullet spawns at clipped position
owner.game.addBullet(this, owner, { position, ... });
```

**Effect:** If a player shoots while their barrel is inside a wall, the bullet spawns **outside** the wall instead of glitching through.

### Multi-Bullet Projectiles (Shotguns)

When `definition.bulletCount > 1` (e.g., shotgun with 8 pellets), the spawn function runs multiple times with different patterns:

```typescript
const projCount = definition.bulletCount ?? 1;  // Default 1

for (let i = 0; i < projCount; i++) {
    let finalSpawnPosition: Vector;
    let rotation: number;

    if (definition.consistentPatterning) {
        // Deterministic cubic pattern (shotguns)
        // Bullets spread in a predictable arc
        rotation = 8 * (i / (projCount - 1) - 0.5) ** 3;  // Cubic spread function
    } else {
        // Random jitter (machine guns with multiple projectiles)
        finalSpawnPosition = randomPointInsideCircle(position, jitter);
        rotation = randomFloat(-1, 1);
    }

    rotation *= spread;  // Scale pattern by current spread
    owner.game.addBullet(this, owner, {
        position: finalSpawnPosition,
        rotation: owner.rotation + HALF_PI + rotation,
        ...
    });
}
```

**Shotgun spread pattern:**
```
8 pellets, consistent patterning:
     • (center, perfect aim)
   •       •
   •       •
     •   •
     •   •

Each pellet's angle offset computed as cubic interpolation: 8×(i/7 - 0.5)³
```

---

## Reload System

### Reload Flow

```
Precondition:  ammo < capacity AND player has ammo in inventory AND no action in progress
    ↓
GunItem.reload(skipFireDelayCheck)
    ↓
Create ReloadAction(player, this)
    → action.execute() scheduled after (reloadTime / reloadMod) seconds
    ↓
While action running:
    → player.action = reloadAction
    → player.attackSpeed = 0.5 (ReloadAction.speedMultiplier)
    → setPartialDirty() (sync to client)
    ↓
When action completes:
    → player.action = undefined
    → ammo refilled to capacity (or extendedCapacity)
    → ammo pool decremented from inventory
    → gunFireAnimation = None
```

### Reload Duration Modifiers

| Modifier | Effect | Formula |
|----------|--------|---------|
| `definition.reloadTime` | Base reload duration (seconds) | Always applied |
| `owner.reloadMod` | Perk/effect modifier | `reloadTime / reloadMod` |
| `PerkIds.CombatExpert` | Reload +25% faster | `reloadMod = 1.25` |
| `PerkIds.TacticalRifleman` | Rifle reload +30% faster | Type-gated |
| `PerkIds.FieldMedic` | Healing faster (not reload) | Different subsystem |

**Example:**
```
G19 with CombatExpert perk:
  reloadTime = 1.5 seconds
  reloadMod = 1.25 (perk bonus)
  Actual duration = 1.5 / 1.25 = 1.2 seconds
```

### Full Reload vs. Partial Reload

**Partial reload** (most guns):
```
Reload fills magazine to capacity:
  ammo before: 3/15
  ammo after:  15/15  (filled)
  pool:        120 - 12 = 108
```

**Full reload** (some shotguns, sniper rifles):
```
When reloadFullOnEmpty: true AND ammo <= 0:
  Use fullReloadTime instead of reloadTime
  Example: DP-12 gets 2.9s full reload instead of 1.8s
```

**Pump-action style** (shotsPerReload):
```
Some guns reload only N rounds at a time, not the entire magazine.
Example: Pump shotgun
  capacity: 8
  shotsPerReload: 1
  
Each reload action adds 1 round, not all 8.
Reload twice = 2 rounds added.
```

### Ammo Trimming on Gun Swap/Downgrade

When a player **switches from a higher-capacity gun to a lower-capacity gun**, the magazine may exceed the new gun's capacity:

```
Before switch: M4 with 30 ammo in mag (capacity 30)
Switch to:     G19 with capacity 15
After switch:  G19 with 15 ammo in mag (trimmed from 30)

Extra 15 ammo are NOT returned to the pool — they're simply lost.
```

This is a **gotcha** — players should be aware of magazine size differences.

---

## Accuracy & Spread System

### Base Spread (Stationary vs. Moving)

Every gun has two spread values:
- **`shotSpread`** — spread when player is stationary
- **`moveSpread`** — spread when player is moving

The game decides which to use:

```typescript
const spread = owner.isMoving ? definition.moveSpread : definition.shotSpread;
```

### Spread Accumulation

Spread **does NOT accumulate** per shot by default. Each shot uses the **current spread value**:

```
Shot 1:  spread_degrees = 4°  (stationary) or 8° (moving)
Shot 2:  spread_degrees = 4°  (no increase)
Shot 3:  spread_degrees = 4°  (no increase)
...
Shot N:  spread_degrees = 4°  (always the same)
```

**Exception:** Some mechanics (perks, effects) may modify spread per shot, but the gun definition itself doesn't compound spread.

### Aiming Accuracy Bonus (Scopes)

When a player equips a **scope** (e.g., 2×, 4×), the spread is **reduced by a fixed amount** during aiming:

**From equipment-scope subsystem** (not gun mechanics, but applies to guns):
```
Holding AIM button:
  spread *= 0.5  (50% reduction with any scope)

2× scope:  4° → 2° spread
4× scope:  4° → 2° spread  (same reduction; difference is FOV)
```

---

## Recoil System

### Recoil Mechanics

**Recoil is a speed penalty** applied to the player (not the camera). When a gun fires:

```typescript
owner.recoil.active = true;
owner.recoil.time = owner.game.now + definition.recoilDuration;
owner.recoil.multiplier = definition.recoilMultiplier;
```

**Effect during recoil:**
```
player.effectiveSpeed = baseSpeed * (1 - recoilMultiplier)

Examples:
  baseSpeed = 4 units/tick
  recoilMul = 0.8  (20% slower)
  effectiveSpeed during recoil = 4 × (1 - 0.8) = 0.8 units/tick
  
  So the player moves at 20% normal speed while recoiling.
```

**Duration:** Recoil lasts until `game.now > owner.recoil.time`, then returns to normal.

### Recoil Values by Gun Type

| Gun Type | Recoil Multiplier | Duration (ms) | Effect |
|----------|-------------------|---|---------|
| G19 (pistol) | 0.8 | 90 | Light slowdown, brief |
| AK-74 (rifle) | 0.85 | 150 | Moderate, short |
| M1895 (sniper) | 0.75 | 135 | Heavy slowdown |
| Deagle (high-power pistol) | 0.65 | 150 | Very heavy slowdown |
| RSh-12 (anti-material) | 0.8 | 600 | Longest duration |
| Shotgun | 0.9 | 90 | Minimal impact |

### Recoil & Movement

Recoil **stacks with other speed modifiers**:
```
effectiveSpeed = baseSpeed
    × (1 - recoilMod)        // Recoil penalty
    × knocbackMod             // From knockback (other subsystem)
    × perksSpeedMod           // From perks
    × ...
```

A player under sustained fire + knockback + perks can move very slowly.

---

## Gun Perks & Modifiers

### Perk Application During Firing

Inside `GunItem._useItemNoDelayCheck()`, a **single perk loop** evaluates all player perks and applies modifiers **before bullets are spawned**:

```typescript
const modifiers: {
    damage: number,
    dtc: number,              // Damage-to-construction
    range: number,
    speed: number,
    tracer: { opacity, width, length }
} = { damage: 1, dtc: 1, range: 1, speed: 1, tracer: {...} };

for (const perk of owner.perks) {
    switch (perk.idString) {
        case PerkIds.HollowPoints:
            modifiers.damage *= perk.damageMod;  // +15% damage
            break;
        case PerkIds.SabotRounds:
            modifiers.damage *= perk.damageMod;  // +18% damage
            modifiers.range *= perk.rangeMod;
            modifiers.speed *= perk.speedMod;
            spread *= perk.spreadMod;            // -10% spread
            break;
        case PerkIds.ExtendedMags:
            // Handled in reload(); not in firing
            break;
        ...
    }
}

// Spawn bullets with accumulated modifiers
for (let i = 0; i < projCount; i++) {
    owner.game.addBullet(this, owner, {
        ...,
        modifiers  // Passed to bullet constructor
    });
}
```

### Perk Interactions (Detailed)

#### HollowPoints

| Trigger | Effect | Condition |
|---------|--------|-----------|
| Fire any gun | +15% damage | **Not 12g shotgun with slug pellets** |
| — | — | `definition.casingParticles[0].frame?.includes("slug")` must be false |

#### SabotRounds  

| Trigger | Effect | Condition |
|---------|--------|-----------|
| Fire non-explosive gun | +18% damage, -10% spread, +20% range | **Not:** explosives, on-hit-explosion, projectile-on-hit, or BB ammo type |

#### CloseQuartersCombat

| Trigger | Effect | Condition |
|---------|--------|-----------|
| Fire while near (< 10u) enemy | +20% damage, +25% reload speed, 0.5× recoil | Enemy alive, not downed, not teammate |
| Fire while >10u from enemies | Reload speed = 1.0× (normal) | — |

#### Overclocked

| Trigger | Effect | Condition |
|---------|--------|-----------|
| Fire any gun | -15% spread | — |

#### Toploaded

| Trigger | Effect | Condition |
|---------|--------|.........|
| Fire with low ammo | +30% damage at <50% mag, +15% at <75% | Thresholds: `threshold: [0.5, 1.5], [0.75, 1.15]` |

#### Flechettes

| Trigger | Effect | Condition |
|---------|--------|---------|
| Fire non-explosive gun | Bullet splits into 5 smaller bullets | **Not:** explosives, on-hit-explosion, BB ammo, airdrop summons |
| — | Each sub-bullet does 25% damage | Splits at random deviation ±20° |

#### ExtendedMags

| Trigger | Effect | Condition |
|---------|--------|---------|
| Reload any gun | +50% magazine capacity | Applied to both capacity and extendedCapacity |

#### InfiniteAmmo

| Trigger | Effect | Condition |
|---------|--------|---------|
| Fire by equipping perk | Ammo never depletes | `definition.infiniteAmmo !== true` (perk only) |

---

## Gun Animations

### Firing Animation

When `_useItemNoDelayCheck()` fires a bullet, it sets the player's animation:

```typescript
owner.animation = this._altFire ? AnimationType.GunFireAlt : AnimationType.GunFire;
```

| Animation | Trigger | Duration | Visual |
|-----------|---------|----------|--------|
| `GunFire` | Normal firing | From `definition.fists.animationDuration` | Right arm firing |
| `GunFireAlt` | Dual-wield alternating shot | Same | Left arm firing (dual guns) |
| `GunClick` | Attempt to fire with no ammo | Instant | Click/jam sound |

### Reload Animation

`ReloadAction.execute()` plays the reload animation (client-side), triggered when:

```
reloadAction.player.action = reloadAction
→ setPartialDirty()
→ Client receives action = PlayerActions.Reload
→ Client plays reload animation for `reloadTime / reloadMod` seconds
```

The **animation duration matches the actual reload time**, so it looks synchronized.

### Shot Effects

#### Casing Particles

When a gun fires, casing particles are spawned from the ejection port:

```typescript
definition.casingParticles = [{
    position: Vec(3.5, 0.5),  // Relative to player
    velocity: { x: { min: -8, max: 2 }, y: { min: 2, max: 18 } },
    ejectionDelay: 0,         // Spawn at trigger pull
    on: "fire" | "reload" | "cycle"
}]
```

#### Muzzle Flash

Unless `definition.noMuzzleFlash === true`, a muzzle flash sprite is rendered at the barrel tip.

#### Gas/Smoke Particles

Each gun has optional `gasParticles` config:

```javascript
gasParticlePresets = {
    automatic: { amount: 2, minLife: 1000, maxLife: 2000 },
    shotgun: { amount: 12, minLife: 2000, maxLife: 5000 },
    pistol: { amount: 2, minLife: 1000, maxLife: 2000 }
}
```

Spawned at the muzzle, drifts with random velocity.

#### Camera Shake

Some guns trigger camera shake:

```javascript
definition.cameraShake = {
    duration: 150,   // ms
    intensity: 0.2   // magnitude
}
```

---

## Gun Sounds

### Fire Sound

When a bullet is spawned, the **ballistics module** triggers a fire sound:

```
owner.game.addBullet(...)
→ Bullet spawned (see projectiles-ballistics subsystem)
→ Sound manager queues fire-sound at position
```

Sound is played at bullet spawn position (server-side event sent to client).

### Reload Sound

When `ReloadAction.execute()` completes, a reload sound is synchronously played:

```typescript
// Not directly in gunItem.ts, but in client's gun rendering
// Client receives action = PlayerActions.Reload
→ Play sound "weapons/[gunName]/reload.wav"
```

### Empty Magazine Click

When a player attempts to fire with `ammo <= 0`:

```typescript
if (this.ammo <= 0) {
    owner.animation = AnimationType.GunClick;
    owner.setPartialDirty();
}
→ Client plays "weapons/generic/empty_mag.wav"
```

---

## Network Sync

### Ammo Count Serialization

The client needs to know the current ammo count. It's sent in the **UpdatePacket**:

```
UpdatePacket {
    direction: "server→client"
    players: [{
        id: 123,
        activeItem: { idString, ammo: 15 },  // Current magazine
        items: {
            g19: {...},           // Inventory items
            "9mm": 105            // Ammo pool
        }
    }]
}
```

**Update frequency:** Delta-compressed — only sent when ammo changes.

### Firing State Serialization

When a gun fires, several state changes are sent:

| State | Field | Type | Sent When |
|-------|-------|------|-----------|
| Animation | `player.animation` | `AnimationType` | Firing, reloading, click |
| Dirty weapons flag | `player.dirty.weapons` | `boolean` | Ammo or weapon state changed |
| Recoil active | `player.recoil.active` | `boolean` | Firing (recoil applied) |
| Action | `player.action` | `PlayerActions` | Reload started |

### Dual-Gun State

For dual-gun weapons, `_altFire` determines which barrel fires, but it's **not explicitly serialized**:
- The **animation** changes (`GunFire` vs `GunFireAlt`)
- **Animation alternates** each shot (visual feedback)

Client infers which barrel fires from the animation sequence.

---

## Known Gotchas & Tech Debt

### Fire-Rate Timing Issues

**Problem:** Some guns have fire rates faster than the 40 TPS game tick (25 ms):

```
Game tick: 25 ms
Gun fire delay: 15 ms (CZ-75A auto)

A timeout that expires between ticks won't fire until the next tick!
```

**Solution:** Use native `setTimeout` for burst and auto timers:
```typescript
this._burstTimeout = setTimeout(
    this._useItemNoDelayCheck.bind(this, false),
    definition.burstProperties.burstCooldown
);  // NOT game.addTimeout() — that would be misaligned
```

**Implication:** These timers can "slip" if the server is under load.

### Spread Accumulation Misconception

**Gotcha:** The **gun definition doesn't accumulate spread**. Each shot uses the current spread value, not a running total.

If you want spread to accumulate per consecutive shot, you'd need to **modify `GunItem` to track consecutive shots per position/direction**, which doesn't exist yet.

### Magazine Trimming on Downgrade

**Gotcha:** When swapping guns, **excess ammo in the magazine is lost**:

```
Swap: AK-74 (30 cap) with 28 ammo  →  G19 (15 cap)
Result: G19 with 15 ammo (8 rounds lost forever)
```

Players should reload to full before swapping to avoid waste.

### Dual-Gun Alternation Reset

**Gotcha:** `_altFire` doesn't reset if the player **switches weapons mid-burst**:

```
Fire Dual G19 (left barrel):   _altFire = true
Switch to different gun:       (no reset)
Switch back to Dual G19:       _altFire still = true  (might expect reset)
```

### Reload Guards are Strict

`reload()` returns without starting if **any guard fails**:

```typescript
if (
    definition.infiniteAmmo
    || alreadyAtCapacity()
    || !hasAmmoInPool()       // No ammo in inventory
    || action !== undefined    // Example: in revive
    || notActiveItem()
    || firingTooEarly()        // fireDelay not elapsed
                               //  (unless skipFireDelayCheck=true)
    || owner.downed
) return;  // Silent fail — no error or feedback
```

**Implication:** Spamming reload button is fine; extras just do nothing.

### Bullet Spread Pattern for Shotguns

**Gotcha:** The cubic spread pattern for consistent patterning can produce tight clusters in the center and sparse edges:

```
Shotgun pellet spread:  8 pellets, cubic function  
                        8×(i/7 - 0.5)³

i=0: 8×(-0.5)³ = -1.0     ← very left
i=1: 8×(-3/7)³ ≈ -0.27
i=2: 8×(-1/7)³ ≈ -0.02
i=3: 8×(1/7)³  ≈ +0.02   ← tight center
i=7: 8×(0.5)³  = +1.0    ← very right
```

Most pellets cluster near center; few on edges. This is **intentional** for tighter aim.

### Ammo Trimming on Magazine Capacity Changes

**Gotcha:** If a player hits a perk that increases magazine capacity mid-reload:

```
Before reload:  G19 capacity=15, ExtendedMags perk acquired (capacity→24)
During reload:  ReloadAction targets 24
After reload:   ammo=24 (fills to new extended capacity)
```

This works correctly because capacity is recalculated in `ReloadAction.execute()`.

### Fire-Rate Modifier Stacking

**Gotcha:** Fire rate modifiers **multiply the fireDelay**, not add to it:

```
fireDelay = 110 ms
fireRateMod = 0.8  (from AMP perk, +25% faster)

Actual fireDelay = 110 × 0.8 = 88 ms  (NOT 110 - 25)
```

**Multiple modifiers stack multiplicatively:**
```
Base: 110 ms
Perk 1: ×0.8
Perk 2: ×0.9
Actual: 110 × 0.8 × 0.9 = 79.2 ms
```

---

## Cross-Tier References

### Tier 1 — Architecture & Data Model
- [docs/architecture.md](../../../../architecture.md) — System overview, subsystem layout
- [docs/datamodel.md](../../../../datamodel.md) — Entity schema, gun definition structure as part of the object registry

### Tier 2 — Related Subsystems
- [docs/subsystems/inventory/README.md](../README.md) — Subsystem overview, ammo management, item lifecycle
- [docs/subsystems/inventory/patterns.md](../patterns.md) — Inventory patterns (e.g., swapping, dropping, pooling)
- [docs/subsystems/projectiles-ballistics/README.md](../../projectiles-ballistics/) — Bullet spawning, damage calculation, hit effects
- [docs/subsystems/perks-passive/README.md](../../perks-passive/) — Perk system, how perks are applied to actions
- [docs/subsystems/game-objects-server/README.md](../../game-objects-server/) — Player object, state, dirty flags
- [docs/subsystems/serialization-system/README.md](../../serialization-system/) — Binary packet encoding (UpdatePacket, ammo)

### Tier 3 — Related Modules
- [docs/subsystems/inventory/modules/gun-item.md](gun-item.md) — Legacy GunItem documentation (parallel resource)
- [docs/subsystems/inventory/modules/actions.md](actions.md) — ReloadAction, how reload state is managed
- [docs/subsystems/inventory/modules/equip-pickup.md](equip-pickup.md) — Gun pickup/equip flow
- [docs/subsystems/projectiles-ballistics/modules/...](../../projectiles-ballistics/modules/) — Bullet mechanics
- [docs/subsystems/perks-passive/modules/perk-effects.md](../../perks-passive/modules/perk-effects.md) — Perk rule application
- [docs/subsystems/core-math-physics/modules/vectors-hitbox.md](../../core-math-physics/modules/vectors-hitbox.md) — Vector math, wall-clip line intersection

---

## Summary

The **Gun Mechanics & Firing System** is a tightly integrated module that handles:

1. **Definition loading** from the `Guns` registry — each gun is a static configuration object
2. **Fire mode dispatch** (Single / Burst / Auto) with different timing models
3. **Spread calculation** per-shot with FSA reset and movement penalties
4. **Bullet spawning** at the barrel with wall-clip prevention
5. **Perk application** via a unified modifier loop before bullets are created
6. **Dual-gun alternation** with barrel offset tracking
7. **Ammo management** (magazine + inventory pool) with reload sequencing
8. **Recoil imposition** as a speed penalty on the player
9. **Fire-rate enforcement** using buffered inputs and native timers for precision
10. **Network synchronization** of ammo, animation, and weapon state to clients

All of this runs **server-authoritative** — the client sends input, the server decides whether to fire and what bullets to create, and sends the authoritative result back.
