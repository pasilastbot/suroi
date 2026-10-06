# Requirement flat-file state

- **Snapshot at:** 2026-10-06T10:00:00Z
- **Source store/revision:** git suroi@5f67ec7f8abd6ee84c39ad739f1f406d9f23e0dc (dirty)
- **Context / release:** Base / full sweep + coverage batch GRP-COV-001…008 (≥60% source disposition — see `file-state/coverage/SUROI-ONBOARD-source-disposition.json`)

---

## UR-CORE-001 — Join and play a multiplayer battle royale match

- **Kind / status:** UR / PENDING_VERIFICATION
- **Statement / source:** A player can discover a server, join a match over WebSocket, send input each tick, receive authoritative world updates, and play until elimination or victory. / DOC:docs/architecture.md; CODE:common/src/packets/packet.ts
- **Owner / release:** Base
- **Epic membership:** none
- **Declared relations:** scenarios require SR-CORE-001, SR-CORE-002, SR-CORE-003, SR-CORE-004
- **UR content:**
  - **Actor:** Player (browser client)
  - **Context:** Suroi client–server session
  - **Outcome:** Participate in an authoritative online match with visible world state synced from the server
  - **Scenarios:**
    - **UR-CORE-001-S1:** Given the main menu and a reachable game server, when the player joins a game, then the client receives map and joined state and enters gameplay.
    - **UR-CORE-001-S2:** Given an active match, when the player provides movement and combat input, then the server simulates the world and the client renders updates from `UpdatePacket`.
- **SR content:** N/A
- **Candidate packet:** N/A
- **Onboarding authority:** GATE-BASELINE-001; group GRP-CORE-001; receipt in `file-state/sources/SUROI-ONBOARD-2026-10-06.json`

### Trace references

| Evidence class | Target | Code | Test case | RED result | Passing result | Outcome | Environment | Fingerprint | Validity/revision |
|---|---|---|---|---|---|---|---|---|---|
| UR upper | UR-CORE-001-S1 | CODE:client/src/scripts/game.ts; CODE:server/src/game.ts | — | — | — | SKIP | — | onboarding@5f67ec7 | INHERITED_UNVERIFIED |
| UR upper | UR-CORE-001-S2 | CODE:common/src/packets/inputPacket.ts; CODE:common/src/packets/updatePacket.ts | — | — | — | SKIP | — | onboarding@5f67ec7 | INHERITED_UNVERIFIED |

### Gates and delivery

- **Confirmation gates:** N/A
- **Entry gates:** N/A (existing-baseline path)
- **Start/review gates:** N/A
- **Completion gates:** pending `rdd-reverse-engineer-verify` / accept
- **Delivered revision:** not delivered
- **Gaps / deferrals / blockers / notes:** No automated E2E test linked yet.

---

## SR-CORE-001 — Protocol version must match on join

- **Kind / status:** SR / PENDING_VERIFICATION
- **Statement / source:** Client and server refuse or fail join when `GameConstants.protocolVersion` differs. / DOC:docs/architecture.md; CODE:common/src/constants.ts
- **Owner / release:** Base
- **Epic membership:** none
- **Declared relations:** serves UR-CORE-001 (scenario S1)
- **UR content:** N/A
- **SR content:**
  - **Boundary:** Join handshake / connection setup
  - **Behavior:** Shared protocol version constant is enforced on both packages before gameplay packets are accepted
  - **Scope:** Version constant and join path only; not individual packet field layouts
- **Candidate packet:** N/A
- **Onboarding authority:** GATE-BASELINE-001; GRP-CORE-001

### Trace references

| Evidence class | Target | Code | Test case | RED result | Passing result | Outcome | Environment | Fingerprint | Validity/revision |
|---|---|---|---|---|---|---|---|---|---|
| SR lower | protocol match | CODE:common/src/constants.ts | — | — | — | SKIP | — | onboarding@5f67ec7 | INHERITED_UNVERIFIED |

### Gates and delivery

- **Completion gates:** pending verify pass
- **Delivered revision:** not delivered
- **Gaps / deferrals / blockers / notes:** —

---

## SR-CORE-002 — Server holds authoritative simulation

- **Kind / status:** SR / PENDING_VERIFICATION
- **Statement / source:** Game rules (damage, loot, gas, positions) are computed on the server tick loop. / DOC:docs/architecture.md; CODE:server/src/game.ts
- **Owner / release:** Base
- **Declared relations:** serves UR-CORE-001 (S2)
- **SR content:**
  - **Boundary:** `Game.tick()` and server object model
  - **Behavior:** Server state is source of truth; client displays interpolated snapshots
- **Onboarding authority:** GATE-BASELINE-001; GRP-CORE-001

### Trace references

| Evidence class | Target | Code | Test case | RED result | Passing result | Outcome | Environment | Fingerprint | Validity/revision |
|---|---|---|---|---|---|---|---|---|---|
| SR lower | authority | CODE:server/src/game.ts | — | — | — | SKIP | — | onboarding@5f67ec7 | INHERITED_UNVERIFIED |

### Gates and delivery

- **Delivered revision:** not delivered

---

## SR-CORE-003 — Client sends InputPacket intent

- **Kind / status:** SR / PENDING_VERIFICATION
- **Statement / source:** Client encodes player intent in `InputPacket` via shared packet codecs. / CODE:common/src/packets/inputPacket.ts; CODE:client/src/scripts/game.ts
- **Owner / release:** Base
- **Declared relations:** serves UR-CORE-001 (S2)
- **Onboarding authority:** GATE-BASELINE-001; GRP-CORE-001

### Trace references

| Evidence class | Target | Code | Test case | RED result | Passing result | Outcome | Environment | Fingerprint | Validity/revision |
|---|---|---|---|---|---|---|---|---|---|
| SR lower | input path | CODE:common/src/packets/inputPacket.ts | — | — | — | SKIP | — | onboarding@5f67ec7 | INHERITED_UNVERIFIED |

### Gates and delivery

- **Delivered revision:** not delivered

---

## SR-CORE-004 — Client applies UpdatePacket for render state

- **Kind / status:** SR / PENDING_VERIFICATION
- **Statement / source:** Client deserializes `UpdatePacket` and updates displayed objects. / CODE:common/src/packets/updatePacket.ts; CODE:client/src/scripts/game.ts
- **Owner / release:** Base
- **Declared relations:** serves UR-CORE-001 (S2)
- **Onboarding authority:** GATE-BASELINE-001; GRP-CORE-001

### Trace references

| Evidence class | Target | Code | Test case | RED result | Passing result | Outcome | Environment | Fingerprint | Validity/revision |
|---|---|---|---|---|---|---|---|---|---|
| SR lower | update path | CODE:common/src/packets/updatePacket.ts | — | — | — | SKIP | — | onboarding@5f67ec7 | INHERITED_UNVERIFIED |

### Gates and delivery

- **Delivered revision:** not delivered

---

## UR-CHAR-001 — Select a playable character with distinct look and passives

