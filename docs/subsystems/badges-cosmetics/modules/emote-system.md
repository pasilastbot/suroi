# Emote System Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/badges-cosmetics/README.md -->
<!-- @source: common/src/definitions/emotes.ts, server/src/objects/player.ts -->

## Purpose
Manages emote playback (gesture animations), definition structure, activation mechanics, network synchronization, and rate limiting to prevent spam and maintain server performance.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| `common/src/definitions/emotes.ts` | Emote definitions, categories, badges mapping | Medium |
| `server/src/objects/player.ts` | Emote activation, duration tracking, rate limit checks | High |
| `client/src/scripts/ui.ts` | Emote wheel UI, selection handlers, playback triggers | Medium |
| `client/src/scripts/game.ts` | Client-side emote rendering and animation playback | Medium |
| `common/src/packets/` | UpdatePacket emote field serialization | Low |

## Business Rules

### Emote Definition Structure
- **Categories:** EmoteCategory enum defines 7 categories
  - `People` — character/face emotes (Happy Face, Thumbs Up, etc.)
  - `Text` — word/text emotes (GG, LOL, GGEZ, etc.)
  - `Memes` — meme/trend emotes (Rickroll, etc.)
  - `Icons` — miscellaneous symbol emotes
  - `Misc` — uncategorized emotes
  - `Team` — team-only emotes (available only in team modes)
  - `Weapon` — weapon-specific emotes (gun wave, melee gesture, etc.)
- **Properties per emote:**
  - `idString` — unique identifier (e.g., `"happy_face"`)
  - `category` — EmoteCategory enum value
  - `hideInLoadout?` — if true, emote hidden from loadout UI (event-only emotes)
  - Sprite/image asset linked to spritesheet

### Emote Activation
- **Trigger:** Player activates emote via emote wheel (keybind + UI) or command
  - Both client and server trigger emote (client for UI confirmation, server for network)
- **Duration:** Fixed duration per emote, tracked on server
  - Emote "plays" for set duration (e.g., 2–3 seconds)
  - Other players see animation during this window
  - Duration stored in definition or hardcoded per category
- **Animation sequence:** Client-side emote animation mapped to frame range
  - Spritesheet contains emote animation frames
  - Animation loops or plays once based on emote type
  - Player animation state set to `AnimationType.Emote`

### Rate Limiting (Spam Prevention)
- **Per-player cooldown:** Minimum interval between emotes (e.g., 500 ms)
  - Prevents player spamming emote wheel rapidly
  - Cooldown stored as `lastEmoteTime: number` on Player object
  - Check: `now - lastEmoteTime < EMOTE_COOLDOWN` → reject
- **Per-server rate limit:** Aggregate emote count per tick
  - Total UpdatePackets with emote data limited per game tick
  - If tick emote count exceeds threshold → defer or drop oldest
- **Bandwidth consideration:** Emote data in UpdatePacket only if emote active
  - UpdatePacket.emote field: `{ idString: string; duration: number }`
  - Sending every tick = bandwidth cost; skip ticks where emote inactive

### Network Synchronization
- **UpdatePacket field:** Player's current emote encoded in update
  - `emote?: { idString: string; duration: number; progress: number }`
  - Only present if emote active (not null); omitted if no active emote
  - Sent with every player update during emote duration
- **Server-side tracking:** `player.activeEmote` tracks current emote
  - Set when player activates emote
  - Decremented each tick; nulled when duration expires
  - Cleared when player dies, is downed, or is revived (interruption)
- **Client-side playback:** Clients receive emote in UpdatePacket
  - Look up emote definition by idString
  - Play animation for received duration
  - Other players see animation overlaid on player model

### Badge Integration
- **Emote badges:** Some badges are emoted (BDG_* prefix)
  - Function: `isEmoteBadge(badge: BadgeDefinition | string): boolean`
  - Emote badges trigger emote playback linked to badge
  - Example: "BDG_happy_face" → plays "happy_face" emote
