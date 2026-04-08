# Packets — KillPacket Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/packets/README.md -->
<!-- @source: common/src/packets/killPacket.ts -->
<!-- @updated: 2026-03-04 -->

## Purpose

This module documents the kill packet: sent when a player is downed or killed. Contains victim, attacker, damage source, weapon, and killstreak. The client uses it for killfeed and death UI.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| @file common/src/packets/killPacket.ts | `KillPacket`, `KillData`, `DamageSources` | Medium |

## KillData Structure

| Field | Purpose |
|-------|---------|
| `victimId` | Victim object ID |
| `attackerId?` | Attacker object ID |
| `creditedId?` | Player credited for kill (team assist) |
| `kills?` | Attacker's new kill count |
| `damageSource` | DamageSources enum |
| `weaponUsed?` | Gun/Melee/Throwable/Explosion/Obstacle definition |
| `killstreak?` | Attacker's killstreak |
| `downed` | Player downed (not yet dead) |
| `killed` | Player killed (final death) |

## DamageSources

| Value | Source |
|-------|--------|
| Gun | Bullet from gun |
| Melee | Melee weapon |
| Throwable | Grenade, C4, etc. |
| Explosion | Barrel, obstacle explosion |
| Gas | Gas zone damage |
| Obstacle | Obstacle collision |
| BleedOut | Downed bleed-out |
| FinallyKilled | Finish off downed player |
| Disconnect | Player left while downed |

## Serialization

- Packed `kfData` byte: damageSource (5 bits), killed, downed, credited state
- Optional: attackerId, creditedId, kills, weaponUsed, killstreak
- Weapon serialized by definition type (Gun, Melee, Throwable, Explosion, Obstacle)

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Packets overview
- **Tier 2:** [../ui/](../../ui/) — Killfeed display
- **Tier 2:** [../game-loop/](../../game-loop/) — Damage application, death logic
