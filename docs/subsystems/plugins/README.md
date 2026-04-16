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
       └─ dynamic import('./plugins/<name>')
            └─ loadPlugin(PluginClass)
                 └─ new PluginClass(game)   ← calls initListeners()
                      └─ this.on(Events.X, handler)  ← stored per-plugin

Game runtime
  └─ game.pluginManager.emit(Events.X, payload)
       └─ for each registered plugin
            └─ dispatch matching listeners in order
                 └─ handler(payload, eventControl)
```

Each `Game` instance owns exactly one `PluginManager`. Plugins are registered
per game, not globally. `loadPlugins()` is called before `game_created` is
emitted (and before the tick loop starts), so plugins are active from the very
first tick.

## GamePlugin Base Class

Defined in `server/src/pluginManager.ts`.

```typescript
export abstract class GamePlugin {
    constructor(public readonly game: Game)

    /** Override to call this.on() for each event the plugin handles */
    protected abstract initListeners(): void

    /** Register a listener for the given event type */
    on<Ev extends EventTypes>(eventType: Ev, cb: EventHandler<Ev>): void

    /** Remove a specific listener, or all listeners for an event */
    off<Ev extends EventTypes>(eventType: Ev, cb?: EventHandler<Ev>): void
}
```

**Key points:**
- The `game` property exposes the full `Game` instance — plugins can read and
  mutate grid objects, players, the map, gas state, etc.
- `initListeners()` is called from the `GamePlugin` constructor, so `this.on()`
  calls inside it are always safe.
- Errors thrown inside a handler are caught and logged; they do not kill the
  dispatch loop.

### Handler signature

```typescript
type EventHandler<Ev extends EventTypes> =
    (...[data, event]: [...ArgsFor<Ev>, ...EventData<Ev>]) => void
