# Plugin System — Creating Plugins Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/plugins/README.md -->
<!-- @source: server/src/pluginManager.ts, server/src/plugins/ -->
<!-- @updated: 2026-03-04 -->

## Purpose

This module documents how to create and load server plugins: `GamePlugin` base class, `initListeners`, `on`/`off`, config loading, and the plugin lifecycle.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| @file server/src/pluginManager.ts | `GamePlugin`, `PluginManager`, `loadPlugin`, `loadPlugins` | Medium |
| @file server/src/plugins/*.ts | Built-in plugins (placeObject, teleport, etc.) | Low |

## GamePlugin

- **Constructor:** Receives `game: Game`, calls `initListeners()`
- **initListeners()** — Abstract; register event handlers via `this.on()`
- **on(eventType, cb)** — Subscribe to event
- **off(eventType, cb?)** — Unsubscribe; omit cb to remove all for event

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

## Loading

- **Config:** `server/config.json` → `"plugins": ["placeObjectPlugin", "teleportPlugin"]`
- **Import:** `import(\`./plugins/${plugin}\`).default` — dynamic import from `server/src/plugins/`
- **loadPlugin(pluginClass)** — Instantiate, add to set; warns if already loaded
- **loadPlugins()** — Called at server start; loads all from config

## Cancellation

- For cancellable events, handler receives `event` with `event.cancel()`
- `emit()` returns the `GamePlugin` that cancelled, or `undefined`

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Plugin overview
- **Tier 3:** [events.md](events.md) — Event catalog
- **Tier 1:** [docs/development.md](../../../development.md) — Config
