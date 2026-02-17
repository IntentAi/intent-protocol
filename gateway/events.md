# Gateway Events

Events are dispatched via opcode 0 (Dispatch) with an event type in the `t` field.

## Event Structure

All dispatch events follow this structure:

```typescript
{
  op: 0,
  t: string,    // Event name (e.g., "MESSAGE_CREATE")
  s: number,    // Sequence number for resume
  d: object     // Event payload (varies by event type)
}
```

## Event Table

| Event | Payload | Status |
|-------|---------|--------|
| MESSAGE_CREATE | `{ message: Message }` | Implemented |
| MESSAGE_UPDATE | `{ message: Message }` | Implemented |
| MESSAGE_DELETE | `{ id, channel_id }` | Implemented |
| SERVER_CREATE | `{ server: Server }` | Implemented |
| CHANNEL_CREATE | `{ channel: Channel }` | Implemented |
| SERVER_UPDATE | `{ server: Server }` | Planned |
| SERVER_DELETE | `{ server_id }` | Planned |
| CHANNEL_UPDATE | `{ channel: Channel }` | Planned |
| CHANNEL_DELETE | `{ channel_id, server_id }` | Planned |
| VOICE_STATE_UPDATE | Voice state object | Planned |
| VOICE_SERVER_UPDATE | Voice server info | Planned |
| TYPING_START | `{ channel_id, user_id }` | Planned |
| PRESENCE_UPDATE | Presence object | Planned |
| MEMBER_ADD | `{ server_id, user }` | Planned |
| MEMBER_REMOVE | `{ server_id, user_id }` | Planned |

## Implemented Events

### MESSAGE_CREATE

Dispatched when a message is sent in a channel the client can see.

**Payload:**

```typescript
{
  message: {
    id: string,
    channel_id: string,
    author: {
      id: string,
      username: string,
      display_name: string,
      avatar_url: string | null,
      created_at: string
    },
    content: string,
    created_at: string,
    edited_at: string | null
  }
}
```

**Example:**

```json
{
  "op": 0,
  "t": "MESSAGE_CREATE",
  "s": 42,
  "d": {
    "message": {
      "id": "1234567890",
      "channel_id": "5555555555",
      "author": {
        "id": "9876543210",
        "username": "alice",
        "display_name": "Alice",
        "avatar_url": null,
        "created_at": "2024-01-01T00:00:00Z"
      },
      "content": "Hello everyone!",
      "created_at": "2024-01-15T12:00:00Z",
      "edited_at": null
    }
  }
}
```

**When it fires:**
- User sends a message in any channel you have access to
- Bot sends a message
- Webhook posts a message

**Notes:**
- The full author object is included (not just user ID)
- Attachments and embeds will be added in a future phase

### MESSAGE_UPDATE

Dispatched when a message is edited.

**Payload:**

```typescript
{
  message: Message  // Full message object with updated content
}
```

**Example:**

```json
{
  "op": 0,
  "t": "MESSAGE_UPDATE",
  "s": 43,
  "d": {
    "message": {
      "id": "1234567890",
      "channel_id": "5555555555",
      "author": {
        "id": "9876543210",
        "username": "alice",
        "display_name": "Alice",
        "avatar_url": null,
        "created_at": "2024-01-01T00:00:00Z"
      },
      "content": "Hello everyone! (edited)",
      "created_at": "2024-01-15T12:00:00Z",
      "edited_at": "2024-01-15T12:05:00Z"
    }
  }
}
```

**When it fires:**
- User edits their own message
- Bot edits a message
- Embed is added/removed (link preview generated)

**Notes:**
- The `edited_at` field is set to the edit timestamp
- Full message object is sent, not a partial update

### MESSAGE_DELETE

Dispatched when a message is deleted.

**Payload:**

```typescript
{
  id: string,           // Message ID
  channel_id: string    // Channel the message was in
}
```

**Example:**

```json
{
  "op": 0,
  "t": "MESSAGE_DELETE",
  "s": 44,
  "d": {
    "id": "1234567890",
    "channel_id": "5555555555"
  }
}
```

**When it fires:**
- User deletes their own message
- Moderator deletes someone's message
- Bot deletes a message

**Notes:**
- Only the ID is sent, not the message content
- If you need the content, you must have cached it locally

### SERVER_CREATE

Dispatched when you join or create a server.

**Payload:**