```

- First argument: the event payload (typed per event — see [Event Payload
  Types](#event-payload-types)). Absent for `player_will_connect` (payload is
  `never`).
- Last argument: an event-control object, always present:
  - `stopImmediatePropagation()` — stop later plugins from receiving this event
  - `cancel()` — mark event as cancelled *(cancellable events only)*
  - `isCancelled: boolean` — current cancellation state *(cancellable events only)*

## Events

`Events` is a plain `const` object (not an enum) whose keys are the event
names. Each value carries a `cancellable` flag produced by `makeEvent()`.

The table below lists all 48 events, grouped by namespace, with their
cancellability and when they fire.

### Player lifecycle

| Event | Cancellable | When it fires |
|-------|:-----------:|---------------|
| `player_will_connect` | ✓ | Before `Player` object is created. **Cancel** closes the websocket. Payload: `never`. |
| `player_did_connect` | — | After `Player` is instantiated and placed on the map. |
| `player_will_join` | ✓ | Start of `Game.activatePlayer()` — `JoinedPacket` sent but player not yet in any collection. **Cancel** disconnects the player. |
| `player_did_join` | — | End of `Game.activatePlayer()` — player fully added to all collections; game started if needed. |
| `player_disconnect` | — | After `Game.removePlayer()` — socket closed, player despawned, removed from team. |
| `player_update` | — | End of the player's first update pass (physics, health/adren/zoom, item use). |
| `player_start_attacking` | ✓ | First tick a player starts attacking. **Cancel** skips `useItem()` and leaves `startedAttacking` set (re-fires next tick). |
| `player_stop_attacking` | ✓ | First tick a player stops attacking. **Cancel** skips `stopUse()` and leaves `stoppedAttacking` set. |
| `player_input` | — | Every time an `InputPacket` is received; all side-effects already applied. |
| `player_will_emote` | ✓ | Player is about to use an emote. **Cancel** prevents it. |
| `player_did_emote` | — | Player successfully used an emote. |
| `player_will_map_ping` | ✓ | Player is about to use a map ping. **Cancel** prevents it. |
| `player_did_map_ping` | — | Player successfully used a map ping. |
| `player_damage` | — | Player is about to receive damage (unclamped, before protective modifiers). |
| `player_will_piercing_damaged` | ✓ | Before piercing damage is applied (regular damage is also routed here after modifier reduction). **Cancel** prevents the damage. |
| `player_did_piercing_damaged` | — | After piercing damage applied; death check has not yet run. |
| `player_will_die` | — | `health <= 0`, `dead` flag not yet set. |
| `player_did_die` | — | Player fully dead — all cleanup done (inventory cleared, death marker created, game-over packet sent, etc.). |
| `player_did_win` | — | Fired for each winning player after win emote and game-over packet sent. |

### Inventory

| Event | Cancellable | When it fires |
|-------|:-----------:|---------------|
| `inv_item_equip` | — | An inventory item is equipped. When swapping, old item fires `unequip` before new fires `equip`. |
| `inv_item_unequip` | — | An inventory item is unequipped. |
| `inv_item_use` | ✓ | Item is "used" (fire/cook) — after pre-flight checks pass. **Cancel** emulates a pre-flight fail (no `lastUse` side-effect). |
| `inv_item_stop_use` | ✓ | `InventoryItem.stopUse()` is called. **Cancel** stops the rest of `stopUse()` but does not reset the `attacking` flag. |
| `inv_item_stats_changed` | — | A weapon's stats have changed. |
| `inv_item_modifiers_changed` | — | A weapon's modifiers have changed, usually following stat changes. |

### Obstacle

| Event | Cancellable | When it fires |
|-------|:-----------:|---------------|
| `obstacle_will_generate` | ✓ | Start of `GameMap.generateObstacle()` (map generation + airdrop landings). **Cancel** prevents generation. |
| `obstacle_did_generate` | — | End of `GameMap.generateObstacle()`. |
| `obstacle_will_damage` | ✓ | Obstacle is about to sustain damage (not fired for invalid attempts). **Cancel** prevents damage. |
| `obstacle_did_damage` | — | After obstacle damaged; death logic not yet checked. |
| `obstacle_will_destroy` | ✓ | Obstacle is about to be destroyed. **Cancel** prevents all side-effects (loot/explosions/decals) while still making it vanish. |
| `obstacle_did_destroy` | — | After obstacle fully destroyed with all side-effects applied. |
| `obstacle_will_interact` | ✓ | Player about to interact with obstacle (`canInteract()` already returned `true`). **Cancel** prevents interaction. |
| `obstacle_did_interact` | — | After player successfully interacted with obstacle. |

### Loot

| Event | Cancellable | When it fires |
|-------|:-----------:|---------------|
| `loot_will_generate` | ✓ | Before `Game.addLoot()` creates the loot object. **Cancel** prevents generation. |
| `loot_did_generate` | — | After loot object created and added. |
| `loot_will_interact` | ✓ | Player about to pick up loot (`canInteract()` passed, redundancy check not yet run). **Cancel** prevents pickup. |
| `loot_did_interact` | — | After player successfully picked up loot (inventory mutated, `PickupPacket` sent, loot removed). |

### Building

| Event | Cancellable | When it fires |
|-------|:-----------:|---------------|
| `building_will_generate` | ✓ | Start of `GameMap.generateBuilding()`. **Cancel** prevents the building from generating. |
| `building_did_generate` | — | End of `GameMap.generateBuilding()`. |
| `building_will_damage_ceiling` | ✓ | A building's ceiling is about to take damage (usually from a destroyed wall). **Cancel** prevents the damage. |
| `building_did_damage_ceiling` | — | After ceiling damage applied. |
| `building_did_destroy_ceiling` | — | Building ceiling destroyed (building marked dead + partially dirty). |

### Airdrop

| Event | Cancellable | When it fires |
|-------|:-----------:|---------------|
| `airdrop_will_summon` | ✓ | `Game.summonAirdrop()` called; actual landing position not yet decided. **Cancel** prevents summon. |
| `airdrop_did_summon` | — | After position decided and airdrop scheduled. |
| `airdrop_landed` | — | Parachute hits ground; airdrop crate generated (obstacle events have already fired); no particles or crush damage yet applied. |

### Game lifecycle

| Event | Cancellable | When it fires |
|-------|:-----------:|---------------|
| `game_created` | — | Near end of `Game` constructor — grid, map, gas, and plugins are fully loaded; tick loop not yet started. |
| `game_tick` | — | End of each game tick — all tick side-effects done, next tick not yet scheduled. |
| `game_end` | — | A player/team is determined to have won; server not stopped yet. |

## Event Payload Types

Full types are in the `EventDataMap` interface in `server/src/pluginManager.ts`.

```typescript
// Supporting interfaces
interface DamageParams {
    readonly amount: number
    readonly source?: GameObject | DamageSources.*
    readonly weaponUsed?: GunItem | MeleeItem | ThrowableItem | Explosion | Obstacle
}
interface PlayerDamageEvent extends DamageParams { readonly player: Player }
interface ObstacleDamageEvent extends DamageParams {
    readonly obstacle: Obstacle
    readonly position?: Vector
}
```

| Event | Payload type |
|-------|-------------|
| `player_will_connect` | `never` (no argument passed) |
| `player_did_connect` | `Player` |
| `player_will_join` | `{ player: Player, joinPacket: JoinData }` |
| `player_did_join` | `{ player: Player, joinPacket: JoinData }` |
| `player_disconnect` | `Player` |
| `player_update` | `Player` |
| `player_start_attacking` | `Player` |
| `player_stop_attacking` | `Player` |
| `player_input` | `{ player: Player, packet: InputData }` |
| `player_will_emote` | `{ player: Player, emote: EmoteDefinition }` |
| `player_did_emote` | `{ player: Player, emote: EmoteDefinition }` |
| `player_will_map_ping` | `{ player: Player, ping: PlayerPing, position: Vector }` |
| `player_did_map_ping` | `{ player: Player, ping: PlayerPing, position: Vector }` |
| `player_damage` | `PlayerDamageEvent` |
| `player_will_piercing_damaged` | `PlayerDamageEvent` |
| `player_did_piercing_damaged` | `PlayerDamageEvent` |
| `player_will_die` | `Omit<PlayerDamageEvent, "amount">` |
| `player_did_die` | `Omit<PlayerDamageEvent, "amount">` |
| `player_did_win` | `Player` |
| `inv_item_equip` | `InventoryItem` |
| `inv_item_unequip` | `InventoryItem` |
| `inv_item_use` | `InventoryItem` |
| `inv_item_stop_use` | `InventoryItem` |
| `inv_item_stats_changed` | `{ item: InventoryItem, oldStats, newStats, diff }` |
| `inv_item_modifiers_changed` | `{ item: InventoryItem, oldMods: PlayerModifiers, newMods: PlayerModifiers, diff }` |
| `obstacle_will_generate` | `{ type: ObstacleDefinition, position, rotation, layer, scale, variation?, lootSpawnOffset?, parentBuilding?, puzzlePiece?, locked?, activated?, waterOverlay? }` |
| `obstacle_did_generate` | `Obstacle` |
| `obstacle_will_damage` | `ObstacleDamageEvent` |
| `obstacle_did_damage` | `ObstacleDamageEvent` |
| `obstacle_will_destroy` | `ObstacleDamageEvent` |
| `obstacle_did_destroy` | `ObstacleDamageEvent` |
| `obstacle_will_interact` | `{ obstacle: Obstacle, player?: Player }` |
| `obstacle_did_interact` | `{ obstacle: Obstacle, player?: Player }` |
| `loot_will_generate` | `{ definition: LootDefinition, position, layer, count?, pushVel?, jitterSpawn?, data? }` |
| `loot_did_generate` | `{ loot: Loot, position, layer, count?, pushVel?, jitterSpawn? }` |
| `loot_will_interact` | `{ loot: Loot, canPickup?: boolean \| InventoryMessages, player: Player }` |
| `loot_did_interact` | `{ loot: Loot, player: Player }` |
| `building_will_generate` | `{ definition: BuildingDefinition, position, orientation: Orientation, layer }` |
| `building_did_generate` | `Building` |
| `building_will_damage_ceiling` | `{ building: Building, damage: number }` |
| `building_did_damage_ceiling` | `{ building: Building, damage: number }` |
| `building_did_destroy_ceiling` | `Building` |
| `airdrop_will_summon` | `{ position: Vector }` (desired position) |
| `airdrop_did_summon` | `{ airdrop: Airdrop, position: Vector }` (actual position) |
| `airdrop_landed` | `Airdrop` |
| `game_created` | `Game` |
| `game_tick` | `Game` |
| `game_end` | `Game` |

## PluginManager

Defined in `server/src/pluginManager.ts`. One instance lives on `game.pluginManager`.

```typescript
class PluginManager {
    constructor(readonly game: Game)

