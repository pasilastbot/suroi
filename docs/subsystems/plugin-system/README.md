# Plugin System

<!-- @tier: 2 -->
<!-- @parent: docs/architecture.md -->
<!-- @source: server/src/pluginManager.ts -->

## Purpose

Server-side extensibility via a typed event system. Plugins receive game events and can read game state, modify behavior (damage amounts, item definitions), cancel actions, and communicate with players. Alternative to hardcoding custom game rules; enables experimental features and developer tools without modifying core code.

## Key Files & Entry Points

| File | Purpose |
|------|---------|
| `server/src/pluginManager.ts` | Event definitions (`Events`), `GamePlugin` base class, `PluginManager` orchestrator |
| `server/src/plugins/` | Example plugins bundled with server |
| `server/config.example.json` | Plugin loading configuration (list of enabled plugins) |
| `server/config.schema.json` | Schema for `plugins` config array |

## Architecture

### Loading & Initialization

```
Server startup
  → Game constructor → PluginManager(game)
  → PluginManager.loadPlugins() [async]
  → for each plugin name in config.plugins:
    → import(./plugins/${name}) → get default export
    → new PluginClass(game) → calls initListeners()
    → register in _plugins Set
```

Plugins are loaded in **config array order**, during game instantiation (before game tick loop starts).

### Event System

1. **Event Definition**: `Events` object defines ~35 event types using `makeEvent(cancellable?)` factory
   - `makeEvent(true)` → cancellable event (handler can prevent side effects)
   - `makeEvent()` → informational event (read-only)

2. **Type Safety**: `EventDataMap<EventName>` interface maps each event to its payload type
   - EventHandler receives typed data + event control object
   - TypeScript enforces correct callback signatures per event

3. **Dispatch**: `PluginManager.emit(eventType, ...data)` fires event to all registered plugins
   - Iterates plugins in load order
   - Each plugin's handler invoked via `pluginDispatchers` WeakMap
   - Errors in handlers logged but don't crash dispatcher
   - Returns first plugin that cancelled (or undefined)

4. **Handler Registration**: `GamePlugin.on(eventType, callback)`
   - Stores callbacks in `_events` map (Set per event type)
   - Multiple handlers per event allowed; executed in registration order

### Plugin Lifecycle

```
Plugin init
  ↓ [extends GamePlugin, calls initListeners()]
Plugin.initListeners() → this.on(eventType, handler)
  ↓ [register event listeners in _events]
Plugins ready for execution
  ↓ [game loop begins]

Per game tick:
  game.update()
    → player actions fire events
    → PluginManager.emit(eventType, ...)
    → plugin handlers execute

Server shutdown:
  → plugins discarded
  → (no cleanup hook; plugins must handle cleanup in listeners)
```

## Event System

Plugins listen to typed events via `this.on(eventTypeName, handler)`. Each event has:
- **Name**: Snake_case identifier (e.g., `player_did_die`)
- **Cancellable**: Some events can be cancelled to prevent side effects
- **Data**: Typed object specific to the event
- **Control object**: Provides `cancel()`, `stopImmediatePropagation()`, `isCancelled` readonly

### Event Categories & Reference

#### Player Events

| Event | Cancellable | Data | Fired When |
|-------|-------------|------|-----------|
| `player_will_connect` | ✓ | `never` | WebSocket connection initiated; cancelling closes socket |
| `player_did_connect` | ✗ | `Player` | Player object created and placed |
| `player_will_join` | ✓ | `{ player, joinPacket }` | Before player added to game collections; cancelling disconnects |
| `player_did_join` | ✗ | `{ player, joinPacket }` | After player added to all collections |
| `player_disconnect` | ✗ | `Player` | After player removed from game and teams |
| `player_update` | ✗ | `Player` | After player's first update pass (physics, health, adrenaline, items) |
| `player_start_attacking` | ✓ | `Player` | First tick of attack; cancelling prevents item use |
| `player_stop_attacking` | ✓ | `Player` | First tick after attack ends; cancelling prevents item stopUse |
| `player_input` | ✗ | `{ player, packet }` | After InputPacket processed; all side effects applied |
| `player_will_emote` | ✓ | `{ player, emote }` | Before emote used; cancelling prevents emote |
| `player_did_emote` | ✗ | `{ player, emote }` | After emote used |
| `player_will_map_ping` | ✓ | `{ player, ping, position }` | Before map ping used; cancelling prevents ping |
| `player_did_map_ping` | ✗ | `{ player, ping, position }` | After map ping used |
| `player_damage` | ✗ | `DamageParams & { player }` | Before piercing damage applied (informational) |
| `player_will_piercing_damaged` | ✓ | `DamageParams & { player }` | Before damage applied; cancelling prevents damage |
| `player_did_piercing_damaged` | ✗ | `DamageParams & { player }` | After damage applied; no death check yet |
| `player_will_die` | ✗ | `DamageParams & { player }` (health ≤ 0) | Before death processing; no flags set yet |
| `player_did_die` | ✗ | `DamageParams & { player }` | After all death cleanup (inventory cleared, removed from game) |
| `player_did_win` | ✗ | `Player` | Player won the match |

