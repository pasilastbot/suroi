# Settings & Menus Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/ui-management/README.md -->
<!-- @source: client/src/scripts/ui.ts, client/index.html -->

## Purpose

Manages main menu state machine, settings panel, audio/graphics/input configuration, loadout selection, and team mode selection before game start.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| [client/src/scripts/ui.ts](../../../../../client/src/scripts/ui.ts) | Menu state machine, button event handlers, settings logic, team/loadout selection | High |
| [client/index.html](../../../../../client/index.html) | UI structure: menu containers, buttons, input fields, settings panels | Medium |
| [client/src/scripts/managers/uiManager.ts](../../../../../client/src/scripts/managers/uiManager.ts) | HUD elements rendered during gameplay (not menu) | High |
| [client/src/scripts/config.ts](../../../../../client/src/scripts/config.ts) | Server list, region selection, game configuration | Medium |

## Business Rules

- **State Machine**: Menu, settings, team creation, game entry → game starts
- **Settings Categories**:
  - **Volume**: Master, SFX, music sliders
  - **Graphics**: Renderer selection (WebGL1/2), resolution, antialias, shadows
  - **Input**: Key binding, mouse sensitivity, gamepad support, mobile UI
  - **Gameplay**: Auto-pickup, anonymize names, etc.
- **Loadout Selection**: Player picks skin, emote wheel, melee, primary gun, secondary gun
- **Team Modes**: Solo, Duo, Squad team creation with player slots
- **Game Entry**: Username input, server/region selection, optional game code
- **Mobile Differences**: Simplified menu, touch-friendly buttons, no keybind editor
- **Button Lock**: `buttonsLocked` state prevents rapid clicking during transitions
- **Team Socket**: WebSocket connection to team creation server for multi-player lobby

## Data Lineage

```
Player Input (clicks, text input, slider changes)
  ↓
Event handlers in ui.ts (button callbacks, input listeners)
  ↓
State update:
  1. Read form inputs (username, region, settings)
  2. Validate values (min/max, string length)
  3. Store in localStorage or in-memory state
  ↓
Team Creation Flow (optional):
  - Create team via teamSocket WebSocket
  - Generate shareable team URL
  - Wait for teammates to join
  - All ready → trigger game start
  ↓
Game Start Trigger (via JoinPacket):
  - Send username, loadout, region, team ID to game server
  - Transition from menu → game scene
```

## Configuration

| Setting | Effect | Source |
|---------|--------|--------|
| `autoPickup` | Auto-pick up items on proximity | `ui.ts` boolean flag |
| `teamMode` | Solo=1, Duo=2, Squad=4 players | Selected via menu buttons |
| `selectedRegion` | Game server region (e.g., "us-east") | Region list from `Config.regions` |
| `buttonsLocked` | Prevent button clicks during transitions | `ui.ts` boolean, set via `lockPlayButtons()` / `unlockPlayButtons()` |
| `joinedTeam` | Status of team socket connection | `ui.ts` boolean, updated on team creation |
| Settings Preset | Volume/graphics saved in localStorage | Browser persistent storage |

## Complex Functions

### `setUpUI()` — @file client/src/scripts/ui.ts
**Purpose:** Initialize all menu UI elements, attach event handlers, restore settings from storage.

**Implicit behavior:**
- Called once at game startup (before visible page render)
- Loads saved settings from `localStorage`
- Initializes region/server list from `Config.regions`
- Attaches click handlers to all menu buttons
- Creates dropdown menus for graphics/input options
- Does NOT start game (only sets up click listeners)

### `resetPlayButtons()` — @file client/src/scripts/ui.ts
**Purpose:** Unlock play/team buttons after state transition completes.

**Implicit behavior:**
- Checks `buttonsLocked` before allowing click
- If locked, returns early (prevents double-clicks)
- Used after game start, region switch, or team creation

### `unlockPlayButtons()` / `lockPlayButtons()` — @file client/src/scripts/ui.ts
**Purpose:** Control button interactivity during loading or transitions.

**Implicit behavior:**
- `lockPlayButtons()` → set `buttonsLocked = true`
- `unlockPlayButtons()` → set `buttonsLocked = false`
- Called before async operations (server fetch, team creation)

### Team Socket Connection — @file client/src/scripts/ui.ts
**Purpose:** Establish WebSocket to team creation server for multiplayer lobby.

**Implicit behavior:**
- Opens socket on "Create Team" button click
- Sends `JoinTeamPacket` with player username
- Listens for team join updates (`CustomTeamMessage`)
- When all players "ready" → triggers game start via `JoinPacket`
- Socket closes on disconnect or timeout

### Menu Button Click Handlers — @file client/src/scripts/ui.ts
**Purpose:** Transition between menu states (Main → Settings → Game Start).

**Implicit behavior:**
- Click "Play Solo/Duo/Squad" → set `teamMode` → show region list
- Click region → fetch server data → show username/code entry
- Click "Start Game" → lock buttons → send `JoinPacket` to game server
- Settings button → show settings panel, restore current values, attach sliders
- Back button → restore previous menu state

## Related Documents

- **Tier 2:** [../README.md](../README.md) — UI Management subsystem overview
- **Tier 2:** [../../client-managers/README.md](../../client-managers/README.md) — Other client-side managers
- **Tier 1:** [../../../../architecture.md](../../../../architecture.md) — Client architecture overview
- **Patterns:** [../patterns.md](../patterns.md) — UI state management patterns