    /** Emit an event; returns the GamePlugin that cancelled it, or undefined */
    emit<Ev extends EventTypes>(eventType: Ev, ...args: ArgsFor<Ev>): GamePlugin | undefined

    /** Instantiate a plugin class and register it (no-op if already loaded) */
    loadPlugin(pluginClass: new (game: Game) => GamePlugin): void

    /** Remove a plugin instance from the active set */
    unloadPlugin(plugin: GamePlugin): void

    /** Read Config.plugins, dynamically import each file, call loadPlugin() */
    async loadPlugins(): Promise<void>
}
```

`emit()` iterates all registered plugins in insertion order. Any plugin can
call `stopImmediatePropagation()` on cancellable or non-cancellable events to
prevent later plugins from receiving the event. For cancellable events,
`emit()` returns the `GamePlugin` instance that called `cancel()` (or
`undefined` if not cancelled); callers in `game.ts` use this return value to
gate side-effects.

## Plugin Loading

Plugins are loaded **once per `Game` instance**, in the `Game` constructor
(`server/src/game.ts` line 255), before `game_created` is fired and before the
tick loop starts.

```typescript
// server/src/game.ts (constructor)
void this.pluginManager.loadPlugins();
// ...
this.pluginManager.emit("game_created", this);
this.tick(); // loop starts here
```

`loadPlugins()` reads `Config.plugins: string[]` — a list of filenames
(without extension) relative to `server/src/plugins/`. Each name is resolved
via dynamic `import()`:

```typescript
// server/src/pluginManager.ts — PluginManager.loadPlugins()
for (const plugin of Config.plugins) {
    const pluginClass = (await import(`./plugins/${plugin}`)).default;
    this.loadPlugin(pluginClass);
}
```

Configuration is via `server/config.json` (validated against
`server/config.schema.json`):

```json
{
  "plugins": ["speedTogglePlugin", "teleportPlugin"]
}
```

Each entry must match the filename of a default-exported `GamePlugin` subclass
in `server/src/plugins/`. If a plugin class is already loaded, `loadPlugin()`
logs a warning and skips it.

## Built-in Plugins

All four built-in plugins are development/demonstration tools. None are
activated by default (no `config.json` ships with them enabled).

| Plugin | File | Purpose |
|--------|------|---------|
| `PlaceObjectPlugin` | `server/src/plugins/placeObjectPlugin.ts` | Dev map-building tool. Spawns a chosen obstacle at each connected player's cursor position (updated every tick). Emoting rotates the obstacle by 1 step (0–3). Attacking logs the obstacle's normalized position and rotation to stdout in a format suitable for copy-pasting into map definition files. |
| `SpeedTogglePlugin` | `server/src/plugins/speedTogglePlugin.ts` | Toggles the emoting player's `baseSpeed` between the default value and 12× the default. Useful for rapid movement during testing. |
| `TeleportPlugin` | `server/src/plugins/teleportPlugin.ts` | Teleports the player to their map-ping position by setting `player.position` and marking the player as partially dirty. |
| `WeaponSwapPlugin` | `server/src/plugins/weaponSwapPlugin.ts` | When a player dies, gives their killer a random weapon of the same category (gun/melee/throwable) in the active slot. Guns are given their full ammo capacity. Excludes killstreak and `wearerAttributes` weapons. |

## Data Flow

```
Player action / game tick
  │
  ├─ game.ts calls pluginManager.emit(Events.<X>, payload)
  │
  ├─ PluginManager iterates _plugins (Set, insertion order)
  │    └─ calls pluginDispatchers.get(plugin)(eventType, payload, eventControl)
  │         └─ plugin's stored handlers for that event run in registration order
  │              └─ handler(payload, eventControl)
  │                   ├─ handler reads/mutates game state via this.game.*
  │                   ├─ handler may call eventControl.cancel()
  │                   └─ handler may call eventControl.stopImmediatePropagation()
  │
  └─ emit() returns cancelSource (GamePlugin | undefined)
       └─ game.ts gates the side-effect: if (cancelSource) return;
