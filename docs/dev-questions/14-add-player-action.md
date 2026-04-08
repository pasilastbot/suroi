# Q: How do I add a new timed player action (like a new type of interaction or heal)?

<!-- @tags: inventory, actions, server, player -->
<!-- @related: docs/subsystems/inventory/modules/actions.md -->

## Overview

Timed player actions use the `Action` base class in
`server/src/inventory/action.ts`. An action runs for a fixed duration, can be
cancelled, and calls `execute()` on completion.

## 1. Create the action class

```typescript
// server/src/inventory/action.ts — add to the existing file

export class MyCustomAction extends Action {
    constructor(player: Player) {
        super(
            player,
            3.0,    // duration in seconds
        );
        // Optional: reduce speed during action
        // this.speedMultiplier = 0.7;
    }

    override execute(): void {
        super.execute();  // clears player.action

        // Apply the effect on completion
        this.player.health = Math.min(
            this.player.health + 50,
            this.player.maxHealth
        );
        this.player.setDirty();  // mark player for UpdatePacket
    }
}
```

## 2. Trigger the action from InputPacket

Player actions are triggered in `server/src/objects/player.ts`, in the
`processInputs()` method, based on `InputActions` from the input packet.

If you need a new trigger, first add a new `InputAction` type:

### 2a. Add to the InputActions enum

```typescript
// common/src/constants.ts
export enum InputActions {
    // ...existing...
    MyAction,   // ← append
}
```

### 2b. Handle in player.ts

```typescript
// server/src/objects/player.ts — in processInputs()
case InputActions.MyAction:
    if (this.action === undefined) {  // no action already running
        this.action = new MyCustomAction(this);
    }
    break;
```

### 2c. Send from client

```typescript
// client/src/scripts/managers/inputManager.ts or wherever you trigger it
InputManager.addAction({ type: InputActions.MyAction });
```

## 3. Bump the protocol version

```typescript
// common/src/constants.ts
protocolVersion: 73,  // ← increment
```

Adding to `InputActions` changes the `InputPacket` format.

## 4. Validate

```bash
bun validateDefinitions
bun dev
```

## How Actions Work

```
Action created:
    → schedules game.addTimeout(execute, duration * 1000)
    → player.action = this
    → player.setDirty()

Action cancelled (player downed, another action started, cancel input):
    → clearTimeout()
    → player.action = undefined
    → player.setDirty()

Action completed:
    → execute() called
    → super.execute() clears player.action
    → Apply effect
```

The client shows the action progress bar based on the serialized `player.action`
field in `UpdatePacket` PlayerData (action type + elapsed time).

## Existing Action Types for Reference

| Class | Duration | Effect |
|-------|----------|--------|
| `HealingAction` | from item def | Heal + consume item |
| `ReloadAction` | from gun def | Refill magazine |
| `ReviveAction` | `GameConstants.player.reviveTime` | Revive downed teammate |

## Related

- [Actions Module](../subsystems/inventory/modules/actions.md) — full reference with all action types
- [Inventory](../subsystems/inventory/README.md) — action lifecycle
- [Game Loop](../subsystems/game-loop/README.md) — `addTimeout` scheduling
