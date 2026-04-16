# GunItem Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/inventory/README.md -->
<!-- @source: server/src/inventory/gunItem.ts -->

## Purpose
Implements all server-side gun firing logic: fire mode dispatch, fire rate enforcement, wall-clip bullet spawning, perk modifier stacking per shot, dual-gun barrel alternation, cyclic fire-delay override, and auto/burst rescheduling via native `setTimeout`.

## Key Files
| File | Purpose | Complexity |
|------|---------|------------|
| `server/src/inventory/gunItem.ts` | `GunItem` class — all gun state and firing logic | High |
| `server/src/inventory/inventoryItem.ts` | `InventoryItemBase` — base class with `_bufferAttack`, modifier system, `useItem` lifecycle | Medium |
| `common/src/definitions/items/guns.ts` | `GunDefinition` interface — static gun configuration read at fire-time | Medium |
| `server/src/inventory/action.ts` | `ReloadAction` — executes reload and chains partial reloads | Medium |

## Business Rules

- **Fire-delay gating** — `_bufferAttack` compares `game.now - lastUse` against `fireDelay * owner.fireRateMod`; inputs arriving up to 200 ms early are buffered via a `game.addTimeout`, inputs beyond 200 ms are dropped silently.
- **No-ammo handling** — When `ammo <= 0` before firing, `AnimationType.GunClick` is played and the function returns early without spawning bullets.
- **Infinite-ammo exception** — Guns with `definition.infiniteAmmo === true` skip the post-shot ammo decrement entirely.
- **Auto-reload on empty** — After the ammo decrement drives `ammo` to zero, a `game.addTimeout` is queued (delay = `fireDelay * fireRateMod`) to call `reload(true)`, which starts a `ReloadAction`.
- **FSA (first-shot accuracy)** — Spread is forced to `0` if `game.now - lastUse >= definition.fsaReset`. If `fsaReset` is `undefined`, FSA never resets (spread always accumulates normally).
- **Movement spread** — Uses `moveSpread` when `owner.isMoving`, `shotSpread` otherwise; both halved before applying to rotation offset.
- **Burst fire** — `_consecutiveShots` counts shots within a burst. When it reaches `burstProperties.shotsPerBurst`, the burst cooldown is applied and the next burst is triggered via `setTimeout` (native Node.js, not game timeout).
- **Auto-fire** — If `fireMode !== Single` (or `owner.isMobile`), subsequent shots are scheduled via `setTimeout` (`_autoFireTimeout`), scaled by `owner.fireRateMod`.
- **Dual-gun alternation** — `_altFire` toggles after every shot (non-burst) or after every complete burst; bullet spawn offset is `±definition.leftRightOffset * _altFire ? -1 : 1`.
- **Wall-clip prevention** — Before spawning, a line trace from gun origin to intended spawn position checks the spatial grid for obstacles/buildings. If an object is found closer than the intended position, the spawn point is clipped to the intersection minus a `0.2 + jitterRadius` retraction along the gun direction.
- **Cycle delay (DP-12)** — Guns with `definition.cycle` replace `fireDelay` with `definition.cycle.delay` every `cycle.shotsRequired` shots; the previous delay is restored on other shots.
- **Plugin override** — The `"inv_item_use"` plugin event is emitted before bullet spawning; a non-`undefined` return value aborts the fire entirely.
- **Reload guards** — `reload()` checks: `infiniteAmmo`, already at capacity (Extended Mags aware), no ammo in inventory, ongoing action, wrong active item, fire-delay not elapsed (unless `skipFireDelayCheck = true`), and `owner.downed`.

## GunDefinition — Fields Actually Used in GunItem

The following fields are directly read inside `gunItem.ts`. Fields that are part of `GunDefinition` but only consumed by other subsystems (rendering, loot spawn, etc.) are excluded.

