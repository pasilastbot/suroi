# Debug Tools Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/console-debug/README.md -->
<!-- @source: client/src/scripts/console/commands.ts, client/src/scripts/console/variables.ts -->

## Purpose

Provides built-in debug commands for developers: toggle rendering features, spawn items, enable godmode, display performance metrics, network debug info, and built-in cheats for testing gameplay.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| [client/src/scripts/console/commands.ts](../../../../../client/src/scripts/console/commands.ts) | Command registry, command executor, built-in commands (50+ commands) | High |
| [client/src/scripts/console/variables.ts](../../../../../client/src/scripts/console/variables.ts) | Console variables (CVars) and configuration flags | High |
| [client/src/scripts/console/gameConsole.ts](../../../../../client/src/scripts/console/gameConsole.ts) | Console parser, command execution, error handling, output display | High |
| [client/src/scripts/console/internals.ts](../../../../../client/src/scripts/console/internals.ts) | Internal console utilities, helper functions | Medium |

## Business Rules

- **Command Registry**: Commands registered via `Command.createCommand()` or `Command.createInvertiblePair()`
- **Invertible Commands**: Toggle commands with +name (enable) and -name (disable) syntax
- **Command Validation**: Type checking on arguments; mismatched types rejected with error message
- **Built-in Commands**: ~50 commands for debugging, testing, admin operations
- **CVars**: Console variables with types (int, string, bool, enum); persistent in localStorage
- **Cheats**: Godmode, unlimited ammo, instant reload, spawn items, teleport (only in DEBUG_CLIENT builds)
- **Debug Output**: Commands can print to in-game console or return structured output
- **Performance Metrics**: FPS, ping, latency, packet statistics, memory usage
- **Network Debug**: Packet size, update frequency, interpolation state, player prediction
- **Admin Only**: Some commands restricted to authenticated users (if server enforces)

## Data Lineage

```
Player opens Console (Ctrl+Shift+K or mobile button)
  ↓
Type command: "/vomit [argument]"
  ↓
GameConsole.parseCommand(input)
  1. Extract command name and arguments
  2. Lookup command in registry
  3. Validate argument count and types
  4. Call command executor function
  ↓
Command Executor runs:
  - Perform action (spawn item, toggle rendering, etc.)
  - Read game state if needed (player position, inventory)
  - Modify game state or display output
  ↓
Return result:
  - Success: undefined or output message
  - Error: error object with message
  ↓
GameConsole outputs result to in-game console display
```

## Configuration

| Setting | Effect | Source |
|---------|--------|--------|
| `DEBUG_CLIENT` | Enable cheat commands and debug features | Build-time constant |
| `allowConsoleCommands` | Enable/disable console (true on desktop, false on mobile by default) | CVar: `cv_allow_console` |
| `cv_anonymize_player_names` | Hide player names (return `Player_<id>` instead) | CVar, persisted in localStorage |
| `db_show_hitboxes` | Render collision hitboxes on screen | CVar (debug flag) |
| `db_godmode` | Take no damage (cheat) | CVar (debug flag) |
| `db_infiniteAmmo` | Never consume ammo (cheat) | CVar (debug flag) |

## Complex Functions

### Command Registry Pattern — @file client/src/scripts/console/commands.ts
**Purpose:** Define a new debug command with name, signature, documentation, and executor.

**Pattern**:
```typescript
Command.createCommand(
    "vomit", // command name
    function() { /* executor code */ }, // no args
    {
        short: "Vomit",
        long: "Makes the player vomit (removes inventory items)",
        allowOnlyWhenGameStarted: true,
        signatures: [
            {
                args: [],
                noexcept: true
            }
        ]
    }
);
```

**Implicit behavior:**
- Command stored in global registry
- Called via console input: `/vomit`
- Signature validates argument count (0 args here)
- `allowOnlyWhenGameStarted: true` prevents execution in menu
- Result displayed in console

### Invertible Command Pair — @file client/src/scripts/console/commands.ts
**Purpose:** Create toggle commands (+name / -name format).

**Pattern**:
```typescript
Command.createInvertiblePair(
    "rendering_rendering",
    function() { /* enable rendering */ },
    function() { /* disable rendering */ },
    { /* info for ON version */ },
    { /* optional: info for OFF version */ }
);
```

**Implicit behavior:**
- Two commands created: `+rendering_rendering` (enable) and `-rendering_rendering` (disable)
- Each command has inverse reference: `cmd.inverse` points to the other
- Stored separately in registry
- Called via: `/+rendering_rendering` or `/-rendering_rendering`

### `GameConsole.executeCommand(name, ...args)` — @file client/src/scripts/console/gameConsole.ts
**Purpose:** Parse and execute a debug command.

**Implicit behavior:**
- Splits input string by spaces → command name + arguments
- Trims whitespace and special characters
- Looks up command in registry
- Validates argument count and types against signature
- Calls executor with parsed arguments
- Catches exceptions; displays error message in console
- Returns result or error to caller

### Built-in Cheat Commands — @file client/src/scripts/console/commands.ts
**Purpose:** Developer testing commands (only in DEBUG_CLIENT builds).

**Common Cheats**:
- `/godmode` (CVar: `db_godmode = true`) → take no damage, no knockback
- `/infiniteAmmo` (CVar: `db_infiniteAmmo = true`) → never consume ammunition
- `/giveItem [itemID]` → spawn item in inventory
- `/spawn_item [itemID] [count]` → drop item at player position
- `/tp [x] [y]` → teleport player to coordinates
- `/kill_nearby [radius]` → kill all enemies within radius (admin)
- `/toggle_hitboxes` → show/hide collision and terrain hitboxes on minimap
- `/toggle_rendering [feature]` → enable/disable rendering layer (shadows, particles, etc.)

### Performance Metrics Display — @file client/src/scripts/console/commands.ts
**Purpose:** Show FPS, ping, packet statistics, frame time breakdown.

**Metrics Displayed**:
- **FPS**: Frames per second (game render loop rate)
- **Ping**: Latency to game server (round-trip milliseconds)
- **Packet Size**: Average bytes per UpdatePacket
- **Memory**: Heap used / heap max (if available)
- **Frame Time**: Milliseconds since last frame update (budget: 40 TPS = 25 ms per tick)
- **Update Rate**: How many server updates per second
- **Interpolation**: Current interpolation time for smooth animation

**Implementation**: Command reads from `Game.netGraph` and timing statistics, formats as text display

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Console & debug subsystem overview
- **Tier 2:** [../../game-loop/README.md](../../game-loop/README.md) — Game loop (cheats modify game state)
- **Tier 1:** [../../../../development.md](../../../../development.md) — Development guide (includes console command list)
- **Patterns:** [../patterns.md](../patterns.md) (if created) — Console design patterns
