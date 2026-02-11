# Gateway Opcodes

WebSocket gateway uses MessagePack binary protocol with opcode-based messages.

## Opcode List

| Opcode | Name | Direction | Description |
|--------|------|-----------|-------------|
| 0 | Dispatch | Server → Client | Event dispatch |
| 1 | Heartbeat | Client → Server | Heartbeat ping |
| 2 | Identify | Client → Server | Initial connection |
| 3 | Presence Update | Client → Server | Update presence |
| 4 | Voice State Update | Client → Server | Voice state change |
| 5 | Resume | Client → Server | Resume connection |
| 6 | Reconnect | Server → Client | Server requests reconnect |
| 7 | Request Guild Members | Client → Server | Request member list |
| 8 | Invalid Session | Server → Client | Session invalid |
| 9 | Hello | Server → Client | Connection established |
| 10 | Heartbeat ACK | Server → Client | Heartbeat acknowledged |
| 11 | Voice Signal | Bidirectional | WebRTC signaling |

 Full opcode documentation in development
