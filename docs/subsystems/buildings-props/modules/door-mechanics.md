# Door Mechanics Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/buildings-props/README.md -->
<!-- @source: server/src/objects/obstacle.ts, server/src/objects/building.ts -->

## Purpose
Manages door state transitions (open/closed/locked), swing animations, activation mechanics via player interaction or mode triggers, sound effects, hitbox switching, and network synchronization for all doors in the game world.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| `server/src/objects/obstacle.ts` | Door object class, state machine, activation logic | High |
| `server/src/objects/building.ts` | Building door integration, unlock mechanics | Medium |
| `common/src/utils/math.ts` | Door hitbox calculations (calculateDoorHitboxes) | High |
| `server/src/data/gameModes.ts` | Mode-specific unlock stages and door properties | Medium |
| `common/src/definitions/obstacles.ts` | Door definition structure and unlock behavior | Medium |

## Business Rules

### Door State Machine
Doors have three primary states:

```
┌──────────┐
│  CLOSED  │  (locked or unlocked)
└────┬─────┘
     │ [onClick: player interact]
     │ [unlockStage trigger: mode-specific unlock]
     ├─────────────────────────────────┐
     │                                 │
     v                                 v
┌─────────┐                      ┌──────────┐
│ OPENING │ (0.5 sec animation)  │ LOCKED   │ (stays until unlock)
└────┬────┘                      └────┬─────┘
     │ [animation complete]            │ [stage unlock, APKey, etc.]
     v                                 v
┌────────┐                      ┌──────────┐
│  OPEN  │ [openOnce → stuck]   │ UNLOCKING│ (instant or anim)
└────┬───┘                      └────┬─────┘
     │ [no re-close; openOnce=true]  │
     │ [time expiry if openOnce=false]│
     │                                v
     │ [on openOnce=false: auto-close after N sec]
     └────────────────────────────────┘

```

### Door Properties

**Hitbox switching:**
- **Closed state:** `door.closedHitbox` — blocking collision box (prevents player/bullets)
- **Open state:** `door.openHitbox` — reduced hitbox (e.g., just the door frame edges)
- **Alternate open:** `door.openAltHitbox?` — optional second hitbox for specific door types
  - Used for barn doors, sliding gates with side collision areas
  - If present, collision with either hitbox blocks movement
- **Height getter:** If door closed, `height = Infinity` (uncrossable); if open, `height = 0.2` (jumpable)

**Locked state:**
- `door.locked: boolean` — whether door requires unlock key or mode progression
- `door.operationStyle: 'pushable' | 'locked_before_unlock' | 'bunker'`
  - `pushable` — any player can open door (no lock)
  - `locked_before_unlock` — door stays locked until unlock condition met
  - `bunker` — H.U.N.T.E.D. mode bunker (requires APKey or stage unlock)
- **Unlock triggers:**
  - Mode unlock stage (`mode.unlockStage`) progression (Halloween, H.U.N.T.E.D. modes)
  - APKey item (`unlockableWithKey`; not lockableWithKey)
  - Bunker unlock (`perkOnDestroy: Infected` → player becomes bunker-opener)
  - Golden airdrop stage (`mode.forcedGoldAirdropStage`)

**One-time opening:**
- `door.openOnce: boolean` — if true, door opens and cannot re-close
  - Example: bunker doors in H.U.N.T.E.D. mode (open once per game)
  - Prevents tactical door closure for defensive holds
  - State persists for game duration (never reverts to locked)

### Activation Mechanics

**Player interaction:**
- Player enters interactive radius around door
- Presses interaction key (E / gamepad button)
- Server receives `ObjectInteraction` packet with door ID
- Server validates:
  1. Door exists (not destroyed)
  2. Player within range (16–32 units depending on door type)
  3. Door not already opening (state transition in progress)
  4. Player not downed or dead
  5. If locked: check unlock condition (key inventory, stage reached, etc.)
- If valid:
  - Set `door.isOpen = true`
  - Start swing animation (0.5 second clock)
  - Switch hitbox from `closedHitbox` to `openHitbox`
  - Emit sound effect (creaking, locking mechanism, etc.)
  - Mark door dirty (included in next UpdatePacket)
- If invalid:
  - Send "cannot interact" feedback (e.g., "Door is locked" message)
  - No state change

**Automatic unlock (mode-driven):**
- Game progresses to stage N (e.g., Stage 2 of Halloween mode)
- All doors with `unlockableWithStage` and matching stage auto-unlock (no interaction needed)
- Used for bunker progression in Halloween
- Triggered in `game.tick()` when `game.stage >= door.unlockStage`

### Sound Effects

**Door sounds triggered on state change:**