```

## Interfaces & Contracts

### `GamePlugin` base class (`server/src/pluginManager.ts`)

All plugins inherit from this abstract base:

```typescript
abstract class GamePlugin {
    constructor(public readonly game: Game)
    protected abstract initListeners(): void
    protected on<Ev extends EventTypes>(eventType: Ev, cb: EventHandler<Ev>): void
    protected off<Ev extends EventTypes>(eventType: Ev, cb?: EventHandler<Ev>): void
}
```

### Plugin access to Game instance

Inside a plugin, `this.game` provides access to the full `Game` instance,
which includes (non-exhaustive):

| Property | Type | Purpose |
|----------|------|---------|
| `this.game.grid` | `Grid` | Spatial hash — `addObject`, `removeObject`, `updateObject`, `intersectsHitbox` |
| `this.game.map` | `GameMap` | Map dimensions, obstacle/building generation |
| `this.game.gas` | `Gas` | Gas state and stage data |
| `this.game.pluginManager` | `PluginManager` | Emit custom events or load/unload other plugins |
| `this.game.players` | `Set<Player>` | All connected and active players |
| `this.game.livingPlayers` | `Set<Player>` | Players still alive |
| `this.game.id` | `number` | Game instance ID |

## Dependencies

- **Depends on:**
  - [Game Loop](../game-loop/) — `game.ts` is the source of all `emit()` calls; plugins run inside the tick
  - Object model (`server/src/objects/`) — payload types reference `Player`, `Obstacle`, `Building`, `Loot`, `Airdrop`
  - `@common/utils/misc` — `ExtendedMap` used for per-plugin handler storage

- **Depended on by:** Nothing. Plugins are opt-in. The core game loop has no
  dependency on any specific plugin being present; `emit()` is a no-op when no
  listeners are registered.

## Module Index (Tier 3)

No separate Tier 3 modules documented yet. All plugin event definitions and dispatch logic are in the single [server/src/pluginManager.ts](../../../../../../server/src/pluginManager.ts) file.

## Known Gotchas

- **`player_will_connect` has no payload.** Its `EventDataMap` type is `never`,
  so handler receives only the event-control object. The return value of
  `emit()` is used to close the socket.
- **Events fire even with zero plugins loaded.** `emit()` short-circuits
  cleanly, so there is no overhead concern from unused event hooks.
- **Plugin order matters for `stopImmediatePropagation`.** Plugins are stored
  in a `Set` and iterated in insertion order (load order from `Config.plugins`).
- **`obstacle_will_generate` fires for airdrops too**, not only during initial
  map generation. Same for `obstacle_did_generate`.
- **Built-in plugins are dev tools only** — they do not have production use
  cases and should not be enabled in production `config.json`.

## Related Documents

- **Tier 1:** [docs/architecture.md](../../architecture.md) — System overview
- **Tier 2:** [Game Loop](../game-loop/) — event emission source (`game.ts`)
- **Patterns:** [patterns.md](patterns.md) — Reusable plugin patterns
