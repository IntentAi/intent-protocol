# Discord API Mapping - Events

Gateway event name mappings between Intent and Discord.

## Terminology

Intent uses "Server" where Discord uses "Guild". Event names reflect this.

## Message Events

| Intent | Discord | Status |
|--------|---------|--------|
| `MESSAGE_CREATE` | `MESSAGE_CREATE` | Implemented |
| `MESSAGE_UPDATE` | `MESSAGE_UPDATE` | Implemented |
| `MESSAGE_DELETE` | `MESSAGE_DELETE` | Implemented |

## Server Events

| Intent | Discord | Status |
|--------|---------|--------|
| `SERVER_CREATE` | `GUILD_CREATE` | Implemented |
| `SERVER_UPDATE` | `GUILD_UPDATE` | Planned |
| `SERVER_DELETE` | `GUILD_DELETE` | Planned |

## Channel Events

| Intent | Discord | Status |
|--------|---------|--------|
| `CHANNEL_CREATE` | `CHANNEL_CREATE` | Implemented |
| `CHANNEL_UPDATE` | `CHANNEL_UPDATE` | Planned |
| `CHANNEL_DELETE` | `CHANNEL_DELETE` | Planned |

## Member Events

| Intent | Discord | Status |
|--------|---------|--------|
| `MEMBER_ADD` | `GUILD_MEMBER_ADD` | Planned |
| `MEMBER_REMOVE` | `GUILD_MEMBER_REMOVE` | Planned |

## Presence Events

| Intent | Discord | Status |
|--------|---------|--------|
| `PRESENCE_UPDATE` | `PRESENCE_UPDATE` | Planned |
| `TYPING_START` | `TYPING_START` | Planned |

## Voice Events

| Intent | Discord | Status |
|--------|---------|--------|
| `VOICE_STATE_UPDATE` | `VOICE_STATE_UPDATE` | Planned |
| `VOICE_SERVER_UPDATE` | `VOICE_SERVER_UPDATE` | Planned |

## Connection Events

| Intent | Discord | Status |
|--------|---------|--------|
| `READY` (opcode 3) | `READY` (dispatch) | Implemented |

**Note:** Intent sends READY via opcode 3. Future versions will use dispatch like Discord.

## Not Supported

These Discord events have no Intent equivalent:

| Discord Event | Reason |
|---------------|--------|
| `GUILD_BAN_ADD/REMOVE` | Different moderation system |
| `GUILD_EMOJIS_UPDATE` | Custom emoji not planned |
| `GUILD_STICKERS_UPDATE` | Stickers not planned |
| `GUILD_INTEGRATIONS_UPDATE` | No integrations |
| `STAGE_INSTANCE_*` | No stage channels |
| `GUILD_SCHEDULED_EVENT_*` | No scheduled events |
| `THREAD_*` | No threads (Phase 1) |
| `AUTO_MODERATION_*` | Different approach |

## Payload Differences

Most event payloads are structurally similar. Key differences:

1. **Field naming:** `guild_id` → `server_id`
2. **Object nesting:** Intent wraps data (e.g., `{ message: Message }` vs flat)
3. **Missing fields:** Intent Phase 1 omits some Discord fields (see objects.md)

## Migration Notes

For bot developers:

```javascript
// Discord
client.on('GUILD_CREATE', (guild) => { ... })

// Intent
client.on('SERVER_CREATE', (payload) => {
  const server = payload.server  // Note: wrapped in object
})
```
