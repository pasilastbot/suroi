# Client Auxiliary Managers

<!-- @tier: 2 -->
<!-- @parent: docs/architecture.md -->
<!-- @modules: docs/subsystems/client-managers/modules/ -->
<!-- @source: client/src/scripts/managers/ -->

## Purpose

Five singleton UI managers for secondary game systems: minimap display with pings, gas zone visualization, emote/ping wheel selection, perk tracking, and optional screen recording. These managers are initialized once at game startup and update independently based on game state and user input.

## Key Files & Entry Points

| Manager | File | Responsibility |
|---------|------|-----------------|
| **MapManager** | [client/src/scripts/managers/mapManager.ts](mapManager.ts) | Minimap rendering, map pings, terrain, objective markers, teammate indicators |
| **GasManager** | [client/src/scripts/managers/gasManager.ts](gasManager.ts) | Gas ring visuals, danger timer, warning messages, stage transitions |
| **EmoteWheelManager** | [client/src/scripts/managers/emoteWheelManager.ts](emoteWheelManager.ts) | Emote selection radial menu, keyboard/gamepad/touch navigation |
| **MapPingWheelManager** | [client/src/scripts/managers/emoteWheelManager.ts](emoteWheelManager.ts) | Map ping selection (extends EmoteWheelManager), player pings, ping placement |
| **PerkManager** | [client/src/scripts/managers/perkManager.ts](perkManager.ts) | Active perk tracking, perk list management, definition lookups |
| **ScreenRecordManager** | [client/src/scripts/managers/screenRecordManager.ts](screenRecordManager.ts) | In-game video recording via MediaRecorder API, download to disk |

## Architecture

Each manager is a singleton class instantiated once and accessed globally. They are initialized during `Game.init()` at startup:

```
Game.init()
  ├─ MapManager.init()
  ├─ GasManager.init()
  ├─ EmoteWheelManager.init()
  └─ ScreenRecordManager.init()
```

Managers integrate with the main game loop:
- **updateFrom()** — receive state changes from [Networking](../networking/) (UpdatePacket)
- **update()** — called each frame to animate time-dependent visuals (gas lerp, recording timer)
- **reset()** — called on game end or restart

Keyboard input via [Input Management](../input-management/) triggers manager actions:
- `cv_toggle_map` → `MapManager.toggleMinimap()`
- `cv_open_emotes` → `EmoteWheelManager.enabled = true`
- `cv_open_pings` → `MapPingWheelManager.enabled = true`

## MapManager

### Responsibilities
- Render **minimap** in viewport corner as 2D bird's-eye view
- Display **player position** and facing orientation on minimap
- Render **gas zone boundary** (safe zone outline, warning ring)
- Support **player pings** — dynamic map markers with type icon
- Display **teammate indicators** — colored position dots for squad members
- Render **terrain** — beach outline, rivers, grass, buildings
- Handle **minimap expansion** — toggle between small corner view and full-screen map

### Data Flow

```
Server UpdatePacket (gas position, safe zone, objects)
  ↓
MapManager.updateFromPacket()
  ├─ Gas position → GasRender.update() (for boundary ring)
  ├─ Player position → indicator.setVPos()
  ├─ Teammate positions → updateTeammates()
  └─ Ping list → pings collection
  ↓
Game loop: frame()
  └─ MapManager.container.sortChildren()
      (layers: terrain, gas ring, safe zone, pings, indicators, teammates)
```

### Key Methods

| Method | Purpose |
|--------|---------|
| `init()` | Create containers, graphics, mask; initialize GasRender; setup terrain |
| `renderMap()` | Async render terrain to RenderTexture (buildings, rivers, ground graphics) |
| `updateFromPacket(packet)` | Sync gas, player, teammate, ping state from server update |
| `drawTerrain(ctx, scale, gridLineWidth)` | Draw beach/grass/river/building terrain onto Graphics object |
| `isInOcean(position)` | Check if position is outside beach hitbox (in ocean) |
| `distanceToShoreSquared(position)` | Distance to nearest beach point (for water damage calc) |
| `updateTeammates(team)` | Add/update teammate indicators with color coding |
| `resize()` | Recalculate minimap bounds when viewport size changes |
| `toggle()` / `toggleMinimap()` | Toggle expanded or visibility state |

### State & Collections

