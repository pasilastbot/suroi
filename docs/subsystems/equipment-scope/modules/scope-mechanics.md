# Equipment & Scopes — Scope Mechanics Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/equipment-scope/README.md -->
<!-- @source: server/src/inventory/inventory.ts | client/src/scripts/managers/uiManager.ts -->

## Purpose

Implements scope equipping, unequipping, zoom animation, magnification levels, crosshair rendering per scope type, and field-of-view (FOV) reduction.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| `server/src/inventory/inventory.ts` | Scope getter/setter, active scope tracking | High |
| `client/src/scripts/managers/uiManager.ts` | Crosshair rendering, zoom animation, FOV update | High |
| `common/src/definitions/items/scopes.ts` | Scope definition schema (magnification, crosshair type) | Medium |
| `client/src/scripts/game.ts` | Camera FOV application during zoom | Medium |

## Business Rules

- **Scope Equipping:** Inventory tracks active scope via `inventory.scope` property
  - Default scope: `DEFAULT_SCOPE` (iron sights or built-in gun sight)
  - Scope set via `inventory.scope = scopeDefinition` or string IDstring
  - Scope persists across weapon swaps (global state per player)
  
- **Effective Scope Override:** Temporary scope substitution via `player.effectiveScope`
  - Examples: Smoke/gas clouds snap scope to `DEFAULT_SCOPE` (can't ADS through smoke)
  - Building interiors force `DEFAULT_SCOPE` (no ADS in tight spaces)
  - Persists for smoke duration; automatically reverts
  
- **Magnification:** Each scope defines zoom level (e.g., 2×, 4×, 8×)
  - Maps to camera FOV reduction: `baseFOV / magnification`
  - Example: Base FOV 90°, 2× scope → 45° (narrow view)
  
- **Zoom Animation:** Client-side easing of FOV transition
  - Non-instant to prevent disorientation
  - Duration: ~200–300 ms (scope-dependent, configurable)
  
- **ADS (Aim Down Sights) Mechanics:**
  - ADS triggered by right-click (secondary attack)
  - Gun must have scope in inventory (if default scope only, subtle ADS)
  - Camera zooms; crosshair changes to scope-specific design
  
- **Crosshair Types:**
  - **Iron Sights:** Simple circle with center dot
  - **Holographic/Red Dot:** Reticle pattern (custom sprite)
  - **Thermal:** Heat-map tinting (affects world rendering, not just UI)
  - **ACOG/Sniper:** Precise grid reticle
  
- **Layer/Smoke Scope Snapping:** Smoke/gas effects can force scope reset
  - Triggered by `SyncedParticleDefinition.snapScopeTo` field
  - Snap occurs before particle expires (prevents ADS through obstruction)

## Data Lineage

```
Scope State Management
│
├─ Server (Inventory)
│  ├─ inventory._scope = ScopeDefinition (current active scope)
│  ├─ Synced to client via UpdatePacket (dirty flag: inventory.items)
│  └─ Overridden by player.effectiveScope (temp override)
│
├─ Client (UIManager)
│  ├─ Receive UpdatePacket with scope change
│  ├─ Trigger zoom animation
│  │  ├─ Start: currentFOV
│  │  ├─ Target: baseFOV / scope.magnification
│  │  ├─ Duration: scope.zoomDuration ?? 300 ms
│  │  └─ Easing: easeInOutQuad (smooth acceleration/deceleration)
│  │
│  ├─ Update crosshair sprite
│  │  ├─ Hide previous crosshair
│  │  ├─ Load new crosshair image (from scope definition)
│  │  └─ Position at screen center
│  │
│  └─ Apply FOV to camera
│     ├─ Calculate zoom: baseFOV / magnification
│     ├─ Animate over 300 ms
│     └─ Update game.camera.zoom property
│
└─ Dynamic Override
   ├─ If player.effectiveScope != inventory.scope:
   │  └─ Use effectiveScope for zoom/crosshair (temporary)
   │
   └─ When override expires:
      └─ Revert to inventory.scope
```

## Complex Functions

### Scope Getter & Setter — @file server/src/inventory/inventory.ts:60–65

**Code:**
```typescript
private _scope!: ScopeDefinition;

get scope(): ScopeDefinition { return this._scope; }

set scope(scope: ReifiableDef<ScopeDefinition>) {
    this._scope = Loots.reify<ScopeDefinition>(scope);
    this.owner.dirty.items = true;  // Mark for network sync
}
```

**Implicit Behavior:**
- Setter triggers lazy reification (converts IDstring to object if needed)
- Sets dirty flag to notify server to broadcast scope change to all clients
- No immediate zoom animation (handled client-side on packet arrival)
- Scope change can occur mid-gunfight (no animation lock prevents swapping)

### Effective Scope Override — @file server/src/objects/player.ts:504–510

**Code:**
```typescript
get effectiveScope(): ScopeDefinition { return this._tempScope ?? this._scope; }

set effectiveScope(target: ReifiableDef<ScopeDefinition>) {
    this._tempScope = target ? Scopes.reify(target) : undefined;
}
```

**Pattern:**
- `_tempScope` nullish-coalesced: if set, use temp; otherwise use primary scope
- Setting to `undefined` clears override (reverts to primary)
- Used by smoke/gas/building checks (temporary FOV reset)

**Example (Smoke Cloud):**
```typescript
if (snapScopeTo && syncedParticle._lifetime - age >= scopeOutPreMs) {
    scopeTarget ??= snapScopeTo;  // Snap scope first time entering smoke
}
// ...later
this.effectiveScope = scopeTarget ?? this.inventory.scope;  // Apply or revert
```

### Zoom Animation — @file client/src/scripts/managers/uiManager.ts

**Conceptual Implementation:**
```typescript
// On scope change packet:
const oldFOV = this.game.camera.zoom;
const scopeMagnification = newScope.magnification ?? 1;
const targetFOV = baseFOV / scopeMagnification;

const duration = newScope.zoomDuration ?? 300;  // milliseconds

new Tween(this.game.camera, { zoom: targetFOV }, duration, easeInOutQuad);
```

**Animation Properties:**
- Start FOV: Current camera zoom (smooth from previous scope)
- Target FOV: `baseFOV / magnification`
- Easing: Ease-in-out-quad (acceleration then deceleration)
- Duration: Scope-defined (typically 200–500 ms)

**Example (4× Sniper Scope):**
- Base FOV: 90°
- Magnification: 4×
- Target FOV: 90 / 4 = 22.5°
- Duration: 400 ms
- Easing: easeInOutQuad (smooth acceleration/deceleration)

### Crosshair Rendering — @file client/src/scripts/managers/uiManager.ts

**Pattern (Pseudo-code):**
```typescript
function updateCrosshair(scope: ScopeDefinition): void {
    // Hide old crosshair
    this.previousCrosshair?.destroy();
    
    // Load new crosshair sprite
    if (scope.image !== undefined) {
        this.currentCrosshair = new SuroiSprite(scope.image);
        this.currentCrosshair.setVPos(screenCenter);  // Center of screen
        this.crosshairContainer.addChild(this.currentCrosshair);
    } else {
        // Iron sights: draw simple circle
        this.drawIronSights();
    }
}
```

**Scope Image Types:**
- **String IDstring:** Asset name in sprite registry (e.g., `"holographic_1x_crosshair"`)
- **Undefined:** Fall back to iron sights (circle + center dot)
- **Custom assets:** Loaded via Vite spritesheet plugin

### Field-of-View Reduction — @file client/src/scripts/game.ts

**Formula:**
```typescript
const magnification = scope.magnification ?? 1;
const zoomFactor = 1 / magnification;  // 2× scope = 0.5× FOV
this.camera.zoom = this.baseZoom * zoomFactor;
```

**Implication:**
- Zoom > 1 = narrower view (magnified)
- Zoom < 1 = wider view (unscoped/iron sights)
- Applied to all subsequent renders until scope changes

**Example (Progression):**
- Iron sights (1×): zoom = 1.0 (90° view)
- Red dot (1.5×): zoom = 1.5 (60° view, subtle magnification)
- Sniper (8×): zoom = 8.0 (11.25° view, extreme magnification)

### Scope Change Network Sync — @file server/src/objects/player.ts

**Dirty Flag:**
```typescript
set scope(scope: ReifiableDef<ScopeDefinition>) {
    this._scope = Loots.reify<ScopeDefinition>(scope);
    this.owner.dirty.items = true;  // Trigger UpdatePacket broadcast
}
```

**Broadcast:**
- Server, game loop: checks `dirty.items` flag
- Broadcasts `UpdatePacket` to all clients with new scope IDstring
- Client unpacks packet, triggers zoom animation

## Dependencies

**Internal:**
- [Inventory System](../../inventory/) — Scope storage and dirty flag
- [Perks & Passives](../../perks-passive/) — No direct dependency (scopes are equipment)
- [Particle System / Synced Particles](../../particle-system/) — Smoke/gas scope snapping
- [Camera Management](../../camera-management/) — FOV application and animation

**External:**
- Scope Definition Schema (`@common/definitions/items/scopes`)
- Tween system (smooth animation)
- Spritesheet asset registry (crosshair images)

## Configuration

| Setting | Effect | Type | Path |
|---------|--------|------|------|
| `ScopeDefinition.magnification` | Zoom level (1× = no zoom, 4× = 4× zoom) | number | Per-scope |
| `ScopeDefinition.image` | Crosshair sprite IDstring | string \| undefined | Per-scope |
| `ScopeDefinition.zoomDuration` | Zoom animation duration (ms) | number | Per-scope |
| `DEFAULT_SCOPE` | Fallback scope (iron sights) | string | `common/src/definitions/items/scopes.ts` |
| `Player.baseZoom` | Unscoped FOV (reciprocal of magnification) | number | 1.0 |
| `Camera.baseFOV` | Unscoped field-of-view (degrees) | number | 90 (estimate) |

## Implementation Gotchas

- **Sync Delay:** Scope change synced via UpdatePacket; network latency means server state leads client by ~100 ms
- **Animation Overlap:** Rapid scope swaps can queue multiple tween animations (oldest interrupted by newest)
- **Smoke Snap Timing:** `snapScopeTo` check must occur before scope is applied to camera (order-dependent)
- **Crosshair Null Case:** Undefined `image` field should default to iron sights, not blank screen
- **Magnification Edge Cases:** Magnification ≤ 0 causes division-by-zero in zoom calculation (safeguard with `?? 1`)

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Equipment & Scopes subsystem
- **Tier 1:** [../../../architecture.md](../../../architecture.md) — System architecture
- **Inventory:** [../../inventory/README.md](../../inventory/README.md) — Scope storage
- **Camera:** [../../camera-management/README.md](../../camera-management/README.md) — FOV and zoom
- **Particles:** [../../particle-system/README.md](../../particle-system/README.md) — Smoke/gas interactions
- **UI:** [../../ui-management/README.md](../../ui-management/README.md) — Crosshair rendering
