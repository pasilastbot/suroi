# Console & Debug System

<!-- @tier: 2 -->
<!-- @parent: docs/architecture.md -->
<!-- @modules: docs/subsystems/console-debug/modules/ -->
<!-- @source: client/src/scripts/console/ -->

## Purpose

In-game developer console and debug visualization system. Provides:
- **Text command interface** — type commands to inspect and manipulate game state
- **Console variables (CVars)** — runtime settings (graphics, audio, gameplay, debug flags)
- **Debug rendering** — overlay hitbox/grid visualization
- **Performance monitoring** — real-time FPS, network latency, and update packet size graphs
- **Server debug packets** — remote cheat commands from server (speed override, noclip, invulnerability, spawn dummy/loot)

## Key Files & Entry Points

| File | Purpose |
|------|---------|
| [`client/src/scripts/console/gameConsole.ts`](../../client/src/scripts/console/gameConsole.ts) | Console UI window (FloatingWindow), output log, input prompt, autocomplete |
| [`client/src/scripts/console/variables.ts`](../../client/src/scripts/console/variables.ts) | Console variables (CVars) registry, type casters, default game settings |
| [`client/src/scripts/console/commands.ts`](../../client/src/scripts/console/commands.ts) | Built-in commands (movement, inventory, combat, screenshots, utilities) |
| [`client/src/scripts/console/internals.ts`](../../client/src/scripts/console/internals.ts) | Query parsing, command chaining (`;`, `&`, `\|`), error handling |
| [`client/src/scripts/utils/debugRenderer.ts`](../../client/src/scripts/utils/debugRenderer.ts) | Debug shape rendering (hitboxes, lines, rays, circles) |
| [`client/src/scripts/utils/graph/graph.ts`](../../client/src/scripts/utils/graph/graph.ts) | Generic data visualization (time-series graphs with history buffer) |
| [`client/src/scripts/utils/graph/netGraph.ts`](../../client/src/scripts/utils/graph/netGraph.ts) | Network performance metrics (latency, packet size) |
| [`common/src/packets/debugPacket.ts`](../../../../../common/src/packets/debugPacket.ts) | Server → Client debug commands (speed, zoom, noclip, invulnerable, spawn) |

## Architecture

### Console UI Flow

```
Player presses backtick (`)
  ↓
GameConsole.toggle() → show input field with focus
  ↓
Player types: "give ak47"
  ↓
Press Enter
  ↓
GameConsole.execute("give ak47")
  ├─→ Parse via extractCommandsAndArgs()
  │   (handles chaining with ; & |)
  │
  ├─→ Look up "give" in GameConsole.commands registry
  │
  ├─→ Call Command.executor("ak47")
  │   └─→ Handler sends packet/modifies state
  │
  └─→ Log result to output
```

### Console Variable (CVar) Architecture

CVar is a typed, observable container for configuration:

```typescript
ConVar<T> {
  value: T
  flags: { archive, readonly, cheat, replicated }
  changeListeners: (newVal, oldVal, cvar) => void
}
```

**Registry:** `CVarCasters` maps CVar name → type caster function  
**Loader:** `defaultClientCVars` provides initial values from config/defaults  
**Persistence:** Archive flag marks CVars to save to local storage

### Debug Renderer

Maintains a list of shapes to render each frame:

```
DebugRenderer.addHitbox(hitbox, "red", 0.8)
DebugRenderer.addLine(pointA, pointB, "blue")
DebugRenderer.addRay(position, angle, length, "yellow")
DebugRenderer.addCircle(radius, position, "green")
  ↓
Graphics object queued for rendering
  ↓
Renders as overlay on top of game world (pixel space)
```

### Performance Graphs

Graph class renders time-series data with scrolling history:

```
Graph<[fps]>
  ├─ Container (positioned, sized)
  ├─ Graphics (background, border, curve)
  ├─ Title text
  └─ Labels (e.g., "FPS: 60")

