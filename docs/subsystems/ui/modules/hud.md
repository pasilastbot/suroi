# UI — HUD Module

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/ui/README.md -->
<!-- @source: client/src/scripts/managers/uiManager.ts -->
<!-- @updated: 2026-03-04 -->

## Purpose

This module documents the in-game HUD: health, adrenaline, inventory (weapons, items, scope), killfeed, teammate list, and crosshair. Updated from `UpdatePacket` PlayerData and KillPacket.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| @file client/src/scripts/managers/uiManager.ts | `UIManager` — HUD state, DOM updates | High |

## HUD Elements

| Element | Source | Purpose |
|---------|--------|---------|
| Health bar | PlayerData.health | Current/max health |
| Adrenaline bar | PlayerData.adrenaline | Adrenaline level |
| Weapon slots | PlayerData.inventory.weapons | Active weapon, locked state |
| Inventory items | PlayerData.inventory.items | Healing, ammo, armor |
| Scope | PlayerData.inventory.scope | Zoom level |
| Killfeed | KillPacket | Recent kills with weapon icons |
| Teammate list | PlayerData.teammates | Names, health, status |
| Crosshair | InputManager | Aim reticle |

## Update Flow

```
UpdatePacket.playerData
    → UIManager.updateUI(playerData)
    → Update health, adrenaline, inventory, scope
    → updateWeaponSlots(), updateInventory()

KillPacket
    → Add to killfeed
    → Show weapon icon, victim, attacker
```

## Key Concepts

- **Locked slots** — Bitmask prevents accidental weapon swap
- **cv_anonymize_player_names** — Replaces names with defaultName_id
- **TEAMMATE_COLORS** — Teammate panel colors

## Related Documents

- **Tier 2:** [../README.md](../README.md) — UI overview
- **Tier 3:** [translations.md](translations.md) — i18n for HUD strings
- **Tier 2:** [../packets/](../../packets/) — PlayerData, KillPacket
