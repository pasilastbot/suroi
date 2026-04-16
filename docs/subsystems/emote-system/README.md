# Emote System

<!-- @tier: 2 -->
<!-- @parent: docs/architecture.md -->
<!-- @modules: docs/subsystems/emote-system/modules/ -->
<!-- @source: server/src/objects/emote.ts, common/src/definitions/emotes.ts, client/src/scripts/managers/emoteWheelManager.ts -->

## Purpose

The Emote System enables players to display cosmetic gestures, reactions, and social expressions via a radial selection wheel. Emotes are synchronized across all players and include 100+ emoji-style graphics with optional sound effects. The system features rate-limiting to prevent spam, a dynamically-calculated radial UI, and client-side animation sequencing (pop-up, dwell, fade-out).

## Key Files & Entry Points

| File | Purpose |
|------|---------|
| `server/src/objects/emote.ts` | Emote entity lifecycle: instantiation and serialization |
| `common/src/definitions/emotes.ts` | EmoteDefinition registry: 100+ emotes organized by category (People, Icons, Memes, Text, Misc, Team, Weapon) |
| `client/src/scripts/managers/emoteWheelManager.ts` | Radial selection UI: wheel display, mouse tracking, slot sizing, selection logic |
| `server/src/objects/player.ts` | Emote sending and rate-limiting: `sendEmote()`, `emoteRateLimit()`, loadout storage |
| `client/src/scripts/objects/player.ts` | Animation and audio playback: `showEmote()`, tweens, sound effects |
| `common/src/packets/inputPacket.ts` | Input serialization: `InputActions.Emote` packet encoding/decoding |
| `common/src/packets/updatePacket.ts` | Network synchronization: `EmoteSerialization`, emote broadcast flag |
| `server/src/game.ts` | Game loop integration: `game.emotes[ ]` array, tick cleanup, plugin hooks |

## Architecture

### Conceptual Flow

```
Player Input (Wheel Selection)
    ↓
EmoteWheelManager.emitEmote(emote)
    ↓
InputManager.addAction({ type: Emote, emote: definition })
    ↓
InputPacket encoded & sent to server
    ↓
Server: Player.sendEmote(definition)
    → Rate-limit check
    → Plugin event: "player_will_emote"
    → Create Emote(definition, player)
    → Add to game.emotes[ ]
    → Plugin event: "player_did_emote"
    ↓
UpdatePacket serializes emotes to all clients
    ↓
Client: Game.onUpdatePacket() → Player.showEmote(definition)
    → Play sound ("emote" channel, 0.4 falloff, 128 range)
    → Scale-in animation (250ms backOut)
    → 2000ms dwell
    → Scale-out animation (200ms backIn)
    → Cleanup
    ↓
Server: Game.tick() clears game.emotes[ ] at end of tick
```

### Key Components

**Emote Definition** (`EmoteDefinition`)
- Extends `ObjectDefinition` with `category: EmoteCategory` enum
- Categories: People, Text, Memes, Icons, Misc, Team, Weapon
- Team/Weapon categories marked `hideInLoadout: true` (system-generated, not player-selectable)
- Binary-encoded as uint8 index in ObjectDefinitions registry

**Emote Entity** (`Emote` class)
- Minimal: stores playerID, definition, player reference
- Exists only for one game tick (cleared at end of tick)
- Used for network synchronization and plugin hooks

**Rate-Limiting**
- Tracks `player.emoteCount` per session
- Threshold: 10 emotes in rapid succession
- Punishment: 5-second block (sets `player.blockEmoting = true`)
- Shared with map pings (both increment `emoteCount`)

**Wheel UI** (`EmoteWheelManager`)
- Radial menu centered at mouse position (desktop) or screen center (mobile)
- Dynamic slot calculation based on emote count
- Uses sine law for proper circular distribution of emote sprites
- Ring dimensions:
  - Outer radius: 130 px
  - Inner radius: 20 px
  - Center close button with X icon
  - Ticks connecting center to outer ring
- Desktop: mouse hover selects slot, click emits
- Mobile: touch on emote slot to emit

## Emote Categories & Catalog

### People (55 emotes)
Emoji-style faces with expressions:
- **Basic:** Happy Face, Sad Face, Neutral Face, Angry Face, Crying Face
- **Gestures:** Thumbs Up, Thumbs Down, Wave, Facepalm, Shrugging Face, Saluting Face
- **Special:** Alien, Devil Face, Creepy Clown, Zombie Face, Headshot
- **Emotions:** Heart Face, Cool Face, Nervous Face, Sweating Face, Hot Face, Cold Face, etc.
- **Meta:** Thousand Yard Stare, Monocle Face, Nerd Face, Side Eye Face

