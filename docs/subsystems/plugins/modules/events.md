# Plugin System — Events Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/plugins/README.md -->
<!-- @source: server/src/pluginManager.ts, server/src/game.ts -->
<!-- @updated: 2026-03-04 -->

## Purpose

This module documents the event-based plugin system. Plugins register handlers for game events; some events are cancellable and can prevent default behavior.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| @file server/src/pluginManager.ts | `Events` enum, `PluginManager.emit()`, handler registration | High |
| @file server/src/game.ts | `pluginManager` — emit at key points in game flow | High |

## Event Categories

### Connection & Join
| Event | Cancellable | When |
|-------|-------------|------|
| `player_will_connect` | Yes | Before Player created; reject closes socket |
| `player_did_connect` | No | After Player instantiated |
| `player_will_join` | Yes | Start of activatePlayer; reject disconnects |
| `player_did_join` | No | End of activatePlayer |
| `player_disconnect` | No | After removePlayer |

### Player Lifecycle
| Event | Cancellable | When |
|-------|-------------|------|
| `player_update` | No | End of first update pass |
| `player_start_attacking` | Yes | First tick attacking |
| `player_stop_attacking` | Yes | First tick not attacking |
| `player_input` | No | After InputPacket processed |
| `player_will_emote` | Yes | Before emote used |
| `player_did_emote` | No | After emote used |
| `player_will_map_ping` | Yes | Before map ping used |
| `player_did_map_ping` | No | After map ping used |

### Damage & Death
| Event | Cancellable | When |
|-------|-------------|------|
| `player_damage` | No | Before damage applied |
| `player_will_piercing_damaged` | Yes | Before piercing damage |
| `player_did_piercing_damaged` | No | After piercing damage |
| `player_will_die` | No | Before death (health ≤ 0) |
| `player_did_die` | No | After death cleanup |

### Game & Airdrop
| Event | Cancellable | When |
|-------|-------------|------|
| `game_created` | No | Game constructor |
| `game_tick` | No | Start of each tick |
| `game_end` | No | Game over |
| `player_did_win` | No | Winner determined |
| `airdrop_will_summon` | Yes | Before airdrop spawned |
| `airdrop_did_summon` | No | After airdrop spawned |

### Inventory
| Event | Cancellable | When |
|-------|-------------|------|
| `inv_item_use` | Yes | Item use started |
| `inv_item_stop_use` | No | Item use stopped |
| `inv_item_stats_changed` | No | Item modifiers updated |

## Cancellation

- Cancellable events: handler returns truthy to cancel
- Cancelled events skip default behavior (e.g. `player_will_connect` → close socket)

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Plugin overview
- **Tier 2:** [../game-loop/](../../game-loop/) — Where events fire in tick