| Event | Effect | Duration |
|-------|--------|----------|
| Door opens (creaking) | 0.3 sec creak audio | 0.3 s |
| Door locked (rattle) | 0.2 sec rattle sound | 0.2 s |
| Door slams (auto-close) | 0.2 sec slam sound | 0.2 s |

- Audio file referenced from definition: `definition.sound` (e.g., `"door_creak_wood"`)
- Played via SoundManager at door position
- 3D audio: volume/pan based on camera distance and angle

### Network Synchronization

**Include in UpdatePacket:**
- `ObstaclePartial.isOpen: boolean` — current door state
- `ObstaclePartial.locked: boolean` — if state changed (lock/unlock)
- Only serialized if door is "dirty" (state changed)
  
**Full vs. Partial updates:**
- **Full:** New door spawned; includes all hitbox data, unlock stage, operation style
- **Partial:** Door state changed (open/close/lock); only `isOpen` and `locked` flags

**Client-side rendering:**
- Client receives `ObstacleUpdate.isOpen` flag
- Play animation: swing door from closed position to open position
- Update collision hitbox in spatial grid (client uses CulledGrid for rendering)
- Sync audio playback (play creak sound if door was previously closed)

## Data Lineage

```
Player Interaction Input
  ↓
[Server validates: range, state, unlock condition]
  ↓
door.isOpen = true
  ↓
door.hitbox = door.openHitbox
  ↓
door.powered = true (for motorized doors)
  ↓
Emit sound event
  ↓
Mark door.setDirty()
  ↓
[Next tick: included in UpdatePacket.obstacleUpdates]
  ↓
Client receives update
  ↓
[Animate: close hitbox → open hitbox]
  ↓
Door rendered as open
```

## Dependencies

- **Internal:**
  - Buildings & Props — Building container for doors
  - Collision/Hitbox — Hitbox calculations and spatial collision
  - Game Loop — Stage/mode progression triggers
- **External:**
  - [Core Math & Physics](../../core-math-physics/README.md) — `calculateDoorHitboxes()` rotation and position calculation
  - [Networking](../../networking/README.md) — UpdatePacket obstacle serialization
  - [Game Objects Server](../../game-objects-server/README.md) — BaseGameObject lifecycle
  - [Sound Management](../../sound-management/README.md) — Door audio playback
  - [Game Modes](../../game-modes/README.md) — Mode unlock stages and triggers

## Complex Functions

### Hitbox Calculation
**Function:** `calculateDoorHitboxes(doorDef: ObstacleDefinition, position: Vector, rotation: Orientation): { closedHitbox: Hitbox; openHitbox: Hitbox; openAltHitbox?: Hitbox }`  
**Source:** `common/src/utils/math.ts:calculateDoorHitboxes()`  
**Purpose:** Pre-compute closed and open hitboxes based on definition and rotation

**Logic:**
```
1. Get base hitbox dimensions from doorDef (width, height, offset)
2. Apply rotation to hitbox (rotate offset vector by door rotation angle)
3. Calculate closedHitbox at rotated position
4. Calculate openHitbox = reduced hitbox (door "pushed aside")
   - For horizontal swing: openHitbox.width reduced, height same
   - For vertical swing: openHitbox.height reduced, width same
   - Offset adjusted so door doesn't "walk through" frame
5. If doorDef has openAltHitbox config: calculate alternate hitbox
   - Used for double-swinging doors (collision on both sides when open)
6. Return { closedHitbox, openHitbox, openAltHitbox }
```

**Implicit behavior:**
- Hitbox pre-calculated at door creation (not per-frame)
- Rotation determines swing axis (important for correct collision)
- Offset vectors ensure door doesn't penetrate walls
- Both hitboxes use absolute coordinates (not relative to door object)

**Called by:**
- Obstacle constructor — during door setup
- Map generation — validates door placement

### Door Interaction Handler
**Function:** `onInteract(player: Player): void`  
**Source:** `server/src/objects/obstacle.ts (as method on Obstacle with isDoor check)`  
**Purpose:** Handle player interaction with door

**Logic:**
```
if (!this.door) return  // door doesn't exist

if (this.door.isOpen) return  // already open
if (this.door.powered) return  // motorized, player can't interact

// Check unlock condition
if (this.door.locked) {
  if (!this.checkUnlockCondition(player)) {
    player.notify("This door is locked")
    return
  }
  this.door.locked = false
}

// Swing open
this.door.isOpen = true
this.door.hitbox = this.door.openHitbox
this.playSound("door_creak")

// If openOnce is true, door never closes
if (!this.door.openOnce) {
  this.scheduleClose(DOOR_AUTO_CLOSE_TIME)  // re-close after 5 sec
}

this.setDirty()
```

