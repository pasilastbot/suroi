# Plugin Development Module

<!-- @tier: 3 -->
<!-- @parent: ../README.md -->
<!-- @source: server/src/pluginManager.ts, server/src/plugins/*.ts, server/src/game.ts -->

## Purpose

Documents the plugin system architecture, lifecycle, and event API for extending game behavior without modifying core code.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| `server/src/pluginManager.ts` | `GamePlugin` base class, `PluginManager` event bus, lifecycle | High |
| `server/src/plugins/placeObjectPlugin.ts` | Example plugin: place objects for dev | Medium |
| `server/src/plugins/speedTogglePlugin.ts` | Example plugin: speed modification | Medium |
| `server/src/plugins/teleportPlugin.ts` | Example plugin: teleportation support | Medium |
| `server/src/plugins/weaponSwapPlugin.ts` | Example plugin: instant weapon swap | Medium |
| `server/src/game.ts` | Game loop integration, event dispatching | High |

## Business Rules

- **Inheritance Required:** All plugins extend `GamePlugin` base class
- **Event-Driven:** Plugins subscribe to ~30+ typed events (player_did_join, player_update, etc.)
- **Cancellable Events:** Some events allow `event.cancel()` to prevent default behavior
- **Static Lifetime:** Plugins loaded at startup (specified in `config.json`); no live reload
- **Per-Game Isolation:** Each game instance has independent plugin instances
- **Manager Pattern:** `PluginManager` holds all plugins and dispatches events to subscribers

## Plugin Lifecycle

```
Server Startup
    ↓
PluginManager initializes
    ↓
For each plugin name in config.plugins[]:
  - Dynamically import plugin module
  - Call plugin.constructor() with (game: Game)
  - Call plugin.initListeners() (plugin subscribes to events)
    ↓
During Game Loop:
  - Events fire (e.g., player_did_join, player_update)
  - PluginManager calls all subscribed listeners
    ↓
On Server Shutdown:
  - Plugins unsubscribe (optional cleanup)
  - Resources released
```

## GamePlugin Base Class

**Methods:**

| Method | Purpose | When Called |
|--------|---------|------------|
| `constructor(game: Game)` | Initialize with game reference | Plugin instantiation |
| `initListeners()` | Subscribe to events | After construction |
| `on(event, listener)` | Subscribe to event (protected) | Inside `initListeners()` |
| `off(event, listener)` | Unsubscribe from event (protected) | Cleanup (rarely used) |

**Properties:**

| Property | Type | Purpose |
|----------|------|---------|
| `game` | Game | Reference to game instance |
| `protected listeners` | Map<string, Set<function>> | Active subscriptions |

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| `server/src/pluginManager.ts` | `GamePlugin` base class, `PluginManager` event bus, lifecycle | High |
| `server/src/plugins/placeObjectPlugin.ts` | Example: place objects for dev | Medium |
| `server/src/plugins/speedTogglePlugin.ts` | Example: speed modification | Medium |

## Plugin Events Reference

**Player Events:**

| Event | Data | Cancellable | Docs |
|-------|------|-----------|------|
| `player_will_connect` | ws, joinPacket | ✓ | Before player object created |
| `player_did_connect` | player | — | After player spawned |
| `player_will_join` | player | ✓ | Before game.activatePlayer() |
| `player_did_join` | player | — | After fully activated |
| `player_disconnect` | player | — | On socket close |
| `player_update` | player | — | End of player update tick |
| `player_start_attacking` | player | ✓ | First tick of attack |
| `player_stop_attacking` | player | ✓ | First tick of non-attack |
| `player_input` | player | — | After input processed |
| `player_will_emote` | player, emote | ✓ | Before emote use |
| `player_did_emote` | player, emote | — | After emote sent |
| `player_damage` | player, damageParams | — | Before damage applied |
| `player_will_piercing_damaged` | player, damageParams | ✓ | Before final damage |
| `player_did_piercing_damaged` | player, damageParams | — | After damage applied |
| `player_will_die` | player | — | Before death cleanup |
| `player_did_die` | player, killData | — | After death complete |
| `player_did_win` | player | — | On victory |

**Inventory Events:**

| Event | Data | Cancellable |
|-------|------|-----------|
| `inv_item_equip` | player, item | — |
| `inv_item_unequip` | player, item | — |
| `inv_item_use` | player, item | ✓ |
| `inv_item_stats_changed` | player, item | — |

**Obstacle/Projectile/Misc Events:** (Additional ~20+ events)

## Complex Functions

### `GamePlugin.on(eventName: string, listener: Function): void` — @file server/src/pluginManager.ts  
**Purpose:** Subscribe to an event.

**Implicit Behavior:**
1. Resolve event name to actual event definition
2. Register listener in plugin's subscriptions map
3. Listener called every time event fires (with typed data)
4. If event is cancellable, listener can call `event.cancel()`

**Example:**
```typescript
this.on("player_did_join", ({ player }) => {
  console.log(`Player ${player.name} joined`);
});
```

**Type Safety:** Event names are validated at compile time via event type union

**File:** `server/src/pluginManager.ts:80–120`

### `PluginManager.emit(eventName: string, data: EventData): void` — @file server/src/pluginManager.ts  
**Purpose:** Dispatch event to all subscribed listeners.

**Implicit Behavior:**
1. Look up all listeners for event
2. For each listener, call with event data
3. If event is cancellable, check if any listener called cancel()
4. Return whether event was cancelled (for caller to honor)

**Gotcha:** Event-driven code can be hard to trace; use console logging to debug listener order

**Performance:** O(n) where n = listeners subscribed to event (typically 1–3)

**File:** `server/src/pluginManager.ts:130–150`

## Example Plugin: PlaceObject

**Purpose:** Development tool—allow devs to place objects at cursor for level design.

**Implementation:**
```typescript
export default class PlaceObjectPlugin extends GamePlugin {
  obstacleToPlace = "house_column";  // Configurable
  private playerToObstacle = new ExtendedMap<Player, Obstacle>();

  protected override initListeners(): void {
    this.on("player_did_join", ({ player }) => {
      // Create obstacle paired with player
      const obstacle = new Obstacle(player.game, this.obstacleToPlace, player.position);
      this.playerToObstacle.set(player, obstacle);
      this.game.grid.addObject(obstacle);
    });

    this.on("player_disconnect", ({ player }) => {
      // Clean up obstacle on logout
      if (this.playerToObstacle.has(player)) {
        this.game.grid.removeObject(this.playerToObstacle.get(player)!);
      }
    });

    this.on("player_update", ({ player }) => {
      // Move obstacle to cursor position
      const obstacle = this.playerToObstacle.get(player);
      if (obstacle) {
        obstacle.position = player.cursorPosition;
        this.game.grid.updateObject(obstacle);
      }
    });

    this.on("player_start_attacking", ({ player }) => {
      // Rotate on attack (click)
      const obstacle = this.playerToObstacle.get(player);
      if (obstacle) {
        obstacle.rotation = (obstacle.rotation + 1) % 4;
      }
    });
  }
}
```

## Plugin Loading (Config)

**In `config.json`:**
```json
{
  "plugins": [
    "placeObjectPlugin",
    "speedTogglePlugin",
    "teleportPlugin"
  ]
}
```

**At startup:**
```typescript
// PluginManager iterates config.plugins
for (const pluginName of Config.plugins || []) {
  const plugin = await import(`./plugins/${pluginName}.ts`);
  const instance = new plugin.default(game);
  instance.initListeners();
}
```

## Plugin Configuration

**Pattern:** Plugins expose configurable properties:
```typescript
export default class MyPlugin extends GamePlugin {
  readonly speed = 1.5;  // Configurable
  readonly teleportRadius = 100;

  protected override initListeners(): void {
    // Use this.speed, this.teleportRadius, etc.
  }
}
```

**Future Enhancement:** Config schema for plugin parameters (not yet implemented).

## Debugging Plugins

**Console Output:**
```typescript
this.on("player_did_join", ({ player }) => {
  this.game.log(`[MyPlugin] Player joined: ${player.name}`);
});
```

**Enable Debug Mode:**
- Set env variable `DEBUG=suroi:*` (if logging set up)
- Check server logs for plugin init messages

**Test Locally:**
- Start server with dev plugins enabled in `config.json`
- Trigger events (join game, move, attack)
- Observe console output

## Related Documents

- **Tier 2:** [Plugin System Subsystem](../README.md) — Plugin architecture overview
- **Tier 2:** [Plugins Subsystem](../../plugins/README.md) — Bundled plugin catalog
- **Tier 1:** [Architecture](../../../architecture.md) — Event system design
- **Tier 1:** [Development Guide](../../../development.md) — Plugin development setup
- **Patterns:** [Plugin Patterns](../patterns.md) — Common plugin implementation patterns
