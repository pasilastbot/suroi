# Melee Combat — Attack Mechanics & Hit Detection

<!-- @tier: 3 -->
<!-- @parent: ../README.md -->
<!-- @source: server/src/inventory/meleeItem.ts -->

## Purpose

Server-side melee attack execution and hit detection. Covers swing initiation, hitbox creation, target filtering with wall occlusion checks, damage multiplier application including perk modifiers, multi-hit scheduling, and cooldown enforcement. This module defines how a melee swing connects with targets and applies damage.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| `server/src/inventory/meleeItem.ts` | `MeleeItem` class: `useItem()`, `_useItemNoDelayCheck()`, attack timing and damage application | High |
| `common/src/definitions/items/melees.ts` | `MeleeDefinition` interface and weapon definitions (hatchet, sickle, saw, ice_pick, etc.) | Medium |
| `common/src/utils/gameHelpers.ts` | `getMeleeHitbox()` — hitbox creation; `getMeleeTargets()` — target filtering with raycasting | High |
| `server/src/inventory/inventoryItem.ts` | `_bufferAttack()` — cooldown checking and attack buffering (200 ms tolerance) | Medium |

## Melee Weapon Types

### Weapon Categories by Playstyle

| Type | Examples | Damage | Cooldown | Radius | Specialty |
|------|----------|--------|----------|--------|-----------|
| **Fast/Light** | Sickle, Kukri, Dagger | 20–25 | 160–225ms | 2.5–2.7 | Auto-fire, attack speed |
| **Medium/Balanced** | Hatchet, Baseball Bat, Falchion | 34–41 | 340–450ms | 2.0–4.1 | Balanced damage & speed |
| **Heavy/Slow** | Crowbar, Maul, Saw, Scythe | 40–54 | 420–725ms | 2.0–3.0 | High damage, wall breaking |
| **Specialty** | Pan (reflective), Ice Pick (ice damage), Chainsaw (auto) | 8–42 | 10–500ms | 1.75–4.0 | Unique mechanics |

### Stat Examples

**Hatchet** (Tier B, balanced melee):
```typescript
{
  idString: "hatchet",
  damage: 40,
  radius: 2,
  offset: Vec(5.2, -0.5),
  cooldown: 420,
  hitDelay: 180,
  obstacleMultiplier: 2.0,
  piercingMultiplier: 1.5,
  speedMultiplier: 1,
  fireMode: undefined // requires click-to-attack
}
```

**Sickle** (Tier B, fast & auto):
```typescript
{
  idString: "sickle",
  damage: 20,
  radius: 2.7,
  offset: Vec(3.1, 0.9),
  cooldown: 160,
  attackCooldown: 140, // shorter if hits
  fireMode: FireMode.Auto, // hold-to-swing
  obstacleMultiplier: 1.25,
  iceMultiplier: 0.1
}
```

**Maul** (Tier S, heavy):
```typescript
{
  idString: "maul",
  damage: 54,
  radius: 2.7,
  offset: Vec(5.4, -0.5),
  cooldown: 450,
  hitDelay: 180,
  maxHardness: 5, // only damages hardness ≤ 5
  stonePiercing: true,
  obstacleMultiplier: 2.0,
  piercingMultiplier: 1.0,
  iceMultiplier: 5.0
}
```

## Attack Animation Timing

### Timeline of a Melee Swing

```
T=0 ms:    Player clicks M (or holds M for auto-fire)
           ↓
           InputPacket received on server
           → Player.attacking = true
           → Player.startedAttacking = true
           ↓
           MeleeItem.useItem() called
           → _bufferAttack(definition.cooldown, ...)
           ↓
           Check: did cooldown pass? (now - lastUse >= cooldown)
           ↓ YES
           _useItemNoDelayCheck() executes immediately
           → owner.animation = AnimationType.Melee
           → owner.setPartialDirty() (broadcast to clients)
           ↓
T=hitDelay: game.addTimeout(..., hitDelay) fires
           (default hitDelay = 50 ms, hatchet = 180 ms)
           ↓
           FOR i = 0 TO numberOfHits-1:
             - Create hitbox: getMeleeHitbox(player, definition)
             - Query grid: game.grid.intersectsHitbox(hitbox)
             - Filter targets: getMeleeTargets(hitbox, definition, ...)
             - Apply damage to each target
             - Schedule next hit with delayBetweenHits (if i < numberOfHits-1)
           ↓
T=hitDelay: If hits.length > 0 and attackCooldown defined:
           Schedule next swing at (hitDelay + attackCooldown)
           ↓
           Otherwise:
           Schedule next swing at (hitDelay + cooldown)
           ↓
           For auto-fire weapons (FireMode.Auto):
           _autoUseTimeoutID = setTimeout(..., timing)
```

