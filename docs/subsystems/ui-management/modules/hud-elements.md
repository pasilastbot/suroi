# HUD Elements & State Synchronization

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/ui-management/README.md -->
<!-- @source: client/src/scripts/managers/uiManager.ts, client/src/scripts/ui.ts -->

## Purpose

HUD Elements implements the player-facing status displays within Suroi: health/shield/adrenaline bars, inventory counters, killfeed messages, minimap, teammate indicators, action timers, and status effect visualizations. All HUD elements update **passively** by receiving game state from `UpdatePacket.playerData` (40 TPS). No user input triggers these displays directly; they reflect server authority.

---

## HUD Architecture — Layer Structure & Rendering

### DOM Organization

All HUD elements are mounted in `client/index.html` as static DOM tree. UIManager **does not create new DOM elements** (except jQuery-wrapped collections and temporary killfeed items); it caches jQuery selectors at load time and updates their properties:

```
#game-ui (root container)
├── #health-bar-container
│   ├── #health-bar (animated width)
│   ├── #health-bar-amount (text: current HP)
│   ├── #health-bar-animation (damage indicator)
│   └── #health-bar-max (text: max HP)
├── #adrenaline-bar-container
│   ├── #adrenaline-bar (animated width)
│   ├── #adrenaline-bar-min-wrapper (visual minimum threshold)
│   └── #adrenaline-bar-amount (text: current adrenaline)
├── #shield-bar (clip-path inset)
├── #infection-bar (clip-path inset)
├── #weapons-container
│   ├── #weapon-slot-1 through #weapon-slot-4
│   │   └── .main-container
│   │       ├── .item-image (gun sprite, fist with skin tint)
│   │       ├── .item-name (localized gun name)
│   │       ├── .item-ammo (magazine rounds)
│   │       └── .active-indicator (bar fill for ammo heat)
│   └── #weapon-ammo-container
│       ├── #weapon-clip-ammo (rounds in mag)
│       ├── #weapon-clip-ammo-count (count)
│       ├── #weapon-inventory-ammo (reserve)
│       └── #weapon-clip-reload-icon
├── #ammos-container (consumable ammo icons)
├── #team-container (dynamically populated TeammateIndicatorUI)
├── #kill-feed (prepend kills, max 5 visible)
├── #kill-msg (modal, high-priority kill message)
├── #killfeed-kill-leader-indicator
├── #kill-leader-leader (name)
├── #kill-leader-count (count)
├── #action-container (reload, medical, revive timer)
│   ├── #action-name (text: "Reloading", "Using Medical Kit", etc.)
│   ├── #action-time (text: remaining seconds, e.g., "2.3")
│   └── #action-timer-anim (SVG progress circle)
├── #spectating-container (visible when dead or spectating)
│   ├── #spectating-msg-player (spectated player name + badge)
│   ├── #btn-spectate-previous
│   ├── #btn-spectate-next
│   ├── #btn-spectate-options-icon (toggle UI, quit spectating)
│   └── #btn-spectate-replay
├── #perk-slot-0 through #perk-slot-2
│   ├── .item-image (perk icon)
│   ├── .item-tooltip (hover: perk name + duration)
│   └── .item-timer (animation fill)
├── #interact-message (lootable item, downed teammate revive prompt)
│   ├── #interact-key (key binding)
│   └── #interact-text (action description)
├── #game-over-overlay (fade-in on death/win)
│   ├── #game-over-text (message: "You Won!" or "Player X Eliminated You")
│   ├── #game-over-rank (position: "#1" or "#42")
│   ├── #chicken-dinner (golden chicken image on victory)
│   ├── #player-game-over-cards (team card list)
│   └── #game-over-team-kills-container (team total kills, team mode only)
├── #ui-players-alive (player count display)
├── #gas-and-debug (debug coordinates, gas warnings)
└── #c4-detonate-btn (conditionally shown if has C4 items)
```