| Field | Type | Purpose |
|-------|------|---------|
| `ammoType` | `ReferenceTo<AmmoDefinition>` | Which ammo pool to consume; checked in `reload()` |
| `capacity` | `number` | Normal magazine size; used as full-mag threshold |
| `extendedCapacity` | `number?` | Magazine size with `ExtendedMags` perk |
| `reloadTime` | `number` (s) | Duration of one reload cycle |
| `shotsPerReload` | `number?` | If set, reload adds this many rounds instead of filling magazine (pump-action style) |
| `infiniteAmmo` | `boolean?` | Skips ammo decrement and inventory-ammo check |
| `reloadFullOnEmpty` | `boolean?` | When `true` and mag runs dry, uses `fullReloadTime` |
| `fullReloadTime` | `number` (s) | Alternative reload time when `reloadFullOnEmpty` fires |
| `fireDelay` | `number` (ms) | Minimum time between shots for Single/Auto; inter-burst-shot cooldown for Burst |
| `fireMode` | `FireMode` | `Single`, `Burst`, or `Auto` |
| `burstProperties.shotsPerBurst` | `number` | Shots per burst before the burst cooldown applies |
| `burstProperties.burstCooldown` | `number` (ms) | Delay between bursts; also used as initial `this.fireDelay` in the constructor |
| `recoilMultiplier` | `number` | Speed multiplier applied to owner during recoil |
| `recoilDuration` | `number` (ms) | Duration of recoil speed penalty |
| `shotSpread` | `number` (degrees) | Max spread half-angle when stationary |
| `moveSpread` | `number` (degrees) | Max spread half-angle when moving |
| `fsaReset` | `number?` (ms) | Time after which spread resets to 0 (first-shot accuracy) |
| `jitterRadius` | `number?` | Radius of random position offset for each bullet spawn point |
| `consistentPatterning` | `boolean?` | Deterministic cubic spread pattern instead of random (shotguns) |
| `bulletCount` | `number?` (default 1) | Projectiles spawned per trigger pull |
| `bulletOffset` | `number?` | Single perpendicular spawn offset for non-dual guns |
| `bulletOffsets` | `number[]?` | Per-remaining-ammo offsets indexed by `ammo - 1` (e.g., double-barrel) |
| `length` | `number` | Barrel length; controls spawn distance and `rangeOverride` calculation |
| `isDual` | `boolean` | Whether this is a dual-wield variant; enables `_altFire` alternation |
| `leftRightOffset` | `number` | Perpendicular offset for dual guns |
| `singleVariant` | `ReferenceTo<GunDefinition>` | Single-gun counterpart — resolved inside `game.addBullet` to find bullet `idString` |
| `ballistics.lastShotFX` | `boolean?` | If `true`, `lastShot: true` is passed to `addBullet` when `ammo === 1` |
| `ballistics.onHitExplosion` | `string?` | Perk guard: Flechettes and SabotRounds skip if set |
| `ballistics.onHitProjectile` | `...?` | Perk guard: Flechettes and SabotRounds skip if set |
| `summonAirdrop` | `boolean?` | Perk guard: Flechettes and SabotRounds skip if set |
| `casingParticles[0].frame` | `string?` | HollowPoints guard: skips 12g shotguns whose frame includes `"slug"` |
| `cycle.delay` | `number` (ms) | Overrides `fireDelay` for one shot every `cycle.shotsRequired` shots |
| `cycle.shotsRequired` | `number` | Shots before cycle delay activates |

## FireMode Enum

Defined as `const enum` in `common/src/constants.ts`:

| Name | Value | Behavior |
|------|-------|----------|
| `FireMode.Single` | `0` | One shot per attack press; no auto-fire timeout (except on mobile where auto-fire is always used) |
| `FireMode.Burst` | `1` | Fires `burstProperties.shotsPerBurst` shots with `fireDelay` between them, then waits `burstCooldown`; `fireDelay` in constructor initialized to `burstCooldown` |
| `FireMode.Auto` | `2` | Fires continuously; after each shot a `_autoFireTimeout` reschedules `_useItemNoDelayCheck` with `fireDelay * fireRateMod` |

## GunItem Class — Properties

