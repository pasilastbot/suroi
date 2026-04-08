# Object Model Subsystem

<!-- @tier: 2 -->
<!-- @parent: docs/architecture.md -->
<!-- @modules: docs/subsystems/objects/modules/ -->
<!-- @source: common/src/utils/gameObject.ts, server/src/objects/, client/src/scripts/objects/ -->
<!-- @updated: 2026-03-04 -->

## Purpose

The Object Model subsystem defines the shared hierarchy and interfaces for all game objects. Both server and client extend a common base (`makeGameObjectTemplate()`) to create category-specific implementations. The server owns authoritative state; the client renders and interpolates.

## Key Files & Entry Points

| Package | File | Purpose |
|---------|------|---------|
| common | `common/src/utils/gameObject.ts` | `makeGameObjectTemplate()`, `CommonObjectMapping`, `BaseGameObject` type |
| common | `common/src/utils/objectsSerializations.ts` | `ObjectsNetData`, `ObjectSerializations` — full/partial serialize per category |
| server | `server/src/objects/gameObject.ts` | `BaseGameObject`, `ObjectMapping` — server base class |
| server | `server/src/objects/player.ts` | `Player` |
| server | `server/src/objects/obstacle.ts` | `Obstacle` |
| server | `server/src/objects/loot.ts` | `Loot` |
| server | `server/src/objects/building.ts` | `Building` |
| server | `server/src/objects/decal.ts` | `Decal` |
| server | `server/src/objects/parachute.ts` | `Parachute` |
| server | `server/src/objects/projectile.ts` | `Projectile` |
| server | `server/src/objects/syncedParticle.ts` | `SyncedParticle` |
| server | `server/src/objects/deathMarker.ts` | `DeathMarker` |
| server | `server/src/objects/explosion.ts` | `Explosion` |
| server | `server/src/objects/bullet.ts` | `Bullet` |
| server | `server/src/objects/emote.ts` | `Emote` |
| server | `server/src/objects/mapIndicator.ts` | `MapIndicator` |
| client | `client/src/scripts/objects/gameObject.ts` | `GameObject` — client base (extends PixiJS `Container`) |
| client | `client/src/scripts/objects/player.ts` | Client `Player` rendering |
| client | `client/src/scripts/objects/*.ts` | Client implementations for each category |

## Architecture

```
common: makeGameObjectTemplate()
    │
    ├── Server: BaseGameObject.derive(ObjectCategory.X)
    │   └── Player, Obstacle, Loot, Building, Decal, Parachute, Projectile, SyncedParticle, DeathMarker
    │
    └── Client: GameObject.derive(ObjectCategory.X)
        └── Same categories, each extends PixiJS Container
```

**Server objects** hold authoritative state, run game logic, and serialize to `UpdatePacket`.
**Client objects** receive `ObjectsNetData` from `UpdatePacket`, update visual state, and render via PixiJS.

## CommonObjectMapping

Every runtime object (server or client) implements at minimum:

```typescript
{
    type: ObjectCategory
    position: Vector
    rotation: number
    dead: boolean
    layer: Layer
    damageable?: boolean
    // + category-specific fields (definition, hitbox, etc.)
}
```

Category-specific extensions (e.g. `Player` has `hitbox`, `activeItemDefinition`, `teamID`) are defined in `CommonObjectMapping`.

## derive() Pattern

New object categories are registered via:

```typescript
// Server
class Player extends BaseGameObject.derive(ObjectCategory.Player) { ... }

// Client
class Player extends GameObject.derive(ObjectCategory.Player) { ... }
```

`derive(category)` creates a subclass and registers it. Only one subclass per category is allowed.

## Serialization (ObjectSerializations)

`ObjectsNetData` defines the wire format for each category. Each category has:
- **Full data** — complete state (sent when object first enters view)
- **Partial data** — delta (sent when only some fields change)

`ObjectSerializations` provides `serializeFull`, `serializePartial`, `deserializeFull`, `deserializePartial` per category.

## Data Flow

```
Server tick
    → For each visible object: serialize full or partial to SuroiByteStream
    → UpdatePacket sends to client

Client receives UpdatePacket
    → For each object: deserialize full or partial
    → GameObject.updateFromData() or updateFull()
    → Interpolate position/rotation (client-side smoothing)
    → PixiJS renders
```

## Protocol Considerations

- **Affects protocol:** Yes. Changes to `ObjectsNetData` or serialization format require `GameConstants.protocolVersion` bump.
- **Full vs partial:** Client must handle both; partial updates assume previous full state.

## Dependencies

- **Depends on:** Definitions (object definitions), Packets (UpdatePacket structure), Constants (ObjectCategory, Layer)
- **Depended on by:** Game loop (server), Rendering (client), Packets (ObjectSerializations)

## Module Index (Tier 3)

For implementation details, see:

- [Server Objects](modules/server-objects.md) — Server-side object lifecycle, collision, damage
- [Client Objects](modules/client-objects.md) — Client rendering, interpolation, PixiJS integration
- [Hitbox](modules/hitbox.md) — Collision geometry (Circle, Rect, Group, Polygon)

## Related Documents

- **Tier 1:** [docs/datamodel.md](../../datamodel.md) — ObjectCategory, CommonObjectMapping
- **Tier 1:** [docs/protocol.md](../../protocol.md) — UpdatePacket, ObjectSerializations
- **Tier 2:** [../definitions/](../definitions/) — Definition types used by objects
- **Tier 2:** [../packets/](../packets/) — UpdatePacket structure
