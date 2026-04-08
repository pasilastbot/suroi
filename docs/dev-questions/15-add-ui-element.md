# Q: How do I add a new UI element to the HUD?

<!-- @tags: UI, HUD, jQuery, uiManager -->
<!-- @related: docs/subsystems/ui/modules/hud.md, docs/subsystems/ui/README.md -->

## Overview

The Suroi HUD uses **jQuery** for DOM manipulation. The `UIManager` class
(`client/src/scripts/managers/uiManager.ts`) is the single place that reads
game state (from `UpdatePacket`) and updates DOM elements.

## 1. Add the HTML element

Open `client/src/index.html` and add your element inside the `#game-ui` or
`#hud` div, in the appropriate position:

```html
<!-- Inside the game HUD section -->
<div id="my-element-container">
    <span id="my-element-label">My Stat</span>
    <span id="my-element-value">0</span>
</div>
```

## 2. Add state to UIManager

```typescript
// client/src/scripts/managers/uiManager.ts — in UIManagerClass

private myElementValue = 0;
```

## 3. Update from game state

The `updateUI(data: PlayerData)` method is called every time `PlayerData` arrives
in `UpdatePacket`. Add your update logic there:

```typescript
// In UIManagerClass.updateUI(data: PlayerData):
if (data.myNewField !== undefined) {
    this.myElementValue = data.myNewField;
    this.updateMyElement();
}
```

If your UI element reacts to a different packet (e.g. `KillPacket`), add the
update call in the appropriate packet handler in `client/src/scripts/game.ts`.

## 4. Add the update method

```typescript
private updateMyElement(): void {
    $("#my-element-value").text(Math.round(this.myElementValue));
    // Or toggle visibility:
    // $("#my-element-container").toggle(this.myElementValue > 0);
    // Or update a progress bar:
    // $("#my-element-bar").css("width", `${this.myElementValue}%`);
}
```

## 5. Add SCSS styling

Create or edit the relevant SCSS file in `client/src/scss/`:

```scss
// client/src/scss/ui/hud.scss (or create a new file and import it)
#my-element-container {
    position: absolute;
    bottom: 100px;
    right: 20px;
    background: rgba(0, 0, 0, 0.6);
    padding: 4px 8px;
    border-radius: 4px;
    color: white;
    font-size: 14px;
}
```

## 6. Add a translation string (if user-visible)

If the label needs to be translated:

```typescript
// In the HTML element, use data-i18n:
// <span id="my-element-label" data-i18n="my_element_key"></span>
```

Then add the key to `TranslationKeys` and HJSON files — see [Add Translation String](16-add-translation-string.md).

## 7. Hide during lobby / show only in-game

```typescript
// Show/hide based on game state in UIManager:
Game.on("game_started", () => {
    $("#my-element-container").show();
});

Game.on("game_ended", () => {
    $("#my-element-container").hide();
});
```

Or add `display: none` in CSS and toggle with `.show()` / `.hide()`.

## 8. Test

```bash
bun dev
```

Open the browser and join a game. The element should appear.

## Important Notes

- **No Svelte components** in the current HUD — everything is jQuery + HTML + SCSS
- Use jQuery selectors (`$()`) consistent with the rest of `uiManager.ts`
- For mobile, check `isMobile.any` and adjust positioning/sizing in SCSS
- The `cv_ui_scale` cvar scales the entire HUD — use `em`/`rem` units where
  possible for automatic scaling compatibility

## If the stat comes from the server

If your new UI element displays a value from the server that doesn't exist yet,
you'll also need to add it to `UpdatePacket` PlayerData — see
[Add Player Stat](19-add-player-stat.md).

## Related

- [HUD Module](../subsystems/ui/modules/hud.md) — existing HUD elements and update flow
- [UI Subsystem](../subsystems/ui/README.md) — overview
- [Translations](16-add-translation-string.md) — i18n for labels
- [Add Player Stat](19-add-player-stat.md) — if you need new data from the server