update(timestamp, value) → add to history buffer (FIFO)
render() → redraw curve across maxHistory samples
```

## Console Variables (CVars)

Real CVars define game behavior at runtime:

### Loadout & Identity
| Variable | Type | Default | Scope |
|----------|------|---------|-------|
| `cv_player_name` | string | "" | Client |
| `cv_loadout_skin` | string | (default skin) | Client |
| `cv_loadout_badge` | string | "" | Client |
| `cv_loadout_crosshair` | int | 0 | Client |
| `cv_loadout_*_emote` | string | (varies) | Client |

### Audio
| Variable | Type | Default | Range |
|----------|------|---------|-------|
| `cv_master_volume` | number | 1.0 | 0.0–1.0 |
| `cv_sfx_volume` | number | 1.0 | 0.0–1.0 |
| `cv_ambience_volume` | number | 1.0 | 0.0–1.0 |
| `cv_music_volume` | number | 1.0 | 0.0–1.0 |
| `cv_mute_audio` | boolean | false | — |

### Graphics & Rendering
| Variable | Type | Options | Purpose |
|----------|------|---------|---------|
| `cv_renderer` | choice | `"webgl1"` \| `"webgl2"` \| `"webgpu"` | Rendering API |
| `cv_renderer_res` | choice | `"auto"` \| `"0.5"` \| `"1"` \| `"2"` \| `"3"` | Resolution scale |
| `cv_antialias` | boolean | true | —  |
| `cv_high_res_textures` | boolean | false | Load 4096×4096 spritesheet |
| `cv_cooler_graphics` | boolean | false | Extra visual effects |
| `cv_camera_shake_fx` | boolean | true | Screen shake on events |
| `cv_explosion_shockwaves` | boolean | true | Shockwave visuals |
| `cv_blood_splatter` | boolean | true | Gore |
| `cv_ambient_particles` | boolean | true | Fog, falling leaves, etc. |
| `cv_blur_splash` | boolean | true | Damage screen blur |

### UI & HUD
| Variable | Type | Default | Purpose |
|----------|------|---------|---------|
| `cv_draw_hud` | boolean | true | Hide/show all UI |
| `cv_ui_scale` | number | 1.0 | HUD scaling |
| `cv_map_expanded` | boolean | false | Big map state |
| `cv_minimap_minimized` | boolean | false | Minimap collapsed |
| `cv_minimap_transparency` | number | 0.8 | Minimap opacity |
| `cv_killfeed_style` | choice | `"icon"` \| `"text"` | Kill notification format |
| `cv_weapon_slot_style` | choice | `"simple"` \| `"colored"` | Weapon slot styling |
| `cv_weapon_compare` | boolean | false | Show weapon stats popup |
| `cv_hide_emotes` | boolean | false | Hide player emotes |

### Controls & Input
| Variable | Type | Default | Purpose |
|----------|------|---------|---------|
| `cv_loop_scope_selection` | boolean | false | Cycle scopes with scroll wheel |
| `cv_responsive_rotation` | boolean | true | Smooth aim rotation |
| `cv_movement_smoothing` | boolean | true | Smooth player movement animation |
| `cv_autopickup` | boolean | true | Auto-pickup nearby loot |
| `cv_autopickup_dual_guns` | boolean | false | Auto-equip second gun |

### Performance Display
| Variable | Type | Default | Purpose |
|----------|------|---------|---------|
| `pf_show_fps` | boolean | false | Show frames per second |
| `pf_show_ping` | boolean | false | Show network latency |
| `pf_show_pos` | boolean | false | Show player coordinates |
| `pf_show_inout` | boolean | false | Show input/output bytes |
| `pf_net_graph` | choice | 0 | `0` = off, `1` = label only, `2` = graph + label |

### Debug Visualization
| Variable | Type | Default | Purpose |
|----------|------|---------|---------|
| `db_show_hitboxes` | boolean | false | Show all hitboxes |
| `db_show_hitboxes_players` | boolean | false | Player collision shapes only |
| `db_show_hitboxes_obstacles` | boolean | false | Obstacles/props only |
| `db_show_hitboxes_stairs` | boolean | false | Stairs collision |
| `db_show_hitboxes_loot` | boolean | false | Loot item collision |
| `db_show_hitboxes_buildings` | boolean | false | Building collision |
| `db_show_hitboxes_buildings_ceilings` | boolean | false | Ceiling collision |
| `db_show_hitboxes_synced_particles` | boolean | false | Network particle collision |
| `db_show_hitboxes_terrain` | boolean | false | Terrain collision |

### Debug Gameplay Overrides
| Variable | Type | Default | Cheat Flag |
|----------|------|---------|-----------|
| `db_speed_override` | number | 0 | ✓ Override movement speed |
| `db_override_zoom` | boolean | false | ✓ Force custom zoom |
| `db_zoom_override` | number | 0 | ✓ Zoom level (0–255) |
| `db_no_clip` | boolean | false | ✓ Walk through walls |
| `db_invulnerable` | boolean | false | ✓ Take no damage |

### Console UI
| Variable | Type | Purpose |
|----------|------|---------|
| `cv_console_width` | number | Console window width (px) |
| `cv_console_height` | number | Console window height (px) |
| `cv_console_left` | number | Console X position |
| `cv_console_top` | number | Console Y position |
| `cv_console_open` | boolean | Console visible |

### Mobile-Specific
| Variable | Type | Purpose |
|----------|------|---------|
| `mb_controls_enabled` | boolean | Enable on-screen joysticks |
| `mb_joystick_size` | number | Joystick radius |
| `mb_haptics` | boolean | Vibration feedback |
| `mb_shake_to_reload` | boolean | Shake device to reload weapon |

## Built-in Commands

### Movement (Invertible Pairs)

Invertible commands have `+cmd` (start) and `-cmd` (stop) variants:

| Command | Arguments | Effect |
|---------|-----------|--------|
| `+up` / `-up` | none | Move up / stop |
| `+left` / `-left` | none | Move left / stop (spectate previous when watching) |
| `+down` / `-down` | none | Move down / stop |
| `+right` / `-right` | none | Move right / stop (spectate next when watching) |

### Inventory Management

| Command | Arguments | Effect |
|---------|-----------|--------|
| `slot` | int (0–2) | Switch to weapon slot by index |
| `last_item` | none | Equip previous active weapon |
| `other_weapon` | none | Swap to other weapon or melee |
| `swap_gun_slots` | none | Exchange slot 0 ↔ slot 1 guns |
| `cycle_items` | int (offset) | Cycle weapons by offset, skipping empty slots |
| `lock_slot` | int (optional) | Lock current or specified slot (prevents item swap) |
| `unlock_slot` | int (optional) | Unlock current or specified slot |
| `toggle_slot_lock` | int (optional) | Toggle lock on slot |
| `loot` | none | Pick up closest item in range |
| `interact` | none | Interact with closest interactable object |

### Combat & Actions

| Command | Arguments | Effect |
|---------|-----------|--------|
| `+attack` / `-attack` | none | Start / stop attacking (hold down) |
| `use` | idString | Use consumable, scope, or throwable by definition ID |
| `reload` | none | Reload current weapon |
| `explode_c4` | none | Detonate all deployed C4 |
| `cancel_action` | none | Stop current reload/consumable use |

### Game Map

| Command | Arguments | Effect |
|---------|-----------|--------|
| `+view_map` / `-view_map` | none | Show / hide fullscreen map |

### Screenshots

| Command | Arguments | Effect |
|---------|-----------|--------|
| `screenshot_map` | none | Screenshot minimap, open as blob image |
| `screenshot_game` | none | Screenshot game viewport (no HUD), open as blob image |

### Console Utilities

| Command | Arguments | Effect |
|---------|-----------|--------|
| `clear` | none | Clear console output log |
| `clear_history` | none | Clear console command history |
| `echo` | string... | Print message to console |
| `disconnect` | none | Leave current game |

## Debug Visualization

### Hitbox Overlay

Enable with `db_show_hitboxes 1`. Displays collision shapes:

```
db_show_hitboxes = true
  → DebugRenderer.addHitbox() called for each object
  → Color: varies by object type
  → Opacity: 0.6–0.8
  → Rendered as overlay on top of world
