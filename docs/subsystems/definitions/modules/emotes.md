# Definitions — Emotes Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/definitions/README.md -->
<!-- @source: common/src/definitions/emotes.ts -->
<!-- @updated: 2026-03-04 -->

## Purpose

This module documents emote definitions: player emotes (faces, icons, memes, text), `EmoteCategory`, loadout slots (8), and the badge/weapon emote overlap (ammo, healing, weapons as emotes).

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| @file common/src/definitions/emotes.ts | `Emotes`, `EmoteDefinition`, `EmoteCategory` | Medium |

## EmoteDefinition Fields

| Field | Purpose |
|-------|---------|
| `category` | EmoteCategory enum |
| `hideInLoadout?` | Hide from loadout picker (weapon/ammo/healing emotes) |

## EmoteCategory

| Category | Examples |
|----------|----------|
| People | Happy Face, Sad Face, Thumbs Up, etc. |
| Icons | Suroi Logo, Skull, Trophy |
| Memes | Troll Face, Pog, RIP |
| Text | gg, ez, oof, real, fake |
| Misc | Fire, Heart, Penguin |
| Team | Ammo, healing items (hideInLoadout) |
| Weapon | Guns, melees, throwables (hideInLoadout) |

## Auto-Generated Emotes

- **Ammo, HealingItems** — Each becomes Team category emote, `hideInLoadout: true`
- **Guns, Melees, Throwables** — Each becomes Weapon category emote, `hideInLoadout: true`

## Badge Overlap

- **isEmoteBadge(badge)** — True if badge idString matches an emote (badge format: `emote_<idString>`)
- **getBadgeIdString** — Strips `emote_` prefix for emote badges
- Used for loadout: badge can show emote

## Serialization

- JoinPacket, JoinedPacket — 8 emote slots (EmoteDefinition | undefined)
- Emotes.writeToStream / readFromStream for definition index

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Definitions overview
- **Tier 2:** [../packets/](../../packets/) — JoinPacket, JoinedPacket (emotes)
- **Tier 2:** [../definitions/](../definitions/) — Badges