| Property | Type | Purpose |
|----------|------|---------|
| `position` | Vec | Player position on world map |
| `_position`, `_lastPosition` | Vec | Interpolated player positions for smooth rendering |
| `pings` | Set<MapPing> | Active map pings from all players |
| `indicators` | Map<id, {sprite, tween}> | Animated objective/loot markers |
| `teammateIndicators` | Map<playerId, SuroiSprite> | Teammate position indicators |
| `indicator` | SuroiSprite | Player's own minimap position indicator |
| `terrain` | Terrain | Hitbox/river/beach data for current map |
| `container` | Container | Root PixiJS display object for entire minimap |
| `mask` | Graphics | Circular clipping mask for minimap border |
| `expanded` | bool | Is minimap fullscreen or corner-sized (getter/setter) |
| `visible` | bool | Is minimap shown or hidden (getter/setter) |

### Layer Structure (zIndex ordering)

```
1000: player.indicator (topmost)
 999: teammate.indicators
 998: pings
 997: map.indicators (objectives)
 997: safe.zone
 996: gas.ring (GasRender)
 --: terrain, buildings, rivers (sorted by Z_INDEX_COUNT)
  0: background
```

## GasManager

### Responsibilities
- **Gas state machine** — synchronize with server (Inactive → Waiting → Advancing)
- **Visual overlay** — render expanding gas ring with configurable color
- **Timing display** — show minutes:seconds until stage advance
- **Warning UI** — cyan-colored message during advancing phase, white during waiting
- **Gas interpolation** — smooth lerp between old/new positions every frame

### Data Flow

```
Server UpdatePacket (gas.state, gas.position, gas.radius, gasProgress ∈ [0, 1])
  ↓
GasManager.updateFrom(data)
  ├─ If advancing: position = lerp(oldPos, newPos, gasProgress)
  ├─ If advancing: radius = lerp(oldRadius, newRadius, gasProgress)
  └─ UI: show message + timer based on state
  ↓
Game loop: frame()
  └─ GasRender.update()
      └─ Interpolate position based on elapsed time
         (client-side extrapolation between server ticks)
```

### Key Methods

| Method | Purpose |
|--------|---------|
| `init()` | Cache jQuery DOM references for timer + message UI elements |
| `updateFrom(data)` | Parse gas state, position, radius, progress from server packet |
| `reset()` | Clear timer when game ends |
| **[GasRender]** | |
| `update()` | Calculate interpolated gas center + radius; redraw circle mask |

### Gas State Timeline

| State | Behavior | UI Display |
|-------|----------|------------|
| **Inactive** | Game over or no gas stages remaining | "Gas Inactive" (white) |
| **Waiting** | Countdown before ring starts advancing | "Gas advancing in M:SS" (white) |
| **Advancing** | Ring moves toward safe zone; deals damage | "Gas Advancing" (cyan, fades after 5s) |

### Key Properties

| Property | Type | Purpose |
|----------|------|---------|
| `state` | GasState enum | Current gas phase (Inactive, Waiting, Advancing) |
| `currentDuration` | number | Total milliseconds for current phase |
| `oldPosition`, `newPosition` | Vec | Gas center before/after stage advance |
| `oldRadius`, `newRadius` | number | Gas radius before/after advance |
| `position`, `radius` | Vec, number | **Current** (interpolated) gas boundary |
| `gasProgress` | [0, 1] | Server's interpolation factor for current advance |
| `time` | number \| undefined | Remaining milliseconds in phase (MM:SS) |

### GasRender Details

`GasRender` is a PixiJS Graphics object that renders the gas zone as a **circular hole cut from a massive outer rectangle**:

```javascript
// Overdraw large rectangle
ctx.beginPath()
  .moveTo(-100000, -100000)
  .lineTo(+100000, -100000)
  // ... 4 corners
  .closePath()
  .fill(gasColor)

// Cut out circular hole
ctx.moveTo(0, 1)
for (i = 0; i < 512; i++) {
  ctx.lineTo(sin(2π * i/512), cos(2π * i/512))
}
ctx.closePath()
ctx.cut()  // Creates the hole via path manipulation
```

This approach ensures smooth performance even as gas radius changes every frame.

## EmoteWheelManager & MapPingWheelManager

### Responsibilities (EmoteWheelManager)
- **Emote wheel UI** — radial menu with 8+ emote slots arranged in circle
- **Selection feedback** — highlight selected slot, show selection ring
- **Input handling** — keyboard/gamepad angle → select slot; click to emit
- **Mobile support** — touch selection + tap-to-select on mobile
- **Emote sending** — dispatch InputAction packet with selected emote

