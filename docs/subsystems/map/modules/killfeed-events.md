# Killfeed & Events

<!-- @tier: 3 -->
<!-- @parent: ../README.md -->
<!-- @source: server/src/game.ts, common/src/packets/killPacket.ts -->

## Purpose

Documents death event handling, kill attribution, killfeed message generation, and broadcasting to all clients. Covers the flow from damage to network announcement.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| `common/src/packets/killPacket.ts` | KillPacket data structure and serialization | Medium |
| `server/src/game.ts` | Kill event generation, killfeed tracking | High |
| `server/src/objects/player.ts` | Damage handling, death logic, kill credit | High |
| `client/src/scripts/ui.ts` | Killfeed display, animation | Medium |

## Death Packet Structure

The Kill packet broadcasts death events to all players:

```typescript
// common/src/packets/killPacket.ts
export enum DamageSources {
    Gun = 0,
    Melee = 1,
    Throwable = 2,
    Explosion = 3,
    Gas = 4,
    Obstacle = 5,
    BleedOut = 6,
    FinallyKilled = 7,
    Disconnect = 8
}

export interface KillData {
    readonly type: PacketType.Kill;
    
    // WHO DIED
    readonly victimId: number;              // uint16
    
    // WHO KILLED THEM
    readonly attackerId?: number;           // uint16 (undefined = environmental death)
    readonly creditedId?: number;           // uint16 (who gets kill credit)
    
    // ATTACKER STATS
    readonly kills?: number;                // Attacker's kill count after this kill
    readonly killstreak?: number;           // Attacker's current streak
    
    // HOW THEY DIED
    readonly damageSource: DamageSources;   // Enum for cause type
    readonly weaponUsed?: 
        GunDefinition |
        MeleeDefinition |
        ThrowableDefinition |
        ExplosionDefinition |
        ObstacleDefinition;
    
    // STATE
    readonly downed: boolean;               // Victim downed (incapacitated, not dead)
    readonly killed: boolean;               // Victim fully dead
}
```

## Death Packet Serialization

Binary layout (compact encoding):

```
Byte 0-1: kfData (10-bit header)
    bits 9: hasAttackerId
    bits 8-7: creditedState (00=none, 01=victim, 10=other)
    bit 6: downed
    bit 5: killed
    bits 4-0: damageSource (5 bits, 32 values max)

Byte 2-3: victimId (uint16)

[if hasAttackerId]
Byte 4-5: attackerId (uint16)

[if creditedState == 10]
Byte 6-7: creditedId (uint16)

[if hasAttackerId or hasCreditedId]
Byte 8: kills count (uint8)

[Depends on weaponUsed]
...weapon-specific serialization
```

Size: ~6-12 bytes per kill

## Kill Credit Tracking

### Attacker Attribution

When Player A damages Player B:

```typescript
// server/src/objects/player.ts
takeDamage(params: DamageParams): void {
    this._health -= params.amount;
    
    // Track who damaged us
    this.lastDamagedBy = {
        player: damager,
        weaponUsed: params.weaponUsed,
        time: Date.now()
    };
    
    // For downed state (team mode)
    if (this.gameMode === TeamMode && this.health <= 0 && !this.dead) {
        this.downed = true;  // Players can revive you
    }
}
```

### Kill Credit Attribution

When victim reaches 0 HP:

```typescript
// server/src/game.ts / player.ts
player.kill = (finalDamageSource: DamageSources): void => {
    if (player.dead) return;
    
    player.dead = true;
    
    // Determine who gets credit
    let creditedPlayer = undefined;
    let creditedSource = finalDamageSource;
    
    // Last damager within 5 seconds
    if (this.lastDamagedBy?.time && Date.now() - this.lastDamagedBy.time < 5000) {
        creditedPlayer = this.lastDamagedBy.player;
    }
    
    // Special case: blame the finisher, not the one who started the fight
    if (finalDamageSource === DamageSources.Gas) {
        creditedSource = DamageSources.Gas;
        creditedPlayer = undefined;
    }
    
    // Generate kill packet
    const killData: KillData = {
        type: PacketType.Kill,
        victimId: player.id,
        attackerId: creditedPlayer?.id,
        creditedId: creditedPlayer?.id,
        kills: creditedPlayer?.kills,
        damageSource: creditedSource,
        weaponUsed: this.lastDamagedBy?.weaponUsed?.definition,
        downed: false,
        killed: true
    };
    
    // Broadcast to all players
    this.game.packets.push(killData);
};
```