- **Kind / status:** UR / PENDING_VERIFICATION
- **Statement / source:** Players choose among named character skins (Trump, Putin, Niinistö) with unique artwork and optional passive stat modifiers, persisted in loadout and used in-game. / DOC:specs/features/character-selection.md; CODE:common/src/definitions/items/skins.ts
- **Owner / release:** Base
- **Declared relations:** requires SR-CHAR-001, SR-CHAR-002, SR-CHAR-003, SR-CHAR-004, SR-CHAR-005, SR-CHAR-006
- **UR content:**
  - **Actor:** Player
  - **Context:** Main menu loadout and live match
  - **Outcome:** Character choice is visible to the player and other clients and affects stats where modifiers are defined
  - **Scenarios:**
    - **UR-CHAR-001-S1:** Given the main menu, when the player opens Loadout → Skins, then Trump, Putin, and Niinistö appear as selectable entries with character artwork.
    - **UR-CHAR-001-S2:** Given Putin is selected, when the page reloads, then `putin` remains the stored loadout skin.
    - **UR-CHAR-001-S3:** Given Trump is selected, when the player joins a game, then the server accepts the skin and all clients render the Trump sprite.
    - **UR-CHAR-001-S4:** Given a player joined as Trump, when the match runs, then max health reflects the Trump modifier relative to default baseline skin.
- **Onboarding authority:** GATE-BASELINE-001; GRP-CHAR-001

### Trace references

| Evidence class | Target | Code | Test case | RED result | Passing result | Outcome | Environment | Fingerprint | Validity/revision |
|---|---|---|---|---|---|---|---|---|---|
| UR upper | UR-CHAR-001-S1 | CODE:client/src/scripts/ui.ts | — | — | — | SKIP | — | onboarding@5f67ec7 | INHERITED_UNVERIFIED |
| UR upper | UR-CHAR-001-S4 | CODE:server/src/objects/player.ts | TEST:tests/src/skins.test.ts | — | — | SKIP | bun test | onboarding@5f67ec7 | INHERITED_UNVERIFIED |

### Gates and delivery

- **Delivered revision:** not delivered
- **Gaps / deferrals / blockers / notes:** Spec mentions Niinistö maxHealth ×0.92; as-built code sets `adrenDrain: 0.85` only (see SR-CHAR-001). Skates feature out of scope for this batch.

---

## SR-CHAR-001 — Character skins are defined with isCharacter and modifiers

- **Kind / status:** SR / PENDING_VERIFICATION
- **Statement / source:** `SkinDefinition` supports `isCharacter` and optional `modifiers`; Trump, Putin, Niinistö are appended with stable ASCII `idString`s. / CODE:common/src/definitions/items/skins.ts; TEST:tests/src/skins.test.ts
- **Owner / release:** Base
- **Declared relations:** serves UR-CHAR-001
- **SR content:**
  - **Boundary:** Definitions registry / `Skins`
  - **Behavior:** Three characters are loadout-visible; modifiers match definition table (Trump +10% health −3% speed; Putin +3% speed −5% health; Niinistö adrenDrain ×0.85 as implemented)
- **Onboarding authority:** GATE-BASELINE-001; GRP-CHAR-001

### Trace references

| Evidence class | Target | Code | Test case | RED result | Passing result | Outcome | Environment | Fingerprint | Validity/revision |
|---|---|---|---|---|---|---|---|---|---|
| SR lower | definitions | CODE:common/src/definitions/items/skins.ts | TEST:tests/src/skins.test.ts | — | RUN:bun test tests/src/skins.test.ts | SKIP | bun | onboarding@5f67ec7 | INHERITED_UNVERIFIED |

### Gates and delivery

- **Delivered revision:** not delivered

---

## SR-CHAR-002 — Loadout UI persists and previews character choice

- **Kind / status:** SR / PENDING_VERIFICATION
- **Statement / source:** Loadout skins tab lists non-hidden skins; character entries show modifier hint; selection stored in `cv_loadout_skin`. / CODE:client/src/scripts/ui.ts; CODE:client/src/scripts/console/variables.ts
- **Owner / release:** Base
- **Declared relations:** serves UR-CHAR-001 (S1, S2)
- **Onboarding authority:** GATE-BASELINE-001; GRP-CHAR-001

### Trace references

| Evidence class | Target | Code | Test case | RED result | Passing result | Outcome | Environment | Fingerprint | Validity/revision |
|---|---|---|---|---|---|---|---|---|---|
| SR lower | loadout UI | CODE:client/src/scripts/ui.ts | — | — | — | SKIP | — | onboarding@5f67ec7 | INHERITED_UNVERIFIED |

### Gates and delivery

- **Delivered revision:** not delivered

---

## SR-CHAR-003 — Server validates join skin and falls back when invalid

- **Kind / status:** SR / PENDING_VERIFICATION
- **Statement / source:** Join path rejects unknown, hidden, or role-locked skins and uses default skin; valid skins assign `player.loadout.skin`. / CODE:server/src/game.ts; CODE:server/src/server.ts; TEST:tests/src/skins.test.ts
- **Owner / release:** Base
- **Declared relations:** serves UR-CHAR-001 (S3)
- **Onboarding authority:** GATE-BASELINE-001; GRP-CHAR-001

### Trace references

| Evidence class | Target | Code | Test case | RED result | Passing result | Outcome | Environment | Fingerprint | Validity/revision |
|---|---|---|---|---|---|---|---|---|---|
| SR lower | join validation | CODE:server/src/game.ts | TEST:tests/src/skins.test.ts | — | — | SKIP | — | onboarding@5f67ec7 | INHERITED_UNVERIFIED |

### Gates and delivery

- **Delivered revision:** not delivered

---

## SR-CHAR-004 — Server merges skin modifiers into player modifiers

- **Kind / status:** SR / PENDING_VERIFICATION
- **Statement / source:** `updateAndApplyModifiers()` multiplies additive skin `modifiers` into computed player modifiers; join recomputes when skin has modifiers. / CODE:server/src/objects/player.ts; CODE:server/src/game.ts
- **Owner / release:** Base
- **Declared relations:** serves UR-CHAR-001 (S4)
- **Onboarding authority:** GATE-BASELINE-001; GRP-CHAR-001

### Trace references

| Evidence class | Target | Code | Test case | RED result | Passing result | Outcome | Environment | Fingerprint | Validity/revision |
|---|---|---|---|---|---|---|---|---|---|
| SR lower | modifier merge | CODE:server/src/objects/player.ts | TEST:tests/src/skins.test.ts | — | — | SKIP | — | onboarding@5f67ec7 | INHERITED_UNVERIFIED |

### Gates and delivery

- **Delivered revision:** not delivered

---

## SR-CHAR-005 — Random skin pools exclude characters

- **Kind / status:** SR / PENDING_VERIFICATION
- **Statement / source:** Filters that pick random skins exclude `isCharacter` skins (e.g. Halloween random skin pool). / CODE:server/src/objects/player.ts; TEST:tests/src/skins.test.ts
- **Owner / release:** Base
- **Declared relations:** serves UR-CHAR-001
- **Onboarding authority:** GATE-BASELINE-001; GRP-CHAR-001

### Trace references

| Evidence class | Target | Code | Test case | RED result | Passing result | Outcome | Environment | Fingerprint | Validity/revision |
|---|---|---|---|---|---|---|---|---|---|
| SR lower | random pool | CODE:server/src/objects/player.ts | TEST:tests/src/skins.test.ts | — | RUN:bun test tests/src/skins.test.ts | SKIP | bun | onboarding@5f67ec7 | INHERITED_UNVERIFIED |

### Gates and delivery

- **Delivered revision:** not delivered

---

## SR-CHAR-006 — Character skins serialize to other clients