### Responsibilities (MapPingWheelManager)
- Extends EmoteWheelManager but filters to **player pings only** (ping type icons)
- **Position detection** — place ping at clicked world position
- **Minimap pings** — detect clicks on minimap vs world, place accordingly

### Data Flow

```
User input (canvas click, keyboard, gamepad)
  ↓
InputManager (drag angle, click position)
  ↓
EmoteWheelManager
  ├─ show() — display wheel at mouse cursor (desktop) or screen center (mobile)
  ├─ update() — track mouse angle, highlight selected emote slot
  ├─ close() — hide wheel + emit selected emote
  └─ emitEmote() — send InputAction.Emote packet
  ↓
Server processes emote, broadcasts to other players
```

### Key Methods

| Method | Purpose |
|--------|---------|
| `init()` | Create Graphics objects for background, ticks, selection ring, close button |
| `setupSlots()` | Calculate emote positions on circle; create sprites for each emote icon |
| `show()` | Display wheel at mouse cursor; set `active = true` |
| `close(emitEmote?)` | Hide wheel; if `emitEmote=true`, send selected emote packet |
| `emitEmote(emote?)` | Dispatch InputAction for emote to server |
| `update()` | Recalculate selection based on mouse angle; update ring visualization |

### Emote Wheel Geometry

```
                 ↑
                 |
           [slot 0]
        /           \
    [slot 7]      [slot 1]
   /               \
[slot 6]           [slot 2]
   \               /
    [slot 5]      [slot 3]
        \           /
           [slot 4]

Selection ring angle = (slotIndex + 0.5) * slotAngle
Each slot angle = 360° / numEmotes

Emote size calculated via sine rule to fit circle:
  emoteSize = sin(slotAngle) * midpoint / sin((180° - slotAngle) / 2)
```

### Key Properties

| Property | Type | Purpose |
|----------|------|---------|
| `enabled` | bool | Input action activates wheel (getter/setter) |
| `active` | bool | Wheel is currently visible |
| `selection` | number \| undefined | Index of currently selected emote (0 to numEmotes-1) |
| `emotes` | string[] | Array of emote idString references |
| **[MapPingWheelManager]** | |
| `position` | Vec \| undefined | World position where ping will be placed |
| `onMinimap` | bool | Is ping being placed on minimap view? |

### Static Configuration

```typescript
COLORS = {
  background: Color(h=0, s=0, l=9%, a=46%),     // dark near-black
  stroke: Color(h=0, s=0, l=75%, a=50%),        // light gray
  selection: 0xff7500                            // orange highlight
}

DIMENSIONS = {
  outerRingRadius: 130,          // max distance of emotes from center
  innerRingRadius: 20,           // center circle radius
  emotePadding: 8,               // space around each emote icon
  tickPadding: 8,                // tick mark margin
  strokeWidth: 5,                // outline width
  closeButtonSize: 6             // X button size
}
```

## PerkManager

### Responsibilities
- **Perk inventory tracking** — maintain client-side list of active perks
- **Definition lookup** — convert perk idString ↔ PerkDefinition
- **Presence checking** — fast `has(perk)` check for UI conditionals
- **Perk operations** — add/remove perks on loot pickup or death

### Data Flow

```
Server UpdatePacket (playerData.perks[])
  ↓
[not directly called by UpdatePacket - PerkManager is reactive]
  ↓
Game.updatePlayerObject()
  └─ For each new perk: PerkManager.add(perkIdString)
  └─ For each removed perk: PerkManager.remove(perkIdString)
```

Note: PerkManager **does not render UI itself**. UI is handled by [UI Management](../ui-management/). PerkManager is a **data store and query interface**.

### Key Methods

| Method | Signature | Purpose |
|--------|-----------|---------|
| `add(perk)` | `(perk: ReifiableDef<PerkDefinition>) ⇒ void` | Add perk to inventory (idempotent) |
| `remove(perk)` | `(perk: ReifiableDef<PerkDefinition>) ⇒ void` | Remove perk from inventory |
| `has(perk)` | `(perk: ReifiableDef<PerkDefinition>) ⇒ bool` | Check if player has perk |
| `map(perk, mapper)` | `<U>(perk, fn: PerkDef ⇒ U) ⇒ U \| undefined` | Apply function if perk exists |
| `mapOrDefault(perk, mapper, def)` | `<U>(perk, fn, default: U) ⇒ U` | Apply function or return default value |

### Key Properties

| Property | Type | Purpose |
|----------|------|---------|
| `perks` | PerkDefinition[] | List of active perks (max 4 per game settings) |

### Usage Pattern

