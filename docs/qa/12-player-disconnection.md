# Q12: How does the server handle a player disconnecting mid-game — is there a grace period or instant death?

**Answer:** Suroi uses an **instant disconnect detection** with a **12-second combat log grace period**. There is no reconnect window, but the death mechanism is context-aware based on recent combat.

---

## Disconnect Detection (Instant)

The server detects disconnection via the WebSocket `close` event (not a packet). When a client socket closes, `game.removePlayer(player)` is called immediately without delay.

**Key file:** `server/src/game.ts` (lines 850-851) — sets `player.disconnected = true` and begins removal logic

---

## Conditional Death Logic (Combat Log Grace Period)

When a player disconnects while **still alive** (`!player.dead`), the server checks their combat history:

```
If player was damaged within last 12 seconds:
  ├─ Attacker is still alive → Give kill credit to the attacker
  ├─ Attacker is disconnected → Apply disconnect source
  └─ Kill source: attacker's weapon
  
Else (no recent damage):
  └─ Kill source: DamageSources.Disconnect (combat log timeout)
```

**Grace period constant:** `common/src/constants.ts` — `combatLogTimeoutMs: 12000` (12 seconds)

This is a **lookahead grace period**: if a player takes damage and immediately disconnects, the attacker gets the kill within that 12-second window. The purpose is to credit the player who dealt damage, even if the opponent disconnects before the damage-kill death sequence completes.

---

## Corpse Persistence (No Instant Despawn)

Whether the corpse persists depends on the **`canDespawn` flag** set during gameplay:

| Scenario | canDespawn | Outcome |
|----------|-----------|---------|
| **Player joins but leaves before 5-sec timeout** | `true` | Instantly removed from game & spectators list |
| **Player takes any damage** | `false` (set during first damage) | Corpse persists on map as `DeathMarker` |
| **Player dies normally** | `false` (set during death) | Corpse/gravestone visible until game ends |

When `canDespawn = false` but player disconnects:
- Corpse remains on the map (visible to other players)
- Death marker synced to all clients
- Player remains in server memory until game ends (for killfeed/spectating purposes)
- If not in a team, player is **removed from the alive count** (affects game-over condition)

---

## Spectating Behavior

- **Living disconnected players** become spectators (if they rejoin, they spectate living teammates)
- **Dead disconnected players** can be selected as spectate targets by other dead players
- No automatic reconnect mechanism — disconnection is permanent for that game instance

---

## Related Systems

**Team Mode Exceptions:** In team-aware damage logic, team-aware logic prevents "down" state if player is disconnected: disconnected teammates don't count toward the "living teammate" check, so you skip the "down" state and go straight to death.

**Initial Connection Timeout:** If a player's socket is created but `JoinPacket` is not received within 5 seconds, they are automatically kicked with `disconnect("JoinPacket not received after 5 seconds")`.

---

## References

- **Tier 2:** `docs/subsystems/death-spectator/README.md` — Death lifecycle, spectating, kill credit
- **Tier 2:** `docs/subsystems/game-loop/README.md` — Tick structure, player removal integration
- **Tier 2:** `docs/subsystems/networking/README.md` — WebSocket frame structure, packet types
- **Tier 3:** `docs/subsystems/networking/modules/protocol.md` — Connection lifecycle, WebSocket close handling
- **Source:** `server/src/game.ts` — `removePlayer()` method (full disconnect flow)
- **Source:** `server/src/objects/player.ts` — `damage()` method setting `canDespawn = false`
