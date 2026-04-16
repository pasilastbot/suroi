# Equipment & Scope System

<!-- @tier: 2 -->
<!-- @parent: docs/architecture.md -->
<!-- @source: common/src/definitions/items/scopes.ts, server/src/inventory/inventory.ts, server/src/objects/player.ts -->

## Purpose

Manages scope magnification, zoom state transitions, and scope-to-player binding. Scopes provide magnification for long-range aiming: players inherit their active scope's zoom level, which is synchronized to all clients via `UpdatePacket`. The system ensures smooth zoom transitions by handling scope swaps that reduce magnification (prevents visual pop-in).

## Key Files & Entry Points

| File | Purpose |
|------|---------|
| `common/src/definitions/items/scopes.ts` | `ScopeDefinition` — magnification, zoom level constants; `Scopes` registry |
| `server/src/inventory/inventory.ts` | `Inventory.scope` property — active scope storage and setter |
| `server/src/objects/player.ts` | `Player.effectiveScope` — getter/setter for active or temporary zoom state; `_tempScope`, `_scopeTimeout` |
| `common/src/packets/updatePacket.ts` | `PlayerData.zoom` — uint8 serialization of scope zoom level |

## Scope Types

All scopes are instances of `ScopeDefinition` and stored as a flat registry in `Scopes` (see `common/src/definitions/items/scopes.ts`). Each scope has:

- **idString** — unique identifier (e.g., `"1x_scope"`)
- **name** — human-readable (e.g., `"1× Scope"`)
- **zoomLevel** — uint8 value sent to client (e.g., 70 = 1x magnification)
- **noDrop, giveByDefault** — whether scope spawns in loot or is given by default

| Scope | Zoom Level | Notes |
|-------|------------|-------|
| **1× Red Dot** | 70 | Default spawn; given to all players on spawn |
| **2× Scope** | 100 | Lightweight general-purpose scope |
| **4× ACOG** | 130 | Mid-range precision aiming |
| **8× Scope** | 160 | Long-range targeting |
| **16× Scope** | 220 | Extreme-range precision (reserved value: 190 for potential 12× scope) |

See `common/src/definitions/items/scopes.ts` for the scope definitions and registry.

## Scope Assignment & Storage

### Player Carries One Active Scope

Each `Player` instance maintains exactly one active scope via the `Inventory` class:

```typescript
// In server/src/inventory/inventory.ts
class Inventory {
    private _scope!: ScopeDefinition;
    get scope(): ScopeDefinition { return this._scope; }
    set scope(scope: ReifiableDef<ScopeDefinition>) {
        this._scope = Loots.reify<ScopeDefinition>(scope);
        this.owner.dirty.items = true;
    }
}
```

**Initialization:** On spawn, the `Inventory` gives the player their default scope:

```typescript
// From inventory.ts initialization
scope ??= Scopes.definitions[0].idString;  // Defaults to "1x_scope"
```

**Pickup:** When a player picks up a scope item, it replaces the current scope (scopes are singular, not stackable):

```typescript
case DefinitionType.Scope:
    this.scope = item.idString;
    break;
```

### Effective Scope (with Transition Logic)

The `Player` class maintains `effectiveScope` with temporary zoom state to prevent visual pop-in:

```typescript
// In server/src/objects/player.ts
private _scope!: ScopeDefinition;
private _tempScope: ScopeDefinition | undefined;
private _scopeTimeout?: Timeout;

get effectiveScope(): ScopeDefinition { return this._tempScope ?? this._scope; }

set effectiveScope(target: ReifiableDef<ScopeDefinition>) {
    const scope = Scopes.reify(target);
    if (this._scope === scope) return;

    // Prevent pop-in when stepping down from a high-zoom scope
    this._scopeTimeout?.kill();
    if ((this._scope?.zoomLevel ?? 0) > scope.zoomLevel) {
        this._tempScope = this._scope;          // Hold old scope
        this._scope = scope;                    // Update actual scope
        this._scopeTimeout = this.game.addTimeout(() => {
            this._tempScope = undefined;        // Release old scope after 2000ms
            this.updateObjects = true;
        }, 2000);
    } else {
        this._tempScope = undefined;            // No transition needed
        this.updateObjects = true;
    }

    this._scope = scope;
    this.dirty.zoom = true;
}
```

**Why two-scope system?**
- **Zoom-down transition:** When swapping from 8× to 1×, immediately sending zoom=70 to the client would cause visible objects to suddenly "pop in" at full resolution. The temporary scope keeps rendering the 8× zoom for 2000ms while the server knows the player now has a 1× scope, then releases the old zoom.
- **No pop-in on zoom-up:** Swapping from 1× to 8× has no delay — the client's camera animates smoothly to the larger FOV.

## Scope Synchronization

### Zoom in UpdatePacket

The server sends each player's effective scope zoom level in the `UpdatePacket` as a single uint8:

```typescript
// From common/src/packets/updatePacket.ts
const hasZoom = zoom !== undefined;
// ...
if (hasZoom) {
    strm.writeUint8(zoom);  // effectiveScope.zoomLevel
}
```