```typescript
{
  server: {
    id: string,
    name: string,
    icon_url: string | null,
    owner_id: string,
    member_count: number,
    created_at: string
  }
}
```

**Example:**

```json
{
  "op": 0,
  "t": "SERVER_CREATE",
  "s": 45,
  "d": {
    "server": {
      "id": "1234567890",
      "name": "My New Server",
      "icon_url": null,
      "owner_id": "9876543210",
      "member_count": 1,
      "created_at": "2024-01-15T12:00:00Z"
    }
  }
}
```

**When it fires:**
- You create a new server
- You accept an invite to a server
- You're added to a server by someone

**Notes:**
- Also included in initial READY payload for servers you're already in
- Contains the full server object

### CHANNEL_CREATE

Dispatched when a channel is created in a server you're in.

**Payload:**

```typescript
{
  channel: {
    id: string,
    server_id: string,
    name: string,
    type: number,        // 0 = text, 1 = voice, 2 = category
    topic: string | null,
    position: number,
    parent_id: string | null,
    created_at: string
  }
}
```

**Example:**

```json
{
  "op": 0,
  "t": "CHANNEL_CREATE",
  "s": 46,
  "d": {
    "channel": {
      "id": "5555555555",
      "server_id": "1234567890",
      "name": "general",
      "type": 0,
      "topic": "General discussion",
      "position": 0,
      "parent_id": null,
      "created_at": "2024-01-15T12:00:00Z"
    }
  }
}
```

**When it fires:**
- Someone creates a channel in a server you're in
- You create a channel

**Notes:**
- You only receive this if you have permission to view the channel
- Category channels have type 2

## Planned Events

These events are documented for future implementation.

### SERVER_UPDATE

Dispatched when server properties change.

**Planned payload:**

```typescript
{
  server: Server  // Full updated server object
}
```

**Triggers:** Server name, icon, or settings changed

### SERVER_DELETE

Dispatched when you leave or are removed from a server, or the server is deleted.

**Planned payload:**

```typescript
{
  server_id: string
}
```

**Triggers:** You leave, get kicked, or server is deleted

### CHANNEL_UPDATE

Dispatched when channel properties change.

**Planned payload:**

```typescript
{
  channel: Channel  // Full updated channel object
}
```

**Triggers:** Channel name, topic, or position changed

### CHANNEL_DELETE

Dispatched when a channel is deleted.

**Planned payload:**

```typescript
{
  channel_id: string,
  server_id: string
}
```

**Triggers:** Channel is deleted by someone with permission

### TYPING_START

Dispatched when a user starts typing in a channel.

**Planned payload:**

```typescript
{
  channel_id: string,
  user_id: string,
  timestamp: number    // Unix timestamp
}
```

**Notes:**
- Typing indicator should expire after ~10 seconds
- No TYPING_STOP event - just let it time out

### PRESENCE_UPDATE

Dispatched when a user's presence changes.

**Planned payload:**

```typescript
{
  user_id: string,
  status: "online" | "idle" | "dnd" | "offline"
}
```

### VOICE_STATE_UPDATE

Dispatched when someone joins, leaves, or changes voice state in a server you're in. Broadcast to all members of the server.

**Planned payload:**

```typescript
{
  user_id: string,
  server_id: string,
  channel_id: string | null,  // null when disconnecting
  session_id: string,         // stable across mute/deaf toggles, changes on rejoin
  self_mute: boolean,
  self_deaf: boolean,
  server_mute: boolean,       // set by MUTE_MEMBERS permission holders
  server_deaf: boolean        // set by DEAFEN_MEMBERS permission holders
}
```

**Example — user joins voice:**

```json
{
  "op": 0,
  "t": "VOICE_STATE_UPDATE",
  "s": 58,
  "d": {
    "user_id": "9876543210",
    "server_id": "1234567890",
    "channel_id": "5555555555",
    "session_id": "vs_f7a2c1d8e3b9",
    "self_mute": false,
    "self_deaf": false,
    "server_mute": false,
    "server_deaf": false
  }
}
```

**Example — user disconnects:**

```json
{
  "op": 0,
  "t": "VOICE_STATE_UPDATE",
  "s": 59,
  "d": {
    "user_id": "9876543210",
    "server_id": "1234567890",
    "channel_id": null,
    "session_id": "vs_f7a2c1d8e3b9",
    "self_mute": false,
    "self_deaf": false,
    "server_mute": false,
    "server_deaf": false
  }
}
```

