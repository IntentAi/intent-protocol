# Gateway Opcodes

WebSocket gateway uses MessagePack binary protocol with opcode-based messages.

## Base Payload Structure

All gateway messages follow this structure:

```typescript
{
  op: number,        // Opcode (message type)
  d?: any,           // Data payload (varies by opcode)
  t?: string,        // Event type (Dispatch only)
  s?: number         // Sequence number (Dispatch only)
}
```

## Opcode Table

| Opcode | Name | Direction | Status |
|--------|------|-----------|--------|
| 0 | Dispatch | Server → Client | Implemented |
| 1 | Heartbeat | Client → Server | Implemented |
| 2 | Identify | Client → Server | Implemented |
| 3 | Ready | Server → Client | Implemented |
| 4 | Voice State Update | Client → Server | Planned |
| 5 | Resume | Client → Server | Planned |
| 6 | Reconnect | Server → Client | Planned |
| 7 | Request Guild Members | Client → Server | Planned |
| 8 | Invalid Session | Server → Client | Planned |
| 9 | Hello | Server → Client | Planned |
| 10 | Presence Update | Client → Server | Planned |
| 11 | Heartbeat ACK | Server → Client | Implemented |
| 12 | Voice Signal | Bidirectional | Planned |

**Note:** The current implementation uses opcodes 0, 1, 2, 3, 11 only. Ready (op 3) combines Hello + Ready into a single message. Future versions will implement the full opcode set with Hello (op 9) sent before Identify.

## Opcode Details

### Opcode 0: Dispatch

**Direction:** Server → Client

The server dispatches an event to the client. Used for all real-time events like new messages, channel updates, etc.

**Payload structure:**

```typescript
{
  op: 0,
  t: string,         // Event type name (MESSAGE_CREATE, CHANNEL_UPDATE, etc.)
  s: number,         // Sequence number (for resume tracking)
  d: any             // Event-specific data
}
```

**Example - MESSAGE_CREATE event:**

```json
{
  "op": 0,
  "t": "MESSAGE_CREATE",
  "s": 42,
  "d": {
    "id": "1234567890",
    "channel_id": "5555555555",
    "author": {
      "id": "9876543210",
      "username": "alice",
      "display_name": "Alice"
    },
    "content": "Hello world!",
    "created_at": "2024-01-15T12:00:00Z"
  }
}
```

**Notes:**
- The `s` field is incremented for each dispatch
- Clients should track the latest sequence number for resume support
- See gateway/events.md for all event types

### Opcode 1: Heartbeat

**Direction:** Client → Server

The client sends this periodically to keep the connection alive. The server responds with Heartbeat ACK (opcode 11).

**Payload structure:**

```typescript
{
  op: 1,
  d: null | number   // Last sequence number received (or null)
}
```

**Example:**

```json
{
  "op": 1,
  "d": null
}
```

**Notes:**
- Sent at intervals specified by the server in the Hello/Ready payload
- If client doesn't receive ACK within timeout, connection is considered dead
- Typical interval: 30-45 seconds

### Opcode 2: Identify

**Direction:** Client → Server

The client sends this immediately after connecting to authenticate with a JWT token.

**Payload structure:**

```typescript
{
  op: 2,
  d: {
    token: string,           // JWT authentication token (usr_* or bot_*)
    intents?: number,        // Intent bitfield (optional, future use)
    properties?: {           // Client properties (optional)
      os?: string,
      browser?: string,
      device?: string
    }
  }
}
```

**Example:**

```json
{
  "op": 2,
  "d": {
    "token": "usr_MTIzNDU2Nzg5MC4xNzA1MzIwMDAwLmFiY2Q...",
    "properties": {
      "os": "linux",
      "browser": "chrome",
      "device": "desktop"
    }
  }
}
```

**Notes:**
- This MUST be the first payload sent after WebSocket connection opens
- Server responds with Ready (opcode 9) on success or Invalid Session (opcode 8) on failure
- Properties are optional and used for analytics/debugging

### Opcode 3: Ready

**Direction:** Server → Client

The server sends this after successful authentication. Contains the authenticated user, server list, and heartbeat interval.

**Payload structure:**

```typescript
{
  op: 3,
  d: {
    user: User,              // Authenticated user object
    servers: Server[],       // List of servers user belongs to
    heartbeat_interval: number  // Milliseconds between heartbeats
  }
}
```

**Example:**

