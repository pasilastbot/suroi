# Plugin System — Patterns

<!-- @tier: 2 -->
<!-- @parent: docs/subsystems/plugins/README.md -->

## Pattern: Creating a Plugin

**When to use:** Extending server game behavior without modifying core files.

**Implementation:**

1. Create a new file in `server/src/plugins/<yourPluginName>.ts`.
2. Default-export a class that `extends GamePlugin`.
3. Override `initListeners()` and call `this.on()` for each event you need.
4. Access the `Game` instance via `this.game`.

```typescript
import { GamePlugin } from "../pluginManager";

export default class MyPlugin extends GamePlugin {
    protected override initListeners(): void {
        this.on("player_did_join", ({ player }) => {
            // player: Player — fully joined, present in all collections
            player.health = 50; // start at half health
        });
    }
}
```

5. Enable by name (without extension) in `server/config.json`:

```json
{ "plugins": ["myPlugin"] }
```

**Example files:** `@file server/src/plugins/speedTogglePlugin.ts`

---

## Pattern: Cancelling an Event

**When to use:** Preventing a game side-effect from occurring (e.g., blocking a
player connection, stopping a pickup, suppressing obstacle generation).

**Implementation:**

Check whether the event is cancellable in the
[Events table](README.md#events) — only those marked ✓ support `cancel()`.
Call `event.cancel()` inside your handler. Game code checks the return value of
`pluginManager.emit()` and skips the guarded side-effect when a cancel is
detected.

```typescript
import { GamePlugin } from "../pluginManager";

export default class BlockMeleePlugin extends GamePlugin {
    protected override initListeners(): void {
        // inv_item_use is cancellable — cancel to prevent the attack
        this.on("inv_item_use", (item, event) => {
            if (item.isMelee) {
                event.cancel(); // attack does not fire
            }
        });
    }
}
```

**Stopping propagation** (prevent later plugins from seeing the same event):

```typescript
this.on("player_will_join", ({ player }, event) => {
    if (someCondition) {
        event.stopImmediatePropagation(); // later plugins won't see this event
    }
});
```

**Example files:** `@file server/src/plugins/weaponSwapPlugin.ts` (uses `player_will_die`)

---

## Pattern: Emitting Events from Game Code

**When to use:** Adding a new hookable point inside `game.ts` or other server
files so that plugins can observe or cancel a new action.

**Implementation:**

```typescript
// In game.ts (or any server file with access to `this.pluginManager`):

// Non-cancellable event — fire and forget
this.pluginManager.emit("game_tick", this);

// Cancellable event — check return value
const cancelledBy = this.pluginManager.emit("player_will_join", { player, joinPacket: packet });
if (cancelledBy) {
    player.disconnect();
    return;
}
```

`emit()` returns the `GamePlugin` instance that called `cancel()`, or
`undefined` if the event was not cancelled. Callers use a truthy check on the
return value to gate the side-effect.

**Example files:** `@file server/src/game.ts` (lines 626, 751, 1183, etc.)

---

## Pattern: Accessing Game State from a Plugin

**When to use:** Reading or modifying game objects (players, grid, map, gas)
from inside a plugin handler.

**Implementation:**

`this.game` is a full `Game` reference. Common access points:

```typescript
import { Vec } from "@common/utils/vector";
import { GamePlugin } from "../pluginManager";

export default class TeleportPlugin extends GamePlugin {
    protected override initListeners(): void {
        this.on("player_did_map_ping", ({ player, position }) => {
            // Move the player to the pinged position
            player.position = Vec.clone(position);
            player.updateObjects = true;
            player.setPartialDirty(); // mark for network sync this tick
        });
    }
}
```

Grid operations (add/remove/update objects):

```typescript
this.on("player_did_join", ({ player }) => {
    const obstacle = new Obstacle(player.game, "house_column", player.position);
    this.game.grid.addObject(obstacle);   // registers in spatial hash
});

this.on("player_disconnect", player => {
    this.game.grid.removeObject(obstacle); // deregisters from spatial hash
});
```

**Example files:** `@file server/src/plugins/teleportPlugin.ts`,
`@file server/src/plugins/placeObjectPlugin.ts`

---

## Pattern: Per-Player State in a Plugin

**When to use:** Tracking plugin-specific data associated with individual
players across multiple events (e.g., mapping a player to a placed obstacle).

**Implementation:**

Use `ExtendedMap<Player, T>` (from `@common/utils/misc`) for convenient
`ifPresent()` access and to avoid `undefined` checks:

```typescript
import { ExtendedMap } from "@common/utils/misc";
import { Obstacle } from "../objects/obstacle";
import { Player } from "../objects/player";
import { GamePlugin } from "../pluginManager";

export default class PlaceObjectPlugin extends GamePlugin {
    private readonly _playerToObstacle = new ExtendedMap<Player, Obstacle>();

    protected override initListeners(): void {
        this.on("player_did_join", ({ player }) => {
            const obstacle = new Obstacle(player.game, "house_column", player.position);
            this._playerToObstacle.set(player, obstacle);
            this.game.grid.addObject(obstacle);
        });

        this.on("player_update", player => {
            // ExtendedMap.ifPresent() — only runs if the key exists
            this._playerToObstacle.ifPresent(player, obstacle => {
                obstacle.position = player.position;
                this.game.grid.updateObject(obstacle);
            });
        });

        this.on("player_disconnect", player => {
            this._playerToObstacle.ifPresent(player, obstacle => {
                this.game.grid.removeObject(obstacle);
                this._playerToObstacle.delete(player);
            });
        });
    }
}
```

**Always clean up** in `player_disconnect` — otherwise stale entries accumulate
for the lifetime of the `Game` instance.

**Example files:** `@file server/src/plugins/placeObjectPlugin.ts`

---

## Pattern: Reacting to Player Kills

**When to use:** Granting rewards, swapping weapons, triggering events when a
player eliminates another player.

**Implementation:**

`player_will_die` fires with `{ source, weaponUsed }`. Check whether `source`
is a `Player` to distinguish PvP kills from gas, obstacle, or bleed-out deaths:

```typescript
import { Player } from "../objects/player";
import { GamePlugin } from "../pluginManager";

export default class WeaponSwapPlugin extends GamePlugin {
    protected override initListeners(): void {
        this.on("player_will_die", ({ source }) => {
            if (!(source instanceof Player)) return; // ignore non-PvP deaths

            const inventory = source.inventory;
            const index = source.activeItemIndex;
            // ... give killer a new weapon
            inventory.replaceWeapon(index, newItem);
        });
    }
}
```

**Note:** `player_will_die` fires when `health <= 0` but before the `dead` flag
is set. `player_did_die` fires after all cleanup is done. Use `will_die` for
rewarding the killer (inventory is still intact); use `did_die` for post-death
bookkeeping.

**Example files:** `@file server/src/plugins/weaponSwapPlugin.ts`

---

## Pattern: Modifying Player Speed

**When to use:** Changing movement speed in response to game events (power-ups,
test tools, game modes).

**Implementation:**

Directly mutate `player.baseSpeed`. The value is read each tick during the
physics update pass, so changes take effect immediately:

```typescript
import { GameConstants } from "@common/constants";
import { GamePlugin } from "../pluginManager";

export default class SpeedTogglePlugin extends GamePlugin {
    protected override initListeners(): void {
        this.on("player_did_emote", ({ player }) => {
            const base = GameConstants.player.baseSpeed;
            player.baseSpeed = player.baseSpeed === base ? 12 * base : base;
        });
    }
}
```

**Example files:** `@file server/src/plugins/speedTogglePlugin.ts`
