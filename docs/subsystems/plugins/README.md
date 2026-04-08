# Plugin System Subsystem

<!-- @tier: 2 -->
<!-- @parent: docs/architecture.md -->
<!-- @source: server/src/pluginManager.ts, server/src/plugins/ -->
<!-- @updated: 2026-03-04 -->

## Purpose

The Plugin System provides an event-based extension point for server-side behavior. Plugins subscribe to game events (player connect, damage, obstacle destroy, etc.) and can cancel or modify behavior. Built-in plugins demonstrate teleport, spawn objects, speed toggle, and weapon swap.

## Key Files & Entry Points

| File | Purpose |
|------|---------|
| `server/src/pluginManager.ts` | `PluginManager`, `GamePlugin`, `Events` — event system |
| `server/src/plugins/placeObjectPlugin.ts` | Spawn obstacles at positions |
| `server/src/plugins/teleportPlugin.ts` | Teleport players |
| `server/src/plugins/speedTogglePlugin.ts` | Toggle player speed |
| `server/src/plugins/weaponSwapPlugin.ts` | Swap weapons between players |
| `server/src/server.ts` | Loads plugins from config |

## Architecture

```
PluginManager (game.pluginManager)
    └── Set<GamePlugin>
            └── Each plugin: on(eventType, handler)

GamePlugin (abstract)
    ├── constructor(game) — calls initListeners()
    ├── initListeners() — abstract, register handlers
    ├── on(eventType, cb) — subscribe
    └── off(eventType, cb?) — unsubscribe

Events.emit(eventType, data)
    → Iterate plugins, call handlers
    → If cancellable and cancel() called: return cancelSource
```

## Module Index (Tier 3)

- [Events](modules/events.md) — Full event catalog, cancellable events, when they fire
- [Creating Plugins](modules/creating-plugins.md) — GamePlugin, initListeners, loading from config

## Event Categories

| Category | Examples |
|----------|----------|
| **Player** | `player_will_connect`, `player_did_join`, `player_update`, `player_damage`, `player_will_die`, `player_did_die` |
| **Inventory** | `inv_item_equip`, `inv_item_use`, `inv_item_stats_changed` |
| **Obstacle** | `obstacle_will_generate`, `obstacle_will_damage`, `obstacle_will_destroy`, `obstacle_will_interact` |
| **Loot** | `loot_will_generate`, `loot_will_interact` |
| **Building** | `building_will_generate`, `building_will_damage_ceiling` |
| **Airdrop** | `airdrop_will_summon`, `airdrop_landed` |
| **Game** | `game_created`, `game_tick`, `game_end` |

## Cancellable Events

Some events support `event.cancel()` to prevent the default behavior:

- `player_will_connect` — prevent join
- `player_will_join` — disconnect player
- `player_start_attacking` — prevent attack
- `player_will_piercing_damaged` — prevent damage
- `obstacle_will_generate` — prevent obstacle spawn
- `obstacle_will_damage` — prevent damage
- `obstacle_will_destroy` — prevent destruction (no loot/explosion)
- `obstacle_will_interact` — prevent interaction
- `loot_will_generate` — prevent loot spawn
- `loot_will_interact` — prevent pickup
- `building_will_generate` — prevent building
- `airdrop_will_summon` — prevent airdrop

## Creating a Plugin

```typescript
import { GamePlugin } from "../pluginManager";
import { Game } from "../game";

export default class MyPlugin extends GamePlugin {
    protected initListeners(): void {
        this.on("player_did_join", (data) => {
            this.game.log(`Player ${data.player.name} joined`);
        });
        this.on("obstacle_will_damage", (data, event) => {
            if (event.cancellable && someCondition) event.cancel();
        });
    }
}
```

## Loading Plugins

Plugins are loaded from `server/config.json`:

```json
{
  "plugins": ["placeObjectPlugin", "teleportPlugin"]
}
```

Plugins are loaded from `server/src/plugins/<name>.ts` via dynamic import.

## Protocol Considerations

- **Affects protocol:** No. Plugins modify game behavior but not packet format unless they add new packet types.

## Dependencies

- **Depends on:** Game (all events originate from game logic)
- **Depended on by:** Game (pluginManager.emit throughout), Config

## Related Documents

- **Tier 2:** [../game-loop/](../game-loop/) — Tick order, when events fire
- **Tier 1:** [docs/development.md](../../development.md) — Config, adding plugins
