# Network Protocol

<!-- @tier: 3 -->
<!-- @parent: ../README.md -->
<!-- @source: server/src/server.ts, client/src/scripts/game.ts -->

## Purpose

Documents the WebSocket protocol used between Suroi client and game server, including connection lifecycle, handshake, frame structure, and error handling.

## Key Files

| File | Purpose | Complexity |
|------|---------|------------|
| `server/src/server.ts` | Main WebSocket server setup, message routing | High |
| `client/src/scripts/game.ts` | Client WebSocket connection, message handlers | High |
| `common/src/packets/packetStream.ts` | Packet serialization/demultiplexing | Medium |
| `server/src/gameManager.ts` | Game instance creation and player routing | High |

## Connection Lifecycle

```
[Client]                              [Server]
  |                                      |
  |-- WebSocket upgrade (HTTP) -------->|
  |<----- 101 Switching Protocols ------|
  |                                      |
  |-- Send JOIN packet with name ------>| [Player joins queue]
  |<------ Send JOINED packet ---------|  [Port to game, send initial state]
  |                                      |
  |-- Send INPUT packet (every 25ms)-->| [Player actions queued]
  |<------ Send UPDATE packet ---------|  [Game state delta]
  |<------ Send UPDATE packet ---------|  (repeats ~40/sec)
  |                                      |
  |<------ Send KILL packet -----------| [Death notification]
  |<------ Send GAMEOVER packet -------| [Game ended]
  |                                      |
  |-- WebSocket close --------------- >|
  |<------ DISCONNECT packet ---------|
```

## WebSocket Connection Setup

### Client → Server
**File:** `client/src/scripts/game.ts:829-850` (approx.)

```typescript
this._socket = new WebSocket(address);
this._socket.binaryType = "arraybuffer";  // Binary frames, not text

this._socket.onopen = (): void => {
    // Ready to send JOIN packet
    const joinPacket = JoinPacket.create({
        name: this.playerName,
        loadout: this.loadout,
        autoFill: this.autoFill,
        gameMode: this.selectedMode
    });
    this.sendPacket(joinPacket);
};
```

### Server → Client
**File:** `server/src/server.ts`

```typescript
wss.on("connection", (socket: WebSocket) => {
    // New WebSocket client connected
    // Server does NOT immediately join them into a game
    
    socket.on("message", (data: ArrayBuffer) => {
        const stream = new PacketStream(data);
        const packet = stream.deserialize();
        
        switch(packet?.type) {
            case PacketType.Join:
                // Route player to available game instance
                gameManager.handleJoin(socket, packet);
                break;
            // ... other packet types
        }
    });
    
    socket.on("close", () => {
        // Clean up player from game if joined
    });
});
```

## Handshake: JOIN → JOINED

### Step 1: Client sends JOIN packet

```typescript
{
    type: PacketType.Join,
    name: "Alice",
    loadout: {
        skin: "hazel_jumpsuit",
        badge: "streamer",
        emote: "wave"
    },
    autoFill: true,
    gameMode: "normal"
}
```

Serialization: ~30-50 bytes (name + loadout refs compressed as IDs)

### Step 2: Server allocates player, routes to game

1. Validate player name (max 16 chars, no profanity)
2. Find/create `Game` instance for this mode/map
3. Create `Player` object with network ID
4. Add player to game's `connectedPlayers` set
5. Register in `Grid` for spatial queries
6. Register in `ObjectPool` for network updates

### Step 3: Server sends JOINED packet

```typescript
interface JoinedData {
    type: PacketType.Joined;
    playerId: number;                        // Network ID for this player
    teams: Team[] | undefined;               // If team mode
    players: Array<{
        id: number;
        name: string;
        position: Vector;
        health: number;
        /// ... additional player state
    }>;
    buildings: Array<(Building definition)>;
    obstacles: Array<(Obstacle definition)>;
    loots: Array<(Loot instance)>;
    emotes: EmoteDefinition[];
    gameData: {
        map: MapName;
        mode: ModeName;
        gas: GasStage;
        //...
    }
}
```