### Team Mode: Different Credit

In team mode, kills may be credited to a teammate who revived you:

```typescript
// Simplified
if (game.isTeamMode) {
    // Attacker and victim on same team → no kill credit
    if (attackerId?.team === victimId?.team) {
        creditedId = undefined;  // Team kill, no credit
    }
    
    // Attacker on different team → normal credit
    else {
        creditedId = attackerId;
    }
}
```

## Kill Reward & Statistics

### Server-Side Kill Tracking

```typescript
// server/src/objects/player.ts
class Player {
    private _kills = 0;
    get kills(): number { return this._kills; }
    set kills(value: number) {
        if (this._kills === value) return;
        this._kills = value;
        this.game.updateKillLeader(this);  // Update leaderboard
    }
    
    kill = (damageSource: DamageSources): void => {
        // ... determine attacker ...
        
        if (attacker) {
            attacker.kills++;  // Increment kill count
            
            // Check if attacker is new kill leader
            if (attacker.kills >= GameConstants.player.killLeaderMinKills) {
                this.game.announcements.push({
                    type: "kill_leader",
                    player: attacker.name,
                    kills: attacker.kills
                });
            }
        }
    };
}
```

### Client-Side Kill Statistics

Killfeed UI displays kills, killstreaks, and milestones:

```typescript
// client/src/scripts/ui.ts (pseudo)
onKillPacket(kill: KillData): void {
    const attacker = this.game.objects.get(kill.attackerId);
    const victim = this.game.objects.get(kill.victimId);
    
    const killMessage = this.generateKillMessage(kill);
    // "Alice killed Bob with M16A4"
    // "Charlie killed Diana with headshot (sniper)"
    
    this.killfeed.addEntry({
        timestamp: Date.now(),
        message: killMessage,
        attackerName: attacker?.name,
        victimName: victim?.name,
        weaponIcon: getWeaponIcon(kill.weaponUsed),
        damageSource: DamageSources[kill.damageSource],
        isPlayerKill: kill.attackerId !== undefined
    });
    
    // Animate killfeed entry (fade in, then fade out after 5 seconds)
    this.killfeed.entries[this.killfeed.entries.length - 1].animate();
}
```

## Team Kill vs Enemy Kill Differentiation

### Team Kill Detection

```typescript
// server/src/game.ts
player.kill = (attacker: Player, damageSource: DamageSources): void => {
    const isTeamKill = game.isTeamMode && attacker.team === player.team;
    
    if (isTeamKill) {
        // Team kill: no reward, may be punished
        attacker.teamKills++;
        if (attacker.teamKills > MAX_TEAM_KILLS_BEFORE_KICK) {
            attacker.socket.close(1000, "Too many team kills");
        }
    } else {
        // Enemy kill: grant reward
        attacker.kills++;
        attacker.gold += KILL_REWARD;
    }
    
    const killPacket: KillData = {
        type: PacketType.Kill,
        victimId: player.id,
        attackerId: isTeamKill ? attacker.id : undefined,  // Don't highlight team kills
        damageSource: damageSource,
        killed: true
    };
    
    this.packets.push(killPacket);
};
```

### Client Display

