# Camera Management — Spectate Mode Module

<!-- @tier: 3 -->
<!-- @parent: ../README.md -->
<!-- @source: client/src/scripts/managers/cameraManager.ts -->

## Purpose
Encapsulates spectator camera behavior: spectator player selection, camera following of spectated target, UI adjustments for spectate mode, death-to-spectate transition logic, and team-mode spectate restrictions.

## Key Files
| File | Purpose | Complexity |
|------|---------|------------|
| `client/src/scripts/managers/cameraManager.ts` | Camera positioning and spectate camera logic | High |
| `client/src/scripts/managers/uiManager.ts` | UI updates when transitioning to spectate | Medium |
| `common/src/packets/spectatePacket.ts` | Spectate data serialization (target player ID, team info) | Low |
| `server/src/objects/player.ts` | Death logic and spectate eligibility | Medium |

## Spectate Modes

Suroi supports two primary spectate scenarios:

### Mode 1: Watching a Teammate (Team Modes)
**Trigger:** Player's HP drops to 0; teammates still alive
**Duration:** Until all teammates dead OR match ends
**Restrictions:** Can only see teammate's visible objects + minimal HUD

**State:**
```typescript
spectatingPlayer: Player | undefined;  // Target player being watched
spectateMode: "teammate" | "observer";
canSwitchSpectateTarget = true;       // Can cycle teammates
```

### Mode 2: Observer/Free Spectate (Any Mode)
**Trigger:** All teammates dead (or solo mode, player dead)
**Duration:** Until match ends
**Restrictions:** Can see full map, all dead players

**State:**
```typescript
spectatingPlayer = undefined;  // Not watching specific player
spectateMode: "observer";
canSwitchSpectateTarget = false;  // No player to switch to
```

## Spectator Player Selection

### Choosing Initial Spectate Target

When player dies, game determines who to spectate:

```
Is game in team mode?
  ├─ YES
  │  ├─ Do I (dead player) have alive teammates?
  │  │  ├─ YES → Spectate first alive teammate
  │  │  └─ NO → Fall through to observer mode
  │  │
  │  └─ NO (solo mode)
  │     └─ Fall through to observer mode
  │
  └─ All dead or solo
     └─ Observer mode (free camera)
```

**Code location:** @file server/src/objects/player.ts (death method)
```typescript
die() {
  if (this.game.isTeamMode && this.team?.players.size > 0) {
    // Find first alive teammate
    const aliveMate = Array.from(this.team.players).find(p => !p.dead);
    this.spectatingPlayer = aliveMate;  // Watch them
  } else {
    // Observer mode
    this.spectatingPlayer = undefined;
    this.spectateMode = "observer";
  }
}
```

### Switching Spectate Target

**In team mode:** Player can manually cycle through alive teammates via input (e.g., Tab key):

```typescript
// Input packet includes "cycle_spectate_target" action
if (inputData.cycleSpectate) {
  const teammates = Array.from(this.team.players).filter(p => !p.dead);
  const currentIndex = teammates.indexOf(this.spectatingPlayer);
  const nextIndex = (currentIndex + 1) % teammates.length;
  this.spectatingPlayer = teammates[nextIndex];
}
```

**In observer mode:** No cycling; camera is free-roaming

## Camera Following Teammate

### Camera Position Calculation

When watching a teammate, camera renders from their perspective:

```typescript
// cameraManager.ts
if (spectatingPlayer) {
  // Position camera to match spectated player's view
  const targetPosition = spectatingPlayer.position;
  
  // Smoothly interpolate (don't snap instantly)
  this.position = Vec.lerp(
    this.position,
    targetPosition,
    0.1  // Smooth factor
  );
  
  // Zoom matches their equipped scope
  const zoom = spectatingPlayer.activeItem?.scope?.zoomLevel ?? DEFAULT_SCOPE.zoomLevel;
  this.zoom = zoom;
  
  // Layer matches their floor
  this._currentLayer = spectatingPlayer.layer;
}
```