- **Kind / status:** SR / PENDING_VERIFICATION
- **Statement / source:** Player skin is written on the wire via `Skins.writeToStream` in object serialization; client player renderer applies skin assets. / CODE:common/src/utils/objectsSerializations.ts; CODE:client/src/scripts/objects/player.ts
- **Owner / release:** Base
- **Declared relations:** serves UR-CHAR-001 (S3)
- **Onboarding authority:** GATE-BASELINE-001; GRP-CHAR-001

### Trace references

| Evidence class | Target | Code | Test case | RED result | Passing result | Outcome | Environment | Fingerprint | Validity/revision |
|---|---|---|---|---|---|---|---|---|---|
| SR lower | network + render | CODE:common/src/utils/objectsSerializations.ts | — | — | — | SKIP | — | onboarding@5f67ec7 | INHERITED_UNVERIFIED |

### Gates and delivery

- **Delivered revision:** not delivered

---

# Full codebase sweep (GRP-DEF-001 … GRP-TST-001) — same run SUROI-ONBOARD-2026-10-06

## UR-DEF-001 — Static definitions describe all spawnable content

- **Kind / status:** UR / PENDING_VERIFICATION
- **Statement / source:** Guns, obstacles, buildings, items, modes, and cosmetics are declared in TypeScript definition arrays and consumed by server and client. / DOC:docs/datamodel.md; DOC:docs/subsystems/definitions/README.md
- **Declared relations:** requires SR-DEF-001, SR-DEF-002, SR-DEF-003
- **UR content:** Actor: Content author / build pipeline. Outcome: Invalid content fails validation before release; valid content resolves by `idString` and wire index.
- **Scenarios:** UR-DEF-001-S1: When `bun validateDefinitions` runs, then all definition families and map/loot/gas data pass schema checks.
- **Onboarding authority:** GATE-BASELINE-001; GRP-DEF-001

### Trace references
| Evidence class | Target | Code | Test case | Outcome | Validity |
|---|---|---|---|---|---|
| UR upper | S1 | TEST:tests/src/validateDefinitions.ts | RUN:bun validateDefinitions | SKIP | INHERITED_UNVERIFIED |

### Gates and delivery
- **Delivered revision:** not delivered

---

## SR-DEF-001 — ObjectDefinitions schema and DefinitionType discrimination

- **Kind / status:** SR / PENDING_VERIFICATION
- **Statement / source:** `ObjectDefinitions<T>` assigns stable indices; `DefinitionType` drives inventory and serialization behavior. / CODE:common/src/utils/objectDefinitions.ts
- **Declared relations:** serves UR-DEF-001
- **Onboarding authority:** GRP-DEF-001

### Trace references
| Evidence class | Target | Code | Test case | Outcome | Validity |
|---|---|---|---|---|---|
| SR lower | schema | CODE:common/src/utils/objectDefinitions.ts | TEST:tests/src/validateDefinitions.ts | SKIP | INHERITED_UNVERIFIED |

### Gates and delivery
- **Delivered revision:** not delivered

---

## SR-DEF-002 — Loot and item families aggregate into Loots registry

- **Kind / status:** SR / PENDING_VERIFICATION
- **Statement / source:** `loots.ts` and `common/src/definitions/items/` export registries used at runtime. / CODE:common/src/definitions/loots.ts
- **Declared relations:** serves UR-DEF-001
- **Onboarding authority:** GRP-DEF-001

### Gates and delivery
- **Delivered revision:** not delivered

---

## SR-DEF-003 — ReferenceTo reify resolves cross-definition links

- **Kind / status:** SR / PENDING_VERIFICATION
- **Statement / source:** Invalid `ReferenceTo` targets fail validation or reify. / CODE:common/src/utils/objectDefinitions.ts; TEST:tests/src/validateDefinitions.ts
- **Declared relations:** serves UR-DEF-001
- **Onboarding authority:** GRP-DEF-001

### Gates and delivery
- **Delivered revision:** not delivered

---

## UR-PKT-001 — Binary PacketStream multiplexes gameplay messages

- **Kind / status:** UR / PENDING_VERIFICATION
- **Statement / source:** Client and game worker exchange typed packets over `/play` WebSocket. / DOC:docs/subsystems/packets/README.md; DOC:docs/protocol.md
- **Declared relations:** requires SR-PKT-001, SR-PKT-002, SR-PKT-003
- **UR content:** Scenarios: S1 join sends JoinPacket and receives Joined+Map; S2 each tick InputPacket out and UpdatePacket in.
- **Onboarding authority:** GRP-PKT-001

### Gates and delivery
- **Delivered revision:** not delivered

---

## SR-PKT-001 — PacketStream serializes type index and packet body

- **Kind / status:** SR / PENDING_VERIFICATION
- **Statement / source:** First byte selects handler from `Packets` array. / CODE:common/src/packets/packetStream.ts; CODE:common/src/packets/packet.ts
- **Declared relations:** serves UR-PKT-001
- **Onboarding authority:** GRP-PKT-001

### Gates and delivery
- **Delivered revision:** not delivered

---

## SR-PKT-002 — SuroiByteStream encodes compact fields

- **Kind / status:** SR / PENDING_VERIFICATION
- **Statement / source:** Game-specific binary helpers built on ByteStream. / CODE:common/src/utils/suroiByteStream.ts; CODE:common/src/utils/byteStream.ts
- **Declared relations:** serves UR-PKT-001
- **Onboarding authority:** GRP-PKT-001

### Gates and delivery
- **Delivered revision:** not delivered

---

## SR-PKT-003 — Join and update packets carry handshake and world deltas

- **Kind / status:** SR / PENDING_VERIFICATION
- **Statement / source:** JoinPacket/JoinedPacket/MapPacket and UpdatePacket implement join and tick sync. / CODE:common/src/packets/joinPacket.ts; CODE:common/src/packets/updatePacket.ts
- **Declared relations:** serves UR-PKT-001
- **Onboarding authority:** GRP-PKT-001

### Gates and delivery
- **Delivered revision:** not delivered

---

## UR-OBJ-001 — World entities sync via ObjectCategory and net IDs

- **Kind / status:** UR / PENDING_VERIFICATION
- **Statement / source:** Players, obstacles, loot, and other categories share templates and partial/full serialization. / DOC:docs/subsystems/objects/README.md
- **Declared relations:** requires SR-OBJ-001, SR-OBJ-002, SR-OBJ-003, SR-OBJ-004
- **Onboarding authority:** GRP-OBJ-001

### Gates and delivery
- **Delivered revision:** not delivered

---

## SR-OBJ-001 — makeGameObjectTemplate derives server and client classes

- **Kind / status:** SR / PENDING_VERIFICATION
- **Statement / source:** Shared template pattern in common. / CODE:common/src/utils/gameObject.ts
- **Declared relations:** serves UR-OBJ-001
- **Onboarding authority:** GRP-OBJ-001

### Gates and delivery
- **Delivered revision:** not delivered

---

## SR-OBJ-002 — ObjectsNetData serializes per-category fields

- **Kind / status:** SR / PENDING_VERIFICATION
- **Statement / source:** Wire layouts for each ObjectCategory. / CODE:common/src/utils/objectsSerializations.ts
- **Declared relations:** serves UR-OBJ-001
- **Onboarding authority:** GRP-OBJ-001

### Gates and delivery
- **Delivered revision:** not delivered

---

## SR-OBJ-003 — Server ObjectMapping runs authoritative simulation

