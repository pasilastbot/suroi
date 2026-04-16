# Q7: Where does the boundary between server-authoritative logic and client-side prediction sit?

**Answer:**

Suroi is **server-authoritative for nearly all game logic**. Client-side prediction is **minimal** — there is no client-side bullet/damage prediction, movement correction system, or rollback mechanism. The client mostly sends input and renders what the server tells it to render.

## Authority Boundary

```
╔═══════════════════════════════════════════════════════╗
║ SERVER (40 TPS tick loop)                             ║
├───────────────────────────────────────────────────────┤
│ ✅ Authoritative:                                     │
│ • All movement physics                                │
│ • All bullet/projectile simulation & collision        │
│ • All damage calculation & armor reduction            │
│ • All inventory actions (equip, reload)               │
│ • All explosion raycasts & damage                     │
│ • All collision detection (hitboxes)                  │
│ • All game-state mutations (health, downed, etc)      │
│ • Kill detection & ranking                            │
╚═══════════════════════════════════════════════════════╝
         ▼ UpdatePacket (25ms binary WebSocket)
╔═══════════════════════════════════════════════════════╗
║ CLIENT (PixiJS rendering at unconstrained FPS)       ║
├───────────────────────────────────────────────────────┤
│ ❌ NO client-side prediction for:                     │
│ • Bullets (no raycast simulation)                     │
│ • Projectiles (no physics replay)                     │
│ • Damage calculation                                  │
│ • Movement validation                                 │
│ • Weapon firing outcome                               │
│                                                       │
│ ✅ Client-side visual updates:                        │
│ • Position/rotation interpolation between ticks       │
│ • Responsive mouse aiming (decoupled from server)     │
│ • Local input buffering                               │
│ • Animation playback                                  │
╚═══════════════════════════════════════════════════════╝
```

## What Gets Predicted vs. What Doesn't

### NO Prediction — Server-Authoritative Only

| Action | Client Role | Server Role | Confirmation |
|--------|-------------|-------------|--------------|
| **Bullet fire** | Sends attack flag | Spawns bullets, raycasts collision | UpdatePacket results (25ms delay) |
| **Projectile/grenade** | Sends attack flag | Simulates physics, bounce, detonation | UpdatePacket with state |
| **Damage application** | Receives outcome | Calculates base × armor × perks; checks kill | UpdatePacket with health deltas |
| **Movement collision** | Sends input | Checks obstacles, applies pushback | UpdatePacket with corrected position |
| **Explosion damage** | Sees effect | Ray-casts LOS; applies falloff | UpdatePacket for each affected object |
| **Inventory actions** | Clicks slot | Validates prerequisites, executes | UpdatePacket with updated state |

**Key principle:** The client never re-runs physics simulations. It receives authoritative deltas from the server and renders them.

### Minimal Prediction — Visual Smoothing Only

The client performs **optical prediction** to mask 25ms latency, but this is render-time interpolation, not game-state prediction:

| Element | Prediction Method | Source |
|---------|-------------------|--------|
| **Own position** | Interpolate between UpdatePacket ticks using velocity | Server UpdatePacket |
| **Own rotation** | Continuous mouse tracking (decoupled from ticks) | Local input |
| **Other players** | Interpolate between last-two UpdatePacket positions | Server UpdatePacket |
| **Gun fire anim** | Play frame sequence immediately; actual bullets server-only | Local timing |
| **Menu/Health** | Display from UpdatePacket immediately (authoritative) | UpdatePacket |

## Movement: Input Sent, Position Confirmed

### Client Side

```typescript
// Client: Every frame (unconstrained FPS)
if (movement changed OR actions queued OR 100ms timer expired) {
    send InputPacket {
        movement: { up, down, left, right },
        attacking: boolean,
        rotation (if turned): angle,
        actions: [InputActions]
    };
}

// Client: Render loop (unconstrained FPS)
Game.tick() {
    localPlayer.position = lerp(
        localPlayer.position,
        targetPositionFromServer,
        this.frame.deltaTime
    );
}
```

**No client-side physics:** The client does NOT:
- Run collision-checking
- Apply drag/friction
- Calculate velocity
- Validate the move is legal

### Server Side

Player.handleInput() processes InputPacket:

```typescript
player.handleInput(inputPacket) {
    player.movement = inputPacket.movement;
}

player.update() {
    // Calculate speed
    let speed = baseSpeed × shoulderMultiplier × perkModifiers;
    
    // Apply movement
    let vector = [(right - left) × speed, (down - up) × speed];
    
    // Collision check
    let obstacles = grid.getObstaclesAtPosition(newPos);
    newPos = resolve_collision(newPos, obstacle);
    
    // Update position
    this.position = newPos;
    this.setPartialDirty();  // → next UpdatePacket
}
```

### Conflict Resolution: Server Wins

If client predicts a move that server rejects:
1. Client sends movement input
2. Server checks collision → move invalid
3. Server returns UpdatePacket with original position
4. Client receives → syncs to server position
5. Player sees tiny visual stutter (usually unnoticeable at 40 TPS)

