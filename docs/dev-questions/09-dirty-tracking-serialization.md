# Q: How does dirty tracking work and how does the server decide what to send to each client each tick?

<!-- @tags: game-loop, serialization, protocol, UpdatePacket -->
<!-- @related: docs/subsystems/game-loop/modules/tick-serialization.md, docs/subsystems/packets/modules/update-packet.md -->

## Core Idea

The server doesn't send the full game state every tick — it only sends what
**changed**. Objects mark themselves dirty when state changes, and the server
serializes only those objects.

## Two Dirty Sets

Every `Game` maintains two sets:

| Set | When Used | What Gets Sent |
|-----|-----------|----------------|
| `partialDirtyObjects` | A field changed (health, position, etc.) | Delta update — only changed fields |
| `fullDirtyObjects` | Object first appears or major change (respawn, etc.) | Full state — all fields |

If an object is in **both** sets, only the full serialization is sent.

## How Objects Mark Themselves Dirty

```typescript
// server/src/objects/gameObject.ts
this.setDirty();      // → partialDirtyObjects
this.setFullDirty();  // → fullDirtyObjects
```

For example, when a player's health changes:

```typescript
// inside player.ts, damage()
this.setDirty();
```

When a new obstacle spawns:

```typescript
// server registers it
this.setFullDirty();
```

## Serialization (per tick, step 12)

```
For each object in partialDirtyObjects (skip if also in fullDirtyObjects):
    → object.serializePartial(stream)  → shared partial buffer

For each object in fullDirtyObjects:
    → object.serializeFull(stream)     → shared full buffer
```

These buffers are computed **once** and shared across all players.

## Per-Player Packet Assembly (tick step 13)

Each player's `secondUpdate()` builds their `UpdatePacket`:

1. Filter objects visible to this player (in their view radius)
2. Pick the relevant slice of the shared partial/full buffers
3. Add player-specific data (health, inventory, adrenaline)
4. Send

A player only receives updates for objects they can **see**. Objects outside
the viewport are not included even if dirty.

## Cleanup (tick step 16)

Both dirty sets are cleared at the end of every tick.

## Practical Implication: Adding a New Field

When you add a new serializable field to a server object:

1. Add the field to `serializePartial()` or `serializeFull()` (in
   `common/src/utils/objectsSerializations.ts`)
2. Call `setDirty()` whenever that field changes
3. Bump `GameConstants.protocolVersion`

## Related

- [Tick & Serialization](../subsystems/game-loop/modules/tick-serialization.md) — full tick order and serialization pipeline
- [UpdatePacket](../subsystems/packets/modules/update-packet.md) — packet structure and DataSplit categories
- [Game Loop](../subsystems/game-loop/README.md) — full 18-step tick order