**Triggers:**
- User sends opcode 4 to join, move, or leave a voice channel
- User toggles self-mute or self-deaf
- Admin mutes/deafens a user (requires `MUTE_MEMBERS` / `DEAFEN_MEMBERS`)
- Admin moves a user to another channel (requires `MOVE_MEMBERS`)
- User disconnects ungracefully (SFU timeout after 15 seconds)

### VOICE_SERVER_UPDATE

Dispatched privately to a user after they send opcode 4 to join a voice channel. Contains everything needed to establish the WebRTC connection with the SFU.

**Planned payload:**

```typescript
{
  server_id: string,
  channel_id: string,
  endpoint: string,          // SFU WebSocket URL
  token: string,             // single-use voice token, valid 60 seconds
  ice_servers: IceServer[]   // STUN/TURN configuration for NAT traversal
}
```

**IceServer structure:**

```typescript
{
  urls: string[],            // e.g. ["stun:stun.intent.chat:3478"]
  username?: string,         // TURN only, time-limited credential
  credential?: string        // TURN only, HMAC-based secret
}
```

**Example:**

```json
{
  "op": 0,
  "t": "VOICE_SERVER_UPDATE",
  "s": 59,
  "d": {
    "server_id": "1234567890",
    "channel_id": "5555555555",
    "endpoint": "wss://us-east.voice.intent.chat",
    "token": "vt_a8f3bc91d2e4...",
    "ice_servers": [
      {
        "urls": ["stun:stun.intent.chat:3478"]
      },
      {
        "urls": [
          "turn:turn.intent.chat:3478?transport=udp",
          "turn:turn.intent.chat:443?transport=tcp"
        ],
        "username": "1708012800:user_9876543210",
        "credential": "a1b2c3d4e5f6..."
      }
    ]
  }
}
```

**Notes:**
- Only sent to the user who requested the voice connection, not broadcast.
- The `token` is a signed JWT binding user + server + channel. Must be included in the first opcode 12 offer.
- If the token expires before use, send opcode 4 again to get a fresh one.
- TURN credentials are time-limited (12 hour TTL). TCP on port 443 is a fallback for restrictive networks.
- See `voice/signaling.md` for the full connection flow after receiving this event.

### MEMBER_ADD

Dispatched when someone joins a server you're in.

**Planned payload:**

```typescript
{
  server_id: string,
  user: User,
  joined_at: string
}
```

### MEMBER_REMOVE

Dispatched when someone leaves a server you're in.

**Planned payload:**

```typescript
{
  server_id: string,
  user_id: string
}
```

## Event Ordering

Events are delivered in order per connection. The sequence number (`s` field) increments for each dispatch.

**Guarantees:**
- Events for the same resource arrive in order
- Sequence numbers are always increasing
- No gaps in sequence numbers during a session

**After disconnect:**
- Use Resume (opcode 5) with your last sequence number
- Server replays missed events in order
- If session expired, you get a fresh READY

## Intents (Future)

When implemented, intents will control which events you receive:

| Intent | Events |
|--------|--------|
| SERVERS | SERVER_CREATE, SERVER_UPDATE, SERVER_DELETE |
| CHANNELS | CHANNEL_CREATE, CHANNEL_UPDATE, CHANNEL_DELETE |
| SERVER_MESSAGES | MESSAGE_CREATE, MESSAGE_UPDATE, MESSAGE_DELETE (in servers) |
| DIRECT_MESSAGES | MESSAGE_CREATE, MESSAGE_UPDATE, MESSAGE_DELETE (in DMs) |
| MEMBERS | MEMBER_ADD, MEMBER_REMOVE |
| PRESENCE | PRESENCE_UPDATE |
| VOICE | VOICE_STATE_UPDATE |
| MESSAGE_CONTENT | Includes message content (privileged) |

Without intents, you receive all events you have permission to see.

## Best Practices

**Handle events idempotently:**
- The same event might arrive twice after reconnect
- Check if you already have the resource before adding

**Cache locally:**
- MESSAGE_DELETE only sends the ID
- Keep local cache if you need deleted content

**Validate payloads:**
- Future server versions might add fields
- Ignore unknown fields, don't error on them

**Update state atomically:**
- Process events in order
- Don't partially apply updates
