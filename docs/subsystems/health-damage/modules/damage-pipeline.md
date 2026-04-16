# Core Gameplay — Health & Damage Application Pipeline

<!-- @tier: 3 -->
<!-- @parent: ../README.md -->
<!-- @source: server/src/objects/player.ts, server/src/game.ts -->

## Purpose

This module documents the complete damage application pipeline in Suroi: how players take damage, how armor/shields mitigate it, how health updates, and how kills are detected and credited. The pipeline covers health fundamentals, damage sources, status effects, formula calculations, shield system, and kill packet generation.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| `server/src/objects/player.ts` | Health state, `damage()` / `piercingDamage()` methods, armor reduction, death detection, kill credit logic | High |
| `server/src/objects/gameObject.ts` | `DamageParams` interface, abstract `damage()` method signature | Medium |
| `server/src/objects/bullet.ts` | `DamageRecord` interface, damage queuing per-bullet | Medium |
| `server/src/game.ts` | Damage event loop, `DamageRecord` processing, kill leader tracking, `KillPacket` emission | High |
| `common/src/packets/killPacket.ts` | `KillPacket` structure, `DamageSources` enum, kill credit serialization | Medium |
| `common/src/constants.ts` | `GameConstants.player` — default health, max shield, knockback scaling | Low |
| `common/src/definitions/items/armors.ts` | Armor definitions with `damageReduction` (helmet/vest mitigation %) | Low |

---

## Section 1: Health System Fundamentals

### Health Properties

```typescript
// From Player class (server/src/objects/player.ts:148-180)

private _maxHealth = GameConstants.player.defaultHealth;  // 100 HP default
private _health = this._maxHealth;                         // Current health (0-maxHealth)
private _normalizedHealth: number;                         // 0-1 for client UI

get health(): number { return this._health; }
set health(health: number) {
    const clamped = Numeric.min(health, this._maxHealth);
    if (this._health === clamped) return;
    
    this._health = clamped;
    this.dirty.health = true;
    this._normalizedHealth = Numeric.remap(this.health, 0, this.maxHealth, 0, 1);
    
    // Low health triggers particles at < 30%
    if (this.emitLowHealthParticles !== (this._health / this._maxHealth < 0.3)) {
        this.emitLowHealthParticles = this._health / this._maxHealth < 0.3;
    }
}
```

**Key facts:**
- Default max health: `GameConstants.player.defaultHealth` (100 HP)
- Health is **clamped** to `[0, maxHealth]` range
- **Normalized health** (0-1) is calculated whenever health changes
- Health updates mark player as **dirty** for network sync
- **Low health threshold**: < 30% triggers visual effects (particle emission)

### Max Health Modifiers

```typescript
// Perks can increase max health
case PerkIds.Lycanthropy:
    newModifiers.maxHealth *= perk.healthMod;
    newModifiers.hpRegen += perk.regenRate;
    
case PerkIds.ExperimentalTreatment:
    newModifiers.maxHealth *= perk.healthMod;
```

Max health is recalculated whenever:
- A weapon is equipped (weapon modifiers applied)
- A perk is added/removed
- Perks with kill limits activate (e.g., Engorged increases max health per kill)

---

## Section 2: Damage Sources

Damage can originate from multiple sources, tracked via `DamageSources` enum:

```typescript
// From common/src/packets/killPacket.ts
export enum DamageSources {
    // Player-to-player
    Gun = 0,
    Melee = 1,
    Throwable = 2,
    Explosion = 3,
    
    // Environment
    Gas = 4,
    Obstacle = 5,
    
    // Special
    BleedOut = 6,        // Bleed damage from status effect
    FinallyKilled = 7,   // Down state timeout → death
    Disconnect = 8       // Forced death for AFK
}
```

### Damage Source Classification

| Source | Originator | Tracking |
|--------|-----------|----------|
| **Gun** | Weapon (bullet hit) — player-inflicted | Damage done stats per gun |
| **Melee** | Melee weapon hit — player-inflicted | Damage done stats per melee |
| **Throwable** | Grenade/throwable explosion — player-inflicted | Damage done stats |
| **Explosion** | On-hit explosion (e.g., shotgun pellets) — player-inflicted | Damage done stats |
| **Gas** | Environmental damage from gas zone | No player credit |
| **Obstacle** | Collision damage from building/prop | No player credit |
| **BleedOut** | Bleed status effect from source weapon | Credits source as down reason |
| **FinallyKilled** | Down state timeout (10 sec) | No kill credit |
| **Disconnect** | Player disconnect while down | No kill credit |