#### Inventory Events

| Event | Cancellable | Data | Fired When |
|-------|-------------|------|-----------|
| `inv_item_equip` | ✗ | `InventoryItem` | Item equipped (when swapping, old unequipped first) |
| `inv_item_unequip` | ✗ | `InventoryItem` | Item unequipped |
| `inv_item_use` | ✓ | `InventoryItem` | Item used (gun: attack, grenade: cook start); cancelling prevents use and resets attacking flag |
| `inv_item_stop_use` | ✓ | `InventoryItem` | Item stopUse called; cancelling prevents stopUse (attacking flag unchanged) |
| `inv_item_stats_changed` | ✗ | `{ item, oldStats, newStats, diff }` | Item stats changed (e.g., due to armor modifiers) |
| `inv_item_modifiers_changed` | ✗ | `{ item, oldMods, newMods, diff }` | Item modifiers changed |

#### Obstacle Events

| Event | Cancellable | Data | Fired When |
|-------|-------------|------|-----------|
| `obstacle_will_generate` | ✓ | `{ type, position, rotation, layer, scale, variation?, lootSpawnOffset?, locked?, activated? }` | Start of obstacle generation (map gen + airdrops); cancelling prevents spawn |
| `obstacle_did_generate` | ✗ | `Obstacle` | End of obstacle generation |
| `obstacle_will_damage` | ✓ | `DamageParams & { obstacle, position? }` | Before obstacle damage applied; cancelling prevents damage |
| `obstacle_did_damage` | ✗ | `DamageParams & { obstacle, position? }` | After damage applied; before death checks |
| `obstacle_will_destroy` | ✓ | `DamageParams & { obstacle, position? }` | Before destruction side effects (loot, particles); cancelling prevents effects |
| `obstacle_did_destroy` | ✗ | `DamageParams & { obstacle, position? }` | After all destruction side effects spawned |
| `obstacle_will_interact` | ✓ | `{ obstacle, player? }` | Before interaction (all validity checks passed); cancelling prevents interaction |
| `obstacle_did_interact` | ✗ | `{ obstacle, player? }` | After interaction |

#### Loot Events

| Event | Cancellable | Data | Fired When |
|-------|-------------|------|-----------|
| `loot_will_generate` | ✓ | `{ definition, position, layer, count?, pushVel?, jitterSpawn?, data? }` | Before loot object created; cancelling prevents generation |
| `loot_did_generate` | ✗ | `{ loot, position, layer, ... }` | After loot created and added to game |
| `loot_will_interact` | ✓ | `{ loot, canPickup?, player }` | Before player picks up loot (validity checks passed); cancelling prevents pickup |
| `loot_did_interact` | ✗ | `{ loot, player }` | After inventory updated and loot removed |

#### Building Events

| Event | Cancellable | Data | Fired When |
|-------|-------------|------|-----------|
| `building_will_generate` | ✓ | `{ definition, position, orientation, layer }` | Start of building generation; cancelling prevents generation |
| `building_did_generate` | ✗ | `Building` | End of building generation |
| `building_will_damage_ceiling` | ✓ | `{ building, damage }` | Before ceiling damage applied; cancelling prevents damage |
| `building_did_damage_ceiling` | ✗ | `{ building, damage }` | After ceiling damage applied |
| `building_did_destroy_ceiling` | ✗ | `Building` | Ceiling destroyed (too many walls destroyed) |

