# Q: How do I create and load a new server plugin?

<!-- @tags: plugins, events, server -->
<!-- @related: docs/subsystems/plugins/modules/creating-plugins.md -->

## 1. Create the plugin file

Create `server/src/plugins/myPlugin.ts`:

```typescript
import { GamePlugin } from "../pluginManager";

export default class MyPlugin extends GamePlugin {
    protected initListeners(): void {
        this.on("player_did_join", ({ player }) => {
            this.game.log(`${player.name} joined`);
        });

        // Cancellable event — prevent obstacle damage under some condition
        this.on("obstacle_will_damage", (data, event) => {
            if (someCondition) event.cancel();
        });
    }
}
```

Rules:
- Extend `GamePlugin` (from `server/src/pluginManager.ts`)
- Override `initListeners()` — this is called in the constructor
- Use `this.on(eventType, handler)` to subscribe
- `this.game` gives access to the running `Game` instance
- Default export the class

## 2. Register in config

Edit `server/config.json`:

```json
{
    "plugins": ["myPlugin"]
}
```

The plugin manager does `import('./plugins/myPlugin').default` — filename must
match the config string exactly (no `.ts` extension).

## 3. Verify

```bash
bun dev:server
```

The server logs plugin loading at startup. Check for errors.

## Available Events

See the full event catalog at
[plugins/modules/events.md](../subsystems/plugins/modules/events.md).

Key events: `player_did_join`, `player_will_die`, `obstacle_will_damage`,
`game_end`, `bullet_will_hit`, and more.

## Cancellable Events

When the event handler signature is `(data, event)`, `event.cancel()` stops
the default behavior. `emit()` returns the plugin that cancelled, or `undefined`.

## Related

- [Creating Plugins](../subsystems/plugins/modules/creating-plugins.md) — full reference with code example
- [Plugin Events](../subsystems/plugins/modules/events.md) — complete event catalog
- [Plugin System](../subsystems/plugins/README.md) — architecture overview