### Key Timing Fields

| Field | Type | Default | Purpose |
|-------|------|---------|---------|
| `hitDelay` | ms | 50 | Wait before first hit in swing sequence |
| `numberOfHits` | count | 1 | Total simultaneous or staggered hits per swing |
| `delayBetweenHits` | ms | 0 | Wait between consecutive hits (if numberOfHits > 1) |
| `cooldown` | ms | required | Time between swings (default if hits.length = 0) |
| `attackCooldown` | ms | optional | Faster cooldown if swing hits something (encourages aggression) |

### Multi-Hit Weapons (Example: Saw)

**Saw definition:**
```typescript
{
  damage: 23,
  cooldown: 725,
  numberOfHits: 2,
  hitDelay: 250,
  delayBetweenHits: 415
}
```

**Swing timeline:**
```
T=0:   Swing starts
T=250: First hit resolves:
       - Hitbox check for targets
       - Damage applied
       ↓
T=665: (250 + 415) Second hit resolves:
       - Hitbox check for targets (same or different)
       - Damage applied
       ↓
T=915: (665 + 250) Cooldown ends, next swing allowed
```

If first hit connects: next swing in 725 ms (full cooldown).
If first hit misses: second hit might connect, then cooldown resumes from T=665.

## Hit Detection Algorithm

### Step 1: Hitbox Creation

**Function:** `getMeleeHitbox(player, definition)` @ [gameHelpers.ts](../../../../common/src/utils/gameHelpers.ts)

```typescript
export function getMeleeHitbox(
  player: CommonObjectMapping[ObjectCategory.Player],
  definition: MeleeDefinition
): CircleHitbox {
  const position = Vec.add(
    player.position,
    Vec.scale(Vec.rotate(definition.offset, player.rotation), player.sizeMod)
  );
  return new CircleHitbox(definition.radius * player.sizeMod, position);
}
```

**Inputs:**
- `player.position` — player center in world coordinates
- `definition.offset` — local offset (e.g., `Vec(5.2, -0.5)` for hatchet = ahead & slightly right)
- `player.rotation` — player facing angle (radians)
- `definition.radius` — unscaled hitbox radius (e.g., `2.0`)
- `player.sizeMod` — size multiplier (perks/effects can change this)

**Output:** `CircleHitbox` centered at rotated offset position, radius = `definition.radius * player.sizeMod`

**Example (Hatchet):**
```
Player at (100, 100), facing 0 radians (right), sizeMod = 1.0
Hatchet offset: Vec(5.2, -0.5)
Rotated offset: Vec(5.2, -0.5) at 0 radians = Vec(5.2, -0.5)
Hitbox center: (100 + 5.2, 100 - 0.5) = (105.2, 99.5)
Hitbox radius: 2.0 * 1.0 = 2.0 units
```

### Step 2: Spatial Query

```typescript
const objects = owner.game.grid.intersectsHitbox(hitbox);
```

Spatial grid returns all objects in cells overlapping the hitbox. Reduces O(n) to O(k) where k ≈ objects in nearby cells.

### Step 3: Target Filtering

**Function:** `getMeleeTargets<T>(hitbox, definition, player, teamMode, objects)` @ [gameHelpers.ts](../../../../common/src/utils/gameHelpers.ts)

For each object in grid:

1. **Type check:** Must be `Player | Obstacle | Building | Projectile` (C4 only)
2. **State check:** `!object.dead && object.damageable`
3. **Self-check:** `object !== player` (don't hit self)
4. **Obstacle filter:** Skip stairs, skip obstacles with `noMeleeCollision` flag
5. **Layer check:** `adjacentOrEquivLayer(object, player.layer)` — same or adjacent layer
6. **Hitbox intersection:** `hitbox.getIntersection(object.hitbox)` — circular overlap
7. **Raycasting** (wall occlusion):
   - For **Players/C4:** Raycast from player to target. If a wall is between, target fails if wall distance < target distance (with 0.05 unit tolerance).
   - For **Obstacles:** Raycast to check if another obstacle blocks it.
8. **Sorting:** Sort by distance (closest first)
9. **Team filter:** In team mode, deprioritize teammates
10. **Capping:** Return only `definition.maxTargets` closest targets (default: 1, uncapped if `Infinity`)

### Step 4: Damage Application

```typescript
for (const hit of hits) {
  const target = hit.object;
  let multiplier = 1;

  // Perk modifiers (exclusive: Lycanthropy XOR Berserker)
  if (!owner.hasPerk(PerkIds.Lycanthropy)) {
    multiplier *= owner.mapPerkOrDefault(
      PerkIds.Berserker,
      ({ damageMod }) => damageMod,
      1
    );
  }
  multiplier *= owner.mapPerkOrDefault(
    PerkIds.Lycanthropy,
    ({ damageMod }) => damageMod,
    1
  );

  // Target-specific multipliers
  if (target.isObstacle) {
    if (definition.piercingMultiplier !== undefined && target.definition.impenetrable) {
      multiplier *= definition.piercingMultiplier;
    } else {
      multiplier *= definition.obstacleMultiplier;
    }

    if (target.definition.material === "ice") {
      multiplier *= definition.iceMultiplier ?? 0.01;
    }
  }

  if (target.isProjectile) {
    multiplier *= definition.obstacleMultiplier;
  }

  // Apply damage
  const actualDamage = definition.damage * multiplier;
  target.damage({
    amount: actualDamage,
    source: owner,
    weaponUsed: this
  });

  // Obstacle interaction (trigger button press)
  if (target.isObstacle && !target.dead) {
    target.interact(owner);
  }
}
```

## Range & Knockback

### Effective Range

**Melee range** is determined by `radius * sizeMod`:

```
Effective Radius = definition.radius * player.sizeMod
Range = player.position to hitbox.position + radius
```

**Examples:**
| Weapon | Radius | Effective Range (sizeMod=1.0) | Reach |
|--------|--------|-------------------------------|-------|
| Sickle | 2.7 | 2.7 units | Melee only |
| Hatchet | 2.0 | 2.0 units | Tight reach |
| Falchion | 4.1 | 4.1 units | Extended reach |
| Baseball Bat | 4.0 | 4.0 units | Wide swing |

**Range tuning:** Larger `radius` = longer reach but easier to hit unintended targets.

### Knockback

**Server:** Knockback calculation happens in `target.damage()` handler (not in melee module itself).

**Client:** When `AnimationType.Melee` is played, the client displays hit effect particles and plays sound (see Animation Feedback below).

**Note:** Melee does NOT directly apply knockback velocity. Damage system handles knockback based on weapon type (melee vs gun).

## Damage Calculation

### Formula

```
actualDamage = definition.damage × (perkMod × typeMod)
```

### Perk Modifiers

| Perk | Qualifier | Effect | Stacking |
|------|-----------|--------|----------|
| **Berserker** | Normal perk | `damageMod` (typically 1.1–1.5x) | Multiplicative with Lycanthropy? No: `if (!hasPerk(Lycanthropy))` |
| **Lycanthropy** | Special perk | `damageMod` (typically 1.5–2.0x) | **Exclusive with Berserker** — only one applies |
| **Close Quarters Combat** | Normal perk | Triggers when enemy <`cutoff` range (TurretGun-specific logic). **Not actively used for melee** | N/A |

**Logic:**
```typescript
if (!owner.hasPerk(PerkIds.Lycanthropy)) {
  multiplier *= owner.mapPerkOrDefault(PerkIds.Berserker, ({ damageMod }) => damageMod, 1);
}
multiplier *= owner.mapPerkOrDefault(PerkIds.Lycanthropy, ({ damageMod }) => damageMod, 1);
```

If player has Lycanthropy: Lycanthropy's `damageMod` applies.
If player has Berserker (no Lycanthropy): Berserker's `damageMod` applies.
Both applied if player has both, but Lycanthropy blocks Berserker evaluation.

### Type Multipliers

#### vs. Players
- No multiplier (always 1.0)

#### vs. Obstacles (normal)
- `definition.obstacleMultiplier` (always required, examples: 1.5–2.2)

#### vs. Obstacles (impenetrable)
- If `definition.piercingMultiplier !== undefined`: use `piercingMultiplier`
- Else: use `obstacleMultiplier` (cannot damage)

#### vs. Obstacles (ice material)
- Also multiply by `definition.iceMultiplier` (default 0.01, can be 2.0+ for specialized weapons)
- Applied **after** normal/piercing multiplier

#### vs. Projectiles (C4 only)
- `definition.obstacleMultiplier`

### Max Hardness Filter

If `definition.maxHardness` is **undefined:**
- Cannot damage obstacles with `hardness` property at all

If `definition.maxHardness` is defined (e.g., Maul has `5`):
- Can only damage obstacles with `hardness ≤ maxHardness`

**Example: Maul vs obstacles**
```typescript
{
  maxHardness: 5,
  damage: 54,
  obstacleMultiplier: 2.0,
  stonePiercing: true // can damage stone
}
```
- Can damage wood (hardness 1), stone (hardness 5), etc.
- Cannot damage titanium (hardness > 5)

### Example Damage Calculations

**Hatchet (40 DMG) vs Player in Berserker:**
```
actualDamage = 40 × (1.1 × 1.0) = 44 DMG
```

**Hatchet vs Wood Wall:**
```
actualDamage = 40 × 2.0 = 80 DMG (obstacleMultiplier)
```

**Hatchet vs Concrete Wall (impenetrable):**
```
actualDamage = 40 × 1.5 = 60 DMG (piercingMultiplier)
```

**Maul vs Ice Wall:**
```
actualDamage = 54 × (2.0 × 5.0) = 540 DMG (obstacleMultiplier × iceMultiplier)
```

## Multi-Hit Prevention

### Mechanism

**No built-in per-swing deduplication.** Same target can be hit **multiple times** in the same swing if:

1. **`numberOfHits > 1`** and target hasn't moved between hit delays
2. **`maxTargets > 1` (Infinity)** and multiple instances of same target exist in grid results

### Default Behavior

Most weapons have `maxTargets: 1` (default), so only the closest valid target is hit per swing hit window. Multi-hit weapons (Saw) have `numberOfHits: 2` but still hit unique targets if they move/separate.

### Example: Saw with 2 hits

```javascript
{
  numberOfHits: 2,
  delayBetweenHits: 415,
  maxTargets: 1
}
```

**Scenario 1 — Same target:**
```
T=250:  First swing hits Player A
        (Player A takes 23 DMG)
        ↓
T=665:  Second swing hits Player A again
        (Player A takes another 23 DMG)
        Total: 46 DMG to Player A
```

**Scenario 2 — Different targets (Player A moves):**
```
T=250:  First swing hits Player A
        (Player A takes 23 DMG)
        ↓ Player A moves away
T=665:  Second swing hits Player B (now closest in hitbox)
        (Player B takes 23 DMG)
        Total: 23 DMG to A, 23 DMG to B
```

### Multi-Target Weapons (maxTargets: Infinity)

**Falchion example:**
```typescript
{
  damage: 41,
  maxTargets: Infinity,
  numberOfHits: 1
}
```

Hits **all** targets in hitbox, not just closest. Can hit 5 players in one swing.

### Prevention Strategies

1. **Small `maxTargets`** (default 1) — ensures only closest target per hit
2. **Disable `numberOfHits`** — single hit per swing prevents stacking
3. **Plugin intercept `inv_item_use`** — event can cancel/modify damage

## Wall Interaction

### Wall Hitting

When a melee swing hits an `Obstacle`:

1. **Check hitbox intersection** — obstacle must overlap swing hitbox
2. **Check raycasting** — if applicable (some obstacles transparent to raycast)
3. **Apply damage** with `obstacleMultiplier` or `piercingMultiplier`
4. **Trigger interaction** — obstacle's `interact()` method called:
   ```typescript
   if (target.isObstacle && !target.dead) {
     target.interact(owner);
   }
   ```

### Damage Transfer

Obstacles can be **chained obstacles** (e.g., doors, windows—break one, reveal next). `interact()` may trigger opening/breaking logic.

### Sound Effects

When obstacle is hit:
- Server emits `hitEffect` with `hitSound` (if defined in weapon)
- Client plays hit sound at target position
- Visual particles (sparks, splinters) depend on material

**Example: Hatchet hitting wood**
```
hitSound: undefined (no special sound)
obstacleMultiplier: 2.0 (80 DMG to wood)
→ Wood plays generic "thud" or splinter sound
→ Sparks/splinter particles at impact
```

**Example: Saw hitting wood**
```
hitSound: "hand_saw_hit"
obstacleMultiplier: 2.0
→ Saw-specific "cutting" sound plays
→ Splinter particles
```

## Animation Feedback

### Server-Side Animation

When `_useItemNoDelayCheck()` executes:

```typescript
owner.animation = AnimationType.Melee;
owner.setPartialDirty();
```

This broadcasts `AnimationType.Melee` to all clients via `UpdatePacket`. Clients receive and invoke `playAnimation(AnimationType.Melee)`.

### Client-Side Visual Feedback

**File:** [client/src/scripts/objects/player.ts](../../../../client/src/scripts/objects/player.ts#L1662)

```typescript
playAnimation(anim: AnimationType): void {
  case AnimationType.Melee: {
    this.updateFistsPosition(false); // hide normal fists
    this.updateWeapon(true);
    const weaponDef = this.activeItem;

    let altFist = Math.random() < 0.5;
    if (!weaponDef.fists.randomFist) altFist = true;
    let previousDuration = 0;

    // Play animation keyframes
    if (weaponDef.animation) {
      for (const frame of weaponDef.animation) {
        const duration = frame.duration;
        this.addTimeout(() => {
          // Tween weapon sprite and fist positions to frame.image and frame.fists
          this.addTween({
            target: this.images.weapon,
            to: { x: frame.image.position.x, y: frame.image.position.y, angle: frame.image.angle },
            duration,
            ease: EaseFunctions.cubicOut
          });
        }, previousDuration);
        previousDuration += duration;
      }
    }

    // Play swing sound at hitDelay
    if (weaponDef.hitDelay) {
      clearTimeout(this._meleeSoundTimeoutID);
      this._meleeSoundTimeoutID = window.setTimeout(() => {
        this.playSound(
          weaponDef.swingSound ?? "swing",
          { falloff: 0.4, maxRange: 96 }
        );
      }, weaponDef.hitDelay);
    } else {
      this.playSound(weaponDef.swingSound ?? "swing", { falloff: 0.4, maxRange: 96 });
    }

    // Apply hit effects
    const hitDelay = weaponDef.hitDelay ?? 50;
    this.addTimeout(() => {
      // For each hit in numberOfHits:
      // Resolve hitbox, create MeleeTargets, apply hit effect particles
      for (let i = 0; i < (weaponDef.numberOfHits ?? 1); i++) {
        this.addTimeout(() => {
          const hitbox = getMeleeHitbox(this, this.activeItem);
          const hits = getMeleeTargets<Player | Obstacle | Projectile | Building>(
            hitbox,
            this.activeItem,
            this,
            Game.isTeamMode,
            Game.objects
          );

          for (const hit of hits) {
            const angle = Math.atan2(hit.direction.y, hit.direction.x);
            if (target.isPlayer) {
              target.hitEffect(hit.position, angle, sound);
            } else {
              target.hitEffect(hit.position, angle);
            }
          }
        }, (i === 0 ? 0 : (weaponDef.delayBetweenHits ?? 0)));
      }
    }, hitDelay);

    break;
  }
}
```

### Animation Definition (Example: Hatchet)

```typescript
{
  idString: "hatchet",
  fists: {
    animationDuration: 150,
    left: Vec(38, -35),  // idle position
    right: Vec(38, 35)
  },
  image: {
    position: Vec(31, 41),
    angle: 190,
  },
  animation: [
    { // Warmup phase
      duration: 100,
      fists: {
        left: Vec(38, -35),
        right: Vec(100, 35)  // swing arm forward
      },
      image: {
        position: Vec(110, 33),
        angle: 40  // rotate weapon up
      }
    },
    { // Recovery phase
      duration: 100,
      fists: {
        left: Vec(38, -35),
        right: Vec(38, 35)  // return to idle
      },
      image: {
        position: Vec(31, 41),
        angle: 190  // return to idle
      }
    }
  ]
}
```

### Hit Effect Particles

When `hitEffect(position, angle, sound?)` is called on target (Player/Obstacle):

**For Players:**
```typescript
target.hitEffect(hit.position, angle, sound);
```
- Blood particles at impact position
- Sound effect (if `weaponDef.hitSound` defined, else generic hit sound)
- Direction angle determines particle spray direction

**For Obstacles:**
```typescript
target.hitEffect(hit.position, angle);
```
- Splinter/spark particles (depends on material: wood, metal, ice, etc.)
- No specific sound from obstacle (client plays server-sent hitSound)

### Animated Weapons (Example: Chainsaw)

```typescript
{
  idString: "chainsaw",
  image: {
    animated: true  // sprite alternates between frames
  },
  animation: [
    { duration: 10, ... }  // very short frame for looping
  ]
}
```

Client code:
```typescript
if (weaponDef.image?.animated) {
  if (this.meleeAttackCounter >= 1) {
    this.meleeAttackCounter--;
  } else {
    this.meleeAttackCounter++;
  }
  this.images.weapon.setFrame(
    `${weaponDef.idString}${this.meleeAttackCounter <= 0 ? "_used" : ""}`
  );
}
```

Weapon sprite flips between two frames (`chainsaw` and `chainsaw_used`) to simulate cutting motion.

## Network Sync

### Attack Initiation

**Client → Server:**
```
InputPacket { action: PlayerActions.Attack, active: true }
```

**Server side:**
```typescript
player.attacking = true;
player.startedAttacking = true;
```

**Event emission:**
```typescript
game.pluginManager.emit("player_start_attacking", player);
```

### Animation Broadcast

**Server → Clients:**
```
UpdatePacket {
  type: ObjectCategory.Player,
  id: player.id,
  data: {
    animation: AnimationType.Melee
  }
}
```

Clients receive `AnimationType.Melee`, trigger `playAnimation()`.

### Damage Registration

**Server-side (no explicit packet):**
- Damage applied directly via `target.damage({ amount, source, weaponUsed })` in `_useItemNoDelayCheck()`
- Target's health updated in `UpdatePacket` for next frame
- If target dies: `KillPacket` sent with killer info

**Client view:**
- Client receives updated health/death via packets
- Hit effect particle plays immediately when `hitEffect()` is called
- No "hit confirmation" packet needed — server-authoritative

### Latency Implications

**High latency (>200 ms):**
- InputPacket delay delayed reaching server
- Attack executes 200+ ms late server-side
- Client-side prediction: clients show local swing animation immediately, server validation is delayed
- If server rejects attack (cooldown, dead, switched items), brief desync visible

**Buffered attacks:**
- If client sends attack input 0–200 ms before cooldown expires, server **buffers** the attack
- `_bufferAttack()` in `inventoryItem.ts` schedules the actual attack for when cooldown allows
- Prevents players from spamming and having attacks dropped

## Cooldown Enforcement

### Cooldown Check (Preflight)

```typescript
const timeToFire = cooldown - (now - this.lastUse);
const timeToSwitch = owner.effectiveSwitchDelay - (now - this.switchDate);

if (timeToFire <= 0 && timeToSwitch <= 0) {
  internalCallback.call(this);  // Attack now
} else {
  const bufferDuration = Numeric.max(timeToFire, timeToSwitch);
  if (bufferDuration >= 200) return;  // Drop attack if wait > 200 ms

  owner.bufferedAttack?.kill();
  owner.bufferedAttack = owner.game.addTimeout(
    () => {
      if (owner.activeItem === this && owner.attacking) {
        this.useItem();  // Re-trigger attack
      }
    },
    bufferDuration
  );
}
```

### Cooldown Timing

**After attack executes:**

```typescript
if (definition.fireMode === FireMode.Auto || owner.isMobile) {
  clearTimeout(this._autoUseTimeoutID);
  this._autoUseTimeoutID = setTimeout(
    () => this._useItemNoDelayCheck(false),
    (
      hits.length && definition.attackCooldown
        ? definition.attackCooldown
        : definition.cooldown
    ) - hitDelay
  );
}
```

**Logic:**
- If swing **hits** something and `attackCooldown` defined: next swing after `attackCooldown - hitDelay` ms
- If swing **misses**: next swing after `cooldown - hitDelay` ms
- `hitDelay` subtracted because next swing's `hitDelay` timer starts immediately

**Example: Sickle (auto-fire)**
```javascript
{
  cooldown: 160,
  attackCooldown: 140,
  hitDelay: 50,
  fireMode: FireMode.Auto
}
```

**Hit scenario:**
```
Player holds M
T=0:   Attack executes, hitDelay timer starts
T=50:  First hit resolves, target takes damage
T=90:  (50 + 140 - 50) Next swing executes
```
Effective: ~90 ms between swings when hitting.

**Miss scenario:**
```
Player holds M, swing misses (no targets in hitbox)
T=0:   Attack executes
T=50:  No hit, start cooldown
T=110: (50 + 160 - 50) Next swing allowed
```
Effective: ~110 ms between swings when missing.

### Non-Auto Weapons

Weapons without `fireMode: FireMode.Auto` require a **new click for each attack**. No automatic re-triggering; client must send new `InputPacket` with `PlayerActions.Attack`.

## Known Gotchas

### 1. **No Per-Swing Hit Deduplication**

**Issue:** Same target can be hit multiple times in one swing if:
- Swing has `numberOfHits > 1` and target hasn't moved
- Swing has `maxTargets: Infinity` and multiple target instances

**Example:**
```typescript
{
  numberOfHits: 2,
  delayBetweenHits: 415,
  maxTargets: 1  // Prevents multi-player hits, but same player can be hit twice
}
```

**Mitigation:** Weapons are tuned so `delayBetweenHits` is long enough for targets to escape, or `numberOfHits: 1` (single hit per swing).

### 2. **Raycasting 0.05 Unit Tolerance**

**Issue:** Wall occlusion check uses arbitrary 0.05 unit tolerance:
```typescript
// arbitrary 0.05 to fix a rare case where you can't hit players that are perfectly
// touching a wall because the distance between the player raycast and wall raycast is really similar
if (toTargetDist - 0.05 > obstacleRaycast.distance) continue;
```

Players standing **exactly on a wall** may sometimes fail to be hit if raycasting distances are too similar.

**Workaround:** Tune weapon `radius` to be generous; rely on tile-based spacing to prevent exact wall alignment.

### 3. **Hitbox is Circular Only**

**Issue:** No cone/fan shape, no vertical (Z) limit.
- Very wide `radius` can hit enemies around corners unexpectedly
- Hits any entity at any height on the same/adjacent layer

**Example:**
```
Player at (100, 100) swings Falchion (radius: 4.1)
Enemy at (103, 100) — hit (distance: 3.1 < 4.1)
Enemy at (100, 103) — hit (distance: 3.0 < 4.1)
Enemy at (100, -10)  — may hit if on same layer (even if on different floor)
```

**Tuning:** Smaller `radius` for precise weapons, larger for sweeping melee (baseball bat).

### 4. **Obstacles with `noCollisions` Break Raycasting Silently**

**Issue:** Bushes, vines, and other `noCollisions` obstacles don't block raycasting, so you can hit enemies inside them:
```typescript
if (object.definition.noCollisions || object.definition.isStair || ...) return false;
```

**Why:** Visual bushes shouldn't prevent swinging sword through them.

**Gotcha:** You can hit enemies inside bushes even though visually the swing doesn't reach.

### 5. **Client UI Cooldown ≠ Server Cooldown**

**Issue:** Client UI shows "ready to attack" but server still enforces cooldown.
```
Client shows: "Ready!" (100% cooldown UI)
Server state: cooldown check fails
User clicks rapidly
Some clicks dropped, others buffered
```

**Cause:** Network latency, plugin intervention, or InventoryItem switching.

**Mitigation:** Trust server-side enforce; UI is visual-only, not authoritative.

### 6. **`speedMultiplier: 1.0` = Normal Speed, Not 100% Slow**

**Confusing naming:**
- `speedMultiplier: 0.5` = 50% move speed (slow)
- `speedMultiplier: 1.0` = 100% move speed (normal, not slowed)
- `speedMultiplier: 1.5` = 150% move speed (fast)

**Applied during attack animation.** After attack ends, speed returns to normal.

### 7. **FireMode.Auto Not Inferred**

**Issue:** Weapon without `fireMode` field defaults to **click-to-attack** even if held:
```typescript
{
  idString: "hatchet",
  cooldown: 420
  // fireMode NOT specified → requires click for each swing
}

{
  idString: "sickle",
  cooldown: 160,
  fireMode: FireMode.Auto  // hold M = continuous swings
}
```

**Implication:** If you want auto-fire, **must explicitly set `fireMode: FireMode.Auto`**.

### 8. **Simultaneous Multi-Target Hits Not Prevented**

**Issue:** Weapons with `maxTargets: Infinity` can hit many players at once in same swing window:
```typescript
{
  maxTargets: Infinity,
  damage: 54
}
```

All valid targets in hitbox take damage at T=hitDelay. No "first come, first served" logic.

**Balancing:** High-damage, high-cooldown weapons (Maul) typically have `maxTargets: 1` to prevent overpowering groups.

### 9. **Network Lag Hit Validation**

**Issue:** Server-side hit detection is authoritative. If client predicts hit but server disagrees (due to lag/position desync), client shows hit effect but damage doesn't apply.

**Scenario:**
```
T=0:   Client shows player in attack range
       Client predicts hit, shows blood particles
T=100: Server receives hitbox check
       Enemy has moved, not in hitbox
       No damage applied, client rollback visible
```

**Mitigation:** Server is authoritative; trust HP updates from server.

## Complex Functions

### `_useItemNoDelayCheck(skipAttackCheck)` — @[server/src/inventory/meleeItem.ts:44](../../../server/src/inventory/meleeItem.ts#L44)

**Purpose:** Execute melee attack after cooldown/delay verification.

**Implicit behavior:**
- Sets `owner.animation = AnimationType.Melee` (broadcasts to all clients)
- Cancels any ongoing action (reload, heal, revive)
- Schedules hit detection after `hitDelay` ms
- For `numberOfHits > 1`, schedules each hit with `delayBetweenHits` stagger
- For `FireMode.Auto`, re-schedules next swing immediately (hold-to-attack behavior)

**Called by:**
- `useItem()` → `_bufferAttack()` → `_useItemNoDelayCheck(true)` (when cooldown OK)
- `_autoUseTimeoutID` callback (for auto-fire weapons)

### `getMeleeTargets<T>(hitbox, definition, player, teamMode, objects)` — @[common/src/utils/gameHelpers.ts:78](../../../common/src/utils/gameHelpers.ts#L78)

**Purpose:** Filter targets from spatial grid, apply raycasting, return sorted by distance.

**Implicit behavior:**
- Raycasts from `player.position` to each potential target to check for wall occlusion
- For **Players/C4:** Target fails if wall between player and target (with 0.05 unit tolerance)
- For **Obstacles:** Raycasts all objects, skips if different obstacle blocks
- Sorts by distance, returns top `definition.maxTargets`

**Edge case:** Teammates in team mode are deprioritized but not excluded.

### `_bufferAttack(cooldown, internalCallback)` — @[server/src/inventory/inventoryItem.ts:267](../../../server/src/inventory/inventoryItem.ts#L267)

**Purpose:** Check cooldown, execute immediately if ready, else buffer for ≤200 ms.

**Implicit behavior:**
- `timeToFire = cooldown - (now - this.lastUse)`
- `timeToSwitch = owner.effectiveSwitchDelay - (now - this.switchDate)`
- If **both** are ≤0: execute immediately
- If wait is 0–200 ms: schedule callback for later
- If wait is >200 ms: drop attack (don't buffer)

**Called by:** `MeleeItem.useItem()`, `GunItem.useItem()`, `ThrowableItem.useItem()`

## Damage Multiplier Summary Table

| Scenario | Formula | Example |
|----------|---------|---------|
| Player hit, no perks | `damage × 1.0` | Hatchet (40) → 40 DMG |
| Player hit, Berserker | `damage × 1.1–1.5` | Hatchet (40) × 1.1 → 44 DMG |
| Player hit, Lycanthropy | `damage × 1.5–2.0` | Hatchet (40) × 1.5 → 60 DMG |
| Obstacle (normal) | `damage × obstacleMultiplier` | Hatchet (40) × 2.0 → 80 DMG to wood |
| Obstacle (impenetrable) | `damage × piercingMultiplier` | Hatchet (40) × 1.5 → 60 DMG to concrete |
| Obstacle (ice) | `damage × obstacleMultiplier × iceMultiplier` | Maul (54) × 2.0 × 5.0 → 540 DMG |
| Projectile (C4) | `damage × obstacleMultiplier` | Hatchet (40) × 2.0 → 80 DMG |

## Related Documents

### Tier 2
- [Melee Combat System](../README.md) — Subsystem overview, attack flow, definitions schema
- [Inventory System](../../inventory/README.md) — Item ownership, active item selection
- [Game Loop](../../game-loop/README.md) — Tick-based attack triggering, `game.addTimeout()`
- [Game Objects (Server)](../../game-objects-server/README.md) — `Player`, `Obstacle` damage handlers
- [Core Math & Physics](../../core-math-physics/README.md) — Spatial grid, raycasting, collision detection
- [Plugin System](../../plugin-system/README.md) — `player_start_attacking`, `inv_item_use` events

### Tier 1
- [Architecture Overview](../../architecture.md) — System design, object categories
- [Data Model](../../datamodel.md) — Damage system, object lifecycle

### Source Files
- **Attack logic:** [server/src/inventory/meleeItem.ts](../../../server/src/inventory/meleeItem.ts)
- **Definitions:** [common/src/definitions/items/melees.ts](../../../common/src/definitions/items/melees.ts)
- **Hit detection:** [common/src/utils/gameHelpers.ts](../../../common/src/utils/gameHelpers.ts)
- **Cooldown/buffer:** [server/src/inventory/inventoryItem.ts](../../../server/src/inventory/inventoryItem.ts)
- **Client animation:** [client/src/scripts/objects/player.ts](../../../client/src/scripts/objects/player.ts) (playAnimation method)