```json
{
  "op": 3,
  "d": {
    "user": {
      "id": "9876543210",
      "username": "alice",
      "display_name": "Alice",
      "avatar_url": null,
      "created_at": "2024-01-01T00:00:00Z"
    },
    "servers": [
      {
        "id": "1234567890",
        "name": "My Server",
        "icon_url": null,
        "owner_id": "9876543210",
        "member_count": 5
      }
    ],
    "heartbeat_interval": 41250
  }
}
```

**Notes:**
- Sent in response to successful Identify (opcode 2)
- Client should start heartbeat loop using heartbeat_interval
- Contains initial state data for populating the UI

### Opcode 4: Voice State Update

**Direction:** Client → Server

The client joins, leaves, or updates voice channel state.

**Payload structure:**

```typescript
{
  op: 4,
  d: {
    server_id: string,       // Server ID
    channel_id: string | null, // Voice channel ID (null to disconnect)
    self_mute: boolean,
    self_deaf: boolean
  }
}
```

**Example:**

```json
{
  "op": 4,
  "d": {
    "server_id": "1234567890",
    "channel_id": "5555555555",
    "self_mute": false,
    "self_deaf": false
  }
}
```

**Notes:**
- Not implemented in Phase 1
- Server responds with VOICE_STATE_UPDATE and VOICE_SERVER_UPDATE events
- Setting channel_id to null disconnects from voice

### Opcode 5: Resume

**Direction:** Client → Server

The client attempts to resume a previous session after disconnecting.

**Payload structure:**

```typescript
{
  op: 5,
  d: {
    token: string,           // JWT token
    session_id: string,      // Session ID from Ready event
    seq: number              // Last sequence number received
  }
}
```

**Example:**

```json
{
  "op": 5,
  "d": {
    "token": "usr_MTIzNDU2Nzg5MC4xNzA1MzIwMDAwLmFiY2Q...",
    "session_id": "abc123def456",
    "seq": 42
  }
}
```

**Notes:**
- Used instead of Identify when reconnecting
- Server replays missed events if session is still valid
- Server sends Invalid Session (opcode 8) if session expired

### Opcode 6: Reconnect

**Direction:** Server → Client

The server requests that the client disconnect and reconnect (used for load balancing, maintenance, etc.).

**Payload structure:**

```typescript
{
  op: 6,
  d: null
}
```

**Example:**

```json
{
  "op": 6,
  "d": null
}
```

**Notes:**
- Client should immediately close connection and reconnect with Resume (opcode 5)
- This is a graceful reconnection request

### Opcode 7: Request Guild Members

**Direction:** Client → Server

