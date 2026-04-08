# Q: When do I need to bump the protocol version and how do I do it?

<!-- @tags: protocol, binary, serialization -->
<!-- @related: docs/protocol.md, docs/subsystems/packets/ -->

## When to Bump

Increment `GameConstants.protocolVersion` any time the **wire format changes**:

- A packet field is added, removed, or reordered
- A definition list changes (items added or removed — because indices shift)
- Any `SuroiByteStream` encoding changes (byte width, range, etc.)

The client sends its version in `JoinPacket`. The server disconnects immediately
on mismatch, preventing deserialization errors. There is no backward compatibility.

## How to Bump

```typescript
// common/src/constants.ts
export const GameConstants = {
    protocolVersion: 73,  // ← increment by 1
    ...
};
```

That's it — one number in one file.

## Important: Definition Index Order

**Never insert a definition in the middle of a list.** Inserting a definition
shifts all subsequent numeric indices. This silently breaks protocol compatibility
for any client/server pair that was compiled with the old ordering.

Always **append** new definitions to the end of their collection array.

## Verification

After bumping, test that the client connects successfully:

```bash
bun dev   # starts client (:3000) and server (:8000)
```

Open the browser — if you see a "version mismatch" disconnect, the client cache
is stale. Hard-refresh or restart Vite.

## Related

- [Protocol Reference](../protocol.md) — full wire format and versioning rules
- [Development Guide](../development.md) § Protocol Version — checklist
- [Packets Subsystem](../subsystems/packets/README.md) — all packet types