| Property | Type | Description |
|----------|------|-------------|
| `ammo` | `number` | Current rounds in the magazine (public) |
| `fireDelay` | `number` | Effective fire delay (ms); mutable — overridden by `cycle.delay` or reset to `burstCooldown` |
| `_consecutiveShots` | `number` (private) | Shots fired in the current burst or continuous sequence |
| `_shots` | `number` (private) | Lifetime shot counter; readable via `shots` getter |
| `_shotsCounter` | `number` (private) | Tracks shots for `cycle.shotsRequired` threshold |
| `_previousFireDelay` | `number?` (private) | Saves pre-cycle `fireDelay` so it can be restored |
| `_reloadTimeout` | `Timeout?` (private) | Game-managed timeout for post-empty auto-reload |
| `_burstTimeout` | `NodeJS.Timeout?` (private) | Native Node.js `setTimeout` handle for burst inter-burst delay |
| `_autoFireTimeout` | `NodeJS.Timeout?` (private) | Native Node.js `setTimeout` handle for repeated auto-fire |
| `_altFire` | `boolean` (private) | Which side fires next for dual-wield guns; `false` = right, `true` = left |

Note: `_burstTimeout` and `_autoFireTimeout` use native `setTimeout` (not the game's tick-aware `addTimeout`) because some guns' fire rates are too fast relative to the 40 TPS game tick.

## Firing Flow

### Entry Point (Player Tick)

```
player.startedAttacking = true (set from client input packet)
    → player tick: this.activeItem.useItem()
        → GunItem.useItem()
            → _bufferAttack(fireDelay * fireRateMod, _useItemNoDelayCheck.bind(this, true))
                → if cooldown elapsed AND switchDelay elapsed:
                    → _useItemNoDelayCheck(skipAttackCheck=true)   [immediate]
                → else if delay < 200ms:
                    → schedule game.addTimeout for remaining delay [buffered]
                → else: drop input
```

### Single Shot (`FireMode.Single`)

```
_useItemNoDelayCheck(skipAttackCheck)
  → guard checks (dead / downed / disconnected / wrong activeItem)
  → ammo <= 0? → GunClick animation, return
  → plugin "inv_item_use" → can abort
  → cancel owner.action, clear _burstTimeout
  → set animation (GunFire / GunFireAlt)
  → setPartialDirty(); dirty.weapons = true
  → ++_consecutiveShots, ++_shots
  → compute spread (FSA check, moveSpread vs shotSpread)
  → compute spawn position + wall-clip retraction
  → rangeOverride = distanceToMouse - definition.length
  → build modifiers, iterate perk switch
  → spawn bulletCount bullets via owner.game.addBullet()
  → apply recoil (recoilMultiplier, recoilDuration)
  → if !infiniteAmmo: --ammo
  → if ammo <= 0: schedule reload timeout, return
  → (no _autoFireTimeout scheduled for Single on non-mobile)
```

### Burst Fire (`FireMode.Burst`)

```
_useItemNoDelayCheck(skipAttackCheck)
  → [same guards + ammo check + animation + perk modifiers + bullet spawn]
  → --ammo
  → if ammo <= 0: schedule reload timeout, return
  → if _consecutiveShots >= burstProperties.shotsPerBurst:
      → _consecutiveShots = 0
      → fireDelay = burstProperties.burstCooldown
      → _burstTimeout = setTimeout(_useItemNoDelayCheck(false), burstCooldown)
      → if isDual: _altFire = !_altFire
      → return
  → else:
      → fireDelay = definition.fireDelay   [intra-burst rate]
      → _autoFireTimeout = setTimeout(_useItemNoDelayCheck(false), fireDelay * fireRateMod)
```

### Full-Auto (`FireMode.Auto`)

```
_useItemNoDelayCheck(skipAttackCheck)
  → [same guards + ammo check + animation + perk modifiers + bullet spawn]
  → --ammo
  → if ammo <= 0: schedule reload timeout, return
  → if isDual: _altFire = !_altFire
  → _autoFireTimeout = setTimeout(_useItemNoDelayCheck(false), fireDelay * fireRateMod)
```

## Ammo System

`ammo` is an instance property initialized to `0` at construction; the `ReloadAction` sets it to `definition.capacity` (or `extendedCapacity` with the `ExtendedMags` perk) when the reload animation completes. The ammo pool consumed is tracked in `owner.inventory.items` using `definition.ammoType` as the key. `reload()` checks `owner.inventory.items.hasItem(definition.ammoType)` before starting a `ReloadAction` (bypassed by `InfiniteAmmo` perk).

## Bullet Spawning

Each shot calls `owner.game.addBullet(this, owner, options)` once per projectile (`definition.bulletCount ?? 1` times). The `options` object passed:

| Field | Source |
|-------|--------|
| `position` | Wall-clip-corrected spawn point at `definition.length` from player |
| `rotation` | `owner.rotation + HALF_PI + spread` (spread is ±random in `[-1, 1] × spreadHalfAngle`) |
| `layer` | `getStartingLayer(position)` — stair-aware layer resolver |
| `rangeOverride` | `owner.distanceToMouse - definition.length` |
| `modifiers` | Accumulated from perk loop; `undefined` if no perk modified them |
| `saturate` | `true` if any perk made `damageMod > 1` |
| `thin` | `true` if any perk made `damageMod < 1` |
| `split` | `true` only for the leading Flechette sub-bullet of a group |
| `shotFX` | `true` for the first bullet (`i === 0`), `false` for subsequent |
| `lastShot` | `definition.ballistics.lastShotFX === true && ammo === 1` |
| `cycle` | `this.fireDelay === definition.cycle?.delay` — `true` when the cycle-delay is active for this shot |

## Dual-Gun Mechanic

`_altFire: boolean` tracks which barrel fires next. When `_altFire` is `false`, offset = `+leftRightOffset`; when `true`, offset = `−leftRightOffset`. The position offset is applied as `Vec.rotate(Vec(0, offset), owner.rotation)` to derive the start position perpendicularly.

After each shot (non-burst) or after a complete burst, `_altFire = !_altFire`. For burst mode, the toggle happens at burst boundary (after `burstCooldown` is applied). The `AnimationType.GunFireAlt` animation is played when `_altFire` is `true`.

## Perk Interactions

All perk logic runs inside a `for...of owner.perks` loop before bullets are spawned. Checks are `switch (perk.idString)`:

| Perk | Condition | Effect |
|------|-----------|--------|
| `Flechettes` (`damageMod 0.4`, `split 3`, `deviation 0.7°`) | Gun has no `ballistics.onHitExplosion`, no `ballistics.onHitProjectile`, no `summonAirdrop`, and `ammoType !== "bb"` | Sets `doSplinterGrouping = true`: each projectile fans into 3 sub-bullets using `(8 * (j/(split-1) - 0.5)^3) * dev + rotation`; `modifiers.damage *= 0.4` |
| `SabotRounds` (`damageMod 0.9`, `rangeMod 1.5`, `speedMod 1.5`, `spreadMod 0.6`, `tracerLengthMod 1.2`) | Same exclusion guards as Flechettes | `modifiers.range *= 1.5`, `modifiers.speed *= 1.5`, `modifiers.damage *= 0.9`, `modifiers.tracer.length *= 1.2`, `spread *= 0.6` |
| `CloseQuartersCombat` (`damageMod 1.2`, `reloadMod 1.3`, `cutoff 50`) | An alive, non-teammate enemy is within 50 units (checked via `grid.intersectsHitbox(CircleHitbox)`) | `modifiers.damage *= 1.2`; `owner.reloadMod = 1.3`. If no enemy in range: `owner.reloadMod = 1` |
| `Toploaded` (thresholds `[[0.2, 1.25], [0.49, 1.1]]`) | Always; uses `extendedCapacity` if `ExtendedMags` is also active | Based on `ratio = 1 - ammo/capacity`: `ratio ≤ 0.2` → `modifiers.damage *= 1.25`; `ratio ≤ 0.49` → `× 1.1`; `ratio > 0.49` → no bonus |
| `HollowPoints` (`damageMod 1.1`) | Skipped for `ammoType === "12g"` guns whose `casingParticles[0].frame` includes `"slug"` | `modifiers.damage *= 1.1` |
| `Overclocked` (`fireRateMod 0.65`, `spreadMod 2.0`) | Always | `spread *= 2.0` (wider spread per shot). The `fireRateMod 0.65` is a player-level modifier consumed via `owner.fireRateMod` when scheduling timeouts — not set inside this loop |
| `LastStand` (`damageMod 1.267`, `healthReq 30`) | `owner.health < 30` | `modifiers.damage *= 1.267` |

After the loop, `saturate = true` if any `damageMod > 1`; `thin = true` if any `damageMod < 1`. These control tracer visual rendering on the client.

## Spread and Accuracy

```
halfSpread = owner.isMoving ? moveSpread/2 : shotSpread/2   (degrees, converted to radians)
if (game.now - lastUse >= fsaReset ?? Infinity):
    spread = 0                   // FSA: first shot always accurate
else:
    spread = halfSpread

per-bullet rotation offset = randomFloat(-1, 1) * spread    // or cubic pattern if consistentPatterning
```

Perk modifiers (`SabotRounds` × 0.6, `Overclocked` × 2) are applied after the base spread is computed. `modifiersModified` is set to `true` only when at least one perk changes the modifiers object; otherwise `modifiers` is passed as `undefined` to `addBullet`.

## Wall Clipping Check

```
startPosition = player.position + rotate(Vec(0, offset), rotation)
targetPosition = player.position + rotate(Vec(length, offset), rotation) * sizeMod

// line-trace against all obstacles + buildings in grid
for each object in grid.intersectsHitbox(RectangleHitbox.fromLine(startPos, targetPos)):
    skip if: dead, no hitbox, not obstacle/building, different layer,
             noCollisions, or isStair
    intersection = object.hitbox.intersectsLine(startPos, targetPos)
    if intersection closer to startPos than current targetPos:
        targetPos = intersection.point - rotate(Vec(0.2 + jitterRadius, 0), rotation)
        // retract 0.2 + jitter units back along barrel axis
```

This prevents bullets from spawning inside walls. The `0.2` constant is a hardcoded retraction margin; `jitterRadius` is added so jittered spawn positions also clear the collision surface.

## Data Lineage

```
Player attack input (client InputPacket)
  → player.startedAttacking = true
  → player tick: activeItem.useItem()
      → GunItem.useItem()
          → _bufferAttack(cooldown, _useItemNoDelayCheck)
              → _useItemNoDelayCheck(skipAttackCheck)
                  → guard checks (dead / downed / disconnected / not activeItem)
                  → ammo <= 0 check → GunClick + return
                  → plugin "inv_item_use" → can abort
                  → animation set, player dirty flags set
                  → _consecutiveShots++, _shots++
                  → cycle.shotsRequired check → fireDelay override
                  → spread = FSA ? 0 : halfSpread (move or shot)
                  → spawn position + wall-clip retraction
                  → rangeOverride = distanceToMouse - length
                  → modifiers built from perk loop
                  → for 0..bulletCount: game.addBullet(this, owner, options)
                  → owner.recoil applied
                  → --ammo (unless infiniteAmmo)
                  → ammo == 0: schedule reload, return
                  → burst boundary or auto: schedule next _useItemNoDelayCheck
  → UpdatePacket marks player as partialDirty (animation + weapon state)
```

## Dependencies

- **Internal:**
  - `InventoryItemBase` (`inventoryItem.ts`) — provides `_bufferAttack`, `lastUse`, `switchDate`, `modifiers`, `stats` (kills/damage), `isActive`/`refreshModifiers`, and the `useItem()` / `stopUse()` / `destroy()` template
  - `ReloadAction` (`action.ts`) — instantiated by `reload()`; manages the reload timer and ammo restoration
- **External:**
  - [Game Loop / Bullets](../../game-loop/README.md) — `game.addBullet()` creates `Bullet` objects that travel and deal damage each tick
  - [Object Definitions](../../object-definitions/README.md) — `GunDefinition` config via `this.definition`; `Bullets.fromString()` resolves bullet `idString` inside `game.addBullet`
  - [Spatial Grid](../../spatial-grid/README.md) — wall-clip check uses `game.grid.intersectsHitbox(RectangleHitbox.fromLine(...))`; CQC perk uses `game.grid.intersectsHitbox(new CircleHitbox(cutoff, ownerPos), layer)`
  - [Networking](../../networking/README.md) — `owner.setPartialDirty()` / `owner.dirty.weapons = true` propagate gun state into the next `UpdatePacket`
  - [Plugin System](../../plugins/README.md) — emits `"inv_item_use"` (pre-fire hook, can cancel) and `"inv_item_stop_use"` (trigger release)

## Complex Functions

### `GunItem._useItemNoDelayCheck(skipAttackCheck)` — @file server/src/inventory/gunItem.ts
**Purpose:** The core fire routine. Validates all preconditions, processes perk modifiers, performs the wall-clip retraction, spawns all projectiles, applies recoil, decrements ammo, and schedules the next shot or reload.
**Implicit behavior:**
- Called with `skipAttackCheck = true` on the first press (via `useItem`) and `false` on timer callbacks; passing `false` allows the method to bail early if the player released fire mid-burst.
- Cancels `_burstTimeout` at entry to prevent double-fires if the burst timer and a new press coincide.
- When burst fires the last shot of a burst and `isDual`, it toggles `_altFire` at the burst boundary, not per-shot.
- The `cycle` flag sent to `addBullet` is `true` when `this.fireDelay === definition.cycle?.delay` — meaning the cycle delay IS currently active for this shot (e.g., DP-12 pump). `owner.isCycling` is set to `!cycle`, so it is `false` when the cycle delay is active and `true` during normal-rate shots.
**Called by:** `GunItem.useItem()` (via `_bufferAttack`), `_burstTimeout` callback, `_autoFireTimeout` callback

### `GunItem.useItem()` — @file server/src/inventory/gunItem.ts
**Purpose:** Public entry from the player tick; delegates to `_bufferAttack` with `fireDelay * owner.fireRateMod`.
**Implicit behavior:** Does not fire directly — input buffering in `_bufferAttack` may defer the actual call by up to 200 ms.
**Called by:** `player.ts` player tick when `startedAttacking` is true

### `GunItem.reload(skipFireDelayCheck)` — @file server/src/inventory/gunItem.ts
**Purpose:** Validates reload preconditions and starts a `ReloadAction` on the player.
**Implicit behavior:** Called with `skipFireDelayCheck = true` from the auto-reload timeout (which already waits `fireDelay`) and with `false` from manual reload key press (double-checks fire delay to prevent reload spam).
**Called by:** Auto-reload `_reloadTimeout`, player input handler (`InputActions.Reload`)

### `GunItem.cancelAllTimers()` — @file server/src/inventory/gunItem.ts
**Purpose:** Kills `_reloadTimeout` (game timeout) and clears `_burstTimeout` + `_autoFireTimeout` (Node.js native timeouts).
**Implicit behavior:** Must be called when the player drops, switches, or disconnects; called by the player weapon-switch logic.
**Called by:** Player weapon switch, `destroy()`

## Configuration

| Setting | Effect | Source |
|---------|--------|--------|
| `definition.fireDelay` | Base time between shots (ms) | Gun definition |
| `definition.burstProperties.burstCooldown` | Delay between bursts; also initial `fireDelay` for Burst guns | Gun definition |
| `definition.burstProperties.shotsPerBurst` | How many shots per burst | Gun definition |
| `definition.reloadTime` | Duration of one reload cycle | Gun definition |
| `definition.fullReloadTime` | Reload time used when `reloadFullOnEmpty` fires | Gun definition |
| `definition.cycle.delay` | Overridden `fireDelay` during cycle | Gun definition |
| `definition.cycle.shotsRequired` | Shots before cycle delay activates | Gun definition |
| `owner.fireRateMod` | Player-level multiplier applied to all fire-delay timeouts | Player / Overclocked perk |
| `owner.reloadMod` | Multiplier applied to reload speed | Player / CQC perk |
| `GameConstants.player.defaultModifiers()` | Starting modifier values (all 1.0) | `common/src/constants.ts` |

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Inventory subsystem overview
- **Tier 3:** [actions.md](actions.md) — `ReloadAction` implementation
- **Tier 1:** [../../../architecture.md](../../../architecture.md) — System architecture
- **Tier 1:** [../../../datamodel.md](../../../datamodel.md) — Core definitions and `GunDefinition` interface overview
- **Patterns:** [../patterns.md](../patterns.md) — Inventory patterns
