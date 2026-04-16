# Client Auxiliary Managers — Chat & Message System

<!-- @tier: 3 -->
<!-- @parent: docs/subsystems/client-managers/README.md -->
<!-- @source: server/src/server.ts, client/src/scripts/game.ts, common/src/typings.ts, client/src/ui.ts -->

## Purpose
Manages chat message sending, receiving, rate limiting, and display: message format parsing, broadcast targeting (team vs all), message queue rendering, and timestamp-based fade-out animations.

## Key Files
| File | Purpose | Complexity |
|------|---------|------------|
| `server/src/server.ts` | WebSocket message handler, custom team messages | High |
| `client/src/scripts/game.ts` | Message display list, rendering queue | High |
| `client/src/ui.ts` | Chat input UI, message HTML assembly, display container | High |
| `common/src/typings.ts` | CustomTeamMessage type definitions | Medium |

## Business Rules

- **Message Format:** `{ playerID?: number; content: string; teamMessage: boolean }`
- **Broadcasting:** `teamMessage=true` → team only; `teamMessage=false` → all players
- **Rate Limit:** Server enforces max messages per player per time window (e.g., 1 msg/2 sec)
- **Message Queue:** Client maintains deque of recent messages; ordered by timestamp
- **Display Duration:** Messages fade after configurable timeout (e.g., 30 seconds)
- **Name Color:** Chat should display sender name with player color (retrieved from UpdatePacket)
- **Spam Prevention:** Duplicate/rapid messages consolidated or throttled
- **Censoring:** Basic profanity filter or keyword blocking (optional, varies by deployment)

## Data Lineage

### Message Send Flow
```
Player types in chat input box
  ↓
Enter key pressed
  ↓
Client validates:
    - Not empty string
    - Length < GameConstants.chat.maxLength (e.g., 200 chars)
    - Not spam (rate limit check)
  ↓
Message object created:
    {
        playerID: this.player.id,
        content: inputValue,
        teamMessage: isTeamToggled
    }
  ↓
Send to server via WebSocket
  ↓
Server receives in WebSocket.message handler
  ↓
Server validation:
    - Player exists and authenticated
    - Not spamming (check RateLimiter)
    - Message not empty
  ↓
If validation passes:
    Create message: { playerID, content, teamMessage }
  ↓
Broadcast logic:
    if teamMessage && player.team:
        Send to team.players only
    else:
        Send to all connected players
  ↓
Client receives in InputPacket (or custom message packet)
  ↓
Add to messageQueue with timestamp
  ↓
Render in UI (next frame or immediately)
```

### Message Display Lifecycle
```
Message added to queue: messageQueue.push(msg)
  ↓
Timestamp recorded: msg.displayTime = now
  ↓
Client tick: update display
  ↓
Render message in chat container:
    DOM: <div class="chat-message">
           <span class="player-name">[colorized]PlayerName</span>
           <span class="message-content">Message text</span>
         </div>
  ↓
Each frame: check if faded
  ↓
Opacity = 1.0 initially
  ↓
At fadeDuration (e.g., 25/30 sec): start fade
  ↓
Fade time = displayDuration - fadeStartTime (e.g., 5 sec)
  ↓
Opacity = 1 - (elapsed / fadeTime)  [0 to 1]
  ↓
After displayDuration (e.g., 30 sec): remove from queue
  ↓
DOM element deleted
```

### Team Message Filtering
```
Game.player.team = Team instance
  ↓
Message received with teamMessage = true
  ↓
Check: player.team exists
  ↓
If yes: display message in chat
  ↓
If no: filter message (shouldn't display)
  ↓
Visually: team messages may have special [TEAM] prefix or color
```

## Complex Functions

### `Game.onChatMessage(message: ChatMessage)` — @file client/src/scripts/game.ts
**Purpose:** Receive chat message from server and add to display queue.

**Implicit behavior:**
1. Validate message object (playerID, content, teamMessage fields present)
2. Lookup player object by playerID (to get name, color)
3. Create message display object:
   ```typescript
   {
       playerName: player.name,
       playerColor: player.skin.color,
       content: message.content,
       teamMessage: message.teamMessage,
       timestamp: Date.now(),
       displayEndTime: Date.now() + CHAT_DISPLAY_DURATION
   }
   ```