#### Airdrop Events

| Event | Cancellable | Data | Fired When |
|-------|-------------|------|-----------|
| `airdrop_will_summon` | ✓ | `{ position }` (desired, terrain-adjusted later) | Before airdrop scheduled; cancelling prevents summon |
| `airdrop_did_summon` | ✗ | `{ airdrop, position }` (actual position) | After airdrop scheduled with final position |
| `airdrop_landed` | ✗ | `Airdrop` | Parachute hit ground, crate generated |

#### Game Events

| Event | Cancellable | Data | Fired When |
|-------|-------------|------|-----------|
| `game_created` | ✗ | `Game` | Game instantiated; grid, map, gas loaded; game loop not yet started |
| `game_tick` | ✗ | `Game` | End of game tick (all side effects applied, next tick not scheduled) |
| `game_end` | ✗ | `Game` | Match ended (player/team won) |

## Plugin Development Pattern

### Basic Structure

```typescript
import { GamePlugin } from "../pluginManager";

export default class MyPlugin extends GamePlugin {
  protected override initListeners(): void {
    // Register all event handlers here
    this.on("player_did_join", ({ player }) => {
      // Handle player join
    });

    this.on("player_will_piercing_damaged", (data, event) => {
      if (shouldCancel) {
        event.cancel(); // Prevent damage
      }
    });
  }
}
```

### Real Examples from Bundled Plugins

**SpeedTogglePlugin** — Toggle player speed on emote:
```typescript
export default class SpeedTogglePlugin extends GamePlugin {
  protected override initListeners(): void {
    this.on("player_did_emote", ({ player }) => {
      const baseSpeed = GameConstants.player.baseSpeed;
      if (player.baseSpeed === baseSpeed) {
        player.baseSpeed = 12 * baseSpeed;
      } else {
        player.baseSpeed = baseSpeed;
      }
    });
  }
}
```

**TeleportPlugin** — Teleport player to map ping location:
```typescript
export default class TeleportPlugin extends GamePlugin {
  protected override initListeners(): void {
    this.on("player_did_map_ping", ({ player, position }) => {
      player.position = Vec.clone(position);
      player.updateObjects = true;
      player.setPartialDirty();
    });
  }
}
```

**WeaponSwapPlugin** — Random weapon on kill:
```typescript
export default class WeaponSwapPlugin extends GamePlugin {
  protected override initListeners(): void {
    this.on("player_will_die", ({ source }) => {
      if (!(source instanceof Player)) return;
      
      const newWeapon = pickRandomInArray(selectableGuns);
      source.inventory.replaceWeapon(source.activeItemIndex, newWeapon);
    });
  }
}
```

**PlaceObjectPlugin** — Development utility: place/rotate obstacle at mouse:
```typescript
export default class PlaceObjectPlugin extends GamePlugin {
  private readonly _playerToObstacle = new ExtendedMap<Player, Obstacle>();

  protected override initListeners(): void {
    this.on("player_did_join", ({ player }) => {
      const obstacle = new Obstacle(player.game, "house_column", player.position);
      this._playerToObstacle.set(player, obstacle);
      this.game.grid.addObject(obstacle);
    });

    this.on("player_did_emote", ({ player }) => {
      this._playerToObstacle.ifPresent(player, obstacle => {
        obstacle.rotation = (obstacle.rotation + 1) % 4;
        this.updateObstacle(obstacle);
      });
    });
  }
}
```

## Configuration

Plugins declared in `server/config.json`:

```json
{
  "plugins": [
    "placeObjectPlugin",
    "speedTogglePlugin",
    "teleportPlugin"
  ]
}
```

