# Q20: Why are definitions serialized as single-byte indices — what happens when a category exceeds 255 items?

**Answer:**

Single-byte (uint8) indices are used as the **default serialization format** for all definition categories because most individual registries contain fewer than 255 items. However, when a category reaches **256+ items**, the system automatically scales to **two bytes (uint16)** per definition index.

---

## How the Scaling Mechanism Works

`ObjectDefinitions` class includes an `overLength: boolean` flag:

```typescript
/**
 * Whether there are more than 256 definitions in this schema, requiring 2 bytes to serialize
 */
readonly overLength: boolean;

constructor(definitions: readonly Def[]) {
    // ... populate maps ...
    this.overLength = idx > 255;  // Set flag if count exceeds 255
}
```

The `writeToStream()` method **dynamically switches encoding based on this flag**:

```typescript
writeToStream(stream: ByteStream, def: ReifiableDef<Def>): void {
    const idx = this.idStringToNumber[idString];
    if (this.overLength) {
        stream.writeUint16(idx);  // 2 bytes for indices 0–65,535
    } else {
        stream.writeUint8(idx);   // 1 byte for indices 0–255
    }
}
```

---

## Scaling Limits

| Bytes | Max Entries | Range |
|-------|------------|-------|
| 1 (uint8) | 256 | 0–255 |
| 2 (uint16) | 65,536 | 0–65,535 |
| *(not used)* | uint24: 16.7M | 0–16,777,215 |

---

## Current Directory Counts

Most **individual categories remain under 255**:
- **Guns:** ~67 entries (single-byte)
- **Obstacles:** 1,000+ entries (dual-variant walls expand count; **overLength=true**, uses 2 bytes)
- **Buildings:** 100+ entries (single-byte likely)
- **Ammos, Melees, Throwables, etc.:** <50 entries each (single-byte)

---

## The `Loots` Meta-Registry (HIGH-RISK CATEGORY)

The `Loots` registry merges ALL loot-droppable items:

```typescript
export const Loots = new ObjectDefinitions<LootDefinition>([
    ...Guns,         // ~67
    ...Ammos,        // ~15
    ...Melees,       // ~50
    ...Throwables,   // ~20
    ...HealingItems, // ~10
    ...Armors,       // ~10
    ...Backpacks,    // ~10
    ...Scopes,       // ~10
    ...Skins,        // ~200+
    ...Perks         // ~50
]);
```

**Total: 400+ entries → `Loots.overLength = true` → Uses uint16 (2 bytes per loot reference)**

---

## Key Takeaway

The serialization system was designed with **built-in, automatic overflow handling**. No game logic breaks if a category exceeds 255—the framework transparently upgrades that category to 2-byte indices while keeping smaller categories at 1 byte. The only impact is a +1 byte per definition reference in network packets for overflowed categories.

---

## References

- **Tier 2:** `docs/subsystems/object-definitions/README.md` — Architecture & registry design
- **Tier 3:** `docs/subsystems/serialization-system/modules/byte-encoding.md` — Primitive encoding details
- **Source:** `common/src/utils/objectDefinitions.ts` — Constructor & serialization logic
- **Source:** `common/src/definitions/loots.ts` — Higher-cardinality composite registry
