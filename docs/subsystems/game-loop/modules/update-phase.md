# Game Loop — Update Phase

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/game-loop/README.md -->
<!-- @source: server/src/objects/player.ts -->

## Purpose

Detailed breakdown of the three-phase player update cycle that executes **once per tick** for each player. These phases handle movement, collision detection, visibility calculation, network serialization, and state cleanup.

## Overview

The game tick allocates three consecutive loops for player updates. Each loop processes all connected (or living) players in sequence:

- **Phase 1:** `player.update()` — Movement, collision, health/status state
- **Phase 2:** `player.secondUpdate()` — Visibility calculation & network sync  
- **Phase 3:** `player.postPacket()` — Dirty flag cleanup

**Timing:** ~25 ms per complete tick at 40 TPS. Player updates consume ~10–15 ms total (varies with player count).

---

## Phase 1: update() — Movement & State

**Called from:** Game.tick() step 13 — loops over `this.livingPlayers`  
**Source:** [server/src/objects/player.ts:1125](server/src/objects/player.ts#L1125)  
**Typical cost:** ~8 ms for all players combined

### Subsections

#### 1a. Building & Smoke Detection

```typescript
// Check if player is inside a building (for ceiling scope, infection damage)
// Check if player overlaps with synced particles (smoke for speed mods)
for (const object of this.nearObjects) {
  if (object.isBuilding && object.scopeHitbox.collidesWith(this._hitbox)) {
    isInsideBuilding = true;
    scopeTarget = object.definition.ceilingScope;
  }
  if (object.isSyncedParticle && object.hitbox.collidesWith(this._hitbox)) {
    syncedParticles.add(object);
  }
}
```

**State modified:** `this.isInsideBuilding`, `scopeTarget`, `syncedParticles` (local)

#### 1b. Speed Calculation

Composite speed multiplier from multiple sources:

```
Speed = baseSpeed
  × floorMultiplier
  × recoilMultiplier
  × perkSpeedMod (Claustrophobic, Advanced Athletics)
  × adrenalineSpeedMod
  × actionSpeedMultiplier
  × itemSpeedMultiplier
  × effectSpeedMultiplier
  × playerModifiers.baseSpeed
```

**Perk interactions:**
- **Claustrophobic:** `× speedMod` if inside building
- **Advanced Athletics:** 
  - Water speed modifier if standing in water
  - Smoke speed modifier if in smoke cloud

**Adrenaline boost** (logarithmic formula):
```
y = b·log(x + d) / log(c) + a
  where x = current adrenaline, y = multiplier ≈ 0.944 to 1.15
```

**State modified:** `speed` (local calculation only)

#### 1c. Movement Vector Calculation

Combines player input (WASD or mobile joystick) with speed:

```typescript
// Handle diagonal movement normalization
let x = +playerMovement.right - +playerMovement.left;
let y = +playerMovement.down - +playerMovement.up;

if (x * y !== 0) {
  // Both x and y are non-zero → diagonal movement
  x *= Math.SQRT1_2;
  y *= Math.SQRT1_2;
}

// Apply Aching Knees perk: reverse movement
if (this.hasPerk(PerkIds.AchingKnees) && this.reversedMovement) {
  movement.x *= -1;
  movement.y *= -1;
}

desiredVelocity = Vec.scale(movement, speed);
```

**On ice:** Uses acceleration-based movement with friction instead of instant velocity.

**State modified:** `this._movementVector`, `movement` (local)

#### 1d. Position Update & Collision Detection

```typescript
// 1. Add movement to position
this.position = Vec.add(this.position, Vec.scale(this.movementVector, dt));

// 2. Update spatial grid
this.nearObjects = this.game.grid.intersectsHitbox(this._hitbox, this.layer);

// 3. Multi-step collision resolution (up to 10 iterations)
for (let step = 0; step < 10; step++) {
  for (const potential of this.nearObjects) {
    if (potential.collidable && potential.hitbox.collidesWith(this._hitbox)) {
      if (isStair) {
        // Layer transition
        potential.handleStairInteraction(this);
      } else if (!this._noClip) {
        // Resolve collision push
        this._hitbox.resolveCollision(potential.hitbox);
        
        // Damage from activated obstacles (spikes, saw blades)
        if (potential.definition.damage) {
          this.damage({
            amount: potential.definition.damage,
            source: DamageSources.Obstacle,
            weaponUsed: potential
          });
        }
      }
    }
  }
  if (!collided) break;
}

// 4. Clamp to world boundaries
this.position.x = clamp(this.position.x, radius, mapWidth - radius);
this.position.y = clamp(this.position.y, radius, mapHeight - radius);
```

**Collision algorithm:** Hitbox sweep-and-resolve collision via `RectangleHitbox.resolveCollision()`. Runs up to 10 corrections per tick to prevent getting stuck.

**State modified:** `this.position`, `this.nearObjects`, `this.floor`, `this.isMoving`, `mapIndicator.position`

#### 1e. Health & Shield Regen

```typescript
// Health regen from modifiers + adrenaline
let toRegen = this._modifiers.hpRegen;

if (this._adrenaline > 0) {
  const adrenRegen = logarithmicFormula(this._adrenaline);
  
  if (this.hasPerk(PerkIds.LacedStimulants)) {
    // Different rates based on health threshold
    adrenRegen *= (this.health <= threshold ? 1 : -carryoverRate);
  }
  toRegen += adrenRegen;
}

this.health += dt / 1000 * toRegen;

// Shield regen (separate)
if (this.hasBubble) {
  this.shield += dt / 1000 * this._modifiers.shieldRegen;
}

// Infection regen (special for Infected perk)
if (this.hasPerk(PerkIds.Infected)) {
  this.infection += dt / 1000;
}
```

**State modified:** `this.health`, `this.adrenaline`, `this.shield`, `this.infection`

#### 1f. Weapon Firing & Item Use

```typescript
// Dequeue queued input packets
if (this.startedAttacking) {
  if (this.game.pluginManager.emit("player_start_attacking", this) === undefined) {
    this.startedAttacking = false;
    this.activeItem.useItem();  // Fire gun or throw item
  }
}

if (this.stoppedAttacking) {
  if (this.game.pluginManager.emit("player_stop_attacking", this) === undefined) {
    this.stoppedAttacking = false;
    this.activeItem.stopUse();
  }
}
```

**State modified:** `startedAttacking`, `stoppedAttacking` (reset), bullets/projectiles added to game

#### 1g. Damage Over Time: Gas, Bleed-Out, Status Effects

```typescript
// Gas damage (scaled by position within zone + time spent outside)
const applyScaleDamageFactor = (now - this.timeWhenLastOutsideOfGas) >= 10000;
if (gas.doDamage && gas.isInGas(this.position)) {
  this.piercingDamage({
    amount: gas.scaledDamage(this.position) 
      + (applyScaleDamageFactor ? extraScaleDamage : 0),
    source: DamageSources.Gas
  });
}

// Knock-down bleed-out (0.002 DPM = 2 DPS; kills in ~100 seconds)
if (this.downed && !this.beingRevivedBy) {
  this.piercingDamage({
    amount: GameConstants.player.bleedOutDPMs * dt,
    source: DamageSources.BleedOut
  });
}

// Status effect damage (from synced particles)
for (const particle of syncedParticles) {
  if (particle.definition.depletePerMs?.health) {
    this.piercingDamage({
      amount: depletion.health * dt,
      source: DamageSources.Gas  // reused for smoke effects
    });
  }
}
```

**State modified:** `this.health`, `this._adrenaline` (from particles), `this.additionalGasDamage`

#### 1h. Perk Updates (Interval-Based)

Perks with `updateInterval` tick every N milliseconds (e.g., `Bloodthirst` every 1000 ms):

```typescript
if (this.perkUpdateMap !== undefined) {
  for (const [perk, lastUpdated] of this.perkUpdateMap) {
    if (game.now - lastUpdated <= perk.updateInterval) continue;
    
    this.perkUpdateMap.set(perk, game.now);
    
    switch (perk.idString) {
      case PerkIds.Bloodthirst:
        this.piercingDamage({
          amount: perk.healthLoss,
          source: DamageSources.BleedOut
        });
        break;
      case PerkIds.BabyPlumpkinPie:
        this.swapWeaponRandomly(undefined, true);
        break;
      case PerkIds.TornPockets:
        // Random ammo drops
        break;
      case PerkIds.Necrosis:
        // Bleed self + infect nearby players
        break;
      // ... many more evil perks ...
    }
  }
}
```

**Affected perks:** `Bloodthirst`, `BabyPlumpkinPie`, `TornPockets`, `RottenPlumpkin`, `Shrouded`, `Necrosis`, `Infected`, `AchingKnees`

**State modified:** `this.health` (multiple perks), `this.inventory`, `this.adrenaline`, perks list

#### 1i. Perk-Based Highlighting: Thermal Goggles & Hollow Points

```typescript
// Thermal Goggles: detect nearby enemy players
if (this.hasPerk(PerkIds.ThermalGoggles)) {
  const detectionHitbox = new CircleHitbox(perk.detectionRadius, this.position);
  for (const player of this.game.grid.intersectsHitbox(detectionHitbox)) {
    if (!player.isPlayer || player === this || player.dead || 
        this.isSameTeam(player)) continue;
    
    if (this.visibleObjects.has(player)) {
      this.highlightedPlayers.push(player);
    } else {
      // Create indicator for detected but not visible players
      this.highlightedIndicators.set(player, 
        new MapIndicator(this.game, "player_indicator", player.position));
    }
  }
}

// Hollow Points: highlight recently hit players
for (const [player, lastHitTime] of this.recentlyHitPlayers ?? []) {
  if (game.now - lastHitTime < highlightDuration) {
    indicator.updatePosition(player.position);
  }
}
```

**State modified:** `this.highlightedPlayers`, `this.highlightedIndicators`, `this.dirty.highlightedPlayers`

#### 1j. Eternal Magnetism Perk: Health & Loot Drain

```typescript
if (this.hasPerk(PerkIds.EternalMagnetism)) {
  const detectionHitbox = new CircleHitbox(perk.radius, this.position);
  
  // Find enemy players & loot in radius
  for (const object of this.game.grid.intersectsHitbox(detectionHitbox)) {
    // Players: drain their health, heal self
    if (object.isPlayer && object.health > minHealth 
        && this.health < maxHealth) {
      object.health -= perk.depletion;
      this.health += perk.depletion;
    }
    
    // Loot: push toward player
    if (object.isLoot) {
      object.push(angle, perk.lootPush);
    }
  }
}
```

**State modified:** `this.health`, `other.health` (enemy), `object.velocity` (loot)

#### 1k. Stuck Projectiles Update

Updates position/rotation of projectiles stuck in the player (e.g., seedshot seeds):

```typescript
if (this.stuckProjectiles) {
  for (const [proj, angle] of this.stuckProjectiles) {
    const finalAngle = rotate(this.rotation + angle);
    proj.position = this.position + offset;
    proj.rotation = finalAngle;
    proj.setPartialDirty();
  }
}
```

**State modified:** stuck projectile `position`, `rotation`, dirty flags

#### 1l. Automatic Doors

Opens/closes doors within 10 units when player is inside a building:

```typescript
for (const door of this.game.grid.intersectsHitbox(new CircleHitbox(10, this.position), this.layer)) {
  if (door.definition.automatic && !door.door.isOpen && isInsideBuilding) {
    if (distance(door, this) < 100) {
      door.interact();  // Open
    }
  }
}
```

**State modified:** door `isOpen` state

#### 1m. Revive Action Range Check

Cancels revive if target player moves >5 units away:

```typescript
if (this.action instanceof ReviveAction) {
  if (squaredDistance(this.position, this.action.target.position) >= 7 ** 2) {
    this.action.cancel();
  }
}
```

**State modified:** `this.action` (cancelled)

### Pseudo-Code Summary

```
update() {
  // 1. Building & smoke detection
  // 2. Speed calculation (8 perk sources + adrenaline)
  // 3. Movement vector from input + perk reversal
  // 4. Position update + 10-step collision resolution
  // 5. Health/shield/infection regen
  // 6. Fire/stop attacking
  // 7. Gas & bleed-out damage
  // 8. Perk interval updates (Bloodthirst, Necrosis, etc.)
  // 9. Thermal Goggles & Hollow Points highlighting
  // 10. Eternal Magnetism field
  // 11. Stuck projectile position tracking
  // 12. Automatic door open/close
  // 13. Revive action range check
}
```

**Total state mutations per Phase 1:**
- `position`, `velocity`, `floor`, `health`, `shield`, `adrenaline`, `infection`
- `isMoving`, `isInsideBuilding`, `effectiveScope`
- Perks list, highlighted players, stuck projectiles, map indicators

---

## Phase 2: secondUpdate() — Visibility & Network Sync

**Called from:** Game.tick() step 15 — loops over `this.connectedPlayers`  
**Source:** [server/src/objects/player.ts:1799](server/src/objects/player.ts#L1799)  
**Typical cost:** ~5 ms for all players combined

### Subsections

#### 2a. Visibility Calculation

Re-calculated every 8 ticks (200 ms) or when `updateObjects` flag is set:

```typescript
this.ticksSinceLastUpdate++;
if (this.ticksSinceLastUpdate > 8 || game.updateObjects || this.updateObjects) {
  this.ticksSinceLastUpdate = 0;
  this.updateObjects = false;
  
  // Compute screen hitbox from zoom level
  const zoom = this._zoomOverride !== 0 ? this._zoomOverride : player.effectiveScope.zoomLevel;
  const dim = zoom * 2 + 8;
  this.screenHitbox = RectangleHitbox.fromRect(dim, dim, player.position);
  
  // Broad-phase: grid query
  const newVisibleObjects = game.grid.intersectsHitbox(this.screenHitbox);
  
  // Delta: compute deleted objects
  packet.deletedObjects = [];
  for (const object of this.visibleObjects) {
    if (newVisibleObjects.has(object)) continue;
    this.visibleObjects.delete(object);
    packet.deletedObjects.push(object.id);
  }
  
  // Delta: compute new objects → full serialization
  for (const object of newVisibleObjects) {
    if (this.visibleObjects.has(object)) continue;
    this.visibleObjects.add(object);
    fullObjects.add(object);  // Mark for full serialization
  }
}
```

**Cost:** Grid queries are O(k) where k = objects in ~64-cell region around player. Avoids full-game O(n²) loop.

**State modified:** `this.visibleObjects`, `this.screenHitbox`, `packet.deletedObjects`, `fullObjects` (local set)

#### 2b. Partial & Full Dirty Object Serialization

```typescript
// Track full dirty objects visible to this player
for (const object of game.fullDirtyObjects) {
  if (!this.visibleObjects.has(object)) continue;
  fullObjects.add(object);
}

packet.fullObjectsCache = fullObjects;

// Track partial dirty objects (exclude if also full-dirty)
packet.partialObjectsCache = [];
for (const object of game.partialDirtyObjects) {
  if (!this.visibleObjects.has(object) || fullObjects.has(object)) continue;
  packet.partialObjectsCache.push(object);
}
```

**Optimization:** Avoids sending duplicate serialization for objects that are both full and partial dirty.

**State modified:** `packet` fields

#### 2c. Player Data Serialization

Conditionally populates `packet.playerData` (spectating player or self):

| Field | Serialized If | Contains |
|-------|---------------|----------|
| `health` | `player.dirty.health` | Normalized health (0–1) |
| `shield` | `player.dirty.shield` | Shield amount |
| `adrenaline` | `player.dirty.adrenaline` | Adrenaline amount |
| `infection` | `player.dirty.infection` | Infection amount |
| `weapons` | `player.dirty.weapons` | Active weapon index + ammo/count for all inventory slots |
| `items` | `player.dirty.items` | Backpack item counts + equipped scope |
| `zoom` | `player.dirty.zoom` | Scope zoom level |
| `perks` | `player.dirty.perks` | Array of equipped perks |
| `teammates` | `player.dirty.teammates` | Array of team members (team mode only) |
| `highlightedPlayers` | `player.dirty.highlightedPlayers` | Players detected by Thermal Goggles / Hollow Points |
| `activeC4s` | `player.dirty.activeC4s` | Boolean: player has active C4 detonators |

**Optimization:** Only serializes fields that changed (dirty flag) vs. previous tick.

**State modified:** `packet` fields

#### 2d. Bullet Culling

Culls bullets (projectiles that spawned this tick) based on trajectory intersection:

```typescript
packet.bullets = [];
for (const bullet of game.newBullets) {
  // Check if bullet trajectory (line segment) intersects player's view rect
  if (!Collision.lineIntersectsRectTest(
    bullet.initialPosition,
    bullet.finalPosition,
    this.screenHitbox.min,
    this.screenHitbox.max
  )) continue;
  
  packet.bullets.push(bullet);
}
```

**Known limitation:** Does not account for movement of the view rect during bullet transit (~0.3–0.8 s), causing rare "ghost bullets" on very slow projectiles (radio, firework launcher).

**State modified:** `packet.bullets`

#### 2e. Explosion Culling

Culls explosions based on distance & viewport intersection:

```typescript
packet.explosions = [];
for (const explosion of game.explosions) {
  if (!this.screenHitbox.isPointInside(explosion.position)
    || squaredDistance(explosion.position, this.position) > GameConstants.explosionMaxDistSquared
  ) continue;
  
  packet.explosions.push(explosion);
}
```

**State modified:** `packet.explosions`

#### 2f. Emotes, Gas, Kill Leader, & Metadata

```typescript
// Emotes: send only if emote player is visible
packet.emotes = [];
for (const emote of game.emotes) {
  if (!this.visibleObjects.has(emote.player)) continue;
  packet.emotes.push(emote);
}

// Gas state (if changed)
if (game.gas.dirty || this._firstPacket) {
  packet.gas = game.gas;
}

// Alive count (if changed)
if (game.aliveCountDirty || this._firstPacket) {
  packet.aliveCount = game.aliveCount;
}

// Kill leader (if changed)
if (game.killLeaderDirty || this._firstPacket) {
  packet.killLeader = {
    id: game.killLeader?.id ?? -1,
    kills: game.killLeader?.kills ?? 0
  };
}

// Planes, map pings, map indicators, new/deleted players
packet.planes = game.planes;
packet.mapPings = [...game.mapPings, ...this._mapPings];
packet.mapIndicators = indicators.filter(i => i.positionDirty || i.definitionDirty || i.dead);
packet.newPlayers = newPlayers.map(p => ({ id, name, badge, ... }));
packet.deletedPlayers = game.deletedPlayers;
```

**State modified:** `packet` fields (remaining)

#### 2g. Packet Serialization & Send

```typescript
// Serialize UpdatePacket to binary
this.sendPacket(packet as MutablePacketDataIn);

// Also send any queued non-UpdatePacket data
this._packetStream.stream.index = 0;
for (const packet of this._packets) {
  this._packetStream.serialize(packet);
}
for (const packet of this.game.packets) {
  this._packetStream.serialize(packet);
}

this._packets.length = 0;
this.sendData(this._packetStream.getBuffer());
```

**Network:** Uses `SuroiByteStream` binary serialization. UpdatePacket includes variable-length:
- Object delta (full/partial), visible players, bullets, explosions, etc.
- Typical frame: ~500–2000 bytes depending on activity

**State modified:** Network buffer sent, cleared local queues

### Pseudo-Code Summary

```
secondUpdate() {
  packet = UpdatePacket.create();
  
  // 1. Visibility (every 8 ticks): grid.intersectsHitbox(screenHitbox)
  //    → build fullObjects set, compute deleted objects
  // 2. Track full/partial dirty objects visible to this player
  // 3. Serialize player data (health, weapons, perks, etc.)
  // 4. Cull & include bullets (trajectory intersection)
  // 5. Cull & include explosions (distance + viewport)
  // 6. Include emotes, gas, kill leader, alive count
  // 7. Include new/deleted players, planes, map pings
  // 8. Serialize packet to binary & send
}
```

**Total state mutations per Phase 2:**
- `this.visibleObjects` (visibility set)
- Network buffer sent (no game state changes)
- `this._firstPacket` flag reset

---

## Phase 3: postPacket() — Cleanup

**Called from:** Game.tick() step 16 — loops over `this.connectedPlayers`  
**Source:** [server/src/objects/player.ts:2073](server/src/objects/player.ts#L2073)  
**Typical cost:** <1 ms

### Implementation

```typescript
postPacket(): void {
  // Reset all dirty flags for this playable state
  for (const key in this.dirty) {
    this.dirty[key as keyof Player["dirty"]] = false;
  }
  
  // Reset animation & action dirty flags
  this._animation.dirty = false;
  this._action.dirty = false;
}
```

**Purpose:** Prevents stale "dirty" flags from queuing unnecessary serialization on next tick.

**Dirty flags reset:**
- `health`, `shield`, `adrenaline`, `infection`
- `weapons`, `items`, `zoom`, `perks`, `layer`
- `highlightedPlayers`, `teammates`, `activeC4s`, `teamID`
- `animation`, `action`

**State modified:** All flags in `this.dirty` object

---

## Detailed Behavior: Damage Application

When any object calls `player.damage(params)`, the damage application sequence is:

```typescript
override damage(params: DamageParams): void {
  const { source, weaponUsed } = params;
  let { amount } = params;
  
  // 1. Invulnerability check
  if (this.invulnerable) return;
  
  // 2. Healing (negative damage)
  if (amount < 0) return this.heal(-amount);
  
  // 3. Emit plugin event
  this.game.pluginManager.emit("player_damage", {
    amount, player: this, source, weaponUsed
  });
  
  // 4. Armor reduction (if shield depleted)
  if (this.shield <= 0) {
    amount *= (1 - (
      (this.inventory.helmet?.damageReduction ?? 0)
      + (this.inventory.vest?.damageReduction ?? 0)
    ));
    
    // Perk: Last Stand —  reduce incoming damage
    amount *= this.mapPerkOrDefault(PerkIds.LastStand, 
      ({ damageReceivedMod }) => damageReceivedMod, 1);
  }
  
  // 5. Apply damage (calls piercingDamage internally)
  this.piercingDamage({ amount, source, weaponUsed });
}

piercingDamage(params: DamageParams): void {
  const { source, weaponUsed } = params;
  let { amount } = params;
  
  // 1. Team-based immunity check
  if (this.game.isTeamMode 
    && source instanceof Player 
    && source.teamID === this.teamID 
    && source.id !== this.id) {
    return;  // Can't damage teammates
  }
  
  // 2. Apply damage to health/shield
  if (this.shield > 0) {
    const initialShield = this.shield;
    this.shield -= amount;
    
    // Overflow: apply remaining damage to health
    if (this.shield < 0) {
      this.damage({ ...params, amount: -this.shield });
    }
  } else {
    this.health -= amount;
  }
  
  // 3. Track damage for statistics
  this.damageTaken += amount;
  
  if (sourceIsPlayer && source !== this) {
    source.damageDone += amount;
    this.lastDamagedBy = { player: source, weapon: weaponUsed, time: now };
  }
  
  // 4. Update weapon stats (if applicable)
  if (weaponUsed instanceof InventoryItemBase) {
    weaponUsed.stats.damage += amount;
    
    // Trigger on-damage effects
    for (const entry of weaponUsed.definition.wearerAttributes?.on?.damageDealt ?? []) {
      if (amount >= entry.limit) {
        source.health += entry.healthRestored ?? 0;
        source.adrenaline += entry.adrenalineRestored ?? 0;
      }
    }
  }
  
  // 5. Check for death
  if (this.health <= 0 && !this.dead) {
    if (this.game.isTeamMode && this._team?.hasAliveMembers() && !this.downed) {
      this.down(source, weaponUsed);  // Knock down, not dead
    } else {
      weaponUsed.stats.kills += 1;  // Track kill
      this.die(params);
    }
  }
  
  // 6. Emit plugin event
  this.game.pluginManager.emit("player_did_piercing_damaged", {
    player: this, amount, source, weaponUsed
  });
  
  // 7. Update modifiers
  this.updateAndApplyModifiers();
}
```

### Pseudo-Flow

```
damage(amount) {
  if (invulnerable) return;
  if (shield > 0) apply to shield, overflow to health;
  else apply to health directly;
  
  if (amount < 0) → heal instead;
  
  armor_reduction = sum(helmet.reduction, vest.reduction);
  amount *= (1 - armor_reduction);
  
  amount *= LastStand perk multiplier;
  
  → piercingDamage(amount);
}

piercingDamage(amount) {
  → update this.health or this.shield;
  → check if health ≤ 0:
     if yes → down() or die();
     
  → track damageTaken, damageDone;
  → update weapon.stats.damage/kills;
  → trigger weapon on-hit effects;
  → emit plugin events;
  → applyModifiers();
}
```

---

## Gotchas & Optimization Notes

### Ordering Guarantees

1. **Position finalized before visibility** — Phase 1 completes all movement, so Phase 2 sees the correct positions.
2. **Visibility recalculated lazily** — Only every 8 ticks (~200 ms) unless `updateObjects` flag is set. This means if a player spawns at tick 5, players won't see them until tick 8 (potentially 150 ms delay).
3. **Dirty flags reset after packet send** — Phase 3 clears flags, so only changes to state between tick N and N+1 trigger serialization.
4. **Damage from bullets queued before `update()` applies** — Bullets fired on tick N deal damage to players in that same tick's Phase 1, but visibility is from 200 ms ago (tick N-8 or so).

### Performance Characteristics

- **Spatial grid:** O(k) where k = cells around player. ~32×32 cell grid → typical k ≈ 9–16 cells.
- **Collision detection:** 10 iterations × O(k) per tick. Amortized O(1) per player in practice.
- **Visibility:** Grid query + `visibleObjects` Set diff. O(k + v) where v = visible objects (~50–200).
- **Perk updates:** Many branching checks. Estimated ~1 ms per player with unusual perk combos.
- **Damage calculation:** O(1) per damage event, but triggers plugin hooks + modifier updates.

### Common Bugs

1. **Ghost bullets** — Slow projectiles (radio, firework) can exit viewport during transit. Culling only checks start/end trajectory, not intermediate positions.
2. **Team damage edge case** — Revive action on teammates uses `piercingDamage()` which checks team immunity. Must explicitly set `source !== teammate` to bypass.
3. **Visibility delay** — 8-tick cache means new players are invisible to others for up to 200 ms. Mitigated by forcing `updateObjects = true` when players join.
4. **Collision loop infinite** — If an object is unmovable and blocked all 10 steps, player gets stuck. Rare in practice due to level design.

---

## Dependencies

### Depends on:

- [Networking](../../networking/) — `UpdatePacket` structure, `SuroiByteStream` serialization
- [Core Math & Physics](../../core-math-physics/) — Vector math, `Hitbox.resolveCollision()`, distance calculations
- [Spatial Grid](../../spatial-grid/) — `Grid.intersectsHitbox()` for visibility & collision
- [Serialization System](../../serialization-system/) — Object serialization (partial/full)
- [Object Definitions](../../object-definitions/) — Perk definitions, armor reduction, scope zoom levels
- [Status Effects & Perks](../../perks-passive/) — Perk logic (Thermal Goggles, Eternal Magnetism, etc.)
- [Inventory](../../inventory/) — Weapon stats tracking, equipped scope/armor

### Depended on by:

- [Game Loop](../README.md) — called from `Game.tick()` every tick
- [Visibility & LOS](../../visibility-los/) — uses visible objects calculated here
- [Networking](../../networking/) — produces `UpdatePacket` for transmission

---

## Related Documents

### Tier 2

- [Game Loop](../README.md) — full subsystem overview and tick orchestration
- [Networking](../../networking/) — packet protocol and binary serialization
- [Spatial Grid](../../spatial-grid/) — broad-phase collision & visibility queries
- [Inventory](../../inventory/) — weapon firing and item usage

### Tier 3

- [tick.md](tick.md) — high-level tick sequence
- Other modules in [Game Loop](../modules/)

### Tier 1

- [Architecture](../../../architecture.md) — system overview
- [Data Model](../../../datamodel.md) — entity relationships

### Patterns

- [patterns.md](../patterns.md) — subsystem patterns (dirty flagging, delta encoding)