- **Kind / status:** SR / PENDING_VERIFICATION
- **Statement / source:** Server game objects in grid with damage and streams. / CODE:server/src/objects/gameObject.ts; CODE:server/src/utils/grid.ts
- **Declared relations:** serves UR-OBJ-001
- **Onboarding authority:** GRP-OBJ-001

### Gates and delivery
- **Delivered revision:** not delivered

---

## SR-OBJ-004 — Client ObjectClassMapping renders Pixi GameObjects

- **Kind / status:** SR / PENDING_VERIFICATION
- **Statement / source:** Client mirrors net data into display objects. / CODE:client/src/scripts/game.ts; CODE:client/src/scripts/objects/gameObject.ts
- **Declared relations:** serves UR-OBJ-001
- **Onboarding authority:** GRP-OBJ-001

### Gates and delivery
- **Delivered revision:** not delivered

---

## UR-LOOP-001 — Match runs at fixed TPS until win or empty

- **Kind / status:** UR / PENDING_VERIFICATION
- **Statement / source:** Game loop advances simulation, gas, and broadcasts updates. / DOC:docs/subsystems/game-loop/README.md
- **Declared relations:** requires SR-LOOP-001, SR-LOOP-002, SR-LOOP-003, SR-LOOP-004
- **UR content:** Scenario: When minimum players met, match starts countdown, gas begins, one winner or last team standing ends game.
- **Onboarding authority:** GRP-LOOP-001

### Gates and delivery
- **Delivered revision:** not delivered

---

## SR-LOOP-001 — Game.tick uses scheduled 40 TPS loop

- **Kind / status:** SR / PENDING_VERIFICATION
- **Statement / source:** Recursive setTimeout with ideal dt compensation. / CODE:server/src/game.ts; CODE:server/src/utils/config.ts
- **Declared relations:** serves UR-LOOP-001
- **Onboarding authority:** GRP-LOOP-001

### Gates and delivery
- **Delivered revision:** not delivered

---

## SR-LOOP-002 — GameManager forks cluster workers per game

- **Kind / status:** SR / PENDING_VERIFICATION
- **Statement / source:** Worker hosts `/play` and one Game instance. / CODE:server/src/gameManager.ts; DOC:docs/multiplayer.md
- **Declared relations:** serves UR-LOOP-001
- **Onboarding authority:** GRP-LOOP-001

### Gates and delivery
- **Delivered revision:** not delivered

---

## SR-LOOP-003 — Grid spatial hash queries nearby objects

- **Kind / status:** SR / PENDING_VERIFICATION
- **Statement / source:** Collision and visibility use grid buckets. / CODE:server/src/utils/grid.ts
- **Declared relations:** serves UR-LOOP-001
- **Onboarding authority:** GRP-LOOP-001

### Gates and delivery
- **Delivered revision:** not delivered

---

## SR-LOOP-004 — IDAllocator assigns reusable object network IDs

- **Kind / status:** SR / PENDING_VERIFICATION
- **Statement / source:** 16-bit pools for game objects. / CODE:server/src/utils/idAllocator.ts
- **Declared relations:** serves UR-OBJ-001, UR-LOOP-001
- **Onboarding authority:** GRP-LOOP-001

### Gates and delivery
- **Delivered revision:** not delivered

---

## UR-INV-001 — Player manages weapons, gear, and timed actions

- **Kind / status:** UR / PENDING_VERIFICATION
- **Statement / source:** Pickup, swap, reload, heal, revive via inventory subsystem. / DOC:docs/subsystems/inventory/README.md
- **Declared relations:** requires SR-INV-001, SR-INV-002, SR-INV-003
- **Onboarding authority:** GRP-INV-001

### Gates and delivery
- **Delivered revision:** not delivered

---

## SR-INV-001 — Inventory slots, armor, backpack, and scope

- **Kind / status:** SR / PENDING_VERIFICATION
- **Statement / source:** Core inventory state on Player. / CODE:server/src/inventory/inventory.ts
- **Declared relations:** serves UR-INV-001
- **Onboarding authority:** GRP-INV-001

### Gates and delivery
- **Delivered revision:** not delivered

---

## SR-INV-002 — GunItem, MeleeItem, ThrowableItem implement combat items

- **Kind / status:** SR / PENDING_VERIFICATION
- **Statement / source:** DefinitionType maps to item classes. / CODE:server/src/inventory/gunItem.ts; CODE:server/src/inventory/meleeItem.ts; CODE:server/src/inventory/throwableItem.ts
- **Declared relations:** serves UR-INV-001
- **Onboarding authority:** GRP-INV-001

### Gates and delivery
- **Delivered revision:** not delivered

---

## SR-INV-003 — Action subclasses gate reload, heal, revive timing

- **Kind / status:** SR / PENDING_VERIFICATION
- **Statement / source:** Timed actions via game timeouts. / CODE:server/src/inventory/action.ts
- **Declared relations:** serves UR-INV-001
- **Onboarding authority:** GRP-INV-001

### Gates and delivery
- **Delivered revision:** not delivered

---

## UR-MAP-001 — Each match gets a generated map with loot placement

- **Kind / status:** UR / PENDING_VERIFICATION
- **Statement / source:** Procedural terrain, structures, and cached MapPacket. / DOC:docs/subsystems/map/README.md
- **Declared relations:** requires SR-MAP-001, SR-MAP-002, SR-MAP-003
- **Onboarding authority:** GRP-MAP-001

### Gates and delivery
- **Delivered revision:** not delivered

---

## SR-MAP-001 — GameMap generates world and MapPacket buffer

- **Kind / status:** SR / PENDING_VERIFICATION
- **Statement / source:** Seeded generation and join buffer. / CODE:server/src/map.ts
- **Declared relations:** serves UR-MAP-001
- **Onboarding authority:** GRP-MAP-001

### Gates and delivery
- **Delivered revision:** not delivered

---

## SR-MAP-002 — Maps registry defines dimensions and onGenerate hooks

- **Kind / status:** SR / PENDING_VERIFICATION
- **Statement / source:** Map names, rivers, clumps, mode linkage. / CODE:server/src/data/maps.ts
- **Declared relations:** serves UR-MAP-001
- **Onboarding authority:** GRP-MAP-001

### Gates and delivery
- **Delivered revision:** not delivered

---

## SR-MAP-003 — Loot tables and helpers roll weighted spawns

- **Kind / status:** SR / PENDING_VERIFICATION
- **Statement / source:** Named tables validated with definitions. / CODE:server/src/data/lootTables.ts; CODE:server/src/utils/lootHelpers.ts
- **Declared relations:** serves UR-MAP-001
- **Onboarding authority:** GRP-MAP-001

### Gates and delivery
- **Delivered revision:** not delivered

---

## UR-GAS-001 — Safe zone shrinks and damages players outside

- **Kind / status:** UR / PENDING_VERIFICATION
- **Statement / source:** Battle royale circle staged by gas timeline. / DOC:docs/subsystems/gas/README.md
- **Declared relations:** requires SR-GAS-001, SR-GAS-002
- **Onboarding authority:** GRP-GAS-001

### Gates and delivery
- **Delivered revision:** not delivered

---

## SR-GAS-001 — Gas advances stages and applies scaled damage

- **Kind / status:** SR / PENDING_VERIFICATION
- **Statement / source:** Gas.tick, advanceGasStage, isInGas. / CODE:server/src/gas.ts; CODE:server/src/data/gasStages.ts
- **Declared relations:** serves UR-GAS-001
- **Onboarding authority:** GRP-GAS-001

