# Q: How do I add a new in-game console command?

<!-- @tags: console, commands, dev -->
<!-- @related: docs/subsystems/console/modules/commands.md -->

## Overview

Console commands are registered in `setUpCommands()` in
`client/src/scripts/console/commands.ts`. They are client-side only.

## Simple Command

```typescript
// In setUpCommands(), in client/src/scripts/console/commands.ts

Command.createCommand(
    "my_command",
    function(arg1, arg2) {
        // arg1, arg2 are string | undefined
        if (arg1 === undefined) {
            return { err: "Missing argument: value" };
        }
        const num = Casters.toNumber(arg1);
        if ("err" in num) return num;   // propagate parse error

        // Do something with num.res
        console.log("my_command called with", num.res);
        // No return value = success
    },
    {
        short: "Brief description",
        long: "Longer description of what this does",
        allowOnlyWhenGameStarted: true,  // false = available in menus too
        signatures: [
            {
                args: [
                    {
                        name: "value",
                        type: ["number"],
                        optional: false,
                    }
                ],
                noexcept: false,  // false = command can return an error
            }
        ]
    }
);
```

## Invertible Command Pair (for toggle/hold binds)

```typescript
Command.createInvertiblePair(
    "my_toggle",
    () => { MyManager.enabled = true; },  // "+my_toggle"
    () => { MyManager.enabled = false; }, // "-my_toggle"
    { short: "Enable X", long: "...", signatures: [{ args: [], noexcept: true }] },
    { short: "Disable X", long: "...", signatures: [{ args: [], noexcept: true }] }
);
```

This creates two commands: `+my_toggle` (bind press) and `-my_toggle` (bind release).

## Command That Sends a Packet to Server

Dev commands often send a `DebugPacket` to trigger server-side behavior:

```typescript
Command.createCommand(
    "teleport",
    function(x, y) {
        // parse x, y...
        Game.sendPacket(/* DebugPacket with teleport data */);
    },
    { /* ... */ }
);
```

## Accessing CVars in Commands

```typescript
const lang = GameConsole.getBuiltInCVar("cv_language"); // reads cvar value
GameConsole.setBuiltInCVar("cv_language", "de");         // sets cvar value
```

## Error Handling

Return `{ err: "message" }` to display an error in the console. Return
`undefined` (or nothing) for success.

```typescript
function(arg) {
    if (badCondition) return { err: "Something went wrong" };
    // success — no return needed
}
```

## Finding the Command in the Console

Commands appear in the autocomplete when you type in the console (press `` ` ``
to open). The `short` description appears in autocomplete suggestions.

## Related

- [Commands Module](../subsystems/console/modules/commands.md) — full reference
- [CVars](18-add-cvar.md) — if your command reads/writes a cvar
- [Input](../subsystems/input/modules/key-bindings.md) — binding commands to keys
