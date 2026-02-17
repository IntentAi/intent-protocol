# Voice Signaling Protocol

WebRTC signaling for Intent voice channels, built on a str0m-based SFU architecture.

## Architecture

Intent uses a Selective Forwarding Unit (SFU) for voice. The server receives each participant's audio stream and forwards it individually to every other participant without mixing or transcoding. This keeps server CPU low and preserves per-user volume control on the client side.

The SFU is built on [str0m](https://github.com/algesten/str0m), a sans-IO Rust WebRTC library. str0m handles SDP negotiation, ICE connectivity, DTLS-SRTP key exchange, and RTP packet routing. The application layer (intent-server) manages session state, permission checks, and participant lifecycle.

```
                     +------------------+
                     |   Intent SFU     |
                     |   (str0m)        |
                     +------------------+
                    /    |    |    |     \
                   /     |    |    |      \
              User A  User B  User C  User D  User E
              (send)  (send)  (send)  (send)  (send)

    Each user sends 1 stream to the SFU.
    SFU forwards N-1 streams to each user.
    No mixing, no transcoding.
```

**Why SFU over mesh or MCU:**

- **Mesh** (peer-to-peer): Each client sends to every other client. Upload bandwidth scales as O(N). Breaks above 4-5 users.
- **MCU** (mixing): Server decodes, mixes, and re-encodes a single stream per client. High CPU cost, adds latency, destroys per-user volume control.
- **SFU** (forwarding): Each client uploads once. Server fans out without processing audio. Scales to 25 users with minimal server load. Clients decode N-1 streams but modern hardware handles that easily.

Maximum participants per voice channel: **25** (24 receive streams * 64 kbps = ~1.5 Mbps downstream worst case).

## Voice Connection Lifecycle

Full sequence from join request to media flow:

```
Client                          Gateway                         SFU
  |                               |                               |
  |-- op 4: Voice State Update -->|                               |
  |   { server_id, channel_id,   |                               |
  |     self_mute, self_deaf }    |                               |
  |                               |-- permission check (CONNECT) -|
  |                               |                               |
  |<-- VOICE_STATE_UPDATE --------|                               |
  |   (broadcast to server)       |                               |
  |                               |                               |
  |<-- VOICE_SERVER_UPDATE -------|                               |
  |   { endpoint, token,         |                               |
  |     ice_servers }             |                               |
  |                               |                               |
  |-- op 12: { type: "offer" } ---------------------------------->|
  |   (SDP offer via gateway)     |                               |
  |                               |                               |
  |<-- op 12: { type: "answer" } ---------------------------------|
  |   (SDP answer via gateway)    |                               |
  |                               |                               |
  |-- op 12: { type: "ice" } ------------------------------------>|
  |<-- op 12: { type: "ice" } ------------------------------------|
  |   (ICE candidate exchange)    |                               |
  |                               |                               |
  |<============== DTLS-SRTP handshake =========================>|
  |                               |                               |
  |<============== RTP/RTCP media flow =========================>|
  |                               |                               |
```

Steps in detail:

1. **Client sends opcode 4** with the target voice channel. Gateway validates the user has `CONNECT` permission on that channel.
2. **Gateway broadcasts VOICE_STATE_UPDATE** to all members of the server so clients can update voice user lists.
3. **Gateway sends VOICE_SERVER_UPDATE** privately to the joining client with the SFU endpoint URL, a single-use voice token (valid 60 seconds), and STUN/TURN server configuration.
4. **Client creates an SDP offer** for an audio-only PeerConnection and sends it via opcode 12 with `type: "offer"`.
5. **SFU responds with an SDP answer** via opcode 12 with `type: "answer"`. The answer includes receive-only tracks for each existing participant.
6. **ICE candidates are exchanged** via opcode 12 with `type: "ice"` in both directions until connectivity is established.
7. **DTLS-SRTP handshake** completes, establishing encrypted media transport.
8. **Media flows.** Client sends Opus audio to the SFU, SFU forwards audio from other participants.

## Opcode 4: Voice State Update

Sent by the client to join, move between, or leave voice channels. Also used to toggle self-mute and self-deaf.

**Direction:** Client -> Server

**Payload:**

```typescript
interface VoiceStateUpdatePayload {
  server_id: string;           // target server
  channel_id: string | null;   // voice channel to join, null to disconnect
  self_mute: boolean;          // client-side mute
  self_deaf: boolean;          // client-side deafen
}
```

**Behaviors:**

| Action | Payload | Result |
|--------|---------|--------|
| Join channel | `channel_id` set, not currently in voice | Permission check, SFU allocation, VOICE_STATE_UPDATE + VOICE_SERVER_UPDATE |
| Move channel | `channel_id` set, already in different channel | Permission check on new channel, teardown old connection, new SFU session |
| Disconnect | `channel_id: null` | Graceful disconnect, track removal, VOICE_STATE_UPDATE broadcast |
| Toggle mute | Same `channel_id`, `self_mute` changed | VOICE_STATE_UPDATE broadcast, no media renegotiation |
| Toggle deaf | Same `channel_id`, `self_deaf` changed | VOICE_STATE_UPDATE broadcast, server stops forwarding streams to client |

**Permission checks:**

- `CONNECT` is required to join any voice channel.
- `SPEAK` is required to transmit audio. Without it, the client connects in listen-only mode.
- Server owner bypasses all permission checks.

**Error responses (via gateway close or dispatch):**

- Missing `CONNECT` permission: gateway error code `4014` (voice permission denied)
- Channel does not exist: gateway error code `4011` (unknown channel)
- Channel is full (25 users): gateway error code `4015` (voice channel full)

## Opcode 12: Voice Signal

Bidirectional signaling opcode that carries WebRTC negotiation messages between the client and SFU, relayed through the gateway.

**Direction:** Bidirectional

**Base payload:**

```typescript
interface VoiceSignalPayload {
  type: "offer" | "answer" | "ice" | "renegotiate" | "speaking";
  // remaining fields depend on type
}
```

### type: "offer"

Client sends an SDP offer to initiate or update the WebRTC session.

**Direction:** Client -> Server

```typescript
interface VoiceSignalOffer {
  type: "offer";
  sdp: string;             // full SDP offer
  voice_token: string;     // token from VOICE_SERVER_UPDATE
}
```

**Example:**

```json
{
  "op": 12,
  "d": {
    "type": "offer",
    "sdp": "v=0\r\no=- 4611731400430051336 2 IN IP4 127.0.0.1\r\n...",
    "voice_token": "vt_a8f3bc91d2e4..."
  }
}
```

### type: "answer"

Server responds to an offer with an SDP answer.

**Direction:** Server -> Client

```typescript
interface VoiceSignalAnswer {
  type: "answer";
  sdp: string;             // full SDP answer
}
```

**Example:**

```json
{
  "op": 12,
  "d": {
    "type": "answer",
    "sdp": "v=0\r\no=- 7614219014086345202 2 IN IP4 10.0.1.5\r\n..."
  }
}
```

### type: "ice"

ICE candidate exchange. Sent in both directions as candidates are gathered.

**Direction:** Bidirectional

```typescript
interface VoiceSignalIce {
  type: "ice";
  candidate: string;       // ICE candidate string (a= line)
  sdp_mid: string;         // media stream identification tag
  sdp_mline_index: number; // index of the m= line
}
```

**Example:**

```json
{
  "op": 12,
  "d": {
    "type": "ice",
    "candidate": "candidate:842163049 1 udp 1677729535 203.0.113.5 24680 typ srflx raddr 192.168.1.5 rport 54321",
    "sdp_mid": "audio",
    "sdp_mline_index": 0
  }
}
```

### type: "renegotiate"

Server pushes a new SDP offer when the session needs updating (participant joined/left, track added/removed). The client must respond with an answer.

**Direction:** Server -> Client

```typescript
interface VoiceSignalRenegotiate {
  type: "renegotiate";
  sdp: string;             // new SDP offer from server
  reason: "participant_joined" | "participant_left" | "track_update";
}
```

**Example:**

```json
{
  "op": 12,
  "d": {
    "type": "renegotiate",
    "sdp": "v=0\r\no=- 7614219014086345202 3 IN IP4 10.0.1.5\r\n...",
    "reason": "participant_joined"
  }
}
```

The client handles this by calling `setRemoteDescription` with the new offer, then generating and sending back an answer via `type: "answer"`.

### type: "speaking"

Indicates whether a user is currently speaking. Used for visual speaking indicators in the client UI.

**Direction:** Server -> Client

```typescript
interface VoiceSignalSpeaking {
  type: "speaking";
  user_id: string;
  speaking: boolean;
}
```

**Example:**

```json
{
  "op": 12,
  "d": {
    "type": "speaking",
    "user_id": "9876543210",
    "speaking": true
  }
}
```

The SFU determines speaking state using RTP audio level headers (RFC 6464). A user is considered speaking when their audio level exceeds -40 dBov. The server applies a 300ms debounce — a user must be below threshold for 300ms before a `speaking: false` is emitted. This prevents flickering indicators during natural speech pauses.

## Voice Server Discovery

When a client sends opcode 4 to join a voice channel, the gateway responds with a `VOICE_SERVER_UPDATE` dispatch event containing everything the client needs to connect:

- **endpoint**: WebSocket URL of the SFU handling this voice session
- **token**: Single-use voice token, valid for 60 seconds. Included in the first opcode 12 offer to authenticate with the SFU.
- **ice_servers**: Array of STUN/TURN server configurations for NAT traversal

The voice token binds to the specific user, server, and channel from the opcode 4 request. It cannot be reused or transferred.

If the token expires before the client sends an offer, the client must send opcode 4 again to get a fresh VOICE_SERVER_UPDATE.

## STUN/TURN Configuration

Voice connections need to traverse NATs and firewalls. The `ice_servers` array in VOICE_SERVER_UPDATE provides the necessary configuration.

**STUN (Session Traversal Utilities for NAT):**

Used for NAT type detection and gathering server-reflexive candidates. Intent runs STUN servers colocated with voice regions. No authentication required for STUN.

**TURN (Traversal Using Relays around NAT):**

Relay fallback for clients behind symmetric NATs or restrictive firewalls where direct connectivity fails. TURN relays media through the server at higher latency but guarantees connectivity.

TURN credentials are time-limited using HMAC-based ephemeral credentials (timestamp-prefixed username + HMAC-SHA1 shared secret). Credentials are valid for 12 hours and included directly in the `ice_servers` array.

**Example ice_servers payload:**

```json
[
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
```

The TURN TCP transport on port 443 exists as a last-resort fallback for networks that block all UDP traffic.

## REST Endpoints

**GET /voice/regions**

Returns available voice server regions. Clients can use this to show latency information or let users override automatic region selection.

```
GET /voice/regions
Authorization: Bearer <token>
```

**Response: 200 OK**

```json
[
  {
    "id": "us-east",
    "name": "US East",
    "endpoint": "wss://us-east.voice.intent.chat",
    "deprecated": false
  },
  {
    "id": "eu-west",
    "name": "EU West",
    "endpoint": "wss://eu-west.voice.intent.chat",
    "deprecated": false
  }
]
```

| Endpoint | Method | Description | Status |
|----------|--------|-------------|--------|
| `/voice/regions` | GET | List voice server regions | Planned |
| `/voice/turn-credentials` | POST | Request fresh TURN credentials | Planned |

**POST /voice/turn-credentials**

Requests fresh TURN credentials outside the normal voice join flow. Useful for ICE restarts when the original credentials have expired.

```
POST /voice/turn-credentials
Authorization: Bearer <token>
Content-Type: application/json

{
  "server_id": "1234567890"
}
```

**Response: 200 OK**

```json
{
  "ice_servers": [
    {
      "urls": ["turn:turn.intent.chat:3478?transport=udp"],
      "username": "1708012800:user_9876543210",
      "credential": "f6e5d4c3b2a1..."
    }
  ],
  "ttl": 43200
}
```

## Mid-Call Participant Join

When a new participant joins an active voice channel, the SFU must update all existing sessions to include the new participant's audio track.

**Flow:**

1. New user sends opcode 4, goes through the normal join flow (VOICE_STATE_UPDATE, VOICE_SERVER_UPDATE, offer/answer).
2. Once the new user's WebRTC session is established, the SFU sends opcode 12 `type: "renegotiate"` with `reason: "participant_joined"` to every existing participant.
3. Each existing participant receives a new SDP offer that includes a receive-only track for the new user.
4. Each existing participant responds with an SDP answer.
5. Media from the new participant starts flowing to all existing participants.

This process runs in parallel for all existing participants. If any single renegotiation fails, only that participant misses the new audio track; others are unaffected.

## ICE Restart

ICE connectivity can degrade due to network changes (Wi-Fi to cellular, IP change, NAT rebinding). When this happens the client or server can trigger an ICE restart.

**When to restart:**

- Client detects ICE connection state transition to `disconnected` or `failed`
- No media received for 10 seconds despite active speakers
- Network interface change detected

**Restart flow:**

1. Client generates a new SDP offer with `iceRestart: true` in the `createOffer` options.
2. Client sends the offer via opcode 12 `type: "offer"` (same as initial negotiation).
3. SFU responds with a new SDP answer containing fresh ICE credentials.
4. ICE candidate exchange proceeds again via opcode 12 `type: "ice"`.
5. Once new connectivity is established, media resumes on the new path.

**If TURN credentials have expired** during the session, the client should call `POST /voice/turn-credentials` to get fresh ones before restarting ICE.

**Fallback after repeated failures:**

If ICE restart fails 3 consecutive times, the client should perform a full rejoin: send opcode 4 with `channel_id: null` (disconnect), wait 1-3 seconds with jitter, then send opcode 4 again with the original channel.

## Speaking Indicators

The SFU detects speaking state server-side using the `ssrc-audio-level` RTP header extension defined in RFC 6464. This avoids clients needing to report their own speaking state, which would be unreliable and gameable.

**Detection parameters:**

- Audio level threshold: -40 dBov (levels above this indicate speech)
- Debounce on: 0ms (speaking starts immediately when threshold exceeded)
- Debounce off: 300ms (must be below threshold for 300ms before speaking stops)

The SFU broadcasts speaking state changes via opcode 12 `type: "speaking"` to all participants in the channel except the speaker themselves (the speaker's own client knows its local state).

Clients should use speaking state to drive visual indicators (glowing avatar borders, pulsing icons). Do not use speaking state for voice activity detection (VAD) gating — that should happen locally on the client before encoding.

## Bitrate Adaptation

The SFU uses Transport-Wide Congestion Control (TWCC) to estimate available bandwidth per receiver and adjusts forwarding accordingly.

**Opus bitrate range:** 32 - 128 kbps

Clients should configure their Opus encoder to use the full adaptive range. The SFU does not transcode or modify audio packets, but it can instruct clients to adjust their send bitrate via RTCP REMB (Receiver Estimated Maximum Bitrate) messages.

**How it works:**

1. SFU collects TWCC feedback from each receiver to estimate their downstream capacity.
2. If a receiver is congested, the SFU sends REMB to the relevant senders, requesting a lower bitrate.
3. Senders adjust their Opus encoder bitrate within the 32-128 kbps range.
4. When congestion clears, REMB values increase and senders can raise their bitrate.

This is per-receiver — one congested participant doesn't degrade audio quality for everyone else.

## Voice State Object

The authoritative representation of a user's voice connection state, broadcast in VOICE_STATE_UPDATE events.

```typescript
interface VoiceState {
  user_id: string;          // who this state belongs to
  server_id: string;        // which server
  channel_id: string | null; // which voice channel, null if disconnected
  session_id: string;       // unique session identifier for this voice connection
  self_mute: boolean;       // user muted themselves
  self_deaf: boolean;       // user deafened themselves
  server_mute: boolean;     // server-side mute (by admin)
  server_deaf: boolean;     // server-side deafen (by admin)
}
```

**Field notes:**

- `session_id` is generated when the user joins voice and remains stable across mute/deaf toggles. It changes on rejoin.
- `server_mute` and `server_deaf` can only be set by users with `MUTE_MEMBERS` and `DEAFEN_MEMBERS` permissions respectively.
- When `server_mute` is true, the SFU drops the user's audio packets rather than forwarding them. The client may still be encoding and sending audio, but nobody hears it.
- When `self_deaf` or `server_deaf` is true, the SFU stops forwarding other participants' audio to that client, saving downstream bandwidth.

## Disconnect and Cleanup

### Graceful Disconnect

Client sends opcode 4 with `channel_id: null`:

1. Gateway broadcasts VOICE_STATE_UPDATE with `channel_id: null` to the server.
2. SFU closes the client's PeerConnection and removes their tracks.
3. SFU sends opcode 12 `type: "renegotiate"` with `reason: "participant_left"` to remaining participants.
4. Remaining participants renegotiate to drop the leaving user's receive track.

### Ungraceful Disconnect

Client disappears (network failure, crash, browser tab closed):

1. SFU detects no heartbeat/RTCP from the client for 15 seconds.
2. SFU marks the session as dead and removes tracks.
3. Gateway broadcasts VOICE_STATE_UPDATE with `channel_id: null`.
4. Renegotiation proceeds with remaining participants as above.

If the client comes back within the 15-second window and successfully restarts ICE, the session is preserved without renegotiation.

### Empty Channel Cleanup

When the last participant disconnects from a voice channel, the SFU tears down all session state for that channel. No resources are held for empty voice channels.

## Security

### Voice Token Validation

The voice token from VOICE_SERVER_UPDATE is a signed JWT containing:

- `user_id`: the authenticated user
- `server_id`: the target server
- `channel_id`: the target voice channel
- `exp`: expiration timestamp (60 seconds from issuance)

The SFU validates this token on the first opcode 12 offer. If invalid or expired, the SFU rejects the connection with a close frame.

### Permission Enforcement

Permissions are checked at two points:

1. **On join (opcode 4):** Gateway checks `CONNECT` and `SPEAK` permissions. If the user lacks `CONNECT`, the request is rejected immediately.
2. **Mid-call:** If a role change removes `SPEAK` from a connected user, the SFU server-mutes them. If `CONNECT` is removed, the SFU disconnects them entirely.

Relevant permissions from the permission system:

| Permission | Bit | Voice behavior |
|------------|-----|----------------|
| `CONNECT` | `1 << 20` | Required to join voice channel |
| `SPEAK` | `1 << 21` | Required to transmit audio (listen-only without) |
| `MUTE_MEMBERS` | `1 << 22` | Server-mute other users |
| `DEAFEN_MEMBERS` | `1 << 23` | Server-deafen other users |
| `MOVE_MEMBERS` | `1 << 24` | Force-move users between voice channels |

### Transport Encryption

All media is encrypted with SRTP (Secure Real-time Transport Protocol), negotiated during the DTLS handshake. The encryption keys are ephemeral and unique per session.

The SFU terminates DTLS-SRTP — it decrypts incoming packets and re-encrypts them for each receiver with their individual session keys. This means the SFU can see unencrypted audio content. This is standard for any SFU architecture.

### Future: End-to-End Encrypted Voice

True E2EE voice (where the SFU cannot access audio content) will be implemented using the Insertable Streams API (WebRTC Encoded Transform). Under E2EE voice:

- Clients encrypt audio frames with MLS group keys before passing them to the WebRTC stack
- The SFU forwards the encrypted packets as-is (it cannot decrypt them)
- Receiving clients decrypt using the shared MLS group key

This builds on the MLS E2EE infrastructure defined in `encryption/mls-spec.md`. Implementation is post-Phase 1.

## Summary

### Opcodes

| Opcode | Name | Direction | Voice role | Status |
|--------|------|-----------|------------|--------|
| 4 | Voice State Update | Client -> Server | Join/leave/mute/deaf | Planned |
| 12 | Voice Signal | Bidirectional | SDP/ICE/speaking | Planned |

### Dispatch Events

| Event | Direction | Voice role | Status |
|-------|-----------|------------|--------|
| VOICE_STATE_UPDATE | Server -> Client | Broadcast voice state changes | Planned |
| VOICE_SERVER_UPDATE | Server -> Client | Provide SFU endpoint + token | Planned |

### REST Endpoints

| Endpoint | Method | Description | Status |
|----------|--------|-------------|--------|
| `/voice/regions` | GET | List voice server regions | Planned |
| `/voice/turn-credentials` | POST | Fresh TURN credentials for ICE restart | Planned |

## Error Handling

| Scenario | Error | Client action |
|----------|-------|---------------|
| Missing CONNECT permission | Gateway close 4014 | Show permission error, don't retry |
| Voice channel full (25 users) | Gateway close 4015 | Show "channel full" message |
| Channel does not exist | Gateway close 4011 | Remove channel from local state |
| Voice token expired | SFU rejects offer | Send opcode 4 again for fresh token |
| Voice server unreachable | SFU endpoint connection fails | Exponential backoff: 1s, 2s, 4s, 8s (max 15s) |
| ICE failure after 3 restarts | ICE state stuck at failed | Full rejoin (disconnect + reconnect) |
| Mid-call permission revoked | SFU disconnects or server-mutes | VOICE_STATE_UPDATE reflects new state |

**Reconnection backoff:**

Use jittered exponential backoff for all voice reconnection attempts. Base delay 1 second, multiplier 2x, max 15 seconds, jitter +/-25%.

## Future Enhancements

These are not part of the current specification but are planned for future phases:

- **Video and screen share:** Additional media tracks negotiated via opcode 12 renegotiation. Video codecs (VP8/VP9/AV1) will be specified in `voice/codecs.md` when implemented. Simulcast for bandwidth adaptation.
- **E2EE voice:** End-to-end encrypted audio using Insertable Streams and MLS group keys. See `encryption/mls-spec.md`.
- **Stage channels:** One-to-many broadcast channels with speaker/audience roles and hand-raise mechanics.
- **Priority speaker:** A permission-gated role that boosts one user's audio and attenuates others, for moderation or presentations.
- **Noise suppression hints:** Server-side metadata to help clients decide whether to enable local noise suppression based on channel size and type.