- **Badge to emote mapping:** Helper `getBadgeIdString(badge)` strips "BDG_" prefix
  - Emotes.definitions.find() matches remaining idString
  - Returns emote definition if found, else badge definition

## Data Lineage

```
Player activates emote (hotkey / UI)
  ↓
[Client-side validation: cooldown check]
  ↓
Emit event to server: "emote_activated"
  ↓
[Server-side validation: cooldown, player alive/not down]
  ↓
Set player.activeEmote = { idString, duration, startTime }
  ↓
Set player.animation = AnimationType.Emote
  ↓
[Each tick: decrement duration, check expiry]
  ↓
Serialize in UpdatePacket.playerData.emote
  ↓
[Clients receive update]
  ↓
Update GameObject in scene for other players
  ↓
Client plays emote animation for duration
  ↓
Animation finishes, emote cleared
```

### Emote Lifecycle Timings

| Event | Time | Handler |
|-------|------|---------|
| Player activates emote | T | `onEmoteActivated()` on server |
| Emote serialized in UpdatePacket | T+0 | Added to player update |
| Client receives UpdatePacket | T+1 | Network latency |
| Client plays animation | T+1..T+1+duration | Emote animation on screen |
| Server emote duration expires | T+duration | Cleared from server state |
| Client animation ends | T+1+duration | Removed from client render |

## Dependencies

- **Internal:**
  - Badges & Cosmetics — Badge definitions and emote badge mapping
  - Player (game object) — Player.activeEmote tracking
  - Animation System — AnimationType.Emote animation playback
- **External:**
  - [Object Definitions](../../object-definitions/README.md) — ObjectDefinitions registry for Emotes
  - [Networking](../../networking/README.md) — UpdatePacket emote field structure
  - [Game Objects Client](../../game-objects-client/README.md) — Client-side emote animation rendering

## Complex Functions

### Emote Activation Logic
**Function:** `emoteActivated(player: Player, emoteIdString: string): void`  
**Source:** `server/src/objects/player.ts` (implied pattern)  
**Purpose:** Validate and execute emote activation on server

**Logic:**
```
1. Check player alive (not downed, not dead)
2. Check cooldown: (now - player.lastEmoteTime) >= EMOTE_COOLDOWN
3. Verify emote exists: Emotes.tryGet(emoteIdString) !== undefined
4. Check mode allows emote category (e.g., Team emotes only in team modes)
5. Set player.activeEmote = { idString: emoteIdString, duration: getDuration(emote) }
6. Set player.lastEmoteTime = now
7. Set player.animation = AnimationType.Emote
8. Mark player dirty for next UpdatePacket
```

**Implicit behavior:**
- If any check fails, silently drop request (no error sent client)
- Cooldown resets even if validation fails (prevent cooldown bypass)
- Emote cancels ongoing action (reload, heal, revive) — no action queueing
- Emote duration is emote-specific OR category-specific (varies per mode)

**Called by:**
- WebSocket message handler — on client emote request
- Plugin event: `emote_activated` — plugins can hook for custom behavior

### Client-Side Animation Dispatch
**Function:** `playEmoteAnimation(player: GameObject, emote: EmoteDefinition): void`  
**Source:** `client/src/scripts/objects/player.ts` (implied pattern)  
**Purpose:** Map emote to spritesheet animation and play

**Logic:**
```
1. Look up spritesheet region for emote.idString
2. Get frame range from animation map (stored in JSON metadata)
3. Create PIXI AnimatedSprite with frames from spritesheet
4. Position overlay on player model (centered above head)
5. Play animation loop or once based on emote type
6. Schedule cleanup after duration + animation time
```

**Implicit behavior:**
- Spritesheet coords must match emote idString exactly (lookup failure = silent drop)
- Animation overlay appears above player model (z-order = player.z + 1)
- If animation longer than emote duration, animation clips (may look abrupt)
- Multiple emotes per frame: animation blends or uses frame priority