### Icons (8 emotes)
Game-branded and trophy sprites:
- Suroi Logo, AEGIS Logo, Flint Logo, NSD Logo
- Skull, Duel, Chicken Dinner, Trophy

### Memes (14 emotes)
Inside jokes and culture references:
- Troll Face, Clueless, Pog, Froog, Boykisser, Leosmug
- awhhmahgawd, Grr, RIP, are you sure
- Suroi General Chat, Muller

### Text (11 emotes)
Chat shortcuts:
- Question Mark, Team = Ban, Hack = Ban
- gg, ez, Hi5, oof, real, fake, Colon Three, Lag

### Misc (14 emotes)
Nature and objects:
- Fire, Heart, Penguin, Squid, Eagle, Whale
- Carrot, Egg, Wilted Rose, Plumpkin, Leek, Tomato, Logged, Sun and Moon

### Team (Ammo & Healing Items)
System-generated with `hideInLoadout: true`
- Item emotes: displayed as category badges, not selectable in wheel
- Used for quick item callouts (e.g., teammate needs ammo)

### Weapon (Guns, Melees, Throwables)
System-generated with `hideInLoadout: true`
- Weapon emotes: item references, hidden from loadout
- Reserved for future weapon callout features

## Emote Wheel UI

### Desktop Flow

```
Player presses 'E' key
  → EmoteWheelManager.enabled = true
  → Wheel appears at mouse position
  → Background circle (dark, semi-transparent) visible
  → Emote slots rendered around ring
  → Inner close button visible
  ↓
Player moves mouse over ring
  → Selection slice highlights (orange stroke arc)
  → Selection index calculated from mouse angle
  ↓
Player clicks or presses Enter
  → EmoteWheelManager.emitEmote(emote[selectionIndex])
  → Wheel closes
  → Emote sent to server
  ↓
If cancelled (moved to inner ring)
  → Close button highlights
  → Release without action
  → Wheel closes without sending emote
```

### Mobile Flow

```
Player taps emote wheel button
  → Wheel appears at screen center
  → Selection UI adapted for touch
  ↓
Player taps an emote sprite
  → Direct pointerdown→pointerup selection
  → MapPingWheelManager briefly takes focus for ping position
  → Emote sent to server
  → Wheel closes
```

### Wheel Configuration

**Slot Angle Calculation**
```
slotAngle = 360° / emoteCount
```
For example:
- 4 slots: 90° apart
- 8 slots: 45° apart
- 6 slots: 60° apart

**Emote Size Calculation** (using sine law)
- Depends on arc angle, ring midpoint, and emote count
- Ensures sprites fit properly without overlap
- Reduces padding (`emotePadding: 8px`) as count increases