4. Add to `this.messageQueue` array
5. Call `UIManager.displayChatMessage()` to render to DOM
6. If queue exceeds max size (e.g., 50 messages), remove oldest

**Called by:** InputPacket handler or custom message packet

**Side effects:**
- DOM updated with new message element
- Message queue modified
- Player name colored in avatar on-screen

**Example:**
```typescript
// Server sends: { playerID: 5, content: "Hello team!", teamMessage: true }
// Game finds player 5: { name: "Warrior", color: rgba(255, 0, 0, 1) }
// Message added:
// {
//   playerName: "Warrior",
//   playerColor: "red",
//   content: "Hello team!",
//   timestamp: 1234567890,
//   displayEndTime: 1234567890 + 30000
// }
```

### `UIManager.displayChatMessage(msg: ChatDisplayMessage)` — @file client/src/ui.ts
**Purpose:** Render chat message to DOM with proper formatting.

**Implicit behavior:**
1. Create DOM element:
   ```html
   <div class="chat-message" data-timestamp="[timestamp]">
       <span class="player-name" style="color: [playerColor]">
           [teamMessage ? "[TEAM] " : ""]PlayerName
       </span>
       <span class="message-content">[sanitized content]</span>
   </div>
   ```
2. Sanitize content (remove HTML tags, prevent XSS)
3. Append to chat container (scrolls to bottom)
4. Set CSS class for styling
5. Schedule fade animation timeout

**Called by:** `onChatMessage()`

**Side effects:**
- DOM modified
- Chat scrolls to bottom
- Animation scheduled

## Message Rate Limiting

| Setting | Effect | Default |
|---------|--------|---------|
| `maxMessagesPerSecond` | Max message frequency | 0.5 (1 msg per 2 sec) |
| `maxMessageLength` | Character limit | 200 |
| `displayDuration` | Time message visible | 30 seconds |
| `fadeStartTime` | When fade begins | 25 seconds (5 sec fade) |

## Configuration
| Setting | Source | Effect |
|---------|--------|--------|
| `CHAT_DISPLAY_DURATION` | constants.ts | How long message stays visible | Default: 30s |
| `CHAT_FADE_TIME` | constants.ts | Duration of fade animation | Default: 5s |
| `MAX_CHAT_MESSAGES` | constants.ts | Max in-game messages stored | Default: 50 |
| `CHAT_MAX_LENGTH` | GameConstants | Max characters per message | Default: 200 |

## Message Types

### Team Message
```typescript
{
    playerID: 3,
    content: "Ammo behind you!",
    teamMessage: true  // Only team sees
}
// Display: [TEAM] PlayerName: Ammo behind you!
```

### Global Message
```typescript
{
    playerID: 3,
    content: "gg wp!",
    teamMessage: false  // All players see
}
// Display: PlayerName: gg wp!
```

### System Message (optional, server-sent)
```typescript
{
    playerID: undefined,  // System-generated
    content: "Player has joined the game",
    teamMessage: false
}
// Display: [SYSTEM] Player has joined the game
// (Different styling, no sender name)
```

## Spam Prevention Strategies

1. **Rate Limiting:** RateLimiter on server, max 1 msg per 2 sec per player
2. **Length Limit:** Max 200 characters per message
3. **Duplicate Detection:** If message exactly matches previous from same player within 5 sec, reject
4. **Keyword Blocking:** Optional list of forbidden words (configurable per deployment)
5. **Cooldown Reset:** After disconnect/reconnect, rate limiter resets

## Related Documents
- **Tier 2:** [../README.md](../README.md) — UI managers, event systems, network integration
- **Tier 2:** [../../ui-management/README.md](../../ui-management/README.md) — HUD elements, display management
- **Tier 2:** [../../team-system/README.md](../../team-system/README.md) — Team data, team operations
- **Tier 2:** [../../networking/README.md](../../networking/README.md) — Packet types, message serialization
- **Tier 1:** [../../../../api-reference.md](../../../../api-reference.md) — Chat packet structure
- **Patterns:** [../../networking/patterns.md](../../networking/patterns.md) (if exists) — Message broadcast patterns