**Called by:**
- `onUpdatePacket()` in Game.ts — when player update received
- Local player emote activation — client-side confirmation visual

### Rate Limit Check
**Function:** `canEmote(player: Player): boolean`  
**Source:** `server/src/objects/player.ts`  
**Purpose:** Check if player can activate emote now

**Logic:**
```
const timeSinceLastEmote = player.game.now - (player.lastEmoteTime ?? 0)
const canReturn = timeSinceLastEmote >= EMOTE_COOLDOWN_MS
  && player.alive
  && !player.downed
  && !(player.action instanceof ReviveAction)  // can't emote while being revived
return canReturn
```

**Implicit behavior:**
- Cooldown is client-side AND server-side (dual validation)
- Server cooldown is authoritative (client can bypass locally, server rejects)
- `lastEmoteTime` never decrements after initial check (prevents race conditions)
- Downed/revived: emote immediately canceled (activeEmote set to null)

**Called by:**
- Emote wheel UI — before sending activation request
- Chat/command handler — before processing emote command
- Emote button click handler — prevents double-click spam

## Configuration

| Setting | Location | Effect | Default |
|---------|----------|--------|---------|
| `EMOTE_COOLDOWN_MS` | `common/src/constants.ts` | Minimum ms between emotes | 500 |
| `EMOTE_DURATION_MS` | Per-category or per-emote | How long emote animation plays | 2000–3000 |
| `EMOTE_MAX_PER_TICK` | `server/src/game.ts` | Max emotes serialized per tick | 20 |
| `hideInLoadout` | `emote: EmoteDefinition` | Hide emote from UI selection | false |

## Edge Cases & Gotchas

### Gotcha: Emote During Action Interruption
- **Issue:** Player activates emote while reviving teammate
  - Server accept emote, starts duration
  - Revive action simultaneously completes
  - Animation state conflict: emote overlay + revive animation both attempt to play
- **Prevention:** Check `!player.action` before accepting emote, or cancel current action

### Gotcha: Emote Persists After Death
- **Issue:** Player dies while emote active
  - Emote duration not cleared on death
  - Player appears as corpse with emote overlay still animating
- **Prevention:** Clear `player.activeEmote` in `player.onDown()` and `player.died()` handlers

### Gotcha: Emote Frame Mismatch
- **Issue:** Spritesheet repacked; emote animation frames change indices
  - Client still uses old frame numbers from cache
  - Emote animation plays wrong frames (shows random sprites)
- **Prevention:** Version spritesheet animations in metadata; cache-bust on deploy

### Gotcha: Badge-to-Emote Mapping Race
- **Issue:** Badge obtained but emote definition not yet loaded in client
  - Emote badge badge applied; `isEmoteBadge()` called
  - Emotes registry not yet populated
  - Returns false; emote doesn't trigger
- **Prevention:** Load emote definitions before badge definitions in definition registry initialization order

### Gotcha: Emote Spam via Ping Wheel
- **Issue:** Emote wheel + map ping wheel can both be activated simultaneously
  - User pings location AND emotes at same time
  - Two network packets sent; server processes both in same tick
  - Two emotes sent to UpdatePacket, exceeding bandwidth budget
- **Prevention:** If emote already active, drop new emote request until previous expires

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Badges & Cosmetics subsystem overview
- **Tier 1:** [../../architecture.md](../../architecture.md) — System architecture
- **Patterns:** [../patterns.md](../patterns.md) — Badge and cosmetic application patterns
- **Related modules:**
  - [Game Objects Client](../../game-objects-client/README.md) — Client-side player animation rendering
  - [Networking](../../networking/README.md) — UpdatePacket structure and emote field serialization
  - [Animation System](../../particle-system/README.md) — Sprite animation and timeline management
