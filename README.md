# Intent Protocol Specification

Open specification for the Intent communication protocol.

This repository contains the complete technical specification for building Intent-compatible clients, bots, and integrations.

## Contents

### REST API (`rest-api/`)
- [**Resources**](rest-api/resources.md) — Servers, channels, messages CRUD with full endpoint specs
- [**Authentication**](rest-api/authentication.md) — Token types, permission bitfields, OAuth2 outline
- [**Rate Limiting**](rest-api/rate-limiting.md) — Per-route buckets, global limits, response headers

### Gateway (`gateway/`)
- [**Opcodes**](gateway/opcodes.md) — 13 opcodes (0-12), connection lifecycle, heartbeat, resume
- [**Events**](gateway/events.md) — 15 event types with payload schemas and examples
- [**MessagePack**](gateway/messagepack.md) — Binary encoding spec, type mappings, wire format, client examples

### Voice (`voice/`)
- [**Signaling**](voice/signaling.md) — WebRTC SFU architecture, connection lifecycle, STUN/TURN, speaking indicators
- [**Codecs**](voice/codecs.md) — Opus configuration, RTP header extensions, RTCP feedback, bitrate profiles

### Encryption (`encryption/`)
- [**MLS E2EE**](encryption/mls-spec.md) — MLS (RFC 9420) for encrypted DMs, key packages, ciphersuite selection

### Webhooks (`webhooks/`)
- [**Format**](webhooks/format.md) — Webhook payloads, Discord-compatible endpoint format

### Discord Mapping (`discord-mapping/`)
- [**Endpoints**](discord-mapping/endpoints.md) — REST endpoint equivalences
- [**Events**](discord-mapping/events.md) — Gateway event name mapping
- [**Objects**](discord-mapping/objects.md) — Object field mapping

### Schemas (`schemas/`)
- [**OpenAPI 3.1**](schemas/openapi.yaml) — Machine-readable spec for code generation and validation

## Purpose

This spec enables:
- Third-party client development
- Bot SDK implementation (see [intent.js](https://github.com/IntentAi/intent.js), [intent.py](https://github.com/IntentAi/intent.py))
- Integration tools
- Migration from Discord

## Status

Phase 1 specification is near-complete. REST API, gateway protocol, voice signaling, and encryption specs are documented. Opcodes 0-3 and 11 are implemented server-side; remaining opcodes (4-10, 12) are specified for future implementation.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md)

Protocol documentation contributions are welcome.

## License

MIT License - See [LICENSE](LICENSE)