The client requests member list for a server (used for large servers where full member list isn't sent in Ready).

**Payload structure:**

```typescript
{
  op: 7,
  d: {
    server_id: string,       // Server ID
    query?: string,          // Username prefix filter (optional)
    limit: number,           // Max members to return (1-1000)
    presences?: boolean      // Include presence data (optional)
  }
}
```

**Example:**

```json
{
  "op": 7,
  "d": {
    "server_id": "1234567890",
    "limit": 100
  }
}
```

**Notes:**
- Not implemented in Phase 1
- Server responds with GUILD_MEMBERS_CHUNK event

### Opcode 8: Invalid Session

**Direction:** Server → Client

The server indicates that the session is invalid (bad token, expired session, etc.).

**Payload structure:**

```typescript
{
  op: 8,
  d: boolean           // Whether the session is resumable
}
```

**Example:**

```json
{
  "op": 8,
  "d": false
}
```

**Notes:**
- If `d` is true, client should attempt Resume (opcode 5)
- If `d` is false, client must start fresh with Identify (opcode 2)
- Client should wait 1-5 seconds with random jitter before reconnecting

### Opcode 9: Hello (Planned)

**Direction:** Server → Client

**Status:** Not yet implemented. Currently combined with Ready (opcode 3).

In future versions, the server will send Hello immediately after connection, before authentication.

**Planned payload:**

```typescript
{
  op: 9,
  d: {
    heartbeat_interval: number
  }
}
```

When implemented:
1. Server sends Hello (op 9)
2. Client starts heartbeat timer
3. Client sends Identify (op 2) or Resume (op 5)
4. Server sends Ready via Dispatch (op 0, t: "READY")

### Opcode 10: Presence Update (Planned)

**Direction:** Client → Server

**Status:** Not yet implemented.

The client updates its presence status (online, idle, dnd, offline).

**Planned payload:**

```typescript
{
  op: 10,
  d: {
    status: "online" | "idle" | "dnd" | "offline",
    since?: number,
    afk: boolean
  }
}
```

### Opcode 11: Heartbeat ACK

**Direction:** Server → Client

The server acknowledges receipt of a heartbeat.

**Payload structure:**

```typescript
{
  op: 11,
  d: null
}
```

**Example:**

```json
{
  "op": 11,
  "d": null
}
```

**Notes:**
- Sent in response to Heartbeat (opcode 1)
- If client doesn't receive ACK after 2-3 heartbeats, connection is considered dead
- Client should close and reconnect if heartbeat timeout occurs

### Opcode 12: Voice Signal (Planned)

**Direction:** Bidirectional

**Status:** Not yet implemented.

WebRTC signaling for voice connections (SDP offers/answers, ICE candidates).

**Planned payload:**

```typescript
{
  op: 12,
  d: {
    type: "offer" | "answer" | "ice",
    sdp?: string,
    candidate?: any
  }
}
```

See voice/signaling.md for full WebRTC flow when implemented.

## Connection Lifecycle

### Initial Connection (Current Implementation)

```
1. Client connects to wss://gateway.intent.chat/
2. Client sends Identify (op 2) with token
3. Server validates token and sends Ready (op 3) with user, servers, heartbeat_interval
4. Client starts heartbeat loop using heartbeat_interval
5. Client sends Heartbeat (op 1)
6. Server responds with Heartbeat ACK (op 11)
7. Connection is now fully established
```

### Initial Connection (Future - Discord-compatible)

```
1. Client connects to wss://gateway.intent.chat/
2. Server sends Hello (op 9) with heartbeat_interval
3. Client starts heartbeat timer
4. Client sends Identify (op 2) with token
5. Server validates token and sends Dispatch (op 0, t: "READY") with user data
6. Client is now fully connected
```

### Heartbeat Loop

```
Client                          Server
  |                               |
  |--- Heartbeat (op 1) --------->|
  |                               |
  |<-- Heartbeat ACK (op 11) -----|
  |                               |
 (wait heartbeat_interval ms)
  |                               |
  |--- Heartbeat (op 1) --------->|
  |                               |
  ...
```

If you send 3 heartbeats without getting an ACK back, the connection is dead. Close it and reconnect.

### Disconnection and Resume

```
1. Connection drops (network issue, server restart, etc.)
2. Client attempts to reconnect
3. Server sends Hello (op 9)
4. Client sends Resume (op 5) with session_id and last sequence
5a. If session valid: Server replays missed events starting from sequence
5b. If session expired: Server sends Invalid Session (op 8, d: false)
6. On Invalid Session: Client sends fresh Identify (op 2)
```

### Graceful Reconnect

```
1. Server sends Reconnect (op 6)
2. Client closes connection
3. Client reconnects with Resume (op 5)
4. Server replays any missed events
```

## Intents System

Intents are not implemented in Phase 1. In future phases, clients will specify intent bitfields in the Identify payload to control which events they receive.

**Planned intents:**

- GUILDS (1 << 0) - Server create/update/delete events
- GUILD_MEMBERS (1 << 1) - Member join/leave events
- GUILD_MESSAGES (1 << 9) - Message events in servers
- DIRECT_MESSAGES (1 << 12) - Message events in DMs
- MESSAGE_CONTENT (1 << 15) - Actual message content (privileged)

Clients without specific intents won't receive the corresponding events. This reduces bandwidth for bots that don't need all events.

## Error Handling

**Connection failures:**
- Implement exponential backoff: 1s, 2s, 4s, 8s, 16s, 30s (max)
- Add random jitter (±25%) to avoid thundering herd
- Max reconnection attempts: unlimited (but with increasing delays)

**Invalid Session:**
- Wait 1-5 seconds with random jitter before reconnecting
- If `d: false`, must send fresh Identify
- If `d: true`, can attempt Resume

**Heartbeat timeout:**
- If no ACK received for 3 consecutive heartbeats, close connection
- Reconnect with Resume to preserve session

## MessagePack Encoding

All gateway messages use MessagePack binary encoding instead of JSON. This reduces payload size by ~30% and improves parsing performance.

Example encoding (JavaScript):

```javascript
import { encode, decode } from '@msgpack/msgpack'

// Encode
const payload = { op: 1, d: null }
const binary = encode(payload)  // Uint8Array
websocket.send(binary)

// Decode
websocket.onmessage = (event) => {
  const buffer = new Uint8Array(event.data)
  const payload = decode(buffer)
  console.log(payload.op, payload.d)
}
```

## Rate Limiting

Gateway connections are not subject to HTTP rate limits, but the following limits apply:

- Maximum 120 gateway commands per 60 seconds (2 per second sustained)
- Maximum 1 Identify or Resume per 5 seconds
- Exceeding limits results in connection termination with close code 4008

These limits prevent abuse while allowing legitimate high-frequency operations.