---

## Section 3: Damage Event Flow

The damage application pipeline follows a **5-phase queue-detect-apply-event-broadcast** flow:

```
[Tick 1] Bullets update & queue DamageRecords
  ↓
[Tick 1] All bullets processed → records accumulated
  ↓
[Tick 1] For each DamageRecord: call object.damage({ amount, source, weapon, position })
  ↓
[Tick 1] damage() applies mitigations → piercingDamage() applies final damage
  ↓
[Tick 1] Health ≤ 0? → down() or die() → emit KillPacket
  ↓
[Tick 2] KillPacket broadcast to all players
```

### Phase 1: Bullet Collision & Damage Queue

```typescript
// server/src/objects/bullet.ts:93-130
// Each bullet returns DamageRecord[] after update()
public update(): DamageRecord[] {
    const records: DamageRecord[] = [];
    for (const collision of this.collisions) {
        const object = collision.object as DamageRecord["object"];
        records.push({
            object,
            damage: this.definition.damage,
            weapon: this.sourceGun,
            source: this.shooter,
            position: collision.position
        });
    }
    return records;
}
```

**Key fact:** Bullets **don't deal damage on-update**. Instead, they queue `DamageRecord`s for deferred processing. This prevents **bullet de-sync** in multi-hit scenarios (e.g., shotgun hitting one target — all pellets collected before damage applied).

### Phase 2: Deferred Damage Application Loop

```typescript
// server/src/game.ts:359-406
// After all bullets update, process all records
let records: DamageRecord[] = [];
for (const bullet of this.bullets) {
    records = records.concat(bullet.update());
}

// Apply damage after ALL bullets have queued records
for (const { object, damage, source, weapon, position } of records) {
    object.damage({
        amount: damage,
        source,
        weaponUsed: weapon,
        position: position
    });
    
    // Handle post-damage effects (on-hit explosions, perk triggers, etc.)
    // ...
}
```

**Why deferred?** Client and server must agree on damage outcomes. If server applied damage during bullet update, some pellets might hit dead objects client-side that are alive server-side.

### Phase 3: Plugin Events & Hook Points

The damage pipeline emits **3 plugin events** at critical junctures:

```typescript
// server/src/objects/player.ts:2626-2643
override damage(params: DamageParams): void {
    // EVENT 1: Player taking damage (before reductions)
    this.game.pluginManager.emit("player_damage", {
        amount,
        player: this,
        source,
        weaponUsed
    });
    
    // Calculate armor reduction...
    // Then call piercingDamage()
}

piercingDamage(params: DamageParams): void {
    // EVENT 2: Before shield/health deduction
    if (this.game.pluginManager.emit("player_will_piercing_damaged", {
        player: this,
        amount,
        source,
        weaponUsed
    })) return;  // Return true to **cancel** damage
    
    // Apply to shield/health...
    
    // EVENT 3: After damage applied
    this.game.pluginManager.emit("player_did_piercing_damaged", {
        player: this,
        amount,
        source,
        weaponUsed
    });
}
```

---

## Section 4: Damage Application Formula

The damage formula applies reductions **sequentially** through 4 stages:

### Stage 1: Shield-Aware Armor Reduction

```typescript
// server/src/objects/player.ts:2626-2655
override damage(params: DamageParams): void {
    // Only apply armor reduction if shield is depleted
    if (this.shield <= 0) {
        // Armor reductions are merged additively:
        // damage *= (1 - (helmet% + vest%)) * perkMod
        amount *= (1 - (
            (this.inventory.helmet?.damageReduction ?? 0) +
            (this.inventory.vest?.damageReduction ?? 0)
        )) * this.mapPerkOrDefault(PerkIds.LastStand, 
            ({ damageReceivedMod }) => damageReceivedMod, 
            1
        );
        
        amount = this._clampDamageAmount(amount);
    }
    
    this.piercingDamage({ amount, source, weaponUsed });
}
```

