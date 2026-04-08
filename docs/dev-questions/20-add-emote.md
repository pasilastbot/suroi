# Q: How do I add a new emote?

<!-- @tags: definitions, emotes, loadout, SVG -->
<!-- @related: docs/subsystems/definitions/modules/emotes.md -->

## 1. Add the SVG asset

Place your emote SVG in:

```
client/public/img/game/shared/<idString>.svg
```

The SVG should be square and clearly recognizable at small sizes (emotes are
displayed as ~48px icons in the loadout picker).

## 2. Validate the SVG

```bash
bun validateSvgs
```

Fix any reported issues before proceeding.

## 3. Add the emote definition

Open `common/src/definitions/emotes.ts` and append to the array:

```typescript
{
    idString: "my_emote",
    name: "My Emote",
    category: EmoteCategory.People,  // or Icons, Memes, Text, Misc
}
```

**Available categories:** `People`, `Icons`, `Memes`, `Text`, `Misc`, `Team`, `Weapon`

Use `hideInLoadout: true` only for emotes that are auto-generated (weapon/ammo/healing
item emotes) and should not appear in the loadout picker.

## 4. Bump the protocol version

```typescript
// common/src/constants.ts
export const GameConstants = {
    protocolVersion: 73,  // ← increment
    ...
};
```

Protocol version must be bumped because `Emotes` is an `ObjectDefinitions`
collection — adding a definition changes serialization indices.

## 5. Validate

```bash
bun validateDefinitions
```

## 6. Test

```bash
bun dev
```

Open the loadout screen in the browser. Your emote should appear in the picker
under its category. Equip it to a slot and test it in-game.

## Notes

- **Auto-generated emotes:** All guns, melees, throwables, ammo types, and
  healing items automatically become `Weapon`/`Team` category emotes with
  `hideInLoadout: true`. You don't need to add these manually.
- **Badge overlap:** If an emote has the same `idString` as a badge with prefix
  `emote_`, `isEmoteBadge()` will match them. Keep idStrings unique.
- **Emote slots:** Players have 8 emote slots (indices 0–7). Emotes are
  serialized in `JoinPacket` and `JoinedPacket`.

## Related

- [Emotes Module](../subsystems/definitions/modules/emotes.md) — full field reference, auto-generation rules
- [Spritesheets](../subsystems/rendering/modules/spritesheets.md) — SVG → spritesheet pipeline
- [Protocol Version](04-protocol-version-bump.md) — when and how to bump