```

**Per-object control:**
- `db_show_hitboxes_players` — player bodies
- `db_show_hitboxes_obstacles` — trees, rocks, barrels
- `db_show_hitboxes_stairs` — stair interactions
- `db_show_hitboxes_loot` — item pickups
- `db_show_hitboxes_buildings` — building walls
- `db_show_hitboxes_buildings_ceilings` — ceiling collision
- `db_show_hitboxes_synced_particles` — network particles
- `db_show_hitboxes_terrain` — terrain collision

### Shape Types

DebugRenderer supports four shape types:

```typescript
addLine(a: Vector, b: Vector, color, alpha)
  → Draw line between two points

addRay(position, angle, length, color, alpha)
  → Draw ray from position in direction

addHitbox(hitbox, color, alpha)
  → Draw CircleHitbox or RectangleHitbox

addCircle(radius, position, color, alpha)
  → Draw circle (wrapper around addHitbox)
```

## Performance Graphs

Real-time monitoring displays:

### FPS Graph

```
pf_show_fps = true
  → Measure frame render time
  → Display: "FPS: 60" or similar
  → Render scrolling history (150 samples)
  → Color green (60 FPS) → red (drops) → blue (capped)
```

### Network Latency Graph

```
pf_net_graph = 1     # Label only: "RTT: 45 ms"
pf_net_graph = 2     # Graph + label with history
  → Measure round-trip time (UpdatePacket ↔ ack)
  → Display moving average
  → Red = high latency, green = low
```

### Network I/O

```
pf_show_inout = true
  → Track bytes sent/received per frame
  → Display: "OUT: 120 B  IN: 2.4 KB"
```

### Zoom & Position

```
pf_show_pos = true
  → Display player coordinates: "X: 512.3  Y: 128.7  R: 45°"
  → Updates every frame