Size: ~500 bytes - 5 KB (depends on player count and map complexity)

**File:** `common/src/packets/joinedPacket.ts`

### Step 4: Client processes JOINED

```typescript
this._socket.onmessage = (message: MessageEvent<ArrayBuffer>): void => {
    const stream = new PacketStream(message.data);
    const packet = stream.deserialize();
    
    if(packet.type === PacketType.Joined) {
        this.playerId = packet.playerId;            // Store our ID
        this.objects.add(packet.players);           // Populate ObjectPool
        this.map.addBuildings(packet.buildings);    // Render map
        this.camera.focusOn(this.playerObject);     // Follow player
    }
};
```

## Input & Update Loop (Frame Sync)

### Player Input Transmission

**File:** `client/src/scripts/game.ts` (input manager)

**Frequency:** Every local frame (~16-60 FPS client-side, buffered until sent)
**Server-side frequency:** Every game tick (40 TPS = every 25 ms)

Client collects input each frame:

```typescript
interface InputData {
    type: PacketType.Input;
    movement: {
        up: boolean;
        down: boolean;
        left: boolean;
        right: boolean;
    };
    attacking: boolean;           // Mouse down / fire weapon
    aimDirection: number;         // Angle to cursor
    action: PlayerAction;         // slot 0-3 = weapon swap, e/r/f = abilities
    dropSlot?: number;            // Drop weapon from slot
    reload?: boolean;             // Reload request
    /// ... more as needed
}
```

Sent as binary packet (4-10 bytes):
- Movement booleans: 4 bits
- Action enum: 4-5 bits
- Aim angle: 2 bytes (uint16)
- Active slot: 2 bits

### Game State Update Transmission

**File:** `server/src/game.ts:531-548` (approx.)

**Frequency:** Every game tick (40 TPS)

```typescript
tick(): void {
    // 1. Process all input packets queued from clients
    // 2. Update player positions, rotations
    // 3. Update bullets, explosions, effects
    // 4. Check collisions
    // 5. Apply damage
    // 6. Serialize game state into UPDATE packet
    
    const updatePacket = {
        type: PacketType.Update,
        playerData: {
            minMax: { maxHealth, maxAdrenaline },
            health: this.health,
            adrenaline: this.adrenaline,
            zoom: this.scope.zoom,
            // ... only dirty fields
        },
        globalData: {
            playerKills: [
                { id: 7, kills: 5 },
                { id: 3, kills: 2 }
            ],
            gasVisualData: { /* current gas circle */ },
            announcements: [ /* new kills, milestones */ ]
        },
        partialDirtyObjects: {
            // Only changed position/rotation
            [ObjectCategory.Player]: [ /* compact delta */ ],
            [ObjectCategory.Obstacle]: [ /* compact delta */ ]
        },
        fullDirtyObjects: {
            // New objects or full refresh needed
            [ObjectCategory.Player]: [ /* full state */ ],
            [ObjectCategory.Loot]: [ /* full state */ ]
        },
        deletedObjects: [42, 87]  // Object IDs to remove clientside
    };
    
    // Broadcast to all connected players
    this.sendPacketToAllConnectedPlayers(updatePacket);
}
```

Delta encoding saves bandwidth:

```typescript
// Partial: Only serialize changed fields
{
    objectId: 7,
    position: [128, 256],   // 4 bytes
    rotation: 1.57,          // 2 bytes
    // health, ammo, etc NOT included if unchanged
}
→ ~6-10 bytes per object

// Full: Send complete state
{
    objectId: 7,
    position: [128, 256],
    rotation: 1.57,
    health: 80,
    skin: "hazel_jumpsuit",
    animation: "running",
    // ... all fields
}
→ ~40-60 bytes per object
```

## Ping/Pong Heartbeat

The UPDATE packet acts as a heartbeat. Both client and server monitor silence:

### Server monitors client heartbeat