```typescript
// client/src/scripts/ui.ts
generateKillMessage(kill: KillData): string {
    const victim = this.getPlayerName(kill.victimId);
    
    if (!kill.attackerId) {
        // Environmental death
        return `${victim} died to ${DamageSources[kill.damageSource]}`;
    }
    
    const attacker = this.getPlayerName(kill.attackerId);
    const weapon = kill.weaponUsed?.name || "unknown";
    
    if (this.isTeamKill(attacker, victim)) {
        // Team kill in red
        return `${attacker} [TEAM KILL] ${victim}`;  // CSS: color: red
    } else {
        // Normal kill
        return `${attacker} killed ${victim} with ${weapon}`;
    }
}
```

## Killfeed Message Generation

### Message Templates

Based on damage source:

```typescript
const killfeedTemplates = {
    [DamageSources.Gun]: (attacker, victim, weapon) =>
        `${attacker} killed ${victim} with ${weapon.name}`,
    
    [DamageSources.Melee]: (attacker, victim, weapon) =>
        `${attacker} sliced ${victim} with ${weapon.name}`,
    
    [DamageSources.Throwable]: (attacker, victim, weapon) =>
        `${attacker} threw ${weapon.name} at ${victim}`,
    
    [DamageSources.Explosion]: (attacker, victim, source) =>
        `${victim} was blown up by ${source.name}`,
    
    [DamageSources.Gas]: (attacker, victim) =>
        `${victim} was killed by the gas`,
    
    [DamageSources.Obstacle]: (attacker, victim, obstacle) =>
        `${victim} hit the ${obstacle.name}`,
    
    [DamageSources.BleedOut]: (attacker, victim) =>
        `${victim} bled out`,
    
    [DamageSources.Disconnect]: (attacker, victim) =>
        `${victim} disconnected`
};
```

## Client Display Integration

### Killfeed UI

```typescript
// client/src/scripts/ui.ts
export class Killfeed {
    private _entries: KillfeedEntry[] = [];
    private _container = new Container();
    
    addEntry(data: {
        message: string;
        attackerName?: string;
        victimName?: string;
        weaponIcon?: Texture;
        timestamp: number;
    }): void {
        const entry = new KillfeedEntry(data);
        this._entries.push(entry);
        this._container.addChild(entry.container);
        
        // Auto-remove after 5 seconds
        Game.addTimeout(() => {
            entry.fadeOut();
            removeFrom(this._entries, entry);
        }, 5000);
    }
}

class KillfeedEntry {
    readonly container = new Container();
    
    constructor(data: any) {
        // Attacker name (white)
        const attackerText = new Text(data.attackerName, { fill: 0xFFFFFF });
        this.container.addChild(attackerText);
        
        // Weapon icon
        if (data.weaponIcon) {
            const icon = new Sprite(data.weaponIcon);
            this.container.addChild(icon);
        }
        
        // Victim name (red)
        const victimText = new Text(data.victimName, { fill: 0xFF0000 });
        this.container.addChild(victimText);
    }
    
    fadeOut(): void {
        // Animate opacity 1 → 0 over 1 second
        Game.addTimeout(() => this.container.alpha -= 0.1, 100);
    }
}
```

## Killfeed Events (Plugin API)

```typescript
// server/src/pluginManager.ts
export type GamePluginEvents = {
    // ... other events ...
    'player_kill': {
        victim: Player,
        attacker?: Player,
        damageSource: DamageSources,
        weaponUsed?: any
    },
    'killfeed_entry': {
        message: string,
        kills: number
    }
};

// In plugin:
game.pluginManager.on('player_kill', (event) => {
    console.log(`${event.attacker?.name} killed ${event.victim.name}`);
});
```

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Map and game events subsystem
- **Tier 1:** [../../../../architecture.md](../../../../architecture.md) — System architecture
- **Tier 3:** [../../networking/modules/packet-types.md](../../networking/modules/packet-types.md) — KillPacket detailed structure
- **Game Loop:** [../../game-loop/README.md](../../game-loop/README.md) — Damage processing and tick flow
- **UI:** [../../ui-management/README.md](../../ui-management/README.md) — Killfeed display and notifications
