# Badges & Cosmetics

<!-- @tier: 2 -->
<!-- @parent: docs/architecture.md -->
<!-- @modules: docs/subsystems/badges-cosmetics/modules/ -->
<!-- @source: common/src/definitions/badges.ts, common/src/definitions/items/skins.ts, common/src/definitions/emotes.ts -->

## Purpose

The Badges & Cosmetics subsystem manages **visual-only player customization** — badges marking roles or achievements, player skins (appearance with color tints), and emotes (animated expressions). These elements have **no gameplay impact** on mechanics, damage, health, or ability. They are purely decorative metadata sent in network packets and rendered client-side.

---

## Key Files & Entry Points

| File | Purpose |
|------|---------|
| `common/src/definitions/badges.ts` | `BadgeDefinition` interface, `Badges` registry — role badges (team, developers, moderators) and player badges (logos, achievements) |
| `common/src/definitions/items/skins.ts` | `SkinDefinition` interface, `Skins` registry — 70+ player skins with color tints (base, fist, backpack), role restrictions, special flags |
| `common/src/definitions/emotes.ts` | `EmoteDefinition` interface, `Emotes` registry, `EmoteCategory` enum — 200+ emotes (People, Text, Memes, Icons, Misc, Team, Weapon categories) and category mapping |
| `server/src/objects/player.ts` | Server-side player loadout (`badge`, `skin`, `emotes`), cosmetic initialization, badge assignment, emote rate-limiting |
| `client/src/scripts/objects/player.ts` | Client-side cosmetic rendering (`_skin`, `emote` container, badge sprite), emote display tweens, cosmetic updates |
| `common/src/packets/updatePacket.ts` | Badge and emote serialization in `UpdatePacket` — binary encoding (1-byte flag for badge presence, emote array with definition + playerID) |

---

## Architecture

### Badge Types

Badges are categorized into two groups:

#### Role Badges (Team/Staff)

Assigned automatically to players with specific roles (from authentication/account data). Each has a `roles` array specifying which roles unlock it:

| Badge Name | Roles | Display | Special |
|------------|-------|---------|---------|
| **Developr** | `["developr", "pap"]` | Team badge | Primary developers |
| **Dev Managr** | `["dev_managr"]` | Team badge | Development manager |
| **Designr** | `["designr"]` | Team badge | Game designers |
| **VIP Designr** | `["vip_designr"]` | Team badge | Senior designer |
| **Sound Designr** | `["sound_designr"]` | Team badge | Audio designer |
| **Moderatr** | `["moderatr"]` | Team badge | Community moderators |
| **Administratr** | `["administratr"]` | Team badge | Server admins |
| **Content Creatr** | `["content_creatr", "lead_content_creatr"]` | Team badge | Content creators |
| **Donatr** | `["donatr"]` | Team badge | Major donors |
| **Marketr** | `["marketr"]` | Team badge | Marketing team |
| **Ownr** | `["hasanger"]` | Team badge | Project owner |

#### Player Badges (Logos, Special)

Assigned via custom logic, gameplay events, or special conditions. No role restriction:

| Badge Name | Condition | Display |
|------------|-----------|---------|
| **Bleh** | Unknown | Cosmetic badge |
| **Froog** | Unknown | Cosmetic badge |
| **AEGIS Logo** | Unknown | Team/Alliance badge |
| **Flint Logo** | Unknown | Team/Alliance badge |
| **NSD Logo** | Unknown | Team/Alliance badge |
| **Suroi Logo** | Unknown | Game logo badge |
| **Duel** | Unknown | Achievement badge |
| **Fire** | Unknown | Achievement badge |
| **Colon Three** | Unknown | Cosmetic badge (joke) |
| **Suroi General Chat** | Unknown | Community badge |

### Badge Serialization & Display

Badges are serialized in `UpdatePacket` as part of player death events and full updates:

```typescript
// UpdatePacket.PlayerDataSerialization
readonly badge?: BadgeDefinition  // optional; only present if player has one

// Network encoding: packed with name color
const hasBadge = badge !== undefined;
strm.writeUint8( (hasColor ? 2 : 0) + (hasBadge ? 1 : 0) );
// Bit layout: (nameColor: 2 bits) + (badge: 1 bit)

if (hasBadge) {
    Badges.writeToStream(strm, player.badge);  // 1 or 2 bytes (index)
}
```

Badge display locations:
- **Killfeed** — Icon in death messages
- **Player death screen** — Achievement summary
- **Spectator UI** — Badge visible on tracked player
- **Leaderboard** — If player's stats are displayed

### Skin System

Player skins are appearance customizations with **color tinting** on body parts:

#### Skin Categories

