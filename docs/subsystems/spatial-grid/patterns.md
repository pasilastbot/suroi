# Spatial Grid — Patterns

<!-- @tier: 2 -->
<!-- @parent: docs/subsystems/spatial-grid/README.md -->

## Pattern: Object Registration

**When to use:** When a new game object enters the world (player join, loot/projectile/
decal/parachute/synced-particle spawn, etc.)

**Implementation:**

```typescript
// Inside Game (server/src/game.ts) — after constructing the object:
const loot = new Loot(this, def, position, count);
this.grid.addObject(loot);          // registers in pool + computes cells
// …
this.grid.addObject(player);        // same pattern for every object type
this.grid.addObject(projectile);
this.grid.addObject(syncedParticle);
this.grid.addObject(decal);
this.grid.addObject(parachute);
```

`addObject` internally calls `updateObject`, so no separate `updateObject` call is needed
at spawn time.

**Example files:** `@file server/src/game.ts` (lines 789, 1041, 1110, 1127, 1168, 1344),
`@file server/src/utils/grid.ts`

---

## Pattern: Object Removal

**When to use:** When a game object is destroyed or leaves the world permanently.

**Implementation:**

```typescript
// Inside Game.removeObject (server/src/game.ts):
removeObject(object: GameObject): void {
    this.grid.removeObject(object);       // removes from cells + pool
    this._idAllocator.give(object.id);    // reclaims the numeric ID
    this.updateObjects = true;
}
```

Always go through `game.removeObject()` rather than calling `grid.removeObject()` directly,
so the ID allocator is kept consistent.

**Example files:** `@file server/src/game.ts` (line 1177),
`@file server/src/utils/grid.ts`

---

## Pattern: Object Movement Update

**When to use:** After any code changes `object._position` (player move, projectile tick,
loot physics settle, obstacle shift, synced-particle animation).

**Implementation:**

```typescript
// Called from each object's own tick / update method:

// server/src/objects/player.ts
this.game.grid.updateObject(this);

// server/src/objects/projectile.ts
this.game.grid.updateObject(this);

// server/src/objects/loot.ts
this.game.grid.updateObject(this);

// server/src/objects/syncedParticle.ts
this.game.grid.updateObject(this);

// server/src/objects/obstacle.ts
this.game.grid.updateObject(this);
```

`updateObject` runs `_removeFromGrid` then re-inserts, so it is a full remove-and-add.
It must be called **after** setting the new position value, not before.

**Example files:** `@file server/src/objects/player.ts` (line 1336),
`@file server/src/objects/loot.ts` (line 183),
`@file server/src/objects/projectile.ts` (line 280),
`@file server/src/objects/syncedParticle.ts` (line 132),
`@file server/src/objects/obstacle.ts` (line 572)

---

## Pattern: Broad-Phase Query (no layer filter)

**When to use:** Finding all objects near a position regardless of which layer they are on
(e.g., checking for nearby players when choosing a spawn location).

**Implementation:**

```typescript
// server/src/game.ts — spawn-position check
const radiusHitbox = new CircleHitbox(minSpawnDist, spawnPosition);
for (const object of this.grid.intersectsHitbox(radiusHitbox)) {
    if (object.isPlayer && !object.dead && …) {
        foundPosition = false;
    }
}
```

`intersectsHitbox` returns a `Set<GameObject>` — every object whose registered grid cells
overlap the hitbox's axis-aligned bounding rectangle. **Narrow-phase filtering** (actual
shape intersection, dead/alive checks, type guards) is the caller's responsibility.

**Example files:** `@file server/src/game.ts` (line 703),
`@file server/src/utils/grid.ts`

---

## Pattern: Broad-Phase Query with Layer Filter

**When to use:** Finding objects on a specific layer (e.g., checking only ground-level
obstacles during airdrop-crate placement).

**Implementation:**

```typescript
// server/src/game.ts — airdrop crate placement
const padded = thisHitbox.clone();
padded.scale(paddingFactor);

for (const object of this.grid.intersectsHitbox(padded, Layer.Ground)) {
    if (
        object.isObstacle
        && !object.dead
        && object.definition.indestructible
        && hitbox.collidesWith(thisHitbox)
    ) {
        // narrow-phase: resolve the actual collision
        thisHitbox.resolveCollision(object.spawnHitbox);
    }
}
```

Passing a `Layer` value to `intersectsHitbox` filters results using
`adjacentOrEquivLayer(object, layer)`, which includes objects on the same layer **and**
adjacent layers. Objects with `layer === undefined` are excluded when a layer is supplied.

**Example files:** `@file server/src/game.ts` (lines 1281, 1303),
`@file server/src/utils/grid.ts`

---

## Pattern: Per-Category Iteration via Pool

**When to use:** Processing every live object of a given type each tick (e.g., update all
`Loot` objects, advance all `Projectile` objects), without a spatial constraint.

**Implementation:**

```typescript
// server/src/game.ts — game tick
for (const loot of this.grid.pool.getCategory(ObjectCategory.Loot)) {
    loot.tick();
}
for (const projectile of this.grid.pool.getCategory(ObjectCategory.Projectile)) {
    projectile.tick();
}
for (const parachute of this.grid.pool.getCategory(ObjectCategory.Parachute)) {
    parachute.tick();
}
for (const syncedParticle of this.grid.pool.getCategory(ObjectCategory.SyncedParticle)) {
    syncedParticle.tick();
}
```

`pool.getCategory()` iterates the flat `ObjectPool` — no spatial filtering occurs. This
is the correct pattern when every instance of a type must be updated, not just nearby ones.

**Example files:** `@file server/src/game.ts` (lines 342–354),
`@file server/src/utils/grid.ts`
