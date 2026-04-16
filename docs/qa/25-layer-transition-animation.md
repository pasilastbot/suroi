# Q25: How does the layer transition animation (200ms sineOut) work when a player walks onto stairs?

**Answer:** The layer transition animation (200ms sineOut) is a coordinated client-server mechanism that smoothly crossfades between the three layer containers (basement, ground, upstairs) when a player's detected layer changes at stairs.

---

## Animation Timing & Parameters

The animation uses:
- **Duration:** 200ms
- **Easing function:** `EaseFunctions.sineOut` — creates a smooth deceleration curve as the layers fade
- **Trigger:** When server detects stair collision and changes player layer

---

## Server-Side Detection

When the player walks onto stairs, the server's collision loop detects the stair obstacle:

```typescript
if (isObstacle && potential.definition.isStair) {
    const oldLayer = this.layer;
    potential.handleStairInteraction(this);
    if (this.layer !== oldLayer) this.setDirty();  // Marks for network sync
    this.activeStair = potential;
}
```

The `handleStairInteraction` method calculates the player's new layer based on their position relative to the stair hitbox:

```typescript
handleStairInteraction(object: GameObject | Bullet): void {
    object.layer = resolveStairInteraction(
        this.definition,
        this.rotation as Orientation,
        this.hitbox as RectangleHitbox,
        this.layer,
        object.position
    );
}
```

The `resolveStairInteraction` function uses edge geometry to determine which layer the player should be on based on whether they're crossing into a higher or lower "floor" of the stair structure.

---

## Client-Side Animation Mechanics

When the `UpdatePacket` arrives with the new layer, the Game singleton calls:

```typescript
updateLayer(newLayer: Layer, initial = false, oldLayer?: Layer): void {
    CameraManager.updateLayer(newLayer, initial, oldLayer);
    // ... update audio and background color
}
```

This triggers the core animation in `CameraManager`:

1. **Layer Container State Change:** The three layer containers get new target alpha values (0 or 1, meaning hidden or visible)

2. **Tween Animation:** A 200ms sineOut tween animates each container's alpha:
```typescript
this.layerTweens[i] = Game.addTween({
    target: container,
    to: { alpha: targetAlpha },
    duration: LAYER_TRANSITION_DELAY,
    ease: EaseFunctions.sineOut,
    onComplete: () => {
        this.layerTweens[i] = undefined;
        container.visible = visible;
    }
});
```

3. **Temporary Layer Container:** During the transition, objects moving between layers are rendered in `tempLayerContainer` to prevent visual glitching. The `getContainer()` method returns this temporary container if a layer change is in progress:

```typescript
getContainer(layer: Layer, oldContainerIndex?: LayerContainer): Container {
    const containerIndex = getLayerContainer(layer, Game.layer);
    if (this.layerTransition && oldContainerIndex !== undefined && containerIndex !== oldContainerIndex) {
        return this.tempLayerContainer;  // Swap buffer for smooth transition
    } else {
        return this.layerContainers[containerIndex];
    }
}
```

4. **Object Re-layering:** After the 200ms delay, all game objects are repositioned to their correct final layer containers. A timeout fires to call `updateLayer()` on all pooled objects:

```typescript
this.objectUpdateTimeout = Game.addTimeout(() => {
    for (const object of Game.objects) {
        object.updateLayer();
    }
}, LAYER_TRANSITION_DELAY);
```

---

## Easing Function: sineOut

The **sineOut easing** produces a natural deceleration: the fade-out/fade-in accelerates quickly at first, then slows at the end, making the transition feel weighted and polished rather than linear.

---

## References

**Documentation:**
- **Tier 2:** `docs/subsystems/camera-management/` — Full viewport and layer mechanics
- **Tier 2:** `docs/subsystems/client-rendering/` — Container hierarchy and rendering pipeline
- **Tier 2:** `docs/subsystems/game-objects-client/` — Object pooling and layer synchronization

**Source Code:**
- `client/src/scripts/game.ts` (line 1389) — updateLayer call
- `client/src/scripts/managers/cameraManager.ts` (lines 128–185) — Animation implementation