**Standard Skins** (40+ publicly available):
- `HAZEL Jumpsuit`, `The Amateur`, `The Pro` — default/common skins
- `Forest Camo`, `Desert Camo`, `Arctic Camo` — camouflage patterns
- `Bloodlust`, `Red Tomato`, `Greenhorn`, `Blue Blood` — color-themed
- `Silver Lining`, `Pot o' Gold`, `Gunmetal` — metallic
- `Algae`, `Twilight Zone`, `Bubblegum`, `Sunrise`, `Sunset` — natural/atmospheric
- (Plus 20+ more: Stratosphere, Mango, Snow Cone, Aquatic, Floral, Sunny, Volcanic, Ashfall, Solar Flare, Beacon, Wave Jumpsuit, Toadstool, Full Moon, Swiss Cheese, Target Practice, Zebra, Tiger, Bee, Armadillo, Printer, Distant Shores, Celestial)

**Hidden Skins** (cosmetics with `hideFromLoadout: true`):
- Event skins: `Peppermint`, `Spearmint`, `Coal`, `Candy Cane`, `Holiday Tree`, `Gingerbread`, `Pumpkified` — seasonal
- Story skins: `Timeless`, `NSD Uniform`, `Veteran`, `Carpenter Uniform`, `Medical Suit`, `Groundskeeper` — role-based
- Legendary/spooky: `Werewolf` (`hideFromLoadout`, `noSwap`, `noDrop`), `Ghillie Suit` — special condition unlocks

**Role Skins** (restricted to specific team roles):
- `Hasanger` Swag — owner
- `Pap` Swag — owner's alternate
- `Developr Swag` — developers
- `Designr Swag` — designers
- `Sound Designr Swag` — sound designers

#### Skin Definition Structure

```typescript
interface SkinDefinition extends ItemDefinition {
    readonly hideFromLoadout?: boolean      // Hidden from UI (event/special)
    readonly grassTint?: boolean            // Applies tint to grass overlay
    readonly baseImage?: string             // Sprite source (e.g., "plain_base")
    readonly fistImage?: string             // Hand sprite source (e.g., "plain_fist")
    readonly baseTint?: number              // Hex color: 0xRRGGBB (body)
    readonly fistTint?: number              // Hex color: hands/arms
    readonly backpackTint?: number          // Hex color: backpack
    readonly hideEquipment?: boolean        // Hide visible equipment on model
    readonly rolesRequired?: string[]       // [] if unrestricted
    readonly hideBlood?: boolean            // Hide blood splatter effects
    readonly noSwap?: boolean               // Cannot change mid-game (e.g., Werewolf)
    readonly sound?: boolean                // Custom audio (rarely used)
}
```

### Emote System

Emotes are **animated facial expressions and gestures** sent with player state. Each player has up to **4 emote slots** in their loadout.

#### Emote Categories

Emotes are grouped by semantic type (`EmoteCategory` enum):

| Category | Examples | Count |
|----------|----------|-------|
| **People** | Happy Face, Sad Face, Thumbs Up, Thumbs Down, Dab, Facepalm, Saluting Face, Thinking Face, etc. | ~50 |
| **Text** | Word emotes: "EPIC", "NOOB", "LOL", "GG", "SALTY", etc. | ~20 |
| **Memes** | Cultural references: "Loss", "Swag", "Stonks", "Ohhh", etc. | ~15 |
| **Icons** | Simple icons: Flag, Heart, Star, Explosion, Skull, Trophy, etc. | ~25 |
| **Misc** | Assorted: Banana, Cookie, Cheese, Beer, Laptop, etc. | ~30 |
| **Team** | Team-colored circles, shapes for communication | ~8 |
| **Weapon** | Gun, Sniper, Shotgun, Melee, Medkit icons | ~10 |

Total: **200+ emotes** available.

#### Emote Serialization

Emotes sent in `UpdatePacket` with rate-limiting:

```typescript
interface EmoteSerialization {
    readonly definition: EmoteDefinition;   // Emote reference
    readonly playerID: number;              // Source player (who's emoting)
}

// Network encoding: array in UpdatePacket when flag set
if (data.emotes?.length) {
    strm.writeArray(
        data.emotes,
        emote => {
            Emotes.writeToStream(strm, emote.definition);  // 1-2 bytes
            strm.writeObjectId(emote.playerID);            // 2 bytes
        }
    );
    flags |= UpdateFlags.Emotes;
}
```

#### Emote Display

Client-side rendering:
- **Emote container** — floats above player head
- **Background sprite** — semi-transparent rounded box
- **Emote image** — centered, scaled to fit
- **Hide tween** — animation that fades out after 3-4 seconds
- **Rate limit** — client blocks emotes after ~4 per second (server enforces too)

### Player Loadout

Server maintains cosmetic loadout per player:

```typescript
// server/src/objects/player.ts
readonly loadout: {
    badge?: BadgeDefinition                      // Optional team/achievement badge
    skin: SkinDefinition                         // Current appearance (mandatory)
    emotes: ReadonlyArray<EmoteDefinition | undefined>  // Up to 4 slots
}
```

