# Voice Signaling Protocol

WebRTC SDP/ICE exchange for voice channels.

## Connection Flow

1. Client joins voice channel (opcode 8)
2. Server responds with connection info (opcode 9)
3. WebRTC SDP offer/answer exchange
4. ICE candidate exchange
5. Media flow begins

 Full signaling specification in development