**Z-Ordering:** All HUD divs are in front of the PixiJS canvas via CSS `z-index`. Teammate health bars (see [Game Objects — Client Rendering](#cross-tier-references)) are rendered **on-canvas** using PixiJS `Graphics`, positioned at player world coordinates.

### jQuery Element Caching

UIManager caches all frequently-accessed DOM elements as readonly jQuery objects in `this.ui` (struct at [line ~165 of uiManager.ts](../../../client/src/scripts/managers/uiManager.ts#L165)):

```typescript
readonly ui = Object.freeze({
    healthBar: $<HTMLDivElement>("#health-bar"),
    healthBarAmount: $<HTMLSpanElement>("#health-bar-amount"),
    healthAnim: $<HTMLDivElement>("#health-bar-animation"),
    adrenalineBar: $<HTMLDivElement>("#adrenaline-bar"),
    adrenalineBarAmount: $<HTMLSpanElement>("#adrenaline-bar-amount"),
    shieldBar: $<HTMLDivElement>("#shield-bar"),
    infectionBar: $<HTMLDivElement>("#infection-bar"),
    weaponsContainer: $<HTMLDivElement>("#weapons-container"),
    ammoCounterContainer: $<HTMLDivElement>("#weapon-ammo-container"),
    killFeed: $<HTMLDivElement>("#kill-feed"),
    // ... ~60 more caches
});
```

**Rationale:** jQuery selector `$()` is O(n) DOM traversal. Caching at init avoids repeated lookups during 40-TPS updates. Object is `Object.freeze()`d to prevent accidental mutation of the cache structure.

---

## Health Bar — Current Health Display, Shield, Injury State

### Structure

```
#health-bar-container
├── #health-bar (primary bar, animated width & color)
├── #health-bar-animation (damage "drain" animation)
├── #health-bar-max (text, shown only if maxHealth > defaultHealth)
└── #health-bar-amount (text, HP value)
```

### Data Flow: Server → Client

1. **Serialization** — Server: `player.health` (normalized 0-1, see [Serialization System](../../serialization-system/)) → binary byte in `UpdatePacket.playerData.health`
2. **Deserialization** — Client: `SuroiByteStream.readUint16()` → remapped from [0, 1] to [0, maxHealth]
3. **Update** — `UIManager.updateUI(playerData)` receives `playerData.health`
4. **Render** — DOM width & color updated synchronously

### Health Thresholds & Colors

```typescript
getHealthColor(normalizedHealth: number, downed?: boolean): string {
    switch (true) {
        case normalizedHealth <= 0.25:
        case downed:
            return "#ff0000"; // Critical (red)
        case normalizedHealth > 0.25 && normalizedHealth < 0.6:
            return `rgb(255, ${((normalizedHealth * 100) - 10) * 4}, ${...})`; // Hurt (gradient yellow-orange)
        case normalizedHealth === 1:
            return "#bdc7d0"; // Healthy (pale gray)
        default:
            return "#f8f9fa"; // Normal (off-white)
    }
}
```

**Visual States:**
- **Healthy** (`>60%`): Pale gray bar, black text
- **Injured** (`25-60%`): Yellow-to-orange gradient bar, white text
- **Critical** (`≤25%` **or** downed): Red bar, white text
- **Flashing** (≤25%): CSS `flashing` class applied for pulsing effect

### Damage Animation

When damage taken ≥1 HP:
1. Calculate old health percentage vs. new health percentage
2. If delta ≥ 1%, animate secondary bar (`#health-bar-animation`) from old% to new% over **500ms** using jQuery `.animate({ width })` 
3. This creates visual "drain" effect showing damage just absorbed

```typescript
if (this._oldHealthPercent - healthPercent >= 1) {
    this.ui.healthAnim
        .width(`${this._oldHealthPercent}%`)
        .animate({ width: `${healthPercent}%` }, 500); // 500ms linear
}
this._oldHealthPercent = healthPercent;
```

### Maximum Health Display

Shown only when player has bonus max health (beyond `GameConstants.player.defaultHealth`):
- Health bar inherits width from max health via parent container
- Text `#health-bar-max` displays the increased maximum (e.g., "150" if boosted by armor/perks)
- Text hidden/shown via `.hide()` / `.show()` based on `minMax.maxHealth` from server

---

## Inventory Display — Ammo Counter, Weapon Slots, Equip Indicator

### Weapon Slots (1-4)

**Structure per slot** (#weapon-slot-{i}, i ∈ [1, 4]):

```html
<div id="weapon-slot-N" class="weapon-slot">
  <div class="main-container">
    <div class="item-image"></div>
    <span class="item-name">Gun Name</span>
    <span class="item-ammo">12</span> <!-- mag rounds or total for throwable -->
  </div>
</div>
```

**CSS Classes (toggled dynamically):**
- `.active` — applied to slot matching `activeWeaponIndex` in inventory
- `.has-item` — visible only if weapon defined at this slot index
- `.locked` — darkened/disabled if slot locked due to inventory capacity
- `.colored` — (optional, if `cv_weapon_slot_style === "colored"`) uses ammo color as background

**Update Method** — `updateWeaponSlots(force = false)`: called every tick if weapons changed

```typescript
for (let i = 0; i < GameConstants.player.maxWeapons; i++) {
    const weapon = inventory.weapons[i];
    const { container, image, ammo, name } = this._getSlotUI(i + 1);
    
    // Check if anything changed (cache miss → need re-render)
    const cached = this.weaponCache[i] ??= {};
    
    // Update DOM only if dirty
    if (weapon?.definition?.idString !== cached.idString) {
        // Load weapon sprite into .item-image
        // Set weapon name localized text
        // For fists: apply skin tint filter
        // For guns: apply ammo-color outline (if colored mode)
    }
    
    if (weapon?.count !== cached.ammo) {
        ammo.text(weapon?.count ?? ""); // magazine rounds or total count
        ammo.css("color", weapon?.count > 0 ? "inherit" : "red"); // red if empty
    }
}
```

**Weapon Properties Rendered:**
- **Image**: Gun sprite from `./img/game/[modeName|shared]/weapons/[idString].svg`
- **Name**: Localized from `translationString` or `idString` (e.g., "M4A1", "Dual MP5")
- **Ammo Count**: Magazine rounds (loaded) + "Reserve" (inventory pool) shown separately for guns
- **Fist Tint**: If equipped, load `SkinDefinition.fistImage` and apply `SkinDefinition.fistTint` or grass tint via `mask-image` filter

### Ammo Counter (Active Weapon)

**Structure** (#weapon-ammo-container):

```html
<div id="weapon-ammo-container">
  <span id="weapon-clip-ammo">12</span>  <!-- magazine rounds -->
  <span id="weapon-clip-ammo-count">/30</span>  <!-- reserve in inventory -->
  <div id="weapon-inventory-ammo">120</div>  <!-- total reserve pool -->
</div>
```

**Behavior:**
- **Visible only** if active weapon is defined and has count
- **Red text** if magazine/reserve is 0 (empty)
- **Reserve label** toggles based on gun type:
  - **Guns**: Show `$inventory.items[ammoType]` (total reserve ammo)
  - **Throwables**: Show "∞" or actual count (no reserve system)
  - **Melee**: Hidden (no ammo)
- **Ephemeral ammo** (e.g., "Fists" punch rounds): Always shows "∞"

**Server → Client Sync:**
- `UpdatePacket.playerData.inventory.weapons[activeIndex].count` = magazine rounds (max `gunDef.magazineSize`)
- `UpdatePacket.playerData.items[ammoType]` = reserve pool in backpack

### Slot Locks

When inventory slots exceed capacity (e.g., backpack too small), locked slots show `.locked` class (darkened):

```typescript
updateSlotLocks(): void {
    for (let i = 0; i < GameConstants.player.maxWeapons; i++) {
        const container = this._getSlotUI(i + 1).container;
        const isLocked = !!(this.inventory.lockedSlots & (1 << i)); // bitfield
        container.toggleClass("locked", isLocked);
    }
}
```

---

## Minimap — Player Position, Teammates, Enemies, Gas Zone

**Note:** Minimap is managed by [MapManager](../../map/), not UIManager. See [Map Subsystem — Minimap Module](../../map/modules/minimap.md) for rendering details.

**UIManager Role:**
- Caches `#game-ui` visibility toggle
- Hides killfeed during big map view via `ui.killFeed.hide()`
- Shows killfeed on mini-map view via `ui.killFeed.show()`
- Passes team data to MapManager for teammate indicator sprites

---

## Scope Display — Crosshair Types, Zoom Level, Rangefinding

### Crosshair System

Crosshair rendering is split between **HUD (DOM) crosshair** for scoped view and **in-game (PixiJS) crosshair** for unscoped aiming. This module covers DOM crosshair only.

**Files:**
- `client/src/utils/crosshairs.ts` — `Crosshairs` enum, `getCrosshair(gunDef)` utility
- `client/src/scripts/managers/cameraManager.ts` — Zoom level, scope switching
- `client/scripts/game.ts` — Crosshair div updates in render loop

**DOM Structure:**

```html
<div id="crosshair" class="crosshair [crosshair-type]">
  <!-- SVG or IMG based on scope type -->
  <!-- SVG for scoped (circle reticle), default SVG for unscoped (simple cross) -->
</div>
```

**Crosshair Types** (enum `Crosshairs`):

| Type | Used By | Appearance |
|------|---------|-----------|
| `Normal` | Unscoped guns | Simple crosshair (+) |
| `Pistol` | Semiauto pistols | Larger crosshair with spread indicator on fire |
| `Rifle` | Sniper, DMR | Scoped circle with windage/elevation marks |
| `Shotgun` | Pump/auto shotguns | Burst spread pattern |
| `Flamethrower` | Incendiary weapons | Cone pattern |
| `Melee` | Fists, sword | Simple melee strike indicator |
| `Grenade` | Throwables | Arc trajectory preview |

**Loading Crosshair Sprite:**

```typescript
const crosshairSrc = `./img/game/${game.modeName}/crosshairs/${gunDef.idString}.svg`;
const crosshairImg = new Image();
crosshairImg.src = crosshairSrc;
$('#crosshair').html(crosshairImg);
```

**Zoom Level Indicator** (scope-specific):
- **Thermal scope**: Shows "4x" or "6x" overlay in corner
- **Night vision**: Shows green tint + NV-specific reticle
- **Standard scope**: Shows magnification text below/beside crosshair

**Server Sync:**
- Crosshair type determined **client-side** from active gun definition (no server message)
- Zoom level: `UpdatePacket.playerData.zoom` (normalized 0-1) → scales crosshair size

---

## Killfeed — Recent Kills, Teammate Kills, Kill Messages, Fade-Out Timing

### DOM Structure

**Container: `#kill-feed`**

```html
<div id="kill-feed">
  <div class="kill-feed-item">
    <img class="kill-icon" src="./img/misc/skull_icon.svg" alt="Kill" />
    <img class="badge-icon" ... />
    attacker_name
    <img class="kill-icon" src="./img/killfeed/gun_killfeed.svg" alt="M4A1" />
    victim_name
  </div>
  <!-- max 5 items visible, prepend new kills, auto-remove oldest -->
</div>
```

**CSS Rules:**
- `-webkit-user-select: none` (prevent text selection)
- Max height constrains visible items to ~5 entries
- Newest items appear at top (prepended via jQuery)
- Oldest items fade out and are removed

### Update Flow: Kill Packet → DOM

**1. Server sends KillPacket:**

```typescript
const killPacket: KillData = {
    victimId: 42,
    attackerId: 7,
    creditedId: 7,
    damageSource: DamageSources.Gun,
    weaponUsed: gunDef,
    kills: 3,        // attacker's kill count (for streak display)
    downed: false,   // victim is downed (not killed)
    killed: true,    // final elimination
    killstreak: "🔥" // optional streak emoji
};
```

**2. Client deserializes KillPacket:**

```
→ Game.processKill(packet)
→ UIManager.processKillPacket(killData)
```

**3. UIManager builds killfeed HTML:**

Based on **damage source** and **CVar `cv_killfeed_style`** (icon | text):

**Case: "text" style (default)**

```html
<!-- Example: normal gun kill -->
<div class="kill-feed-item">
  🔥                                                  <!-- killstreak emoji (if any) -->
  <img src="./img/misc/skull_icon.svg" />           <!-- kill icon -->
  player_foo <img class="badge-icon" ... />         <!-- attacker + badge -->
  "killed"                                            <!-- localized verb -->
  player_bar                                          <!-- victim -->
  "with"                                              <!-- preposition -->
  <img src="./img/killfeed/m4a1_killfeed.svg" />    <!-- weapon icon -->
</div>
```

**Case: "icon" style (compact)**

```html
<div class="kill-feed-item">
  <img src="./img/killfeed/m4a1_killfeed.svg" />
  attacker_name
  <span style="font-size: 80%">
    (3 <img src="./img/misc/skull_icon.svg" height=12 />)  <!-- streak count -->
  </span>
  victim_name
</div>
```

**Special Damage Sources** — Killfeed message varies:

| Source | Message | Icon |
|--------|---------|------|
| `Gun` | "attacker killed victim with gun" | gun sprite |
| `Melee` | "attacker beat victim" | fists |
| `Throwable` (grenade) | "attacker impact-killed victim" | grenade |
| `Explosion` | "attacker explosion-killed victim" | explosion icon |
| `Gas` | "victim died in gas" | gas/cloud |
| `Obstacle` | "victim hit by object" | obstacle sprite |
| `BleedOut` | "victim bled to death" | bandage |
| `FinallyKilled` | "attacker finished victim" | trophy/crown |
| `Disconnect` | "victim disconnected" | X icon |

**Downed vs. Killed** — Two kill types shown:

- **Downed**: Victim loses all HP but not killed → icon is `downed.svg`
- **Killed**: Final elimination (no revive possible) → icon is `skull_icon.svg`

### Animation Sequence

**Entry Animation** (300ms):

```typescript
killFeedItem.css("opacity", 0);
killFeedItem.animate(
    [
        { opacity: 0 },
        { opacity: 1 }
    ],
    { duration: 300, easing: "ease-in" }
);
```

**Position Shift** (for existing items below new kill):

```typescript
const others = this._getKillFeedElements();
others.forEach(otherItem => {
    const oldPos = otherItem.position;
    const newPos = otherItem.element.getBoundingClientRect();
    if (oldPos.y !== newPos.y) {
        // Animate existing item downward
        otherItem.element.animate(
            [{ transform: `translateY(${oldPos.y - newPos.y}px)` }, { transform: "translateY(0px)" }],
            { duration: 300, easing: "ease-in" }
        );
    }
});
```

**Fade-Out & Removal** (7000ms lifetime):

```typescript
setTimeout(() => {
    killFeedItem.animate(
        [
            { opacity: 1, transform: "translateX(0%)" },
            { opacity: 0, transform: "translateY(100%)" }
        ],
        { duration: 300, fill: "backwards", easing: "ease-out" }
    );
    // onfinish callback removes DOM element
}, 7000); // 7 second display lifetime
```

**Max Items** — Only 5 kills visible at once:

```typescript
while (this.ui.killFeed.children().length > 5) {
    this.ui.killFeed.children().last().remove();
}
```

### Kill Modal (High-Priority Message)

When **active player gets kill credit**, show larger modal message above killfeed:

**Condition:**

```typescript
const gotKillCredit = creditedId !== undefined 
    ? activeId === creditedId 
    : activeId === attackerId;
```

**Modal Display** (#kill-msg):

```html
<div id="kill-msg" class="kill-msg-modal">
  <div id="kill-msg-kills">
    <span id="ui-kills">1</span> Kill
  </div>
  <div id="kill-msg-cont">
    <!-- Killfeed entry HTML repeated here -->
  </div>
</div>
```

**Behavior:**
- **Animates in** via `.fadeIn()`
- **Persists** for 3 seconds or until next kill message
- **Auto-hides** via `.fadeOut(3000)` if no new kills
- **Resets counter** on each kill credit

---

## Map Indicators — Named Locations, Pinged Locations, Current Zone

Managed by **MapManager**. See [Map Subsystem — Map Indicators Module](../../map/modules/map-indicators.md).

**UIManager integration:**
- Caches `#player-name` DOM for displaying current location name
- Updates location name when player enters named POI via `playerData.currentLocation` (if sent by server)
- (Currently: location name not sent in UpdatePacket — derived client-side from map data)

---

## Status Indicators — Perks Active, Status Effects, Down State

### Perk Display

**DOM Slots** (#perk-slot-0, #perk-slot-1, #perk-slot-2):

```html
<div id="perk-slot-0" class="perk-slot" data-idstring="perk_id">
  <img class="item-image" src="./img/game/shared/perks/perk_name.svg" />
  <div class="item-tooltip">
    <strong>Perk Name</strong><br>
    Perk description with duration (e.g., "Lasts 45 seconds")
  </div>
  <div class="item-timer"></div>  <!-- animated fill bar (if timed) -->
</div>
```

**Update Flow** — `UpdatePacket.playerData.perks[]`:

```typescript
updateUI(data: PlayerData) {
    if (perks) {
        for (let i = 0; i < GameConstants.player.maxPerks; i++) {
            const newPerk = perks[i];
            if (newPerk !== oldPerks[i]) {
                if (newPerk === undefined) {
                    this.resetPerkSlot(i); // hide slot
                } else {
                    this.updatePerkSlot(newPerk, i); // show + animate
                }
            }
        }
    }
}
```

**Reset Slot** — `resetPerkSlot(index)`:

```typescript
resetPerkSlot(index: number): void {
    const container = $(`#perk-slot-${index}`);
    container.children(".item-tooltip").html("");
    container.children(".item-image").attr("src", "");
    container.css("visibility", "hidden");
    container.off("pointerdown");
}
```

**Update Slot** — `updatePerkSlot(perkDef, index)`:

1. **Load perk icon** from `./img/game/[folder]/perks/[idString].svg`
2. **Set tooltip** with perk name + duration (seconds or update interval)
3. **Apply animation** — perk glows with color based on rarity (`perk-[quality]-colors` animation)
   - Quality: "normal", "rare", "epic", "mythic" (custom CSS animations)
4. **Stop animation after 3 seconds** (perk acquired flash effect)
5. **Show/hide visibility** based on `PerkManager.has(perkDef)` (confirmation perk is active)

**Perk Duration Sources:**

```typescript
let _duration: number;
switch (true) {
    case "duration" in perkDef: _duration = perkDef.duration;           break;
    case "shieldRespawnTime" in perkDef: _duration = perkDef.shieldRespawnTime; break;
    case "highlightDuration" in perkDef: _duration = perkDef.highlightDuration; break;
    case "cooldown" in perkDef: _duration = perkDef.cooldown;           break;
    default: _duration = perkDef.updateInterval ?? 0;
}
```

**Inventory Message** — When perk acquired:

```html
<div id="inventory-message">
  <div id="perk" style="background-image: url(./img/game/shared/loot/loot_background_perk.svg);">
    <img class="perk-img" src="./img/game/shared/perks/perk_id.svg" />
  </div>
  <strong class="perk-name">Perk Name</strong>
</div>
```

Fades in/out over `PERK_MESSAGE_FADE_TIME` (default: 500ms).

### Adrenaline Bar — Energy/Stimulation State

Separate from health bar (see [Health Bar section](#health-bar--current-health-display-shield-injury-state)). Visually distinct:
- **Color**: Always white/light (not damage-state-based)
- **Minimum Threshold**: Can be set by perks/equipment
- **Min Bar** (`#adrenaline-bar-min-wrapper`, `#adrenaline-bar-min`): Shows minimum threshold visually

**Update:**

```typescript
if (hasMinMax) {
    // Set min threshold width
    ui.minAdrenBarWrapper.outerWidth(
        `${100 * this.minAdrenaline / this.maxAdrenaline}%`
    ).show();
    // Set min label text
    ui.minMaxAdren.text(`${safeRound(this.minAdrenaline)}/${safeRound(this.maxAdrenaline)}`);
}

if (hasAdrenaline || hasMinMax) {
    const percent = 100 * this.adrenaline / this.maxAdrenaline;
    ui.adrenalineBar.width(`${percent}%`);
    ui.adrenalineBarAmount.text(safeRound(this.adrenaline));
    // Text color: red if < 7, white otherwise
    ui.adrenalineBarAmount.css("color", this.adrenaline < 7 ? "#ffffff" : "#000000");
}
```

### Downed State Indicator

When player is downed (health = 0, but not killed):

**Visual Changes:**
- **Health bar turns red** (`#ff0000`) — `getHealthColor(..., downed: true)`
- **Health bar flashes** — CSS `flashing` class applied
- **Cannot perform actions** — `blockEmoting = true` prevents emote spam
- **Revive timer visible** (see Action Timer section)
- **Cannot shoot** — active weapon disabled

**Spectating Indicator** — When dead:

```html
<div id="spectating-container">
  <div id="spectating-msg">
    Spectating <span id="spectating-msg-player">player_name</span>
    <img class="badge-icon" ... />
  </div>
  <button id="btn-spectate-previous">← Previous</button>
  <button id="btn-spectate-next">Next →</button>
</div>
```

---

## Game Phase HUD — Game Code, Player Count, Revive Timer, Last Alive Indicator

### Player Count Display

**DOM: `#ui-players-alive`**

```html
<div id="ui-players-alive">
  <img src="./img/misc/player_icon.svg" />
  <span id="players-alive-count">42</span> Players
</div>
```

**Server Sync:** `UpdatePacket.playerCount` (sent periodically as players die/disconnect)

### Game Code Display

**During pre-game phase:**

```html
<div id="game-code-display">
  LOBBY CODE: <span id="game-code-text">ABC123</span>
</div>
```

Shown only in team-mode pre-game. Allows players to share lobby code with teammates.

### Revive Timer

When teammate downed during spectating:

```html
<div id="action-container" class="revive-timer">
  <div id="action-name">Reviving teammate_name</div>
  <div id="action-time">45.2</div> <!-- countdown -->
  <svg id="action-timer-anim" ...>
    <!-- circular progress indicator -->
  </svg>
</div>
```

Driven by `animateAction(name, duration, fake)`:

```typescript
animateAction("Reviving John", 45.0, fake = true): void {
    this.action.fake = true; // don't show "cancel" popup
    this.action.start = Date.now();
    this.action.time = 45.0;
    ui.actionContainer.show();
    ui.actionName.text("Reviving John");
    ui.actionTimer.animate({ "stroke-dashoffset": "0" }, 45000, "linear", () => {
        ui.actionContainer.hide();
    });
}
```

**No-Cancel Revive** — `fake = true` flag prevents "Press [key] to cancel" prompt (only real player actions show cancel).

### Kill Leader Indicator

When player becomes kill leader (or kill count changes):

```html
<div id="killfeed-kill-leader-indicator">
  <img src="./img/misc/crown.svg" />
  <span id="kill-leader-leader">player_name</span>
  <span id="kill-leader-count"> (12 kills)</span>
  <button id="btn-spectate-kill-leader">Spectate</button>
</div>
```

**Update Method** — `updateKillLeader(killLeaderData)`:

```typescript
ui.killLeaderLeader.text(killLeaderName);
ui.killLeaderCount.text(`(${killCount} kills)`);
ui.spectateKillLeader.on("click", () => {
    Game.spectatePlayer(killLeaderID);
});
```

---

## Sync with Server — Health, Ammo, Status Effect Updates via UpdatePacket

### UpdatePacket → PlayerData Flow

**Binary format:** `common/src/packets/updatePacket.ts`

**Key fields synced to HUD:**

| PlayerData Field | HUD Element | Update Frequency |
|------------------|-------------|-----------------|
| `health` | `#health-bar`, `#health-bar-amount` | Every tick if changed |
| `adrenaline` | `#adrenaline-bar`, `#adrenaline-bar-amount` | Every tick if changed |
| `shield` | `#shield-bar` | Every tick if changed |
| `infection` | `#infection-bar` | Every tick if changed |
| `zoom` | Crosshair scale, camera zoom | Every tick if changed |
| `inventory.weapons[]` | Weapon slots 1-4, ammo counter | When weapon equipped/swapped |
| `inventory.activeWeaponIndex` | Active weapon slot highlight | When gun switched |
| `items.items` | Consumable item counts, icons | When item used/picked up |
| `items.scope` | Active scope indicator | When scope equipped |
| `minMax.maxHealth` | Health bar max label | On game start or health boost |
| `minMax.minAdrenaline` | Adrenaline min threshold bar | When equipment changes |
| `minMax.maxAdrenaline` | Adrenaline max label | When equipment changes |
| `teammates[]` | Teammate indicators, health bars | Every tick if team mode |
| `highlightedPlayers[]` | Teammate health bar colors | When teammate health changes |
| `perks[]` | Perk slots 0-2 | When perk acquired/spent |
| `blockEmoting` | Emote button disabled state | When emote/action blocked |
| `activeC4s` | C4 detonation button | When C4 acquired/used |
| `id.spectating` | Spectating UI visibility | When player dies |
| `lockedSlots` | Weapon slot `.locked` class | When inventory fills |

### Network Optimization — Delta Compression

**UpdatePacket uses bitfield flags (`UpdateFlags`) to compress only-changed fields:**

```typescript
enum UpdateFlags {
    PlayerData = 1 << 0,  // if set, player stats follow in stream
    DeletedObjects = 1 << 1,
    Explosions = 1 << 2,
    // ...
}
```

When only health changed (not inventory), `UpdateFlags.PlayerData` set, then only 2 bytes for `health` sent in packet (not full 100-byte PlayerData struct).

**Client-side deserialization** only populates PlayerData fields present in binary stream:

```typescript
if ((flags & UpdateFlags.PlayerData) !== 0) {
    playerData = deserializePlayerData(stream); // Selective field deserialization
}
```

### Ping Tracking in UIManager

**Latency measurement via echo sequence numbers:**

```typescript
updateUI(data: PlayerData): void {
    const sentTime = Game.seqsSent[data.pingSeq];
    if (sentTime !== undefined) {
        const ping = Date.now() - sentTime;
        Game.netGraph.ping.addEntry(ping);
        Game.seqsSent[data.pingSeq] = undefined;
    }
}
```

Maintains **rolling window of last 60 ping samples** for network graph display.

---

## Performance — DOM Update Batching, Dirty Flag Updates, Animation Optimization

### Caching Strategy

All DOM element references cached at UIManager init time (not during updates):

**Cost:** O(1) per update, no DOM traversal
**Tradeoff:** Must manually update all caches if DOM changes (rare after load)

### Dirty Flag Pattern — Only Update Changed Fields

```typescript
const weaponCache = [];
updateWeaponSlots(force = false): void {
    for (let i = 0; i < max; i++) {
        const weapon = inventory.weapons[i];
        const cache = weaponCache[i] ??= {};
        
        // Only write DOM if different from cached value
        if (weapon?.definition?.idString !== cache.idString || force) {
            this._getSlotUI(i + 1).image.css("background-image", ...);
            cache.idString = weapon?.definition?.idString;
        }
    }
}
```

**Benefits:**
- **No re-paints** for unchanged fields (jQuery selector + CSS write only if needed)
- **Conditional animations** — don't animate if value unchanged
- **Lower CPU** — avoids O(n) DOM updates when only one weapon changed

### Animation Performance

**jQuery `.animate()` for smooth transitions** (not CSS transitions):

```typescript
this.ui.healthAnim.animate(
    { width: `${healthPercent}%` },
    500,  // 500ms duration
    "linear"  // easing function
);
```

**PixiJS Web Animations API** for killfeed entries (native browser optimized):

```typescript
killFeedItem.animate(
    [
        { opacity: 0 },
        { opacity: 1 }
    ],
    { duration: 300, iterations: 1, easing: "ease-in" }
);
```

Both use **browser's compositor thread** (GPU acceleration if available), not JavaScript frame loop.

### Batch DOM Updates — updateUI() Pattern

All HUD updates consolidated into single call `updateUI(playerData)` (40 TPS = 25ms budget):

```
Game.processUpdate(packet)           // WebSocket message received
  ↓
Game.processPacketData(playerData)   // Deserialize bytes
  ↓
UIManager.updateUI(playerData)       // All HUD updates here (< 5ms typical)
  ├─ Health bar color + width
  ├─ Adrenaline bar width
  ├─ Weapon slots (only if weapons changed)
  ├─ Ammo counter
  ├─ Teammates health bars
  └─ Perks
  ↓
Game.render()                        // PixiJS render pass
```

Single call = single DOM batch (browser optimizer combines all CSS writes).

### Query Selector Performance

**Avoided:** Repeated `$()` or `document.querySelector()` in loops:

```typescript
// BAD: O(n) DOM traversal per weapon
for (let i = 0; i < 4; i++) {
    $(`#weapon-slot-${i}`).css(...); // Selector search every iteration
}

// GOOD: O(1) cached lookup
this._getSlotUI(i).container.css(...); // Map.get() with memoization
```

**Memoization via `ExtendedMap`:**

```typescript
private readonly _weaponSlotCache = new ExtendedMap<
    number,
    { readonly container: JQuery; readonly image: JQuery; ... }
>();

private _getSlotUI(slot: number) {
    return this._weaponSlotCache.getAndGetDefaultIfAbsent(
        slot,
        () => {
            const container = $(`#weapon-slot-${slot}`);
            return { container, image: container.find(".item-image"), ... };
        }
    );
}
```

---

## Known Gotchas — Stale State, Update Race Conditions, Visual Lag

### 1. **Stale Health on Teammate Bars**

**Problem:** `highlightedPlayers[]` array only sent when health changes. If teammate takes damage but server doesn't send update (due to packet loss), client's teammate health bar doesn't update until next change.

**Mitigation:** Server re-sends full `highlightedPlayers` array periodically (not just on delta).

**Manifestation:** Teammate health bar appears wrong for 1-2 ticks, then corrects.

### 2. **Killfeed Ordering Race Condition**

**Problem:** If two kills happen in same packet (`KillPacket` sent twice in one UpdatePacket), killfeed entries appear in packet order, not game-time order.

**Evidence:** Two kills with same timestamp show in wrong order visually.

**Fix:** Server strictly orders KillPackets by event time before sending.

### 3. **Ammo Counter Desynchronization**

**Problem:** Client updates ammo counter from `playerData.inventory.weapons[activeIndex].count` after shot fired. Server may have already decremented magazine. Visual contradiction: HUD shows (12) but next shot counts come from (11).

**Why:** Network latency. Client fires locally, updates HUD from old playerData, then server's correction arrives.

**Mitigation:** Client uses **optimistic update** — subtract 1 from ammo counter immediately on shot, then correct if server's count differs:

```typescript
// On local fire event
localAmmoCount--;
this.ui.activeAmmo.text(localAmmoCount);

// On UpdatePacket
if (playerData.inventory.weapons[activeWeapon].count !== localAmmoCount) {
    localAmmoCount = playerData.inventory.weapons[activeWeapon].count;
    this.ui.activeAmmo.text(localAmmoCount);
}
```

### 4. **Perk Slot Overflow**

**Problem:** `GameConstants.player.maxPerks = 3`, but if server sends 4 perks (bug), `updatePerkSlot(perk, 4)` accesses non-existent `#perk-slot-4`.

**Current Code:** Silently fails or overwrites slot 0:

```typescript
if (index > GameConstants.player.maxPerks) index = 0; // overwrites first slot!
```

**Fix:** Clamp index to valid range:

```typescript
if (index >= GameConstants.player.maxPerks) {
    console.warn(`Perk index ${index} out of bounds, ignoring`);
    return;
}
```

### 5. **Visual Lag — Client Behind Server**

**Problem:** Killfeed shows kill immediately (client-side), but new player count (from server) arrives 1-2 ticks later. Players see "3 kills" before seeing "Player Y eliminated" in killfeed.

**Root:** UIManager updates from two sources:
- **UpdatePacket.playerData** — immediate (40 TPS)
- **UpdatePacket.playerCount** — delayed (lower frequency, ~1 TPS)

**Manifestation:** Killfeed and player count counter briefly out of sync.

**Mitigation:** Server sends `playerCount` with every `UpdatePacket`, not sparse.

### 6. **Weapon Slot Animation Jank**

**Problem:** Weapon equipped → slot animation triggered (`toggleClass("active")` twice for reflow) → if next weapon switch fires within 300ms, animation interrupted.

**Visual:** Weapon slot flickers instead of smoothly pulsing.

**Fix:** Track animation state, abort old animation before starting new:

```typescript
private _playSlotAnimation(element: JQuery): void {
    element.stop(); // Abort ongoing animations
    element.toggleClass("active");
    element[0].offsetWidth; // Force reflow
    element.toggleClass("active");
}
```

### 7. **Damage Animation Skips Small Changes**

**Problem:** Damage ≥1 HP triggers animation. Poison gas ticks 0.5 HP per tick → multiple ticks aggregate before animation fires → visual stutter when large damage suddenly appears.

**Code:** `if (this._oldHealthPercent - healthPercent >= 1) { animate() }`

**Fix:** Save intermediate health values, animate per-tick:

```typescript
const healthDelta = this._oldHealthPercent - healthPercent;
if (healthDelta >= 0.5) { // Animate even small changes
    this.ui.healthAnim.animate({ width: `${healthPercent}%` }, 200);
}
```

### 8. **Killfeed HTML Injection Vulnerability**

**Problem:** Player names from `getPlayerData(id)` are HTML-ified for color/badge support:

```typescript
const playerName = element.prop("outerHTML"); // returns HTML string like "<span style='color: red'>Player Name</span>"
```

This HTML is then inserted into killfeed via `.html(killfeedHTML)`. If player name contains `<script>`, it executes.

**Safeguard:** `getPlayerData()` uses jQuery, which auto-escapes. But custom HTML construction with `html` template literal could be vulnerable.

**Fix:** Sanitize all user-controlled strings:

```typescript
const safePlayerName = $('<div>').text(playerName).html(); // escape HTML entities
```

---

## Complex Functions — Business Logic & Non-Obvious Behavior

### `updateUI(data: PlayerData)` — @file client/src/scripts/managers/uiManager.ts:700

**Purpose:** Main HUD update entry point, called 40 TPS from `Game.processUpdate()`.

**Implicit behavior:**
- **No error handling** — if `playerData` field missing, silently skips update (trusts server)
- **Teammate list diffing** — compares old teammates vs. new, creates/updates/destroys `TeammateIndicatorUI` instances using `Set.difference` pattern
- **Ping measurement** — correlates outgoing `pingSeq` with server's echo via `Game.seqsSent[]` map
- **Selective updates** — only re-renders DOM if value actually changed (dirty-flag pattern)

### `processKillPacket(data: KillData)` — @file client/src/scripts/managers/uiManager.ts:1379

**Purpose:** Render kill/down event to killfeed and modal.

**Implicit behavior:**
- **Suicide detection** — if `attackerId === undefined`, game-over message changes (e.g., "You fell to your death" vs. "X killed you")
- **Downed vs. killed** — two kill types with different icons and messages; `downed=true, killed=false` shows down-arrow icon
- **Damage source determines message** — switch statement with 8+ cases (Gun, Melee, Throwable, Explosion, Gas, Obstacle, BleedOut, FinallyKilled, Disconnect)
- **Localization** — message template from `TRANSLATIONS` object, then `.getTranslatedString(...)` with player names + weapon name substitutions
- **Kill streak display** — if `killstreak` emoji provided, prepends to message (client doesn't calculate streak, server does)
- **Language special cases** — Turkish/Estonian grammar requires manual string manipulation before display

### `updateWeaponSlots(force = false)` — @file client/src/scripts/managers/uiManager.ts:1017

**Purpose:** Render weapon slots 1-4 based on `inventory.weapons[]` array.

**Implicit behavior:**
- **Fist skin tinting** — if weapon is "fists", load skin definition and apply color filter via CSS `mask-image`
- **Colored mode** — if `cv_weapon_slot_style === "colored"`, set slot background to ammo color (HSL based on `Ammos[ammoType].characteristicColor`)
- **Dual gun naming** — if gun is dual-variant, template string uses `singleVariant` name with "Dual" prefix
- **Image transform scaling** — if `GunDefinition.inventoryScale` defined, apply CSS transform to fit slot
- **Animation trigger** — calls `_playSlotAnimation()` only if weapon changed (not every tick)

### `getHealthColor(normalizedHealth, downed)` — @file client/src/scripts/managers/uiManager.ts:116

**Purpose:** Derive bar color from health percentage and state.

**Implicit behavior:**
- **Downed overrides color** — even if health > 25%, shows red if `downed=true` (visual feedback player can't fight back)
- **Gradient interpolation** — for 25-60% range, color smoothly transitions from yellow to orange:
  ```
  rgb(255, ((normalizedHealth * 100) - 10) * 4, ...)
  ```
  This creates visual urgency scaling with damage taken.
- **Exact boundaries** — 0.25 is critical, 0.6 is normal; not continuous at edges
- **Edge case:** normalizedHealth === 1 gets specific gray `#bdc7d0`, not white

### `getBoundingClientRect()` in killfeed positioning — @file client/src/scripts/managers/uiManager.ts:1304

**Purpose:** Measure killfeed item positions before and after new kill insertion, animate existing items.

**Implicit behavior:**
- **Reflow trigger** — calling `.getBoundingClientRect()` forces browser reflow (expensive, ~1-2ms)
- **Relative positioning** — measures absolute viewport-relative positions, then calculates delta to animate existing items relative to their old positions
- **Animation easing** — uses `ease-in` easing (slow → fast) for downward shift, giving impression of weight
- **Simultaneous animations** — multiple killfeed items animate in parallel (browser handles via requestAnimationFrame batching)

---

## Related Documents

### Cross-Tier References

**Tier 1 (Architecture):**
- [docs/architecture.md](../../../architecture.md) — System overview, layering, client/server split
- [docs/datamodel.md](../../../datamodel.md) — PlayerData structure, health/adrenaline/shield serialization

**Tier 2 (Subsystems):**
- [docs/subsystems/ui-management/README.md](../README.md) — UIManager API, architecture, data flow
- [docs/subsystems/ui-management/patterns.md](../patterns.md) — Singleton pattern, jQuery caching, dirty-flag updates
- [docs/subsystems/networking/](../../networking/) — UpdatePacket deserialization, binary protocol, delta compression
- [docs/subsystems/map/](../../map/) — Minimap rendering, teammate indicator sprites
- [docs/subsystems/game-objects-client/](../../game-objects-client/) — Player health bar rendering (PixiJS)
- [docs/subsystems/input-management/](../../input-management/) — Action timer driven by weapon reload state
- [docs/subsystems/sound-management/](../../sound-management/) — Kill sound effects triggered by processKillPacket()

**Tier 3 (Modules):**
- [docs/subsystems/map/modules/minimap.md](../../map/modules/minimap.md) — Player position, teammate indicators, gas zone rendering
- [docs/subsystems/game-objects-client/modules/player-health-bars.md](../../game-objects-client/modules/player-health-bars.md) — Health bar graphic creation/animation for on-canvas player display
- [docs/subsystems/input-management/modules/action-timer.md](../../input-management/modules/action-timer.md) — Reload/medical action timer coordination

**External references:**
- `common/src/packets/updatePacket.ts` — PlayerData interface, serialization format
- `common/src/packets/killPacket.ts` — KillData type, damage source enum
- `client/src/scripts/utils/translations/` — Localized killfeed messages, translations system
- `client/src/scripts/console/variables.ts` — CVar `cv_killfeed_style`, `cv_weapon_slot_style`, `cv_anonymize_player_names`