**Colors & Styling**
- Background: dark gray, 46% opacity
- Stroke: light gray, 50% opacity, 5px width
- Selection highlight: orange (#FF7500)
- Close button: orange X when not hovering outer ring

## Player Loadout & Selection

### Emote Slots

Player loadout includes 8 emote slots:
```typescript
loadout.emotes: [
  EmoteDefinition | undefined,  // [0–5] Wheel slots (6 total)
  EmoteDefinition | undefined   // [6] Victory emote (auto-sent on win)
  // [7] and beyond: reserved for future quick slots
]
```

The **victory emote** (index 6) is auto-triggered when player wins:
```typescript
// server/src/game.ts, winning logic
player.sendEmote(player.loadout.emotes[6], true)
```
This happens when last player standing calls `sendEmote()` with `isFromServer=true`.

### Validation

Only emotes in slots **0–5** can be selected from the wheel. Attempts to send emotes from slots 6+ are rejected:
```typescript
const indexOf = this.loadout.emotes.indexOf(source);
if (!isFromServer && (indexOf < 0 || indexOf > 5)) return;
```

Server-sent emotes (perks, victories) bypass this check with `isFromServer=true`.

## Emote Lifecycle

### Trigger

**Player Action:** Selects emote from wheel
```
Input → EmoteWheelManager.emitEmote(emote)
      → InputManager.addAction({ type: InputActions.Emote, emote })
      → InputPacket sent to server
```

**Server Action:** `Player.sendEmote(definition, isFromServer = false)`
```typescript
sendEmote(source?: EmoteDefinition, isFromServer = false): void {
    if (this.emoteRateLimit() || !source) return;

    const indexOf = this.loadout.emotes.indexOf(source);
    if (!isFromServer && (indexOf < 0 || indexOf > 5)) return;

    if (this.game.pluginManager.emit("player_will_emote", { 
        player: this, 
        emote: source 
    })) return;

    this.game.emotes.push(new Emote(source, this));

    this.game.pluginManager.emit("player_did_emote", {
        player: this,
        emote: source
    });
}
```

### Network Serialization

**Packet:** `UpdatePacket.emotes: EmoteSerialization[]`
```typescript
interface EmoteSerialization {
    readonly definition: EmoteDefinition  // uint8 index
    readonly playerID: number              // uint16 object ID
}
```

**Encoding** (binary, compact):
- Emote definition: 1 byte (registry index)
- Player ID: 2 bytes (uint16)
- Total: ~3 bytes per emote per update packet

**Broadcast:** Emotes array included in UpdatePacket sent to all players (no network culling; emotes always visible regardless of distance).

### Client Playback

**Step 1: Receive & Decode**
```
UpdatePacket.emotes deserialized
  → EmoteSerialization[] parsed
  → Lookup definition by index
  → Resolve player object by ID
```

**Step 2: Animation**
```
Player.showEmote(emote: EmoteDefinition)
  ↓
Kill any in-flight animation tweens
  → Play sound: "emote" audio channel
     (falloff=0.4, maxRange=128)
  ↓
Show container: container.visible = true
Set scale: (0, 0) → animate to (1, 1)
  → Duration: 250ms
  → Easing: backOut
  ↓
[Dwell 2000ms]
  ↓
Scale down: (1, 1) → (0, 0)
  → Duration: 200ms
  → Easing: backIn
  → cleanup on complete
```

**Step 3: Rendering**
- Emote display positioned above player: Y-offset = -175px
- Container includes:
  - Background sprite ("emote_background")
  - Emote image sprite (definition.idString)
- Background texture selected from Loot registry (shared cosmetic background)
- Z-index: `ZIndexes.Emotes` (above player, below UI)

### Permanent State

- Emote object instantiated in `game.emotes[]` array (server-side)
- Cleared (array reset) at end of every game tick
- Clients store animation state (tween, container) locally
- No persistence beyond single tick

## Rate Limiting

### Spam Prevention

**Trigger Condition**
```typescript
emoteRateLimit(): boolean {
    if (this.blockEmoting) return true;

    this.emoteCount++;

    // After constantly spamming more than 10 emotes, block for 5 seconds
    if (this.emoteCount > GameConstants.player.rateLimitPunishmentTrigger) {
        this.blockEmoting = true;
        this.setDirty();  // Network sync
        
        this.game.addTimeout(() => {
            this.blockEmoting = false;
            this.setDirty();
            this.emoteCount = 0;
        }, GameConstants.player.emotePunishmentTime);
        
        return true;
    }

    return false;
}
```

**Configuration Constants**
- `rateLimitPunishmentTrigger`: 10 (emotes before block)
- `emotePunishmentTime`: 5000 (milliseconds of block)

**Shared with Map Pings**
- Both emotes and map pings increment `player.emoteCount`
- Emoji spam → blocks map pings, and vice versa
- Single unified rate-limit pool

**UI Feedback**
- When `blockEmoting=true`, emote wheel UI tints gray (desaturated)
- Wheel still visible but non-functional
- Counter resets after 5-second timeout

## Interactions with Other Systems

### Game Loop Integration

**During Game Tick** (`game.ts:tick()`)
1. Emotes in `game.emotes[]` array are updated (currently no per-emote update logic)
2. At tick end: `this.emotes.length = 0` (array cleared)

**Serialization Flag**
- UpdatePacket includes emotes if `UpdateFlags.Emotes` set
- Flag set when array contains any emotes

### Plugin Hooks

**Events**
- `"player_will_emote"` — Fired before emote added to array; can cancel via return-true
- `"player_did_emote"` — Fired after emote added; informational

**Usage Example**
```typescript
pluginManager.on("player_did_emote", ({ player, emote }) => {
    // Log, filter, award points, etc.
});
```

### Winning Condition

Auto-triggers victory emote (slot 6) for all remaining alive players:
```typescript
player.sendEmote(player.loadout.emotes[6], true);
```
This happens when:
- Game is `over: true`
- Only one team alive (or one player in free-for-all)
- 5+ seconds since game started

### Input Validation

**Emote Wheel Gating**
- When `UIManager.blockEmoting`, wheel shows but is non-interactive
- Prevents selection even if wheel open (tint feedback)

**Controller Support** (if applicable)
- Emote wheel uses wheel-based navigation (mouse or touch)
- No keyboard emote index shortcuts (design choice to force wheel selection)

### Sound Effects

**Audio Channel:** "emote"  
**Falloff:** 0.4 (spatial audio, reduces with distance)  
**Max Range:** 128 units  
**All emotes use same sound effect** (no per-emote audio variants currently)

## Dependencies

### Depends On

- [Object Definitions](../object-definitions/) — EmoteDefinition registry, binary serialization
- [Game Objects (Server)](../game-objects-server/) — Player entity, loadout storage
- [Networking](../networking/) — InputPacket, UpdatePacket serialization, packet flags
- [Sound Management](../sound-management/) — Audio playback ("emote" channel)
- [UI Management](../ui-management/) — blockEmoting flag, emote wheel button, requestable items state

### Depended On By

- [Game Loop](../game-loop/) — Emote entity creation, tick lifecycle, plugin event integration
- [Client Rendering](../client-rendering/) — Emote sprite sourcing from registry, animation tweening
- [Input Management](../input-management/) — Keyboard input routing to EmoteWheelManager
- [Player Module](../game-objects-server/modules/player.md) — sendEmote() method, loadout field
- [Player Module](../game-objects-client/modules/player.md) — showEmote() animation, rendering

## Known Issues & Gotchas

1. **No per-emote audio variants** — All emotes play the same generic sound effect; individual emote sounds not implemented.

2. **Shared rate limit with map pings** — Emoji spam blocks map pings for 5 seconds (and vice versa); no independent rate limit pools.

3. **Victory emote validation missing** — Victory emote (slot 6) sent with `isFromServer=true`, bypassing slot validation; could allow index out-of-bounds if loadout.emotes corrupted.

4. **No emote cooldown per player** — Only per-player rate limit; no per-emote cooldown or "emote freshness" tracking (same emote can be sent repeatedly once rate-limit expires).

5. **Wheel slot count dynamic** — Wheel size/slot count updates when emote loadout changes, but wheel geometry is recalculated only on `setupSlots()` call; rapid loadout changes mid-wheel-display may cause visual glitches.

6. **Team/Weapon emotes visible in other contexts** — Team and Weapon emotes have `hideInLoadout:true` but are still in Emotes registry; if accidentally sent from server with `isFromServer=true`, they'd render (should verify system-generated emotes are validated).

7. **Emote sprite background fallback** — Emote background texture looked up from Loot registry; if missing, uses default "emote_background"; no error logging if sprite fails to load.

8. **Audio falloff distance** — Hardcoded 128-unit max range with 0.4 falloff; doesn't scale with game-size or follow typical audio manager falloff curves (may sound inconsistent vs. weapon/footstep audio).

9. **No emote animation priority** — If showEmote() called while emote already animating, previous tweens killed without cleanup callbacks; corner case if ShowEmote called in quick succession.

10. **Client-side only animation timing** — Emote duration (2000ms) hardcoded in client; server has no knowledge of how long emote displays, could cause desyncs if client frame-rate drops cause animation skips.

## Related Documents

### Tier 1
- [Data Model](../../datamodel.md) — EmoteCategory enum, definition structure
- [Architecture](../../architecture.md) — Network architecture, serialization overview

### Tier 2
- [Object Definitions](../object-definitions/) — Definition registry, binary encoding
- [Game Objects (Server)](../game-objects-server/) — Entity lifecycle pattern, plugin integration
- [Networking](../networking/) — Packet structure, UpdatePacket flags, serialization protocol
- [Sound Management](../sound-management/) — Audio channels, spatial audio, falloff curves
- [UI Management](../ui-management/) — Wheel button, requestable items, blockEmoting state
- [Game Loop](../game-loop/) — Tick lifecycle, entity cleanup, plugin event bus
- [Client Rendering](../client-rendering/) — Sprite loading, tweening library, Z-indexing

### Tier 3
- [Player (Server)](../game-objects-server/modules/player.md) — sendEmote(), loadout field, rate-limit
- [Player (Client)](../game-objects-client/modules/player.md) — showEmote(), animation rendering
- [Input Packet](../serialization-system/modules/input-packet.md) — InputActions.Emote encoding
- [Update Packet](../serialization-system/modules/update-packet.md) — EmoteSerialization flags