**No rollback system:** Suroi doesn't implement client-side movement prediction with rollback (like Overwatch). Latency is masked by **interpolation**, not prediction.

## Weapon Firing & Bullets: Zero Client Prediction

### Client Input Phase

```typescript
// Client: InputManager
if (mouse down) {
    inputManager.attacking = true;
}

InputManager.update() {
    if (attacking flag changed || actions.length > 0) {
        send InputPacket with { attacking: true };
    }
}

// Client: Display only (NO prediction of bullets)
Game.render() {
    gunSprite.play("fire");  // Show animation only
    // Do NOT create bullets or simulate raycasts
    // Do NOT check collisions
    // Do NOT damage enemies
}
```

**Suroi does NOT show instant client-side hit effects** like some games. Whether the shot hit is determined entirely by the server.

### Server Validation Phase

Gun.\_useItemNoDelayCheck() runs on server:

```typescript
if (owner.attacking && this.canFire()) {
    // 1. Validate fire rate, ammo
    // 2. Calculate spread (FSA reset)
    // 3. Spawn bullets + ray-cast collision
    // 4. Process hits → create DamageRecords
    // 5. Update ammo count
}
```

UpdatePacket includes bullets spawned this tick and damage results (next tick).

## Damage: Calculated Server-Only

All damage follows the server-side pipeline:

```typescript
// Server: Game.tick() processes all DamageRecords
for (const record of game.damageRecords) {
    let finalDamage = damage
        × armor.damageReduction
        × perk.modifiers;
    
    object.damage({ amount: finalDamage });
    
    if (object.health <= 0) {
        object.dead = true;
        game.sendKillPacket();
    }
}
```

**Client does NOT:**
- Calculate armor reduction
- Apply perk modifiers
- Determine kills
- Validate damage interactions

**Client does:**
- Display health bar updates (based on UpdatePacket)
- Play hit feedback sounds (triggered on health delta)

## Inventory: Actions Queued, Server Validates

Example: **Player equips weapon slot 2**

```typescript
// Client: UI interaction
ui.on("slot-click", (slotIndex) => {
    inputManager.addAction(InputActions.EquipItem, { slot: slotIndex });
});
```

```typescript
// Server: Receive action
player.handleAction(InputActions.EquipItem, { slot }) {
    if (!isValid(slot)) return;
    if (!isValid(player.inventory.slots[slot])) return;
    if (player.downed) return;
    
    player.activeGun = player.inventory.slots[slot];
    player.setPartialDirty();
}
```

Server confirms via UpdatePacket with updated `activeSlot`, ammo, etc.

## Conflict/Misprediction Resolution

### No Reconciliation System

Suroi has **no active misprediction correction** like instant feedback confirmation or rollback. Instead:

1. **Interpolation hides movement desync** (25–40ms latency)
2. **UpdatePacket overwrites any prediction** (next tick)
3. **Effects are server-driven** — no optimistic damage numbers

### Known Gotchas

**Melee swing prediction mismatch:**
If client predicts hit but server disagrees (due to lag), client shows hit effect but damage doesn't apply.

**Mitigation:** None currently. Trust server-side validation.

## Example: Shooting a Player

**Timeline:**

```
T=0 ms:     Client: Player clicks mouse (LMB)
            InputManager.attacking = true
            gunFireAnimation.play()  ← visual only
            
T=50 ms:    Client: InputPacket sent { attacking: true }
            
T=65 ms:    Server: Gun.update() checks attacking
            ├─ Validate fire rate, ammo
            ├─ Calculate spread, spawn bullets
            ├─ Ray-cast collision with enemies
            ├─ Create DamageRecord for hit
            └─ Mark enemy partial-dirty
            
T=90 ms:    Server: Processes DamageRecord
            ├─ Subtract armor
            ├─ Apply perk modifiers
            ├─ Enemy's health -= finalDamage
            └─ If health ≤ 0: create KillPacket
            
T=115 ms:   Server: Game sends UpdatePacket
            └─ Includes enemy health delta, bullets, impacts
            
T=140 ms:   Client: Receives UpdatePacket
            ├─ Deserialize enemy's new health
            ├─ Update health bar
            └─ Play hit sound
```

## Summary

| System | Authority | Prediction | Trade-Offs |
|--------|-----------|-----------|------------|
| **Movement** | Server | Interpolation only | 25-40ms latency visible but smooth |
| **Bullets** | Server-only | None | No instant hit feedback |
| **Damage** | Server-only | None | ~50ms delay before health bar updates |
| **Inventory** | Server-only | None | Actions confirmed by server next tick |
| **Mouse aiming** | Client-side | Local input | Responsive rotation; server updates following |

Suroi prioritizes **cheat prevention and consistency** over client-side prediction. This is appropriate for a competitive multiplayer game where fair play matters more than extreme responsiveness.
