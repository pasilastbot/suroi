# UI Management Subsystem

<!-- @tier: 2 -->
<!-- @parent: docs/architecture.md -->
<!-- @modules: docs/subsystems/ui-management/modules/ -->
<!-- @source: client/src/scripts/managers/uiManager.ts -->

## Purpose

Manages all client-side HUD (Heads-Up Display) elements: inventory display, killfeed, teammate indicators, action timers, health/adrenaline/shield bars, spectating UI, perk display, and kill messages. Receives game state updates from `UpdatePacket` and synchronously renders them into the DOM without further messaging.

## Key Files & Entry Points

| File | Purpose |
|------|---------|
| `client/src/scripts/managers/uiManager.ts` | `UIManager` singleton — HUD state, DOM updates, event processing |
| `client/src/scripts/ui.ts` | Root UI setup, event wiring, DOM element caching (`createDropdown()`) |
| `client/src/scripts/uiHelpers.ts` | Shared UI utilities (`createDropdown`, `body` reference) |
| `client/index.html` | DOM structure with HUD element IDs (`#game-ui`, `#kill-feed`, `#health-bar`, etc.) |
| `client/src/scss/` | Stylesheets for all HUD elements (animations, colors, layouts) |

**Singleton:** `export const UIManager = new UIManagerClass()` (instantiated at [module load time](../../../client/src/scripts/managers/uiManager.ts#L1860))

## Architecture

The UIManager is a **passive state container** — it receives data from the game update loop and synchronously mutates the DOM. It does **not** send messages or trigger game logic changes; all communication flows one-way: `Game.processUpdate(packet)` → `UIManager.updateUI(playerData)`.

### Data Flow

```
UpdatePacket (binary, from WebSocket)
    ↓
Game.processUpdate(packet)
    ↓ (extracts PlayerData)
UIManager.updateUI(playerData)
    ├─ Health: playerData.health → DOM #health-bar, #health-bar-amount
    ├─ Ammo: playerData.inventory → DOM #weapon-clip-ammo, #weapon-inventory-ammo
    ├─ Weapons: playerData.inventory.weapons → DOM #weapon-slot-{1-4}
    ├─ Items: playerData.items → DOM item count displays
    ├─ Teammates: playerData.teammates → LiveCollection of TeammateIndicatorUI elements
    └─ Status: playerData.blockEmoting, perks → toggle UI visibility/state
    

KillPacket (binary, from WebSocket)
    ↓
Game.processKill(packet)
    ↓ (extracts KillData)
UIManager.processKillPacket(killData)
    ├─ Adds kill entry to DOM #kill-feed (2-3 per screen, fade-out after 7s)
    └─ Shows modal #kill-msg-modal (fade-out after 3s if no new kill)
```

### HUD Element Categories

| Category | DOM Root | Update Trigger | Actual Elements |
|----------|----------|-----------------|-----------------|
| **Inventory** | `#weapon-ammo-container` | Shot fired, reload, pickup | `#weapon-clip-ammo` (loaded rounds), `#weapon-inventory-ammo` (reserve) |
| **Weapon Slots** | `#weapons-container` | Weapon equip/swap, pickup | `#weapon-slot-{1-4}` with item name, image, ammo counter |
| **Health** | `#health-bar` | Damage, heal, medical item use | Animated bar, color-coded (red ≤25%, yellow 25-60%, white >60%) |
| **Adrenaline** | `#adrenaline-bar` | Stimulation, medical item use | Animated bar, white text if ≥7, red if <7 |
| **Shield** | `#shield-bar` | Body armor equip, damage | Clip-path inset, dynamic width |
| **Infection** | `#infection-bar` | Gas zone ticks | Clip-path inset, dynamic width |
| **Killfeed** | `#kill-feed` | Enemy eliminations | `-webkit-user-select: none`, max 5 simultaneous entries |
| **Kill Modal** | `#kill-msg-modal` | Player gets kill credit | High-priority message with 3s fade-out |
| **Teammates** | `#team-container` | Team update packet | `TeammateIndicatorUI` divs (custom class) |
| **Spectating** | `#spectating-container` | Player death (spectate mode) | Spectating info, prev/next buttons, menu button |
| **Action Timer** | `#action-container` | Reload, use medical, revive | `#action-time` text, `#action-timer-anim` SVG progress circle |
| **Game Over** | `#game-over-overlay` | Match end | Player cards with medals, kills, damage, time alive |
| **Perks** | `#perk-slot-{0-2}` | Perk acquired | Icon, tooltip, timer animation |
| **Interaction** | `#interact-message` | Near lootable object | "Press [key] to loot [item]" |
| **C4 Detonation** | `#c4-detonate-btn` | Has C4 items | Hidden if no C4s, shows "Detonate" on hotkey press |
| **Kill Leader** | `#kill-leader-leader`, `#kill-leader-count` | Leader changes or kill | Leader name badge, kill count, spectate button |
| **Players Alive** | `#ui-players-alive` | Player elimination | Update count display |

## UIManager API

### Core Update Methods

#### `updateUI(data: PlayerData): void`
**Called every game tick from:** `Game.processUpdate()` (40 TPS)

**Responsibilities:**
- **Ping tracking** — extract `data.pingSeq`, calculate RTT from `Game.seqsSent`
- **Player ID & spectating state** — update `Game.activePlayerID`, toggle spectating UI
- **Health/Adrenaline/Shield/Infection bars** — set DOM widths/colors and text
- **Inventory state** — cache weapons, items, scope in `this.inventory`
- **Teammate tracking** — create/update/destroy `TeammateIndicatorUI` instances in `#team-container`
- **Highlighted players** — update teammate health bars
- **Perk slots** — set perk icons/timers
- **Emoting block state** — store `data.blockEmoting` to prevent emote spam

**Key internal caches:**
- `this.inventory` — weapons array, active index, items, scope, locked slots
- `this._weaponSlotCache` — Map of DOM elements for slots 1-4
- `this._itemSlotCache` — Map of DOM elements for consumable items
- `this._scopeSlotCache` — Map of DOM elements for scope slots
- `this._teammateDataCache` — Map of `TeammateIndicatorUI` instances by player ID

#### `processKillPacket(data: KillData): void`
**Called when:** Kill/down event occurs

**Responsibilities:**
- **Killfeed entry** — create HTML, insert into `#kill-feed`, animate, auto-remove after 7s
- **Kill modal** — conditionally show high-priority kill message for 3s if:
  - Active player got kill credit, **or**
  - Teammate got the kill, **or**
  - Active player was the victim
  - Message localizes based on damage source (gun, explosive, gas, obstacle, etc.)
- **Kill streaks** — display streak count if applicable
- **Sound** — trigger via `SoundManager` (if applicable kill type)

#### `processReportPacket(data: ReportData): void`
**Called when:** Report submitted (client-side only, for UI feedback)

Shows temporary modal with report ID, disables report button.

#### `showGameOverScreen(packet: GameOverData): void`
**Called when:** Match ends (player death or win)

**Renders:**
- Victory/defeat message and rank
- Player cards for self + teammates (if team mode)
- Medal display: killed badge, most kills, most damage done/taken (team mode only)
- Total team kills (team mode only)
- Victory music playback
- Screen fade-in animation

#### `updateKillLeader(data: UpdateDataCommon["killLeader"]): void`
**Called when:** Kill leader changes or kill count updates

Shows leader name, kill count, and crown emoji in killfeed when leader changes.

#### `reset(): void`
**Called when:** Game ends or player respawns

Clears all caches, hides overlays, resets state to initial values.

---

### Inventory & Weapon Methods

#### `updateWeapons(): void`
**Responsibilities:**
- Render active weapon ammo counter (`#weapon-clip-ammo`/`#weapon-inventory-ammo`)
- Handle ephemeral ammo (`∞` symbol)
- Check for infinite ammo perk
- Hide ammo counter if unarmed (fists)
- Toggle reserve ammo visibility

#### `updateWeaponSlots(force = false): void`
**Responsibilities:**
- Render each of 4 weapon slots (`#weapon-slot-{1-4}`)
- Show/hide item image, name, ammo count
- Highlight active slot
- Apply color-coding based on ammo type and console var `cv_weapon_slot_style`
- Scale inventory images (`inventoryScale` property)
- Handle dual-wielding display
- Animate slot change if not force-updating

#### `updateSlotLocks(): void`
**Responsibilities:**
- Toggle `locked` class on weapon slots based on `this.inventory.lockedSlots` bitmask
- Locked slots are unequippable (UI visual only; server enforces)

#### `updateItems(): void`
**Responsibilities:**
- Update count displays for consumable items (ammo, healing items)
- Hide item slots if count is 0 and item has `hideUnlessPresent` flag (team mode)
- Update scope slot display

#### `updateRequestableItems(): void`
**Responsible for:**
- Show/hide item slots when ping wheel active (team mode)
- Toggle visibility class on ammo/healing item slots

---

### Perk Management

#### `updatePerkSlot(perkDef: PerkDefinition, index: number): void`
**Responsibilities:**
- Render perk icon at `#perk-slot-{index}`
- Show temporary inventory message with perk name
- Set up timer animation if perk has duration (duration, shield respawn time, highlight duration, cooldown)
- Update tooltip with perk description

#### `resetPerkSlot(index: number): void`
**Clears:**
- Perk icon and tooltip
- Hides slot if not in DEBUG mode

---

### Action Timer (Reload, Revive, etc.)

#### `animateAction(name: string, time: number, fake = false): void`
**Called by:** Weapon reload, medical use, revive, etc.

**Renders:**
- Circular progress SVG at `#action-timer-anim`
- Text label at `#action-name`
- Visibility toggle of `#action-container`
- Animates SVG stroke-dashoffset from 226 to 0 over `time * 1000` ms

**Parameters:**
- `name` — localized action name (e.g., "Reloading", "Using Medical")
- `time` — duration in seconds
- `fake` — if `true`, don't show cancel popup on interrupt (e.g., during revive)

#### `updateAction(): void`
**Called every tick to update countdown time display** (if action in progress)

#### `cancelAction(): void`
**Called when action interrupted** (e.g., player moves during reload)

---

### State Accessors

#### `getRawPlayerName(id: number): string`
Returns player name or `[Unknown Player]` if not cached. Respects `cv_anonymize_player_names` console var.

#### `getPlayerData(id: number): { name: string; badge?: BadgeDefinition }`
Returns player name (as jQuery HTML) + badge definition, respecting anonymization setting.

#### `getHealthColor(normalizedHealth: number, downed?: boolean): string`
Returns CSS color string for health bar:
- **Red** (#ff0000) if ≤25% or downed
- **Yellow gradient** if 25-60%
- **Light gray** (#f8f9fa) if >60%

#### `getTeammateColorIndex(id: number): number | undefined`
Returns teammate color index for nameplate colors (team mode only).

---

## Key Properties

| Property | Type | Purpose |
|----------|------|---------|
| `ui` | Frozen object of JQuery cached DOM elements | Fast, compile-time-verified DOM access |
| `inventory` | Object with weapons array, items, scope, locked slots | Current inventory state cache |
| `teammates` | `PlayerData["teammates"]` | Teammate list from last update |
| `emotes` | `ReadonlyArray<EmoteDefinition \| undefined>` | Assigned emotes for wheel |
| `action` | Object with active, fake, start, time, pos | Current action timer state |
| `hasC4s` | boolean | Whether player has C4s for detonation |
| `blockEmoting` | boolean | Whether emoting is temporarily disabled |
| `reportedPlayerIDs` | Map<number, boolean> | Reported player IDs (prevents duplicate reports) |
| `skinID` | string \| undefined | Current loadout skin (for fist rendering) |

---

## Dependencies

### Depends on:
- **[Client Rendering](../client-rendering/)** — DOM hierarchy, jQuery selector caching, PIXI rendering context
- **[Networking](../networking/)** — `KillPacket` for killfeed, `UpdatePacket` for player data, `GameOverPacket` for match end
- **[Inventory](../inventory/)** — item definitions, ammo types, perk definitions
- **[Input Management](../input-management/)** — hotkey display in interaction prompts, mobile detection
- **[Sound Management](../../../client/src/scripts/managers/soundManager.ts)** — SoundManager for kill sounds, action start sounds
- **[Camera Management](../../../client/src/scripts/managers/cameraManager.ts)** — CameraManager.zoom updates from UIManager

### Depended on by:
- **[Game Loop](../game-loop/)** — calls `UIManager.updateUI()` every tick
- **[Client Rendering](../client-rendering/)** — DOM elements rendered in PixiJS `uiContainer`
- **[Input Management](../input-management/)** — read `UIManager.blockEmoting` to prevent emote input

---

## State Lifecycle

### Game Initialization
```typescript
// UIManager instantiated as singleton at module load
// DOM elements cached in this.ui
// No update until first UpdatePacket
```

### During Gameplay (40 TPS loop)
```typescript
// Every tick:
Game.processUpdate(packet: UpdatePacket)
  → Game.processUpdate.playerData()
    → UIManager.updateUI(playerData)       // sync DOM updates with game state
    → (all bars, slots, counts updated)
    → Game.render()                        // separate render after UI
```

### On Kill Event
```typescript
Game.processUpdate(killPacket)
  → UIManager.processKillPacket(killData)
    → killfeed DOM mutation
    → if (this player killed): show modal
    → 7s or 3s auto-fade-out
```

### On Game End
```typescript
Game.processUpdate(gameOverPacket)
  → UIManager.showGameOverScreen(packet)
    → render player cards, rank, medals
    → fade-in animation after 500ms
    → trigger victory music
```

### On Match Reset
```typescript
Game.reset()
  → UIManager.reset()
    → clear all caches
    → hide overlays
    → prepare for next game
```

---

## Performance Considerations

### DOM Caching
The `ui` object caches jQuery selectors for 50+ elements. This eliminates repeated DOM queries and is the primary performance optimization.

```typescript
readonly ui = Object.freeze({
    healthBar: $<HTMLDivElement>("#health-bar"),
    weaponsContainer: $("#weapons-container"),
    killFeed: $("#kill-feed"),
    // ... 47 more elements
});
```

### Dirty-Flag Optimization
The `weaponCache` tracks last-rendered weapon slot state and only updates DOM if state changed:

```typescript
if (idString !== cache.idString) {
    // only update DOM if weapon id changed
    cache.idString = idString;
    // ... render new weapon
}
```

### Killfeed Cleanup
Max 5 simultaneous killfeed entries enforced:

```typescript
while (this.ui.killFeed.children().length > 5) {
    this.ui.killFeed.children().last().remove();
}
```

### Teammate Indicator Pooling
TeammateIndicatorUI instances are reused between updates via `_teammateDataCache`:

```typescript
const cacheEntry = _teammateDataCache.get(id);
if (cacheEntry !== undefined) {
    cacheEntry.update(newData);   // reuse, just update
    return;
}
const ele = new TeammateIndicatorUI({...});  // new if not cached
_teammateDataCache.set(id, ele);
```

---

## Console Variables (CVars)

UIManager respects these in-game console variables (see `client/src/console/variables.ts`):

| CVar | Type | Effect |
|------|------|--------|
| `cv_anonymize_player_names` | boolean | Hides real player names, shows `DefaultName_ID` format |
| `cv_killfeed_style` | "text" / "icon" | Killfeed display mode (text descriptions vs weapon icons) |
| `cv_weapon_slot_style` | "colored" / "default" | Colored weapon slots by ammo type |
| `cv_language` | string | Localizes all UI text |

---

## Known Issues & Gotchas

### 1. Killfeed Scroll Overflow
**Issue:** If many kills happen in quick succession (>5), oldest entries are cut off. No scroll bar.
**Workaround:** Max 5 entries enforced; rapid kills fade out older ones instead of accumulating.
**File:** [uiManager.ts:1318](../../../client/src/scripts/managers/uiManager.ts#L1318)

### 2. Teammate Indicator Disconnection Lag
**Issue:** If a teammate disconnects, their indicator may persist briefly until next UpdatePacket.
**Root cause:** Teammate data comes from server in UpdatePacket; lag means cache not cleared immediately.
**Mitigation:** Manual `clearTeammateCache()` called on player ID change.

### 3. Mobile Responsive Gaps
**Issue:** Some HUD elements not optimized for mobile (emote button, ping toggle).
**Status:** Toggle visibility based on `InputManager.isMobile`, but layout not fully responsive.
**File:** [uiManager.ts:597](../../../client/src/scripts/managers/uiManager.ts#L597)

### 4. Action Timer Cancel Popup
**Issue:** Reload bar disappears if movement interrupts reload, but no "cancel" message shown unless `fake = false`.
**Why:** Some actions (revives) shouldn't show cancel prompt to reduce visual clutter.
**File:** [uiManager.ts:396-403](../../../client/src/scripts/managers/uiManager.ts#L396)

### 5. Kill Modal Persistence
**Issue:** If player spams quick kills, "Kill" modals may overlap or 3s timeout resets each kill.
**Current behavior:** Latest kill message takes priority, timer refetches every kill inside `fadeIn` callback.
**File:** [uiManager.ts:1813-1820](../../../client/src/scripts/managers/uiManager.ts#L1813)

### 6. Perk Duration Timers
**Issue:** Perk duration display commented out (lines 877-923). Timer loop disabled.
**Status:** Perk timers not yet implemented for display; perks show icon but no duration countdown.
**Future work:** Uncomment perk timer section if duration display needed.

---

## Related Documents

### Tier 1
- [System Architecture](../../architecture.md) — Component map and process model
- [Data Model](../../datamodel.md) — PlayerData, KillData packet schema

### Tier 2 — Subsystems
- [Client Rendering](../client-rendering/) — DOM structure, PixiJS integration, layer management
- [Networking](../networking/) — Packet definitions, binary encoding, WebSocket transport
- [Inventory](../inventory/) — Item definitions, gun properties, perk mechanics
- [Input Management](../input-management/) — Input handling, mobile vs desktop, hotkey display
- [Game Loop](../game-loop/) — Game tick 40 TPS, ProcessUpdate flow

### Tier 3 — Modules
- [UI Management — Killfeed Module](modules/killfeed.md) → Killfeed rendering, animations, localization
- [UI Management — Teammate Indicators Module](modules/teammates.md) → TeammateIndicatorUI class, health bar rendering
- [UI Management — Action Timer Module](modules/action-timer.md) → SVG progress circle animation

### Patterns
- [UI Management — Patterns](patterns.md) → DOM element caching, dirty-flag optimization, jQuery patterns