```typescript
// server/src/objects/player.ts
socket.on("message", () => {
    this.lastInputTime = Date.now();  // Reset timeout
});

// In game.ts tick():
for (const player of this.connectedPlayers) {
    if (Date.now() - player.lastInputTime > TIMEOUT_MS) {  // ~10-15 seconds
        player.disconnect("Heartbeat timeout");
    }
}
```

### Client monitors server heartbeat

```typescript
// client/src/scripts/game.ts
let lastUpdateTime = Date.now();

this._socket.onmessage = (message) => {
    lastUpdateTime = Date.now();  // Reset timeout
};

// In render loop:
if (Date.now() - lastUpdateTime > TIMEOUT_MS) {  // ~5 seconds
    this.disconnect("Server timeout");
}
```

**No explicit ping packet** — the UPDATE packet every 25ms serves as proof of life.

## Reconnection Handling

**Graceful disconnection:**

```typescript
// Client initiates
this._socket.close();
// Server sends DISCONNECT packet first
// Then closes socket

// Client receives
case PacketType.Disconnect:
    this.screens?.showDeathScreen("You were disconnected");
    this.disconnect();
    break;
```

**Timeout disconnection:**

Server kills unresponsive client after heartbeat timeout:

```typescript
// server/src/objects/player.ts
if (Date.now() - this.lastInputTime > HEARTBEAT_TIMEOUT_MS) {
    this.socket.close(1000, "Heartbeat timeout");
    this.kill(DamageSources.Disconnect);
}
```

**Internet outage recovery:**

Client detects silence and reconnects:

```typescript
// client/src/scripts/game.ts
const reconnectButton = new Button("Reconnect");
reconnectButton.on("click", () => {
    gameManager.connect(this.gameURL);  // New WebSocket
    // Will request to rejoin with new socket
});
```

**No built-in rejoin mechanism** — player restarts from lobby after disconnect.

## Bandwidth Characteristics

### Typical frame (40 TPS update cycle):

```
ClientInput:     4-10 bytes/frame (queued, sent every few frames)
ServerUpdate:   80-200 bytes/frame (40 objects × 2-5 bytes delta)
                 + 4-6 bytes object deletion tracking
                 + 2-4 bytes player data (if changed)
                 = ~90-210 bytes/frame

Downstream:     90-210 bytes × 40 FPS = 3.6-8.4 KB/sec per player
Upstream:       4-10 bytes × 10 sends/sec = 0.04-0.1 KB/sec per player
```

### Large packets:

```
JOIN:            ~30-50 bytes
JOINED:          ~0.5-5 KB    (population, map data)
MAP:             ~1-10 KB     (all buildings, obstacles, loot)
GAMEOVER:        ~100-200 bytes
KILL:            ~20-40 bytes (packed kill credit info)
```

## Error Handling & Edge Cases

### Malformed packets

No exception thrown. Stream reads past buffer, client receives default/garbage values:

```typescript
// If server sends truncated UPDATE packet:
const updateStream = new PacketStream(truncatedBuffer);
const update = updateStream.deserialize();
// update.playerData.health might be: undefined, past-end-of-buffer garbage, or 0
```

### Network out-of-order / duplication

**Not handled at protocol layer.** Relies on stateless update approach:

- UPDATE is idempotent — applying same update twice ≈ applying once
- INPUT is aggregated on server tick, old input discarded after 25ms
- KILL is instantaneous, idempotent

### Large burst of clients joining

Server has no per-game player limit. JOIN packets queue in WebSocket backlog. Game loop advances at fixed 40 TPS regardless of connection throughput.

## Related Documents

- **Tier 2:** [../README.md](../README.md) — Networking subsystem overview
- **Tier 1:** [../../../../architecture.md](../../../../architecture.md) — System architecture
- **Tier 3:** [packet-types.md](packet-types.md) — Detailed packet structure reference
- **Game Loop:** [../../game-loop/README.md](../../game-loop/README.md) — Tick timing and update cycle
- **Server:** [../../server-utilities/README.md](../../server-utilities/README.md) — Server infrastructure
