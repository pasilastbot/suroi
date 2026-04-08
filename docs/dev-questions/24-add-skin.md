# Q: How do I add a new player skin?

<!-- @tags: definitions, skins, SVG, spritesheet -->
<!-- @related: docs/subsystems/definitions/modules/items.md, docs/subsystems/rendering/modules/spritesheets.md -->

## 1. Add the skin definition

Open `common/src/definitions/items/skins.ts` and append:

```typescript
{
    idString: "my_skin",
    name: "My Skin",
    defType: DefinitionType.Skin,
    itemType: ItemType.Skin,

    // Optional: restrict visibility in loadout
    // hideFromLoadout: true,   // not shown in skin picker
    // grassTint: true,         // tint skin with grass color (ghillie suit style)
    // role: "server_sender",   // role-restricted skin

    // Optional: custom sounds when this skin moves
    // footstepSound: "wood_footstep",
}
```

## 2. Add SVG assets

Skins consist of multiple SVG layers. The base layer names follow:

```
client/public/img/game/shared/skins/<idString>_base.svg     ← body
client/public/img/game/shared/skins/<idString>_left_fist.svg
client/public/img/game/shared/skins/<idString>_right_fist.svg
```

Some skins may only require the base SVG if fists are shared (the default
fist sprites are used when separate fist SVGs are not present).

Create your SVGs at the standard player scale. Reference the default skin
(`hazel_jumpsuit`) as a size guide.

```bash
bun validateSvgs
```

## 3. Bump the protocol version

```typescript
// common/src/constants.ts
protocolVersion: 73,  // ← increment
```

`Skins` is an `ObjectDefinitions` collection — adding a skin changes indices.
Skins are also serialized in `JoinPacket`.

## 4. Validate

```bash
bun validateDefinitions
```

## 5. Test

```bash
bun dev
```

Open the loadout screen. Your skin should appear in the skin picker. Select
it and join a game to verify the rendering.

## How Skins Work

- The player's skin is selected in the loadout screen and stored in the `cv_loadout_skin` cvar
- It's sent in `JoinPacket` as a definition index into `Skins`
- The server stores it on the `Player` object and broadcasts it in `UpdatePacket`
- The client uses `skin.idString` to look up the sprite frames: `{idString}_base`,
  `{idString}_left_fist`, `{idString}_right_fist`

## Notes

- `grassTint: true` makes the skin tinted with the terrain grass color (for ghillie
  suits). The base SVG should be designed to look good when tinted.
- Role-restricted skins (moderator badge, developer skin) use `role: "..."` and are
  only available to players with that server-assigned role.
- Loot skins (dropped in-world) appear as `loot_<idString>.svg` in the ground.
  Add this only if the skin is droppable.

## Related

- [Items Module](../subsystems/definitions/modules/items.md) — skins.ts
- [Spritesheets](../subsystems/rendering/modules/spritesheets.md) — how SVGs become spritesheet frames
- [Protocol Version](04-protocol-version-bump.md) — JoinPacket carries skin index