**Client receives:**
```typescript
if (hasZoom) {
    data.zoom = strm.readUint8();
}
```

This allows clients to render camera zoom dynamically per player (useful for spectating a zoomed-in player).

### Scope Definition in ItemData

When items are included in `UpdatePacket`, the player's active scope is serialized:

```typescript
const { items: invItems, scope } = items;
// ...
Scopes.writeToStream(strm, scope);    // Binary write using ObjectDefinitions registry
```

Client reads back:
```typescript
scope: Scopes.readFromStream(strm)
```

**Serialization efficient:** Since `Scopes` is a small registry (5 scopes), each scope is written as a single-byte index into the registry, not a string.

## Known Issues & Gotchas

### 1. Scope Zoom Values Are Not Pixel-Perfect Magnification
Zoom levels (70, 100, 130, 160, 220) are **not** simple 1x/2x/4x multipliers — they are quantized values passed to the client's camera system. Exact visual magnification depends on client-side camera easing and viewport dimensions. A "4× scope" (zoomLevel=130) may not yield exactly 4x pixel magnification on all screens.

### 2. Two-Scope Transition Hardcoded to 2000ms
The pop-in prevention timeout is hardcoded to 2000ms. If scope swaps happen frequently or the network is slow, the temporary scope may be released before the client's camera has fully animated to the new zoom state, causing visual glitches.

### 3. Scope Swap Doesn't Interrupt Actions
Picking up a new scope (e.g., finding 8× on the ground) while the player is reloading or in an action will immediately update `player.inventory.scope` but won't call `effectiveScope` setter — the swap is silent and won't trigger the transition timeout. The zoom may appear to change instantly on the next packet cycle.

### 4. No Scope Prediction/Latency Compensation
When a player picks up a scope, the server immediately updates their effective zoom. If the network is laggy, the client may see the zoom change before the scope item appears in their inventory, creating visual desynchronization.

### 5. Scope State Is Per-Player, Not Per-Gun
Unlike weapon attachments in some shooters (e.g., a rifle has a 4× mounted, a pistol has iron sights), Suroi uses a single global scope per player. Swapping guns doesn't change scope — the same 8× you had on your rifle applies to your melee weapon (though melee and throwables don't use zoom).

## Dependencies

### Depends on:
- [Inventory System](../inventory/) — `Inventory.scope` property and item pickup/drop
- [Object Definitions](../object-definitions/) — `ScopeDefinition` schema and registry pattern
- [Networking](../networking/) — `UpdatePacket` serialization / deserialization
- [Game Objects (Server)](../game-objects-server/) — `Player.dirty.zoom`, `Player.game.addTimeout()`

### Depended on by:
- [Game Loop](../game-loop/) — reads `player.effectiveScope.zoomLevel` during tick
- [Camera Management](../camera-management/) — applies zoom visual on client
- [Serialization System](../serialization-system/) — `SuroiByteStream` I/O for scope registry

## Data Flow

```
1. Server: Player picks up 8× scope (loot/drop)
   ↓
2. Inventory.scope setter → executes
   ↓
3. Player.effectiveScope setter → compares zoom levels
   ├─ If zooming down: sets _tempScope, starts 2000ms timeout
   └─ If zooming up: clears _tempScope immediately
   ↓
4. Player.dirty.zoom = true
   ↓
5. Next UpdatePacket cycle:
   ├─ Server sends: zoom = effectiveScope.zoomLevel
   └─ Server sends: scope = inventory.scope (in ItemData)
   ↓
6. Client receives UpdatePacket
   ├─ Reads zoom uint8 → applies camera zoom
   └─ Reads scope definition → updates UI inventory
   ↓
7. Client camera eases from old zoom to new zoom over ~250ms
```

## Scope Definition Format

A `ScopeDefinition` in the common schema inherits from `ItemDefinition`:

```typescript
interface ScopeDefinition extends ItemDefinition {
    readonly defType: DefinitionType.Scope
    readonly zoomLevel: number           // uint8: sent to clients
    readonly giveByDefault?: boolean     // true if spawned on all players at game start
}

// Example in common/src/definitions/items/scopes.ts:
{
    idString: "4x_scope",
    name: "4× Scope",
    defType: DefinitionType.Scope,
    noDrop: false,
    giveByDefault: false,
    zoomLevel: 130
}
```

## Related Documents

### Tier 2
- [Inventory System](../inventory/) — scope storage and item pickup
- [Camera Management](../camera-management/) — client-side camera zoom animation
- [Networking](../networking/) — UpdatePacket structure
- [Game Objects (Server)](../game-objects-server/) — Player class definition
- [Object Definitions](../object-definitions/) — Registry pattern and serialization

### Tier 1
- [docs/architecture.md](../../architecture.md) — System overview
- [docs/datamodel.md](../../datamodel.md) — Item and player data schema

### Patterns
See [Inventory Patterns](../inventory/patterns.md) for how items (including scopes) are picked up and dropped.
