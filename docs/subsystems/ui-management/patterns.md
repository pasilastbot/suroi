# UI Management — Patterns

<!-- @tier: 2 -->
<!-- @parent: docs/subsystems/ui-management/README.md -->

## Pattern: DOM Element Caching with jQuery

**When to use:** Any HUD element that's updated multiple times per second (>5 updates/sec)

**Implementation:**
Define a frozen object at class instantiation that maps English names to jQuery selectors:

```typescript
readonly ui = Object.freeze({
    healthBar: $<HTMLDivElement>("#health-bar"),
    healthBarAmount: $<HTMLSpanElement>("#health-bar-amount"), 
    killFeed: $<HTMLDivElement>("#kill-feed"),
    // ... more elements
});
```

**Benefits:**
- Single DOM query per element at startup, not per-update
- Compile-time-verified element names (IDE autocomplete)
- `Object.freeze()` prevents accidental mutation in production

**Example usage:**
```typescript
// GOOD: cached selector
this.ui.healthBar.width(`${healthPercent}%`);

// BAD: repeated query
$<HTMLDivElement>("#health-bar").width(`${healthPercent}%`);
```

**Related PR:** [Create UIManager singleton caches](https://github.com/hashintel/suroi/pulls/), pattern introduced to reduce DOM query overhead from 50+ queries/tick to 0.

---

## Pattern: Dirty-Flag Optimization

**When to use:** Rendering state that changes infrequently (weapon slots, item counts) but is queried every tick

**Implementation:**
Maintain a cache object alongside the real data. Compare current state to cache before mutating DOM:

```typescript
readonly weaponCache: Array<{
    hasItem?: boolean
    isActive?: boolean
    idString?: string
    ammo?: number
    hasAmmo?: boolean
} | undefined> = new Array(GameConstants.player.maxWeapons);

updateWeaponSlots(force = false): void {
    for (let i = 0; i < max; i++) {
        const weapon = inventory.weapons[i];
        const cache = this.weaponCache[i] ??= {};
        
        const hasItem = weapon !== undefined;
        if (hasItem !== cache.hasItem) {  // ← only update if CHANGED
            cache.hasItem = hasItem;
            container.toggleClass("has-item", hasItem);
        }
    }
}
```

**Benefits:**
- Only 1-2 DOM mutations when weapon actually changes (equip, pickup)
- Zero DOM mutations when weapon stays same (movement, shooting)
- Force flag allows bypassing cache for special reload events

**Anti-pattern:**
```typescript
// BAD: updates DOM every tick even if nothing changed
for (let i = 0; i < 4; i++) {
    container.toggleClass("active", this.inventory.activeWeaponIndex === i);  // every tick!
}
```

---

## Pattern: Map-Based DOM Element Pooling

**When to use:** Dynamic collections of UI elements that appear/disappear (teammate indicators, item slots)

**Implementation:**
Maintain a Map from unique ID to DOM element container. Create on first seen, update on subsequent, delete when not seen:

```typescript
private readonly _weaponSlotCache = new ExtendedMap<number, {
    readonly container: JQuery<HTMLDivElement>
    readonly inner: JQuery<HTMLDivElement>
    readonly name: JQuery<HTMLSpanElement>
    readonly image: JQuery<HTMLImageElement>
    readonly ammo: JQuery<HTMLSpanElement>
}>();

private _getSlotUI(slot: number) {
    return this._weaponSlotCache.getAndGetDefaultIfAbsent(slot, () => {
        // create and cache if not present
        const container = $<HTMLDivElement>(`#weapon-slot-${slot}`);
        return {
            container,
            inner: container.children<HTMLDivElement>(".main-container"),
            name: inner.children(".item-name"),
            image: inner.children(".item-image"),
            ammo: inner.children(".item-ammo")
        };
    });
}
```

**Benefits:**
- Reuse DOM elements across updates (no teardown/rebuild)
- Query child elements once at first access, cache references
- O(1) lookup by ID instead of O(n) DOM search

**Example:** `TeammateIndicatorUI` pooling

```typescript
const _teammateDataCache = new Map<number, TeammateIndicatorUI>();
const notVisited = new Set(_teammateDataCache.keys());

[].forEach((player) => {
    notVisited.delete(player.id);
    const cacheEntry = _teammateDataCache.get(player.id);
    
    if (cacheEntry !== undefined) {
        cacheEntry.update({...player});  // reuse, just update
        return;
    }
    
    const ele = new TeammateIndicatorUI({...});  // first time: create
    _teammateDataCache.set(player.id, ele);
    this.ui.teamContainer.append(ele.container);
});