**Implicit behavior:**
- Check unlock happens INSIDE the interaction, not before
- If locked AND no key/stage: interaction fails silently (no error log)
- openOnce=true: permanently unlocks door (can't be closed again)
- Auto-close only if openOnce=false; otherwise stuck open
- Sound plays synchronously (not queued)

**Called by:**
- Interaction event handler — when player sends ObjectInteraction packet
- Plugin event hook — plugins can override or extend behavior

### Unlock Condition Check
**Function:** `checkUnlockCondition(player: Player): boolean`  
**Source:** `server/src/objects/obstacle.ts`  
**Purpose:** Determine if player can unlock door

**Logic:**
```
// Check all unlock conditions until one succeeds
if (this.definition.unlockableWithStage && this.game.stage >= this.door.unlockStage) {
  return true  // stage unlocked
}
if (this.definition.unlockableWithKey) {
  const hasKey = player.inventory.items.find(item => item.idString === "ap_key")
  if (hasKey) {
    player.inventory.removeItem(hasKey)  // consume key
    return true
  }
}
if (this.game.mode.unlockStage !== undefined && this.game.stage >= this.game.mode.unlockStage) {
  return true  // bunker unlock logic for H.U.N.T.E.D.
}
return false  // no unlock condition met
```

**Implicit behavior:**
- Checks are OR'd (any single condition unlocks)
- APKey consumed on use (removed from inventory)
- Stage unlock is automatic (no item consumption)
- Mode-specific logic (H.U.N.T.E.D. vs. Halloween differ)
- If no unlock method available: always returns false

**Called by:**
- `onInteract()` — before opening door

## Configuration

| Setting | Location | Effect | Default |
|---------|----------|--------|---------|
| `isDoor` | obstacle definition | Marks obstacle as door (enables door logic) | false |
| `operationStyle` | door definition | Door type (pushable vs. locked vs. bunker) | "pushable" |
| `locked` | door definition OR constructor | Initial lock state | false |
| `openOnce` | door definition | If true, can't re-close door | false |
| `unlockableWithStage` | door definition | Can stage unlock open door | false |
| `unlockableWithKey` | door definition | Can APKey open door | false |
| `DOOR_AUTO_CLOSE_TIME` | constants.ts | Auto-close delay (ms) if openOnce=false | 5000 |
| `DOOR_SWING_ANIMATION_TIME` | constants.ts | Door swing animation duration | 500 |

## Edge Cases & Gotchas

### Gotcha: Simultaneous Unlock and Interact
- **Issue:** Player interacts with door EXACTLY when stage unlock triggers
  - Player sends interact packet
  - Server processes stage unlock (door.locked = false)
  - Server processes player interact (tries to unlock, succeeds due to stage)
  - Door opens twice; both events queued
- **Prevention:** Check `if (door.isOpen) return` before interaction handler

### Gotcha: Bunker Door Unlock Condition
- **Issue:** H.U.N.T.E.D. mode bunker doors require "infection" perk or stage
  - Player wants to speed-run bunker unlock
  - Infection perk not found in definition; unlock fails
  - Error silent; player frustrated
- **Prevention:** Log unlock failures in development builds; clearly document unlock requirements

### Gotcha: Open Hitbox Collision
- **Issue:** Open hitbox still causes collision (design flaw)
  - Door openHitbox is supposed to reduce collision area
  - But openHitbox calculated incorrectly; still blocks bullets
  - Players can't shoot through "open" door
- **Prevention:** Validate hitbox dimensions in test suite; ensure openHitbox ⊂ closedHitbox area

### Gotcha: Auto-Close During Combat
- **Issue:** Door auto-closes after 5 seconds while player actively fighting
  - Player opens door for escape route
  - 5 seconds pass; door auto-closes
  - Player trapped inside; loses fight
- **Prevention:** Reset auto-close timer if enemies near door; or manual door closing mechanic

### Gotcha: Motorized Door Activation
- **Issue:** `door.powered = true` for motorized doors
  - Player tries to interact with powered door
  - Server rejects (manual interaction disabled for powered doors)
  - Animation plays anyway (from previous state sync)
  - Visual/gameplay mismatch
- **Prevention:** Don't re-animate if powered; wait for server-side activation

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Buildings & Props subsystem overview
- **Tier 1:** [../../architecture.md](../../architecture.md) — System architecture
- **Patterns:** [../patterns.md](../patterns.md) — Obstacle interaction patterns
- **Related modules:**
  - [Core Math & Physics](../../core-math-physics/README.md) — Hitbox rotation and geometry
  - [Game Modes](../../game-modes/modules/mode-rules.md) — Mode unlock stages
  - [Game Objects Server](../../game-objects-server/README.md) — Object lifecycle and dirty flag system