### Gates and delivery
- **Delivered revision:** not delivered

---

## SR-GAS-002 — Client gasManager visualizes ring from update data

- **Kind / status:** SR / PENDING_VERIFICATION
- **Statement / source:** Overlay synced from server gas fields. / CODE:client/src/scripts/managers/gasManager.ts
- **Declared relations:** serves UR-GAS-001
- **Onboarding authority:** GRP-GAS-001

### Gates and delivery
- **Delivered revision:** not delivered

---

## UR-PLG-001 — Configurable plugins extend game lifecycle events

- **Kind / status:** UR / PENDING_VERIFICATION
- **Statement / source:** Event emitters on Game for join, combat, tick. / DOC:docs/subsystems/plugins/README.md
- **Declared relations:** requires SR-PLG-001, SR-PLG-002
- **Onboarding authority:** GRP-PLG-001

### Gates and delivery
- **Delivered revision:** not delivered

---

## SR-PLG-001 — PluginManager loads config.plugins and emits Events

- **Kind / status:** SR / PENDING_VERIFICATION
- **Statement / source:** Dynamic import of server/src/plugins/*. / CODE:server/src/pluginManager.ts
- **Declared relations:** serves UR-PLG-001
- **Onboarding authority:** GRP-PLG-001

### Gates and delivery
- **Delivered revision:** not delivered

---

## SR-PLG-002 — Bundled plugins hook kills, teleports, dev tools

- **Kind / status:** SR / PENDING_VERIFICATION
- **Statement / source:** weaponSwap, teleport, speedToggle, placeObject plugins. / CODE:server/src/plugins/
- **Declared relations:** serves UR-PLG-001
- **Onboarding authority:** GRP-PLG-001

### Gates and delivery
- **Delivered revision:** not delivered

---

## UR-MP-001 — Lobby discovery, teams, and routed game connections

- **Kind / status:** UR / PENDING_VERIFICATION
- **Statement / source:** Primary server HTTP/JSON team lobby; workers on offset ports. / DOC:docs/multiplayer.md
- **Declared relations:** requires SR-MP-001, SR-MP-002, SR-MP-003
- **UR content:** Scenario: Player fetches server info, optionally joins custom team via JSON WS, connects to worker `/play` on computed port.
- **Onboarding authority:** GRP-MP-001

### Gates and delivery
- **Delivered revision:** not delivered

---

## SR-MP-001 — Primary exposes serverInfo and getGame HTTP APIs

- **Kind / status:** SR / PENDING_VERIFICATION
- **Statement / source:** Discovery endpoints on default port. / CODE:server/src/server.ts
- **Declared relations:** serves UR-MP-001
- **Onboarding authority:** GRP-MP-001

### Gates and delivery
- **Delivered revision:** not delivered

---

## SR-MP-002 — Custom team lobby uses JSON WebSocket on /team

- **Kind / status:** SR / PENDING_VERIFICATION
- **Statement / source:** Team.ts coordinates lobby vs in-game Team. / CODE:server/src/team.ts; CODE:client/src/scripts/ui.ts
- **Declared relations:** serves UR-MP-001
- **Onboarding authority:** GRP-MP-001

### Gates and delivery
- **Delivered revision:** not delivered

---

## SR-MP-003 — Worker listen port is Config.port + gameId + 1

- **Kind / status:** SR / PENDING_VERIFICATION
- **Statement / source:** Client region offset must match for local dev. / DOC:docs/multiplayer.md; CODE:client/src/scripts/config.ts
- **Declared relations:** serves UR-MP-001, UR-CORE-001
- **Onboarding authority:** GRP-MP-001

### Gates and delivery
- **Delivered revision:** not delivered

---

## UR-CLI-001 — Browser client bootstraps Pixi game and WebSocket session

- **Kind / status:** UR / PENDING_VERIFICATION
- **Statement / source:** Vite entry initializes Game singleton. / DOC:docs/subsystems/client/README.md
- **Declared relations:** requires SR-CLI-001, SR-CLI-002
- **Onboarding authority:** GRP-CLI-001

### Gates and delivery
- **Delivered revision:** not delivered

---

## SR-CLI-001 — Game class owns Application, pools, and packet dispatch

- **Kind / status:** SR / PENDING_VERIFICATION
- **Statement / source:** Central client runtime loop. / CODE:client/src/scripts/game.ts
- **Declared relations:** serves UR-CLI-001, UR-CORE-001
- **Onboarding authority:** GRP-CLI-001

### Gates and delivery
- **Delivered revision:** not delivered

---

## SR-CLI-002 — Sound and screen-record managers attach to Game

- **Kind / status:** SR / PENDING_VERIFICATION
- **Statement / source:** Audio and optional capture tooling. / CODE:client/src/scripts/managers/soundManager.ts; CODE:client/src/scripts/managers/screenRecordManager.ts
- **Declared relations:** serves UR-CLI-001
- **Onboarding authority:** GRP-CLI-001

### Gates and delivery
- **Delivered revision:** not delivered

---

## UR-REN-001 — World state renders as layered Pixi display objects

- **Kind / status:** UR / PENDING_VERIFICATION
- **Statement / source:** Sprites, camera, minimap, particles. / DOC:docs/subsystems/rendering/README.md
- **Declared relations:** requires SR-REN-001, SR-REN-002, SR-REN-003
- **Onboarding authority:** GRP-REN-001

### Gates and delivery
- **Delivered revision:** not delivered

---

## SR-REN-001 — CameraManager stacks world layers and zoom

- **Kind / status:** SR / PENDING_VERIFICATION
- **Statement / source:** Camera shake, scope zoom, container hierarchy. / CODE:client/src/scripts/managers/cameraManager.ts
- **Declared relations:** serves UR-REN-001
- **Onboarding authority:** GRP-REN-001

### Gates and delivery
- **Delivered revision:** not delivered

---

## SR-REN-002 — MapManager draws terrain and minimap

- **Kind / status:** SR / PENDING_VERIFICATION
- **Statement / source:** Map assets from MapPacket and mode. / CODE:client/src/scripts/managers/mapManager.ts
- **Declared relations:** serves UR-REN-001
- **Onboarding authority:** GRP-REN-001

### Gates and delivery
- **Delivered revision:** not delivered

---

## SR-REN-003 — SuroiSprite and pixi utils load spritesheets

- **Kind / status:** SR / PENDING_VERIFICATION
- **Statement / source:** Shared Pixi helpers and scale. / CODE:client/src/scripts/utils/pixi.ts
- **Declared relations:** serves UR-REN-001
- **Onboarding authority:** GRP-REN-001

### Gates and delivery
- **Delivered revision:** not delivered

---

## UR-INP-001 — Player controls movement, aim, and actions from devices

- **Kind / status:** UR / PENDING_VERIFICATION
- **Statement / source:** Keyboard, mouse, mobile joysticks → InputPacket. / DOC:docs/subsystems/input/README.md
- **Declared relations:** requires SR-INP-001, SR-INP-002
- **Onboarding authority:** GRP-INP-001

### Gates and delivery
- **Delivered revision:** not delivered

---

## SR-INP-001 — InputManager maps binds to InputActions and packets

- **Kind / status:** SR / PENDING_VERIFICATION
- **Statement / source:** Bi-map from console binds to game actions. / CODE:client/src/scripts/managers/inputManager.ts
- **Declared relations:** serves UR-INP-001, UR-CORE-001
- **Onboarding authority:** GRP-INP-001

### Gates and delivery
- **Delivered revision:** not delivered

---

## SR-INP-002 — Emote and map-ping radial wheels

- **Kind / status:** SR / PENDING_VERIFICATION
- **Statement / source:** EmoteWheelManager and MapPingWheelManager. / CODE:client/src/scripts/managers/emoteWheelManager.ts
- **Declared relations:** serves UR-INP-001
- **Onboarding authority:** GRP-INP-001

### Gates and delivery
- **Delivered revision:** not delivered

---

## UR-UI-001 — Menus, HUD, and localized strings

- **Kind / status:** UR / PENDING_VERIFICATION
- **Statement / source:** Splash, loadout, in-game HUD, HJSON i18n. / DOC:docs/subsystems/ui/README.md
- **Declared relations:** requires SR-UI-001, SR-UI-002, SR-UI-003
- **Onboarding authority:** GRP-UI-001

### Gates and delivery
- **Delivered revision:** not delivered

---

## SR-UI-001 — ui.ts wires splash, regions, and play flow

- **Kind / status:** SR / PENDING_VERIFICATION
- **Statement / source:** DOM lobby connects to Game. / CODE:client/src/scripts/ui.ts; CODE:client/index.html
- **Declared relations:** serves UR-UI-001, UR-MP-001
- **Onboarding authority:** GRP-UI-001

### Gates and delivery
- **Delivered revision:** not delivered

---

## SR-UI-002 — UIManager renders in-game HUD and inventory display

- **Kind / status:** SR / PENDING_VERIFICATION
- **Statement / source:** Health, ammo, kill feed overlays. / CODE:client/src/scripts/managers/uiManager.ts
- **Declared relations:** serves UR-UI-001
- **Onboarding authority:** GRP-UI-001

### Gates and delivery
- **Delivered revision:** not delivered

---

## SR-UI-003 — Translations manifest loads locale HJSON at build time

- **Kind / status:** SR / PENDING_VERIFICATION
- **Statement / source:** virtual:translations-manifest plugin. / CODE:client/vite/plugins/translations-plugin.ts; CODE:client/src/scripts/utils/translations/translations.ts
- **Declared relations:** serves UR-UI-001
- **Onboarding authority:** GRP-UI-001

### Gates and delivery
- **Delivered revision:** not delivered

---

## UR-CON-001 — In-game console configures client via CVars and commands

- **Kind / status:** UR / PENDING_VERIFICATION
- **Statement / source:** Quake-style console, binds, persisted settings. / DOC:docs/subsystems/console/README.md
- **Declared relations:** requires SR-CON-001, SR-CON-002
- **Onboarding authority:** GRP-CON-001

### Gates and delivery
- **Delivered revision:** not delivered

---

## SR-CON-001 — GameConsole executes commands and persists GameSettings

- **Kind / status:** SR / PENDING_VERIFICATION
- **Statement / source:** Floating window UI and query eval. / CODE:client/src/scripts/console/gameConsole.ts
- **Declared relations:** serves UR-CON-001
- **Onboarding authority:** GRP-CON-001

### Gates and delivery
- **Delivered revision:** not delivered

---

## SR-CON-002 — CVars and defaultBinds drive input and rendering toggles

- **Kind / status:** SR / PENDING_VERIFICATION
- **Statement / source:** cv_, pf_, db_ prefixes and loadout CVars. / CODE:client/src/scripts/console/variables.ts; CODE:client/src/scripts/console/commands.ts
- **Declared relations:** serves UR-CON-001, UR-INP-001
- **Onboarding authority:** GRP-CON-001

### Gates and delivery
- **Delivered revision:** not delivered

---

## UR-TST-001 — Offline gates validate definitions, assets, and shared math

- **Kind / status:** UR / PENDING_VERIFICATION
- **Statement / source:** tests package runs before CI merge. / DOC:docs/subsystems/testing/README.md
- **Declared relations:** requires SR-TST-001, SR-TST-002, SR-TST-003, SR-TST-004
- **UR content:** Scenario: validateDefinitions and validateSvgs exit 0 on main branch content.
- **Onboarding authority:** GRP-TST-001

### Trace references
| Evidence class | Target | Code | Test case | Outcome | Validity |
|---|---|---|---|---|---|
| UR upper | validation | TEST:tests/src/validateDefinitions.ts | RUN:bun validateDefinitions | SKIP | INHERITED_UNVERIFIED |
| UR upper | svgs | TEST:tests/src/validateSvgs.ts | RUN:bun validateSvgs | SKIP | INHERITED_UNVERIFIED |

### Gates and delivery
- **Delivered revision:** not delivered

---

## SR-TST-001 — validateDefinitions checks definitions, maps, loot, gas, regions

- **Kind / status:** SR / PENDING_VERIFICATION
- **Statement / source:** Monolithic schema checker. / CODE:tests/src/validateDefinitions.ts; CODE:tests/src/validationUtils.ts
- **Declared relations:** serves UR-TST-001, UR-DEF-001
- **Onboarding authority:** GRP-TST-001

### Trace references
| Evidence class | Target | Test case | Outcome | Validity |
|---|---|---|---|---|
| SR lower | RUN:bun validateDefinitions | PASS (2026-10-06) | INHERITED_UNVERIFIED |

### Gates and delivery
- **Delivered revision:** not delivered

---

## SR-TST-002 — validateSvgs enforces SVG size and structure rules

- **Kind / status:** SR / PENDING_VERIFICATION
- **Statement / source:** Game and killfeed SVG audit. / CODE:tests/src/validateSvgs.ts
- **Declared relations:** serves UR-TST-001
- **Onboarding authority:** GRP-TST-001

### Trace references
| Evidence class | Target | Test case | Outcome | Validity |
|---|---|---|---|---|
| SR lower | RUN:bun validateSvgs | PASS (2026-10-06) | INHERITED_UNVERIFIED |

### Gates and delivery
- **Delivered revision:** not delivered

---

## SR-TST-003 — Bun unit tests cover shared math utilities

- **Kind / status:** SR / PENDING_VERIFICATION
- **Statement / source:** math.test.ts for line intersection. / CODE:tests/src/math.test.ts; CODE:common/src/utils/math.ts
- **Declared relations:** serves UR-TST-001
- **Onboarding authority:** GRP-TST-001

### Trace references
| Evidence class | Target | Test case | Outcome | Validity |
|---|---|---|---|---|
| SR lower | RUN:bun test tests/src/math.test.ts | PASS 8 tests | INHERITED_UNVERIFIED |

### Gates and delivery
- **Delivered revision:** not delivered

---

## SR-TST-004 — stressTest drives multi-bot WebSocket load (manual)

- **Kind / status:** SR / PENDING_VERIFICATION
- **Statement / source:** Requires live server; not CI-default. / CODE:tests/src/stressTest.ts
- **Declared relations:** serves UR-TST-001
- **Onboarding authority:** GRP-TST-001
- **Gaps:** No automated RUN captured in this onboarding pass.

### Gates and delivery
- **Delivered revision:** not delivered

---

## UR-BAL-001 — Combat uses server ballistics and explosions

- **Kind / status:** UR / PENDING_VERIFICATION
- **Statement / source:** Guns fire bullets; explosions deal area damage. / DOC:docs/subsystems/objects/modules/ballistics.md
- **Declared relations:** requires SR-BAL-001, SR-BAL-002
- **Onboarding authority:** GRP-BAL-001

### Gates and delivery
- **Delivered revision:** not delivered

---

## SR-BAL-001 — Server bullets and projectiles simulate hits

- **Kind / status:** SR / PENDING_VERIFICATION
- **Statement / source:** bullet.ts, projectile.ts, baseBullet math. / CODE:server/src/objects/bullet.ts; CODE:common/src/utils/baseBullet.ts
- **Declared relations:** serves UR-BAL-001, UR-INV-001
- **Onboarding authority:** GRP-BAL-001

### Gates and delivery
- **Delivered revision:** not delivered

---

## SR-BAL-002 — Explosions apply area damage and sync to clients

- **Kind / status:** SR / PENDING_VERIFICATION
- **Statement / source:** Server explosion objects and client visual-only explosion.ts. / CODE:server/src/objects/explosion.ts; CODE:client/src/scripts/objects/explosion.ts
- **Declared relations:** serves UR-BAL-001
- **Onboarding authority:** GRP-BAL-001

### Gates and delivery
- **Delivered revision:** not delivered

---

## UR-MOD-001 — Game mode and map selection configure match rules

- **Kind / status:** UR / PENDING_VERIFICATION
- **Statement / source:** ModeDefinition and map/mode pairing. / CODE:common/src/definitions/modes.ts; CODE:server/src/utils/misc.ts
- **Declared relations:** requires SR-MOD-001
- **Onboarding authority:** GRP-MOD-001

### Gates and delivery
- **Delivered revision:** not delivered

---

## SR-MOD-001 — modeFromMap resolves ModeName from map string

- **Kind / status:** SR / PENDING_VERIFICATION
- **Statement / source:** Map switcher and worker env pass map name. / CODE:server/src/utils/misc.ts; CODE:server/src/gameManager.ts
- **Declared relations:** serves UR-MOD-001, UR-MAP-001
- **Onboarding authority:** GRP-MOD-001

### Gates and delivery
- **Delivered revision:** not delivered

---
# Source coverage batch (≥60% disposition target) — SUROI-ONBOARD-2026-10-06
## SR-REN-004 — Client category renderers complete ObjectClassMapping
- **Kind / status:** SR / PENDING_VERIFICATION
- **Statement / source:** As-built implementation spans 10 source files. / CODE:client/src/scripts/objects/building.ts; CODE:client/src/scripts/objects/bullet.ts; CODE:client/src/scripts/objects/deathMarker.ts; CODE:client/src/scripts/objects/decal.ts; CODE:client/src/scripts/objects/loot.ts; CODE:client/src/scripts/objects/obstacle.ts; CODE:client/src/scripts/objects/parachute.ts; CODE:client/src/scripts/objects/plane.ts; CODE:client/src/scripts/objects/projectile.ts; CODE:client/src/scripts/objects/syncedParticle.ts
- **Owner / release:** Base
- **Declared relations:** serves UR-REN-001
- **Onboarding authority:** GATE-BASELINE-001; GRP-COV-001

### Trace references
| Evidence class | Target | Code | Outcome | Validity |
|---|---|---|---|---|
| SR lower | file disposition | CODE:client/src/scripts/objects/building.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:client/src/scripts/objects/bullet.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:client/src/scripts/objects/deathMarker.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:client/src/scripts/objects/decal.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:client/src/scripts/objects/loot.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:client/src/scripts/objects/obstacle.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:client/src/scripts/objects/parachute.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:client/src/scripts/objects/plane.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:client/src/scripts/objects/projectile.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:client/src/scripts/objects/syncedParticle.ts | SKIP | INHERITED_UNVERIFIED |

### Gates and delivery
- **Delivered revision:** not delivered

---
## SR-DEF-004 — Definition content files for loot, world, and cosmetics
- **Kind / status:** SR / PENDING_VERIFICATION
- **Statement / source:** As-built implementation spans 20 source files. / CODE:common/src/definitions/badges.ts; CODE:common/src/definitions/buildings.ts; CODE:common/src/definitions/bullets.ts; CODE:common/src/definitions/decals.ts; CODE:common/src/definitions/emotes.ts; CODE:common/src/definitions/explosions.ts; CODE:common/src/definitions/items/ammos.ts; CODE:common/src/definitions/items/armors.ts; CODE:common/src/definitions/items/backpacks.ts; CODE:common/src/definitions/items/guns.ts; CODE:common/src/definitions/items/healingItems.ts; CODE:common/src/definitions/items/items.ts; CODE:common/src/definitions/items/melees.ts; CODE:common/src/definitions/items/perks.ts; CODE:common/src/definitions/items/scopes.ts; CODE:common/src/definitions/items/throwables.ts; CODE:common/src/definitions/mapIndicators.ts; CODE:common/src/definitions/mapPings.ts; CODE:common/src/definitions/obstacles.ts; CODE:common/src/definitions/syncedParticles.ts
- **Owner / release:** Base
- **Declared relations:** serves UR-DEF-001
- **Onboarding authority:** GATE-BASELINE-001; GRP-COV-002

### Trace references
| Evidence class | Target | Code | Outcome | Validity |
|---|---|---|---|---|
| SR lower | file disposition | CODE:common/src/definitions/badges.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:common/src/definitions/buildings.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:common/src/definitions/bullets.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:common/src/definitions/decals.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:common/src/definitions/emotes.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:common/src/definitions/explosions.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:common/src/definitions/items/ammos.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:common/src/definitions/items/armors.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:common/src/definitions/items/backpacks.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:common/src/definitions/items/guns.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:common/src/definitions/items/healingItems.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:common/src/definitions/items/items.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:common/src/definitions/items/melees.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:common/src/definitions/items/perks.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:common/src/definitions/items/scopes.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:common/src/definitions/items/throwables.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:common/src/definitions/mapIndicators.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:common/src/definitions/mapPings.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:common/src/definitions/obstacles.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:common/src/definitions/syncedParticles.ts | SKIP | INHERITED_UNVERIFIED |

### Gates and delivery
- **Delivered revision:** not delivered

---
## SR-PKT-004 — Gameplay packet types beyond join/input/update
- **Kind / status:** SR / PENDING_VERIFICATION
- **Statement / source:** As-built implementation spans 8 source files. / CODE:common/src/packets/debugPacket.ts; CODE:common/src/packets/gameOverPacket.ts; CODE:common/src/packets/joinedPacket.ts; CODE:common/src/packets/killPacket.ts; CODE:common/src/packets/mapPacket.ts; CODE:common/src/packets/pickupPacket.ts; CODE:common/src/packets/reportPacket.ts; CODE:common/src/packets/spectatePacket.ts
- **Owner / release:** Base
- **Declared relations:** serves UR-PKT-001
- **Onboarding authority:** GATE-BASELINE-001; GRP-COV-003

### Trace references
| Evidence class | Target | Code | Outcome | Validity |
|---|---|---|---|---|
| SR lower | file disposition | CODE:common/src/packets/debugPacket.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:common/src/packets/gameOverPacket.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:common/src/packets/joinedPacket.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:common/src/packets/killPacket.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:common/src/packets/mapPacket.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:common/src/packets/pickupPacket.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:common/src/packets/reportPacket.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:common/src/packets/spectatePacket.ts | SKIP | INHERITED_UNVERIFIED |

### Gates and delivery
- **Delivered revision:** not delivered

---
## SR-UTL-001 — Shared utilities for math, terrain, hitbox, and helpers
- **Kind / status:** SR / PENDING_VERIFICATION
- **Statement / source:** As-built implementation spans 13 source files. / CODE:common/src/defaultInventory.ts; CODE:common/src/typings.ts; CODE:common/src/utils/gameHelpers.ts; CODE:common/src/utils/hitbox.ts; CODE:common/src/utils/isClient.ts; CODE:common/src/utils/layer.ts; CODE:common/src/utils/logging.ts; CODE:common/src/utils/misc.ts; CODE:common/src/utils/objectPool.ts; CODE:common/src/utils/random.ts; CODE:common/src/utils/readDirectory.ts; CODE:common/src/utils/terrain.ts; CODE:common/src/utils/vector.ts
- **Owner / release:** Base
- **Declared relations:** serves UR-DEF-001
- **Onboarding authority:** GATE-BASELINE-001; GRP-COV-004

### Trace references
| Evidence class | Target | Code | Outcome | Validity |
|---|---|---|---|---|
| SR lower | file disposition | CODE:common/src/defaultInventory.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:common/src/typings.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:common/src/utils/gameHelpers.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:common/src/utils/hitbox.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:common/src/utils/isClient.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:common/src/utils/layer.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:common/src/utils/logging.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:common/src/utils/misc.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:common/src/utils/objectPool.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:common/src/utils/random.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:common/src/utils/readDirectory.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:common/src/utils/terrain.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:common/src/utils/vector.ts | SKIP | INHERITED_UNVERIFIED |

### Gates and delivery
- **Delivered revision:** not delivered

---
## SR-OBJ-005 — Server simulated world object classes
- **Kind / status:** SR / PENDING_VERIFICATION
- **Statement / source:** As-built implementation spans 11 source files. / CODE:server/src/inventory/inventoryItem.ts; CODE:server/src/objects/building.ts; CODE:server/src/objects/deathMarker.ts; CODE:server/src/objects/decal.ts; CODE:server/src/objects/emote.ts; CODE:server/src/objects/loot.ts; CODE:server/src/objects/mapIndicator.ts; CODE:server/src/objects/obstacle.ts; CODE:server/src/objects/parachute.ts; CODE:server/src/objects/projectile.ts; CODE:server/src/objects/syncedParticle.ts
- **Owner / release:** Base
- **Declared relations:** serves UR-OBJ-001
- **Onboarding authority:** GATE-BASELINE-001; GRP-COV-005

### Trace references
| Evidence class | Target | Code | Outcome | Validity |
|---|---|---|---|---|
| SR lower | file disposition | CODE:server/src/inventory/inventoryItem.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:server/src/objects/building.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:server/src/objects/deathMarker.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:server/src/objects/decal.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:server/src/objects/emote.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:server/src/objects/loot.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:server/src/objects/mapIndicator.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:server/src/objects/obstacle.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:server/src/objects/parachute.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:server/src/objects/projectile.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:server/src/objects/syncedParticle.ts | SKIP | INHERITED_UNVERIFIED |

### Gates and delivery
- **Delivered revision:** not delivered

---
## SR-CLI-003 — Client managers, console internals, and debug utilities
- **Kind / status:** SR / PENDING_VERIFICATION
- **Statement / source:** As-built implementation spans 14 source files. / CODE:client/src/scripts/console/internals.ts; CODE:client/src/scripts/managers/particleManager.ts; CODE:client/src/scripts/managers/perkManager.ts; CODE:client/src/scripts/uiHelpers.ts; CODE:client/src/scripts/utils/constants.ts; CODE:client/src/scripts/utils/crosshairs.ts; CODE:client/src/scripts/utils/debugMenu.ts; CODE:client/src/scripts/utils/debugRenderer.ts; CODE:client/src/scripts/utils/floatingWindow.ts; CODE:client/src/scripts/utils/graph/graph.ts; CODE:client/src/scripts/utils/graph/netGraph.ts; CODE:client/src/scripts/utils/misc.ts; CODE:client/src/scripts/utils/translations/typings.ts; CODE:client/src/scripts/utils/tween.ts
- **Owner / release:** Base
- **Declared relations:** serves UR-CLI-001
- **Onboarding authority:** GATE-BASELINE-001; GRP-COV-006

### Trace references
| Evidence class | Target | Code | Outcome | Validity |
|---|---|---|---|---|
| SR lower | file disposition | CODE:client/src/scripts/console/internals.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:client/src/scripts/managers/particleManager.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:client/src/scripts/managers/perkManager.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:client/src/scripts/uiHelpers.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:client/src/scripts/utils/constants.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:client/src/scripts/utils/crosshairs.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:client/src/scripts/utils/debugMenu.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:client/src/scripts/utils/debugRenderer.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:client/src/scripts/utils/floatingWindow.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:client/src/scripts/utils/graph/graph.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:client/src/scripts/utils/graph/netGraph.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:client/src/scripts/utils/misc.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:client/src/scripts/utils/translations/typings.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:client/src/scripts/utils/tween.ts | SKIP | INHERITED_UNVERIFIED |

### Gates and delivery
- **Delivered revision:** not delivered

---
## SR-PLG-003 — Stock plugins and server HTTP helpers
- **Kind / status:** SR / PENDING_VERIFICATION
- **Statement / source:** As-built implementation spans 6 source files. / CODE:server/src/plugins/placeObjectPlugin.ts; CODE:server/src/plugins/speedTogglePlugin.ts; CODE:server/src/plugins/teleportPlugin.ts; CODE:server/src/plugins/weaponSwapPlugin.ts; CODE:server/src/utils/config.d.ts; CODE:server/src/utils/serverHelpers.ts
- **Owner / release:** Base
- **Declared relations:** serves UR-PLG-001
- **Onboarding authority:** GATE-BASELINE-001; GRP-COV-007

### Trace references
| Evidence class | Target | Code | Outcome | Validity |
|---|---|---|---|---|
| SR lower | file disposition | CODE:server/src/plugins/placeObjectPlugin.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:server/src/plugins/speedTogglePlugin.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:server/src/plugins/teleportPlugin.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:server/src/plugins/weaponSwapPlugin.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:server/src/utils/config.d.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:server/src/utils/serverHelpers.ts | SKIP | INHERITED_UNVERIFIED |

### Gates and delivery
- **Delivered revision:** not delivered

---
## SR-TST-005 — Client shell entries and character skin tests
- **Kind / status:** SR / PENDING_VERIFICATION
- **Statement / source:** As-built implementation spans 4 source files. / CODE:client/src/changelog.ts; CODE:client/src/dropdown.ts; CODE:client/src/index.ts; CODE:tests/src/skins.test.ts
- **Owner / release:** Base
- **Declared relations:** serves UR-TST-001
- **Onboarding authority:** GATE-BASELINE-001; GRP-COV-008

### Trace references
| Evidence class | Target | Code | Outcome | Validity |
|---|---|---|---|---|
| SR lower | file disposition | CODE:client/src/changelog.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:client/src/dropdown.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:client/src/index.ts | SKIP | INHERITED_UNVERIFIED |
| SR lower | file disposition | CODE:tests/src/skins.test.ts | SKIP | INHERITED_UNVERIFIED |

### Gates and delivery
- **Delivered revision:** not delivered

---