**Formula:**
$$\text{damage} = \text{rawDamage} \times \begin{cases}
(1 - (\text{helmet\%} + \text{vest\%})) \times \text{perkMod} & \text{if shield} \leq 0 \\
\text{rawDamage} & \text{if shield} > 0 \text{ (armor bypassed)}
\end{cases}$$

**Critical fact:** **Armor is bypassed while shield is active.** Once shield depletes, armor begins absorbing damage.

### Armor Reductions (Example)

```typescript
// common/src/definitions/items/armors.ts
// Helmets: 0-25% reduction
// Vests: 0-40% reduction
// Max combined (Level 3 helmet + Level 3 vest): 25% + 40% = 65%
```

Example: 40 damage with Level 2 Vest (20% reduction) + Level 1 Helmet (10% reduction):
- `damage = 40 × (1 - (0.10 + 0.20)) = 40 × 0.70 = 28 HP`

### Stage 2: Health & Shield Deduction

```typescript
// server/src/objects/player.ts:2700-2721
// Case A: Shield active
if (this.shield > 0) {
    const initialShield = this.shield;
    this.shield -= amount;
    
    if (this.shield < 0) {
        const remainingDamage = Math.abs(this.shield);
        this.shield = 0;
        
        // Recursive call: apply remaining damage to health
        this.damage({ ...params, amount: remainingDamage });
    }
}

// Case B: No shield → apply to health
if (this.shield <= 0) {
    this.health -= amount;
}
```