for (const outdated of notVisited) {
    _teammateDataCache.get(outdated)!.destroy();  // cleanup
    _teammateDataCache.delete(outdated);
}
```

---

## Pattern: Conditional DOM Visibility with Ternary/Toggle

**When to use:** Simple show/hide based on state (health bar visible only if has health data, ammo only if gun)

**Implementation:**
Use jQuery `.toggle()` or `.toggleClass()` for boolean visibility:

```typescript
// Show ammo counter only if equipped weapon
if (activeWeapon === undefined || count === undefined) {
    this.ui.ammoCounterContainer.hide();
} else {
    this.ui.ammoCounterContainer.show();
    // ... render ammo
}

// Toggle class based on flag
container.toggleClass("active", isActive);
container.toggleClass("locked", !!lockedFlag);
```

**Benefits:**
- Declarative: intent clear from `toggle(condition)` call
- CSS handles animation (e.g., `opacity: 0; transition: opacity 0.3s`)
- Single jQuery call instead of `if... .addClass() / else .removeClass()`

**Anti-pattern:**
```typescript
// BAD: verbose
if (isActive) {
    container.addClass("active");
} else {
    container.removeClass("active");
}
```

---

## Pattern: Computed Color/Styling Based on Game State

**When to use:** Dynamic CSS values calculated from game state (health bar color, item rarity color, team color)

**Implementation:**
Compute CSS value in TypeScript, set via `.css()` or `.attr()`:

```typescript
getHealthColor(normalizedHealth: number, downed?: boolean): string {
    switch (true) {
        case normalizedHealth <= 0.25:
        case downed:
            return "#ff0000";
        case normalizedHealth > 0.25 && normalizedHealth < 0.6:
            return `rgb(255, ${((normalizedHealth * 100) - 10) * 4}, ${((normalizedHealth * 100) - 10) * 4})`;
        case normalizedHealth === 1:
            return "#bdc7d0";
        default:
            return "#f8f9fa";
    }
}

updateUI(...) {
    const normalizedHealth = health / maxHealth;
    this.ui.healthBar
        .css("background-color", this.getHealthColor(normalizedHealth))
        .toggleClass("flashing", normalizedHealth <= 0.25);
}
```

**Benefits:**
- Single source of truth for color logic (TypeScript)
- Avoids CSS-in-JS bloat (color gradient computed, not hardcoded)
- Easy to test color logic independently

**Example — Weapon slot coloring by ammo type:**

```typescript
if (GameConsole.getBuiltInCVar("cv_weapon_slot_style") === "colored") {
    const color = Ammos.fromString(definition.ammoType).characteristicColor;
    container.css({
        "outline-color": `hsl(${color.hue}, ${color.saturation}%, ${(color.lightness + 50) / 3}%)`,
        "background-color": `hsla(${color.hue}, ${color.saturation}%, ${color.lightness / 2}%, 50%)`,
        "color": `hsla(${color.hue}, ${color.saturation}%, 90%)`
    });
}
```

---

## Pattern: Dropdown Menu Toggle

**When to use:** Any collapsible menu (team options, game menu, settings)

**Implementation:**
Defined in `uiHelpers.ts`:

```typescript
export function createDropdown(selector: string): void {
    const dropdown = {
        main: $(`${selector} .dropdown-content`),
        caret: $(`${selector} button i`),
        active: false,
        show() {
            this.active = true;
            this.main.addClass("active");
            this.caret.removeClass("fa-caret-down").addClass("fa-caret-up");
        },
        hide() {
            this.active = false;
            this.main.removeClass("active");
            this.caret.addClass("fa-caret-down").removeClass("fa-caret-up");
        },
        toggle() {
            if (this.active) this.hide();
            else this.show();
        }
    };
    
    $(`${selector} button`).on("click", ev => {
        dropdown.toggle();
        ev.stopPropagation();
    });
    
    body.on("click", () => { dropdown.hide(); });  // close on outside click
}
```

**Usage:**
```typescript
createDropdown("#team-option-btns");
createDropdown("#game-menu");
```

**CSS (example):**
```scss
.dropdown {
    position: relative;
    
    button i {
        transition: transform 0.2s ease;
    }
    
    .dropdown-content {
        display: none;
        position: absolute;
        background: #333;
        
        &.active {
            display: block;
        }
    }
}
```

---

## Pattern: Animation Frame-Based Countdown Display

**When to use:** Timer that ticks down visually (action timer, perk cooldown, fuel tank)

**Implementation:**
Store start time and duration, compute remaining time in `updateAction()` method:

```typescript
readonly action = {
    active: false,
    fake: false,
    start: -1,
    time: 0
};