```

## DebugPacket (Server → Client)

Server can send cheat commands to clients via `DebugPacket`. Interface:

```typescript
interface DebugPacket {
  readonly type: PacketType.Debug
  readonly speed: number                         // Speed multiplier
  readonly overrideZoom: boolean                 // Force custom zoom
  readonly zoom: number                          // Zoom level (0–255)
  readonly noClip: boolean                       // Walk through walls
  readonly invulnerable: boolean                 // Take no damage
  readonly spawnLootType?: LootDefinition        // Loot to spawn
  readonly spawnDummy: boolean                   // Spawn AI dummy
  readonly dummyVest?: ArmorDefinition           // Armor on dummy
  readonly dummyHelmet?: ArmorDefinition         // Helmet on dummy
  readonly layerOffset: number                   // Rendering depth
}
```

Server sends this only when cheat mode is enabled (admin/developer server).

## Query Parsing & Chaining

Console supports **command chaining** to run multiple commands in sequence:

```
Give two weapons: give ak47 & give mp5
  → Command 1: "give ak47"
    ├─ success → continue
    └─ error → STOP
  → Command 2: "give mp5"

Chain conditional on success: cmd1 & cmd2
  → Run cmd2 only if cmd1 succeeds

Chain conditional on failure: cmd1 | cmd2
  → Run cmd2 only if cmd1 fails

Unconditional chain: cmd1 ; cmd2
  → Run cmd2 regardless of cmd1 result
```

**Right-associativity:** `a & b ; c` is equivalent to `a & (b ; c)`

## Known Issues & Gotchas

1. **Console not sandboxed** — Can execute arbitrary JavaScript via eval-like parsing. Trusts host only (single-player/local debug). Production builds have console disabled.

2. **Debug rendering performance cost** — Hitbox overlay, grid drawing, shape rendering adds CPU overhead. Disable for high-performance testing: `db_show_hitboxes 0`.

3. **CVars not persisted across page reload** — Archive flag marks intention to save, but persistence depends on individual app implementation (localStorage, etc.). Some CVars reset on reload.

4. **Command name typos fail silently** — No "command not found" message. Check console log for errors or use `clear_history` to see previous attempts.

5. **Cheats visible to all clients** — Teleporting via cheats shows to spectators. Other players see your position change (no client-side prediction correction for cheat commands).

6. **DebugPacket requires server cheat mode** — Server-side cheats (noclip, invulnerable) only work if server has cheat mode enabled. Packets are ignored otherwise.

7. **Graph history buffer fixed size** — All graphs have `maxHistory = 150` samples. Older data is discarded (FIFO buffer). Cannot zoom out to see older data.

## Dependencies

### Depends on:
- [Client Rendering](../client-rendering/) — debug shape rendering to screen, PixiJS Graphics integration
- [Networking](../networking/) — DebugPacket sending/receiving, InputPacket for command execution
- [Game Objects (Client)](../game-objects-client/) — access to object positions/hitboxes for debug visualization
- [Input Management](../input-management/) — command input, key binding

### Depended on by:
- None (dev-only subsystem); disabled in production builds

## Module Index (Tier 3)

For implementation details, see:
- [Console UI & Input](modules/console-ui.md) — GameConsole floating window, input field, output log, autocomplete
- [Console Variables](modules/variables.md) — CVar registry, type casters, defaults, persistence
- [Commands & Builtin Functions](modules/commands.md) — command registry, execution, movement/inventory/combat command definitions
- [Debug Renderer](modules/debug-renderer.md) — shape rendering system, hitbox overlay, grid visualization
- [Performance Graphs](modules/graphs.md) — time-series graphing, FPS/latency/I/O monitoring, scrolling history
- [Query Parsing & Chaining](modules/internals.md) — console language parsing, command chaining operators (`;`, `&`, `|`), error handling

## Related Documents

### Tier 1
- [Architecture Overview](../../architecture.md) — System design, client-server model
- [Development Guide](../../development.md) — Console command reference, debug features

### Tier 2 — Related Subsystems
- [Client Rendering](../client-rendering/) — PixiJS integration, debug draw on canvas
- [Networking](../networking/) — DebugPacket protocol, UpdatePacket for cheat verification
- [Game Objects (Client)](../game-objects-client/) — object inspection via console
- [Input Management](../input-management/) — keyboard input, command binding

### Patterns
- **CVar pattern** — Typed, observable configuration container (used across engine: graphics, audio, gameplay, debug)
- **Command pattern** — Reusable command executor with signatures and error handling
- **Graph pattern** — Generic time-series visualization (extensible to any metric: latency, CPU, memory, tickrate)