**Smoothing:** Lerp prevents jarring snaps when target moves quickly
**Scope sync:** Spectator's view is zoomed out/in with target's scope
**Layer sync:** Spectator sees same basement/ground/upstairs layer as target

### Spectate Camera Bounds

Camera is NOT restricted to game bounds (can follow target to edge):

```typescript
// No clamping; follow wherever target goes
const newPos = targetPlayer.position;
this.position = newPos;  // Unbounded follow
```

## Spectator UI Differences

### HUD Changes in Spectate Mode

**Standard gameplay HUD:**
```
╔════════════════════╗
║ Health bar         │
║ Adrenaline        │
║ Inventory (guns)   │
║ Minimap            │
╚════════════════════╝
```

**Teammate spectate HUD:**
```
╔════════════════════╗
║ SPECTATING: [Name] │
│ Health: 45/100     │
│ Adrenaline: 30     │
│ Equipped: M4       │
│ Minimap (limited)  │
║ [← Tab →] Switch   │
╚════════════════════╝
```

**Observer HUD:**
```
╔════════════════════╗
║ OBSERVER MODE      │
│ Alive players: 3   │
│ Minimap (full)     │
│ [Spectate player]  │
╚════════════════════╝
```

**Disabled in spectate mode:**
- Inventory interaction (can't swap weapons)
- Health/adrenaline consumption (can't heal)
- Input actions (firing disabled)
- Crosshair/aim (read-only)

### UI State Management

```typescript
class UIManager {
  updateSpectateUI(mode: "teammate" | "observer") {
    if (mode === "teammate") {
      UI.show("spectate_target_name");
      UI.show("spectate_switch_hint");
      UI.show("spectated_player_health");
      UI.hide("action_buttons");
      UI.hide("inventory");
    } else if (mode === "observer") {
      UI.show("observer_mode_label");
      UI.show("alive_count");
      UI.hide("spectate_target_name");
      UI.show("minimap_full");
    }
  }
}
```

## Death → Spectate Transition

### Sequence of Events

```
1. Player HP drops to 0
   └─ death() called on player object (server)
2. Death logic executes
   ├─ Remove from living players list
   ├─ Determine spectate eligibility
   ├─ Mark player.dead = true
   └─ Emit death event
3. Death event triggers on client
   ├─ Transition camera to spectate mode
   ├─ Update UI (hide health, show spectate label)
   ├─ Disable input (no shooting, moving)
   └─ Start following spectated player
4. Client receives next UpdatePacket
   ├─ Spectate target position
   ├─ Visibility from target's perspective
   └─ HUD state (target's health, ammo)
```

**Timing:** Transition is <1 frame (25ms), instantaneous from player perspective

### Visual Transition

**Option 1: Instant transition**
```typescript
// Death registered
if (this.player.dead) {
  this.camera.enterSpectateMode(this.spectatingPlayer);
  this.ui.hidePlayHUD();
  this.ui.showSpectateHUD();
}
```

**Option 2: Fade transition (smoother)**
```typescript
this.camera.fadeOut(duration: 200);  // Black screen
this.camera.enterSpectateMode(target);
this.camera.fadeIn(duration: 200);   // Fade in from target's view
```

Current implementation: Option 1 (instant)

## Team Spectate Restrictions

### Team Mode Visibility Rules

**Dead player spectating teammate:**
- ✅ Can see: Teammate's visible objects (from teammate's POV)
- ✅ Can see: Teammate's health, ammo, equipped items
- ❌ Cannot see: Other teams' positions (only teammate's can see them)
- ❌ Cannot see: Enemies outside teammate's view range
- ❌ Cannot see: Fog of war bypassed (only teammate's visibility)

**Observer mode:**
- ✅ Can see: All dead players
- ✅ Can see: Map layout (full minimap)
- ❌ Cannot see: Living players' vision (observer is "neutral")

### Spectate Visibility Filtering

Server sends `SpectatePacket` instead of `UpdatePacket` for dead players:

```typescript
// server/src/objects/player.ts
if (deadPlayer.spectatingPlayer) {
  // Build spectate packet (limited visibility)
  const spectatePacket = new SpectatePacket();
  spectatePacket.targetPosition = teammateLoc;
  spectatePacket.targetHealth = teammate.health;
  spectatePacket.visibleObjects = teammate.visibleObjects;  // Filter!
  deadPlayer.sendPacket(spectatePacket);
} else {
  // Observer mode (see more)
  const spectatePacket = new SpectatePacket();
  spectatePacket.mode = "observer";
  spectatePacket.aliveCount = game.aliveCount;
  deadPlayer.sendPacket(spectatePacket);
}
```

**Key:** Server-side visibility filtering prevents cheating (dead players can't see through walls)

## Observer Mode Camera Behavior

### Free Roaming Camera

Once all teammates dead, camera is free to move:

```typescript
if (spectateMode === "observer") {
  // Camera position NOT bound to any player
  // Can pan around freely (if implemented)
  
  // Alternatives:
  // 1. Fixed to last teammate's death location
  // 2. Can pan via arrow keys / mouse
  // 3. Cycles through living players automatically
}
```

**Current implementation:** Camera stays at last spectated player's position; manual pan not implemented

### Observer Minimap

Full map visible to spectators (not in team mode):

```typescript
// Show all map features
minimap.showAllObjects = game.isTeamMode ? false : true;
minimap.showAllPlayers = game.isTeamMode ? false : true;
```

## Complex Functions

### `cameraManager.enterSpectateMode()` — @file client/src/scripts/managers/cameraManager.ts
**Purpose:** Transition camera from active gameplay to spectate view
**Parameters:** `targetPlayer: Player | undefined` (or `undefined` for observer)
**Complexity:** ~50 lines
**Implicit behavior:**
- Fixes position to target
- Clears zoom tween (if mid-zoom from scope change)
- Updates layer visibility
- Sets `this.spectatingPlayer` reference
- Disables player input

### `player.handleDeath()` — @file server/src/objects/player.ts
**Purpose:** Execute death logic (server-side)
**Complexity:** ~100 lines
**Implicit behavior:**
- Checks team mode
- Finds alive teammates (or sets to observer)
- Sets `this.spectatingPlayer`
- Calls `sendGameOverPacket()` to broadcast death to all players
- Triggers spectate mode on client (via game over packet)

### `uiManager.updateSpectateUI()` — @file client/src/scripts/managers/uiManager.ts
**Purpose:** Update HUD for spectate mode
**Parameters:** `targetPlayer: Player | undefined`
**Implicit behavior:**
- Shows/hides spectate UI elements
- Updates teammate name label
- Shows switch hint (Tab key)
- Hides inventory/weapons
- Updates minimap visibility

## Configuration & Tuning

**Camera lerp smooth factor:** @file client/src/scripts/managers/cameraManager.ts
```typescript
const SPECTATE_FOLLOW_SMOOTHING = 0.1;  // 0 = instant, 1 = no movement
// Adjust to make camera follow feel more responsive or dreamy
```

**Spectate UI update interval:**
```typescript
// Update spectated player's health/ammo every tick
// No throttling (max 40 updates per second at 40 TPS)
```

**Observer mode timeout (not implemented):**
```typescript
// Could auto-cycle through living players in observer mode
// Interval: every 5 seconds or so
const OBSERVER_AUTO_CYCLE_INTERVAL = 5000;  // ms
```

## Related Documents
- **Tier 2:** [Camera Management](../README.md) — subsystem overview
- **Tier 2:** [UI Management](../../ui-management/README.md) — HUD state
- **Tier 2:** [Team System](../../team-system/README.md) — team mode rules
- **Tier 2:** [Death & Spectator](../../death-spectator/README.md) — game over mechanics
- **Tier 3:** [Layer Management](../modules/layer-management.md) (when documented) — layer transitions
- **Tier 3:** [Zoom & Scope](../modules/zoom-scope.md) (when documented) — FOV adjustment
- **Patterns:** [../patterns.md](../patterns.md) — subsystem patterns