```typescript
// Check if player has perk
if (PerkManager.has("adrenaline_rush")) {
  // Apply effect
}

// Get perk data and render
PerkManager.map("healing_aura", def => {
  UIManager.renderPerkIcon(def.idString, def.name);
});

// Get perk data with fallback
const perkName = PerkManager.mapOrDefault(
  "unknown_perk",
  def => def.displayName,
  "Unknown Perk"
);
```

## ScreenRecordManager

### Responsibilities
- **Video capture** via browser MediaRecorder API (requires user permission)
- **Recording UI** — show "Recording" indicator + elapsed time
- **Video export** — assemble Blob from chunks, trigger download as `.mp4`
- **Resolution control** — user selection (480p / 720p / 1080p / maximum)

### Data Flow

```
User clicks "Start Recording" button
  ↓
ScreenRecordManager.beginRecording()
  ├─ navigator.mediaDevices.getDisplayMedia()
     (shows browser "Share this tab" dialog)
  ├─ Create MediaRecorder(stream)
  ├─ Attach listeners: dataavailable, start, stop
  └─ mediaRecorder.start()
  ↓
Game loop: frame()
  └─ ScreenRecordManager.update()
      └─ Update recording time display (MM:SS)
  ↓
User clicks "Stop Recording" button
  ↓
ScreenRecordManager.endRecording()
  ├─ mediaRecorder.stop()
  ├─ "stop" event fires
  ├─ Assemble video Blob
  ├─ Create download link (data URI)
  └─ Trigger automatic download (Suroi_YYYY-MM-DD_HH-MM-SS.mp4)
```

### Key Methods

| Method | Purpose |
|--------|---------|
| `init()` | Cache jQuery stop button reference; attach click listener |
| `beginRecording()` | Request screen share permission; create MediaRecorder with audio + video |
| `endRecording()` | Stop recorder; trigger automatic download when "stop" event fires |
| `update()` | Update recording timer UI (MM:SS) each frame |
| `reset()` | Stop recording if game ends (cleanup) |

### Key Properties

| Property | Type | Purpose |
|----------|------|---------|
| `recording` | bool | Is recording currently active? |
| `mediaRecorder` | MediaRecorder \| undefined | Browser API recorder instance |
| `startedTime` | number | Timestamp when recording began |
| `videoData` | Blob[] | Accumulated video frame chunks |

### Browser Compatibility

| Browser | Support | Notes |
|---------|---------|-------|
| **Chrome** ✅ | Full | getDisplayMedia + MediaRecorder fully supported |
| **Firefox** ✅ | Full | Works with `preferCurrentTab: true` |
| **Safari** ❌ | Partial | getDisplayMedia not supported; graceful fallback |
| **Edge** ✅ | Full | Chromium-based, same as Chrome |

### Configuration

```typescript
// Video resolution control via console variable
cv_record_res: "480p" | "720p" | "1080p" | "maximum"

// Ideal height mapping
{
  "480p": 640 pixels
  "720p": 1280 pixels
  "1080p": 1920 pixels
  "maximum": undefined (browser max)
}

// DisplayMedia options
{
  audio: true,
  video: {
    displaySurface: "browser",
    width: { ideal: [...] },
    preferCurrentTab: true,          // stay on this tab
    surfaceSwitching: "exclude"      // don't switch tabs
  }
}
```

## Dependencies

### Depends On

| Subsystem | Why |
|-----------|-----|
| [Client Rendering](../client-rendering/) | PixiJS Container, Graphics, Sprite, RenderTexture for UI layers |
| [Game Objects (Client)](../game-objects-client/) | Player position, teammate list, gas state synchronization |
| [Networking](../networking/) | UpdatePacket: gas position/state, player data, teammate positions |
| [Input Management](../input-management/) | Keyboard actions, canvas click position, mouse angle, gamepad axis |
| [UI Management](../ui-management/) | DOM elements for timers, messages, buttons; HUD coordination |

### Depended On By

| Subsystem | Why |
|-----------|-----|
| [UI Management](../ui-management/) | PerkManager provides perk list; MapManager/GasManager render overlays |
| [Game Objects (Client)](../game-objects-client/) | Uses MapManager.terrain, MapManager.isInOcean() for terrain checks |
| [Input Management](../input-management/) | Responds to emote/ping input actions |

## Known Issues & Gotchas

### MapManager

