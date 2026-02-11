# Intent Protocol Specification

Open specification for the Intent communication protocol.

This repository contains the complete technical specification for building Intent-compatible clients, bots, and integrations.

## Contents

- **REST API** - HTTP endpoints, request/response formats
- **Gateway** - WebSocket protocol, opcodes, events
- **Voice** - WebRTC signaling, SDP/ICE exchange
- **Encryption** - MLS E2EE specification
- **Webhooks** - Webhook payload formats, Discord compatibility
- **Discord Mapping** - API equivalences for migration

## Purpose

This spec enables:
- Third-party client development
- Bot SDK implementation
- Integration tools
- Migration from Discord

## Status

 **Work in Progress** - Being extracted from server implementation.

Documentation is being added incrementally as components are finalized.

## Structure

```
intent-protocol/
├── rest-api/      # REST API specification
├── gateway/      # WebSocket protocol
├── voice/       # Voice/WebRTC specs
├── encryption/     # E2EE specifications
├── webhooks/      # Webhook formats
├── discord-mapping/  # Discord API equivalences
└── schemas/      # JSON schemas
```

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md)

Protocol documentation contributions are welcome!

## License

MIT License - See [LICENSE](LICENSE)