Initialization logic:
1. **Badge** — assigned from player's role metadata if applicable
2. **Skin** — assigned from player preference, or random `default_skin` if none specified
3. **Emotes** — assigned from player preference (default: `happy_face`, `thumbs_up`, `suroi_logo`, `sad_face`)

---

## Data Flow

```
Player Joins Server
    ↓
Authenticate & Load Account Data
    ↓
Assign Badge (if role matches)
    ↓
Load Preferred Skin (or default)
    ↓
Load Preferred Emotes (or default 4)
    ↓
[Game Running]
    ↓
Player Deals Damage → Killfeed Shows Badge (if present)
    ↓
UpdatePacket Serializes: badge + emotes per tick
    ↓
[Client Receives]
    ↓
Render Badge Icon (in HUD, killfeed, spectator UI)
    ↓
Render Player Skin with Color Tints
    ↓
Display Emote Animation (if present) + Hide After 3sec
```

---

## Dependencies

### Depends on:
- [Object Definitions](../object-definitions/) — `BadgeDefinition`, `SkinDefinition`, `EmoteDefinition` registries, deterministic numeric indices for binary serialization
- [Game Objects (Server)](../game-objects-server/) — player loadout storage, badge/skin/emote assignment logic
- [Networking](../networking/) — binary serialization and deserialization in `UpdatePacket` (badges, emotes, skin references)

### Depended on by:
- [Client Rendering](../client-rendering/) — applies skin color tints, renders badge sprites, displays emotes
- [UI Management](../ui-management/) — cosmetic selection UI (skin picker, emote picker), loadout editing
- [Game Objects (Client)](../game-objects-client/) — client-side cosmetic state (`_skin`, `emote` container, tweens)
- [Killfeed / Death Screen](../ui-management/) — badge display in kill messages and achievement summary

---

## Serialization Details

### Cosmetic State in UpdatePacket

**Player data serialization:**
```
[decorations byte]
  - Bit 0: hasColor (name color override)
  - Bit 1: reserved
  - Bit 2: hasBadge
  [if hasBadge] → Badges.writeToStream(badge)
  [if hasColor] → writeUint24(nameColor)
```

**Emote serialization (separate):**
```
[UpdateFlags.Emotes flag]
  [emote count]
  [for each emote]
    → Emotes.writeToStream(definition)
    → writeObjectId(playerID)
```

### Network Efficiency

- **Badge** — 1 byte (flag) + 1 byte (index if ≤255 badges; rare for 2 bytes)
- **Emote** — 1 byte (emote index) + 2 bytes (playerID) per emote, sent sparse
- **Skin** — stored in player object definition (not in cosmetic state), referenced by index

**Optimization notes:**
- Cosmetics cached on client after received once
- Emotes sparse in UpdatePacket — only sent when new emote is displayed
- Badge included every tick if present (required for killfeed state)

---

## Known Issues & Limitations

1. **No cosmetic filtering** — Players cannot disable viewing other players' skins or emotes
2. **Badge assignment manual** — No automatic badge unlock system; requires role-based auth or hardcoded logic
3. **Emote categories not UI-organized** — Emote picker shows flat list, no category filtering in client UI
4. **Skin previews incomplete** — No 3D model preview; players see 2D sprite only
5. **Special skin conditions opaque** — Unlock conditions for hidden skins (Werewolf, Ghillie Suit) not visible to players
6. **No cosmetic rarity tiers** — All cosmetics appear equal with no visual rarity indicator
7. **Emote audio missing** — Emotes display only; no accompanying sound effects
8. **Cross-platform cosmetic purchase not implemented** — Cosmetic unlock system hardcoded per role, not persistent purchase/BP system

---

## Related Documents

### Tier 1
- [Data Model](../../datamodel.md) — Badge enum, Skin enum, Emote definitions
- [Architecture](../../architecture.md) — Player object model, network protocol

### Tier 2
- [Object Definitions](../object-definitions/) — Badge/Skin/Emote registries, binary serialization
- [Game Objects (Server)](../game-objects-server/) — Player class, loadout initialization
- [Game Objects (Client)](../game-objects-client/) — Client Player class, cosmetic rendering
- [Networking](../networking/) — UpdatePacket structure, serialization format
- [UI Management](../ui-management/) — Cosmetic selection menus, killfeed badge display
- [Client Rendering](../client-rendering/) — Sprite rendering, color tinting, emote animation

### Tier 3
- [Module: Badges](modules/badges.md) — Badge registry structure and role assignment
- [Module: Skins](modules/skins.md) — Skin tinting system and hidden/special skins
- [Module: Emotes](modules/emotes.md) — Emote definition categories and rate-limiting