- Array order defines **load order** (first plugin's listeners execute first per event)
- Names are **filenames without extension** (e.g., `speedTogglePlugin` → `server/src/plugins/speedTogglePlugin.ts`)
- Missing files: logged as error, plugin not loaded
- Plugin load errors: caught, logged, server continues
- `config.plugins` omitted or empty: no plugins loaded

## Plugin API Surface

Plugins inherit from `GamePlugin` and have access to:

### Base Class API

| Method | Signature | Purpose |
|--------|-----------|---------|
| `on()` | `on<Ev>(eventType: Ev, handler: EventHandler<Ev>): void` | Register event listener |
| `off()` | `off<Ev>(eventType?: Ev, handler?: EventHandler<Ev>): void` | Unregister listener (all if no args) |
| `constructor` | `constructor(public game: Game)` | `this.game` ref to active Game instance |

### Game Object Access

Via `this.game`, plugins can access:

| Property | Type | Purpose |
|----------|------|---------|
| `game.players` | `Set<Player>` | All players in game |
| `game.objects` | `BaseGameObject[]` | All game objects |
| `game.mapName` | `string` | Current map |
| `game.teamMode` | `TeamMode` | Solo/Duo/Squad |
| `game.grid` | `Grid` | Spatial hash for collision queries |
| `game.airdrop` | `Airdrop \| null` | Current airdrop (if active) |
| `game.gas` | `Gas` | Gas system |
| `game.map` | `GameMap` | Map generation / building / obstacle data |
| `game.log()` | `(msg: string) => void` | Log to console |

Data modifications via plugins are **immediate and net-replicated** (if dirty flags set):
- Modifying `player.position` → `setPartialDirty()` needed to send to clients
- Modifying items/inventory → `dirty.items = true`
- Modifying health/adrenaline → `setPartialDirty()` or `setFullDirty()`

## Dependencies

### Depends On
- [Game Loop](../game-loop/) — Event dispatch points in game tick and player update
- [Game Objects (Server)](../game-objects-server/) — Player, Building, Obstacle, Loot, Airdrop definitions and state
- [Inventory](../inventory/) — Item and inventory state for equipment/use events
- [Spatial Grid](../spatial-grid/) — Grid API for spatial queries
- [Core Math/Physics](../core-math-physics/) — Vec, Vector types used in event data

### Depended On By
- None (optional system)

## Restrictions & Isolation

- **No sandboxing**: Plugins run in **main server process thread**
- **Full access**: Plugins can read/write all game state
- **Single-threaded**: One plugin's heavy computation blocks game tick
- **Load order matters**: First plugin to call `event.cancel()` stops propagation; others might not run
- **No plugin-to-plugin calls**: Plugins can't depend on each other; no inter-plugin API
- **Error handling**: Plugin errors logged but don't crash dispatcher; be defensive
- **No cleanup hook**: Plugins must handle cleanup in their event handlers (e.g., player_disconnect)
- **No reload at runtime**: Plugin list only loaded at game init; must restart to enable/disable

## Known Issues & Gotchas

1. **No error recovery**: Plugin exceptions logged, dispatcher continues, but handler state may be inconsistent
2. **Cancellation chain**: Only first plugin's cancel is returned by `emit()`; later plugins don't know if event was cancelled by an earlier plugin
3. **Event ordering undefined**: If two plugins listen to same event, execution order is registration order within that plugin (sets), not reliable across plugins
4. **Dirty flag bugs**: Plugin modifies object state but forgets to set dirty flags → changes don't replicate to clients
5. **Plugin coupling**: If plugin A expects plugin B to run first, no way to enforce order (only config order hint)
6. **Memory leaks**: Plugin adds listener in `initListeners()` but never calls `off()` → listener stays until game ends
7. **Type-unsafety at plugin level**: TypeScript only helps at plugin compile time; if plugin loaded as js or corrupted, no type guarantees
8. **Player object lifetime**: `player_disconnect` might fire twice or not at all in edge cases (needs verification)

## Related Documents

### Tier 1
- [System Architecture](../../architecture.md) — Extensibility design overview
- [API Reference](../../api-reference.md) — Packet protocol for client-server comms (plugins may broadcast messages)

### Tier 2
- [Game Loop](../game-loop/) — Where events are emitted; tick timing (40 TPS)
- [Game Objects (Server)](../game-objects-server/) — Mutable state exposed to plugins
- [Networking](../networking/) — Dirty flags and replication (how changes broadcast to clients)

### Tier 3
- [`PluginManager` implementation](../server-utilities/) — Event dispatch mechanism, if documented