**Shield breaks health damage:** Shield absorbs damage first. If damage exceeds shield, **recursively call** `damage()` with remaining amount (which applies armor reduction **again**—see gotcha #1).

### Stage 3: Damage Tracking & Weapon Stats

```typescript
// server/src/objects/player.ts:2725-2755
if (amount > 0) {
    this.damageTaken += amount;  // Victim stat
    
    if (canTrackStats && !this.dead) {
        weaponUsed.stats.damage += amount;  // Per-weapon stat
        
        // Apply perk effects for damage dealt
        if (sourceIsPlayer) {
            for (const entry of weaponUsed.definition.wearerAttributes?.damageDealt ?? []) {
                if (weaponUsed.stats.damage >= entry.limit) {
                    source.health += entry.healthRestored ?? 0;
                    source.adrenaline += entry.adrenalineRestored ?? 0;
                }
            }
        }
    }
    
    if (sourceIsPlayer && source !== this) {
        source.damageDone += amount;
        this.lastDamagedBy = {
            player: source,
            weapon: weaponUsed,
            time: this.game.now
        };
    }
}
```

**Weapon stats tracked per-hit:**
- `weaponUsed.stats.damage` — cumulative damage dealt
- `weaponUsed.stats.kills` — kill count (incremented at death only)

---

## Section 5: Shield System

The shield mechanic is **opt-in via perks** (mainly `ExperimentalForcefield`):

```typescript
// server/src/objects/player.ts:229-260
private _maxShield = GameConstants.player.maxShield;  // 0 by default
private _shield = 0;
private _normalizedShield = 0;

get shield(): number { return this._shield; }
set shield(shield: number) {
    const clamped = Numeric.clamp(shield, 0, this._maxShield);
    if (this._shield === clamped) return;
    
    this._shield = clamped;
    this._normalizedShield = Numeric.remap(shield, 0, this.maxShield, 0, 1);
    this.dirty.shield = true;
}
```

### Shield Regeneration (Perks)

```typescript
// server/src/objects/player.ts:2873
case PerkIds.ExperimentalForcefield:
    newModifiers.shieldRegen += perk.shieldRegenRate;
```

Shield regens passively per tick (if `shieldRegen` modifier > 0).

### Shield Interaction with Armor

**Critical rule:** Shield and armor are **mutually exclusive** in the damage formula:
- **With shield** (shield > 0): Armor is **bypassed**, damage hits shield first
- **Without shield** (shield ≤ 0): Armor **applies**, damage hits health

This prevents "shield + armor stacking" and makes armor a fallback protection.

---

## Section 6: Health Recovery

Health can increase via:

### A) Healing Items

```typescript
// Healing items consumed from inventory
// Increases health by item.healAmount (e.g., medkit +25 HP)
this.health += healAmount;  // Clamped to maxHealth
```

### B) Perk-Based Passive Regeneration

```typescript
// server/src/objects/player.ts:2863
case PerkIds.Lycanthropy:
    newModifiers.hpRegen += perk.regenRate;
```

Health regens per tick via `hpRegen` modifier (e.g., Lycanthropy: +5 HP/tick).

### C) Perk-Based Kill Effects

```typescript
// server/src/objects/player.ts:3007-3026
case PerkIds.Bloodthirst:
    source.health += perk.healBonus;      // +50 HP per kill
    source.adrenaline += perk.adrenalineBonus;
    break;

case PerkIds.Overdrive:
    source.health += perk.healBonus;      // +30 HP per K.O. (3+ kills)
    break;
```

---

## Section 7: Status Effects on Damage

### Known Status Effects from Damage Source

Currently **bleed** and **poison** are tracked indirectly via `lastDamagedBy`:

```typescript
// server/src/objects/player.ts:2720
this.lastDamagedBy = {
    player: source,
    weapon: weaponUsed,
    time: this.game.now
};
```

**TODO:** Full status effect pipeline (persistent bleed/poison ticks) would extend `lastDamagedBy` or add a separate `activeStatusEffects` map. Currently, damage is **instantaneous** (no DoT).

### Environment Damage (Gas, Obstacle)

```typescript
// server/src/game.ts (gas tick)
for (const player of this.livingPlayers) {
    if (player.zone contains player) {
        player.damage({
            amount: gasTickDamage,
            source: DamageSources.Gas  // or DamageSources.Obstacle
        });
    }
}
```

Environmental damage bypasses **`lastDamagedBy`** tracking (no killer credit).

---

## Section 8: Kill Detection

### Down State (Team Mode Only)

```typescript
// server/src/objects/player.ts:2756-2762
if (this.health <= 0 && !this.dead) {
    // In team mode: check if teammates alive
    if (
        this.game.isTeamMode
        && this._team?.players.some(p => !p.dead && !p.downed && !p.disconnected && p !== this)
        && !this.downed
        && source !== DamageSources.Disconnect
    ) {
        // Go DOWN instead of dying
        this.down(source, weaponUsed);
    } else {
        // Go DEAD
        this.die(params);
    }
}
```

**Down state rules (team mode only):**
- Health ≤ 0, teammates alive → enter **downed state** (revivable, 100 HP, 10 sec timeout)
- Health ≤ 0, no teammates or down timeout → **die** (final death)

### Death State

```typescript
// server/src/objects/player.ts:2969-3000
die(params: Omit<DamageParams, "amount">): void {
    if (this.health > 0 || this.dead) return;
    
    this.health = 0;
    this.dead = true;
    
    const packet = KillPacket.create();
    packet.victimId = this.id;
    packet.downed = wasDowned;
    packet.killed = true;
    
    // Determine kill credit...
    // Emit KillPacket
}
```

---

## Section 9: Kill Credit Assignment

Kill credit is assigned via `creditedId` and `packet.kills`:

### Case A: Player-to-Player Kill (Direct)

```typescript
// server/src/objects/player.ts:3013-3023
} else if (source instanceof Player && source !== this) {
    this.killedBy = source;
    packet.attackerId = source.id;
    
    // Credit goes to killer (unless downed by teammate)
    if (downedBy && downedBy.teamID === source.teamID) {
        packet.creditedId = downedBy.id;
        packet.kills = ++downedBy.kills;  // Down player gets credit
    } else {
        packet.kills = ++source.kills;     // Killer gets credit
    }
}
```

**Logic:**
- If **downed** by teammate & **killed** by same teammate: down player gets **kill credit**
- Otherwise: killer gets **kill credit**

### Case B: Environmental Kill (Gas/Obstacle)

```typescript
// server/src/objects/player.ts:3001-3012
} else if (
    source === DamageSources.Gas ||
    source === DamageSources.Obstacle ||
    source === DamageSources.BleedOut ||
    source === DamageSources.FinallyKilled ||
    source === DamageSources.Disconnect
) {
    packet.damageSource = source;
    
    // Credit goes to player who downed the victim
    if (downedBy !== undefined) {
        packet.creditedId = downedBy.id;
        packet.kills = ++downedBy.kills;
    }
}
```

**Logic:**
- Environment kill → credit goes to player who **downed** the victim
- If no downer → no kill credit

### Case C: Self-Kill

```typescript
// server/src/objects/player.ts:3031
} else if (source === this) {
    this.lastSelfKillTime = this.game.now;
}
```

Self-damage (explosions, etc.) doesn't increment kill counter.

---

## Section 10: Network Synchronization

### Health Updates in UpdatePacket

```typescript
// server/src/objects/player.ts (serialization)
// Marked as dirty when health changes
if (player.dirty.health || forceInclude) {
    playerData.health = player.normalizedHealth;  // Send 0-1 normalized value
    playerData.maxHealth = player.maxHealth;
}

if (player.dirty.shield || forceInclude) {
    playerData.shield = player.normalizedShield;  // Send 0-1 normalized value
    playerData.maxShield = player.maxShield;
}
```

**Serialization:**
- Health/shield sent as **normalized (0-1)** to fit in `Uint8` in UpdatePacket
- Max values also sent so client can **denormalize** back to raw values

### Kill Packet Structure

```typescript
// common/src/packets/killPacket.ts
interface KillPacket {
    victimId: number;                           // Dead player
    kills: number;                              // Killer's kill count
    damageSource: DamageSources;               // How they died
    attackerId?: number;                        // Attacker (if player-to-player)
    creditedId?: number;                        // Person getting credit (may differ from attacker)
    weaponUsed?: WeaponDefinition;             // Weapon that killed
    killstreak?: number;                        // Killer's streak on this weapon
    downed?: boolean;                           // Was victim downed first?
    killed?: boolean;                           // Was victim killed (vs downed)?
}
```

**Broadcast:** `KillPacket` is added to `game.packets` and sent to **all players** in next `UpdatePacket`.

### Kill Leader Tracking

```typescript
// server/src/game.ts:595-623
updateKillLeader(player: Player): void {
    const killLeader = this.killLeader;
    
    if (
        player.kills > (killLeader?.kills ?? (GameConstants.player.killLeaderMinKills - 1))
        && player !== killLeader
    ) {
        this.killLeader = player;
        this.killLeaderDirty = true;  // Mark for UpdatePacket
    }
}
```

Kill leader only updates if player exceeds threshold (min kills defined in GameConstants).

---

## Section 11: Known Gotchas & Edge Cases

### Gotcha #1: Double Armor Reduction on Shield Break

When shield breaks and remaining damage is applied via **recursive** `damage()` call:

```typescript
// Shield has 20 HP, takes 50 damage
this.shield = 0;                    // Shield breaks
remainingDamage = 30;
this.damage({ amount: 30 });        // Recursive call
// Now armor reduction applies AGAIN to the 30 HP
// Result: health loses ~21 HP (30 × 0.7 with armor), not 30
```

**Impact:** Shield + armor combo is **weaker than expected**. This is likely **intentional** to prevent stacking abuse, but it's unintuitive. The formula is:
- Raw damage → (armor) → shield
- Shield overflow → (armor again) → health

### Gotcha #2: Bleed-Out and Download State Timeout

Down state has a **10-second timeout before forced death**:

```typescript
// server/src/game.ts (implies per-tick countdown)
// Down player not revived by 10 sec → dies with DamageSources.FinallyKilled
```

The `downedBy` player gets credit if victim times out while downed. This can lead to **kill credit delayed across multiple ticks**.

### Gotcha #3: Team Damage Prevention

In team mode, `piercingDamage()` **early-returns** if attacker and victim are teammates:

```typescript
// server/src/objects/player.ts:2676-2684
if (
    this.game.isTeamMode
    && source instanceof Player
    && source.teamID === this.teamID
    && source.id !== this.id
    && !this.disconnected
    && amount > 0
) return;  // Team damage blocked
```

**Effect:** Friendly fire is completely **disabled** in team modes.

### Gotcha #4: Shield Bypasses Armor Until Depleted

```
WITH shield:  damage → shield (no armor)
WITHOUT shield: damage → (armor) → health
```

This asymmetry means armor is **only effective during low-shield states**, which can confuse players expecting cumulative protection.

### Gotcha #5: Damage Clamping

```typescript
// server/src/objects/player.ts:2677
amount = this._clampDamageAmount(amount);
```

Damage is **clamped** to a minimum (likely > 0). Ultra-high armor might reduce damage to exactly 0, which doesn't increment `damageTaken` or `weaponUsed.stats.damage`. This can de-sync stats vs packets.

### Gotcha #6: Kill Credit to Down Player

In team mode, if player A downs player B, and player B dies from timeout/gas while down:
- **Kill credit goes to player A**, not the final damage source
- Only works if downedBy is defined and their `teamID` matches (or they're the killer)

This can lead to **late kill credits** (credit assigned after long timeout).

---

## Section 12: Business Rules Summary

| Rule | Source Code | Effect |
|------|-------------|--------|
| **Health clamped [0, maxHealth]** | `player.ts:172` | Over-healing prevented |
| **Armor bypassed with shield > 0** | `player.ts:2649-2655` | Armor only works when shield depleted |
| **Armor reductions additive** | `player.ts:2649` | helmet% + vest% combined, max ~65% |
| **Shield breaks recursively** | `player.ts:2707-2712` | Overflow damage reapplies armor |
| **Team damage blocked in team mode** | `player.ts:2676-2684` | Friendly fire = 0 damage |
| **Down state only in team mode** | `player.ts:2756-2762` | Solo/duo: health ≤ 0 = instant death |
| **Kill credit to downer (if timeout)** | `player.ts:3006-3012` | Timeout deaths credit the player who downed |
| **Kill credit to killer (if direct)** | `player.ts:3013-3023` | Player-to-player kills credit attacker |
| **Weapon stats track damage dealt** | `player.ts:2729` | Per-weapon stat updates tied to kills |
| **Perk effects on kill** | `player.ts:3006-3026` | Bloodthirst, Overdrive etc. heal on kill |
| **Low health < 30%** | `player.ts:183` | Triggers particle effects |

---

## Section 13: Data Lineage

```
DAMAGE SOURCE (gun, melee, explosion, gas, etc.)
  ↓
DamageRecord created: { object, damage, weapon, source, position }
  ↓
Game.tick() processes DamageRecords in deferred batch
  ↓
Player.damage(params: DamageParams) applied
  ├─→ If shield > 0: no armor reduction
  ├─→ If shield ≤ 0: apply armor reduction
  └─→ Call piercingDamage()
  
Player.piercingDamage(params) applied
  ├─→ Emit "player_will_piercing_damaged" event
  ├─→ If shield > 0: shield -= amount (recursive if breaks)
  ├─→ If shield ≤ 0: health -= amount
  ├─→ Track damage stats (damageTaken, weaponUsed.stats.damage)
  ├─→ Check if health ≤ 0: down() or die()
  ├─→ Emit "player_did_piercing_damaged" event
  └─→ Update modifiers (reapply perk effects)

If health ≤ 0:
  ├─→ down(source, weapon) if in team mode + teammates alive
  │   └─→ Emit KillPacket (downed=true, killed=false)
  └─→ die(source, weapon) otherwise
      ├─→ Determine kill credit (attacker vs downer)
      ├─→ Apply killer perk effects (Bloodthirst, etc.)
      ├─→ Emit KillPacket (downed=false/true, killed=true)
      ├─→ Update kill leader if applicable
      └─→ Broadcast to all clients
      
CLIENT receives UpdatePacket + KillPacket:
  ├─→ Update victim's health/shield
  ├─→ Play death animation + sound
  ├─→ Show kill feed entry
  └─→ Update stats (killer's kill count, etc.)
```

---

## Section 14: Complex Functions

### `damage(params: DamageParams)` — @file server/src/objects/player.ts:2626-2656

**Purpose:** Entry point for damage application. Applies armor/perk reductions before calling `piercingDamage()`.

**Implicit behavior:**
- Treats **negative damage as healing** (calls `heal(-amount)`)
- Emits `player_damage` plugin event
- **Only applies armor reduction if shield ≤ 0**
- Calls `piercingDamage()` with reduced amount

**Assumptions:**
- `DamageParams.amount` is pre-calculated (e.g., already adjusted for hitbox size, bullet count)
- `source` can be `null` (environmental), `Player`, or `DamageSources` enum
- `weaponUsed` is optional (environmental damage has no weapon)

### `piercingDamage(params: DamageParams)` — @file server/src/objects/player.ts:2662-2803

**Purpose:** Core damage application. Bypasses armor (assumes already reduced), deducts shield/health, updates stats, detects death.

**Implicit behavior:**
- **Recursive**: If shield breaks, calls `damage()` with remaining amount
- Invulnerable players take **zero damage** (bypass silently)
- Team damage **soft-blocked** (returns early, no-op)
- **Negative damage heals** if passed through
- Tracks `lastDamagedBy` only for player sources
- Calls `updateAndApplyModifiers()` to reapply perk effects
- Emits 3 plugin events: `player_will_piercing_damaged`, `player_did_piercing_damaged`

**Called by:** `damage()` (after reductions), recursive self-calls (shield overflow)

### `die(params: Omit<DamageParams, "amount">)` — @file server/src/objects/player.ts:2969-3031

**Purpose:** Apply death state and generate `KillPacket`.

**Implicit behavior:**
- **Guards**: Returns early if `health > 0` or already `dead` (idempotent)
- Emits `player_will_die` plugin event
- Increments killer's `kills` counter (unless environmental)
- Applies killer perk effects (Bloodthirst, Overdrive, Engorged, etc.)
- Assigns kill credit via `creditedId` (may differ from `attackerId`)
- Weapon swaps killer if `game.mode.weaponSwap` enabled
- Calls `down()` first if in team mode with alive teammates

**Side effects:**
- Sets `this.dead = true`, `health = 0`, clears `lastDamagedBy`
- Cancels revive actions
- Adds `KillPacket` to `game.packets` (broadcast next tick)
- Updates `killLeaderDirty` flag

### `down(source?, weaponUsed?)` — @file server/src/objects/player.ts:3273-3312

**Purpose:** Enter down state (team mode, team members alive, downed before death).

**Implicit behavior:**
- Only valid in **team mode** (called by `piercingDamage()` conditionally)
- Resets health to **100 HP** (full reset for revivable state)
- Resets adrenaline to `minAdrenaline`
- Cancels active action
- Stops weapon usage
- Sets `this.downed = true`
- Generates `KillPacket` with `downed=true, killed=false`

**Revive timeout:** Down state has **10-second timeout** (managed elsewhere, presumably `game.ts`). If not revived by then, dies with `DamageSources.FinallyKilled`.

---

## Section 15: Related Documents

### Tier 1 Architecture References
- [System Architecture](../../../architecture.md) — Overview of server-client architecture
- [Data Model](../../../datamodel.md) — Player entity schema

### Tier 2 Subsystem References
- [Body Armor & Equipment Subsystem](../../body-armor/README.md) — Armor definitions, mitigation %
- [Perks & Passive System](../../perks-passive/README.md) — Perk effects on health/damage
- [Game Loop Subsystem](../../game-loop/README.md) — Tick processing where damage loop runs
- [Projectiles & Ballistics](../../projectiles-ballistics/README.md) — Bullet generation, DamageRecord creation
- [Team System](../../team-system/README.md) — Down state, revive, team damage rules

### Tier 3 Module References
- [Armor Mitigation Module](../../body-armor/modules/armor-mitigation.md) — Detailed armor reduction formula
- [Perk Effects Module](../../perks-passive/modules/perk-effects.md) — Perk triggers on damage/kill
- [Game Loop — Tick Module](../../game-loop/modules/tick.md) — Damage event loop integration
- [Projectiles & Ballistics Module](../../projectiles-ballistics/modules/collision.md) — DamageRecord generation

### Related Systems
- **Plugin System**: `player_damage`, `player_will_piercing_damaged`, `player_did_piercing_damaged`, `player_will_die` events
- **Network**: `UpdatePacket` (health/shield serialization), `KillPacket` (kill broadcast)
- **Perks**: Bloodthirst, Overdrive, Engorged, ExperimentalForcefield, LastStand

