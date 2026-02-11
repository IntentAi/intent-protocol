# Gateway Events

Events dispatched via opcode 0 (Dispatch).

## Event Types

### Connection
- `READY` - Initial connection complete
- `RESUMED` - Connection resumed

### Servers
- `SERVER_CREATE` - Server created/joined
- `SERVER_UPDATE` - Server updated
- `SERVER_DELETE` - Server deleted/left

### Channels
- `CHANNEL_CREATE` - Channel created
- `CHANNEL_UPDATE` - Channel updated
- `CHANNEL_DELETE` - Channel deleted

### Messages
- `MESSAGE_CREATE` - New message
- `MESSAGE_UPDATE` - Message edited
- `MESSAGE_DELETE` - Message deleted

### Voice
- `VOICE_STATE_UPDATE` - User voice state changed
- `VOICE_SERVER_UPDATE` - Voice server connection info

 Full event schemas in development