animateAction(name: string, time: number, fake = false): void {
    this.action.start = Date.now();
    this.action.time = time;
    this.ui.actionTimer.animate(
        { "stroke-dashoffset": "0" },
        time * 1000,
        "linear",
        () => { this.action.active = false; }
    );
}

updateAction(): void {
    const elapsed = (Date.now() - this.action.start) / 1000;
    const remaining = this.action.time - elapsed;
    if (remaining > 0) {
        this.ui.actionTime.text(remaining.toFixed(1));
    }
}
```

**Benefits:**
- Client-side timing avoids network jitter
- SVG animation handles progress bar, text updates time label
- Both animate simultaneously for smooth visual feedback

**Note:** Called every tick from `Game` update loop (40 TPS).

---

## Pattern: Localized HTML Templates

**When to use:** Killfeed messages, game-over stats, interaction prompts that vary by language

**Implementation:**
Use `getTranslatedString()` helper with template substitution:

```typescript
const { weapon, attacker, victim } = data;
const message = getTranslatedString("kf_message", {
    player: attackerText,
    event: getTranslatedString("kf_killed"),
    victim: victimText,
    with: weapon && getTranslatedString("with"),
    weapon: weaponName
});
```

**File:** Translations loaded from `client/src/translations/*.hjson` (HJSON format)

**Benefits:**
- All text externalized to translation files
- Simple parameter substitution handles name/weapon injection
- Server doesn't handle UI text; pure client-side

**Example translation (en.hjson):**
```hjson
"kf_message": "{player} {event} {victim}{with} {weapon}"
"kf_killed": "killed"
"with": "with"
```

---

## Pattern: Batch DOM Mutation During State Synchronization

**When to use:** Complex updates with many related fields (inventory slot, ammo, name, etc.)

**Implementation:**
Collect all changes, then apply all mutations in one update:

```typescript
const {
    container,
    inner,
    name,
    image,
    ammo
} = this._getSlotUI(i + 1);

// ← all DOM refs cached

// Then apply mutations:
container.toggleClass("has-item", hasItem).toggleClass("active", isActive);
name.text(itemName);
image.css("background-image", weaponImage);
ammo.text(ammoCount);
```

**File:** [uiManager.ts:1046-1127](../../../client/src/scripts/managers/uiManager.ts#L1046) (`updateWeaponSlots()`)

**Benefits:**
- Browser reflow happens once for whole slot, not once per property
- Easier to reason about final state (all mutations visible at once)
- Scales well as slot definition grows (more weapons, more properties)

---

## Pattern: Transient DOM Cleanup with Timer

**When to use:** Temporary UI elements that auto-dismiss (killfeed, notifications, modals)

**Implementation:**
Create element, schedule removal via `setTimeout()`:

```typescript
function addKillFeedEntry(html: string, classes: string[]): void {
    const killFeedItem = $(`<div class="kill-feed-item ${classes.join(" ")}">${html}</div>`);
    this.ui.killFeed.prepend(killFeedItem);
    
    // ... animation setup ...
    
    setTimeout(() => {
        const removeAnimation = killFeedItem.get(0)?.animate([
            { opacity: 1, transform: "translateX(0%)" },
            { opacity: 0, transform: "translateY(100%)" }
        ], {
            duration: 300,
            easing: "ease-out"
        });
        
        if (!removeAnimation) return;
        removeAnimation.onfinish = () => {
            killFeedItem.remove();  // cleanup after animation
        };
    }, 7000);  // appear for 7 seconds
}
```

**Benefits:**
- Element lifecycle is clear: create → animate in → appear → animate out → remove
- No manual cleanup needed after timeout
- Browser GC handles jQuery object when removed

---

## Pattern: Recomputing Derived Data Only on Change

**When to use:** Properties that depend on multiple inputs but only change when one input changes (weapon slot color, health bar width)

**Implementation:**
Cache input state, only recompute if any input changed:

```typescript
private _oldHealthPercent = 100;

updateUI(...) {
    // ... extract health data ...
    const healthPercent = 100 * (health / maxHealth);
    
    if (this._oldHealthPercent !== healthPercent) {  // ← only if changed
        this.ui.healthBar.width(`${healthPercent}%`);
        // ... other mutations ...
        this._oldHealthPercent = healthPercent;
    }
}
```

**Note:** jQuery's `.animate()` is idempotent — calling multiple times with same target width doesn't cause jitter.

---

## Related Documentation

- [UIManager API Reference](README.md#uimanager-api)
- [DOM Element Caching Pattern Details](README.md#dom-caching-optimization)