1. **Minimap is 2D bird's-eye only** — no z-layering or 3D perspective; all sprites flat on one plane
2. **Gas ring jittering** — boundary snaps to 32-pixel grid at each render; smooth visual interpolation can mask discrete position updates
3. **Teammate indicators update latency** — position is 1-2 server ticks behind due to [UpdatePacket](../networking/) serialization scheduling
4. **Corner-case at map edges** — when player is near map boundary, minimap viewport may clip terrain; no wrapping
5. **Ping icons small on zoomed-out minimap** — difficult to see on lo-res screens; no scaling adjustment
6. **Terrain rebuild is async** — `renderMap()` is await-ed; first few frames may show blank minimap before terrain renders

### GasManager

1. **Gas color hardcoded to red** — not configurable per game mode; Game.colors.gas is immutable
2. **Timer display assumes no clock skew** — duration - (duration * progress) can lose precision for very long phases; use `Math.round(time)`
3. **Advancing state color change is instant** — no smooth fade between white/cyan; harsh switch at state boundary
4. **Final stage message** — "final_gas_advancing" vs "gas_advancing" translation may not exist in all languages; fallback to English
5. **GasRender segment count fixed at 512** — suitable for most resolutions but wastes cycles on small minmax or burns GPU on extreme zoom

### EmoteWheelManager & MapPingWheelManager

1. **Emote wheel on mobile is difficult** — radial selection is hard on touch; users often click wrong slot; keyboard preferable
2. **Mouse angle calculated in "pointer space"** — relative to wheel center; if wheel is repositioned mid-selection, angle becomes stale
3. **Keyboard selection index wraps instantly** — no smooth radial animation; selects next slot immediately
4. **Emote preview thumbnails unscaled** — icons stretched if emote definition sprite aspect ratio doesn't match hardcoded size
5. **MapPingWheelManager.position is optional** — if user clicks ping wheel on minimap but selection happens off-minimap, position is undefined; emote sent but ping placement fails silently
6. **Touch selection on mobile forces single choice** — no way to preview different emotes; must commit immediately

### PerkManager

1. **No max perk count enforced** — client accepts unlimited perks; server must enforce max=4; UI truncation doesn't validate
2. **Perk definitions are immutable** — changes to common/src/definitions/perks.ts require full client rebuild
3. **No perk expiry tracking** — perk removal is only via explicit `remove()` call; if server packet is lost, client perk list can desync
4. **Mapper functions don't handle undefined gracefully** — if perk idString doesn't exist in definition registry, `Perks.reify()` throws; no null-safe fallback

### ScreenRecordManager

1. **Safari unsupported** — getDisplayMedia not available in iOS Safari; no graceful error UI; silently fails
2. **Recording stops on full-screen exit** — if user presses F11 (full screen) or switches tabs, browser stops capture; no resume
3. **No video codec selection** — uses browser default (VP8 or H.264); can't force WebM vs MP4
4. **Filename uses browser Date.toLocaleDateString()** — inconsistent formatting across locales; assume ISO YYYY-MM-DD
5. **Large video files not warned** — 1080p 10-min recording = ~500-1000 MB; no quota check before starting
6. **Audio capture requires user permission** — separate from video; getDisplayMedia may request audio twice
7. **MediaRecorder is global** — only one recording at a time; if started twice, first instance silently discards
8. **Download via data URI has 2GB limit** — long recordings (>30 min) fail to download on some browsers

## Related Documents

### Tier 1

- [System Architecture](../../architecture.md) — Component overview, rendering pipeline, game loop
- [Data Model](../../datamodel.md) — Player, GasData, PerkDefinition entities

### Tier 2 — Subsystems

- [Client Rendering](../client-rendering/) — PixiJS layers, sprite batching, dirty-flag rendering
- [Game Objects (Client)](../game-objects-client/) — GameObject lifecycle, state sync from UpdatePacket
- [Networking](../networking/) — Packet types, UpdatePacket structure, server-to-client serialization
- [Input Management](../input-management/) — InputAction types, keyboard/gamepad/touch handlers
- [UI Management](../ui-management/) — UIManager, HUD elements, DOM integration

### Tier 3 — Modules

- [MapManager Module](modules/mapManager.md) — Detailed minimap rendering, terrain graphics, ping placement
- [GasManager Module](modules/gasManager.md) — Gas interpolation, lerp timing, visual effects
- [EmoteWheelManager Module](modules/emoteWheelManager.md) — Radial UI geometry, input selection, mobile support
- [PerkManager Module](modules/perkManager.md) — Perk storage, definition lookup, inventory API
- [ScreenRecordManager Module](modules/screenRecordManager.md) — MediaRecorder lifecycle, video export
