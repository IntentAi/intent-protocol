# REST API Specification

Intent REST API follows RESTful patterns similar to Discord but with cleaner design.

## Base URL

```
https://api.intent.chat/v1
```

## Authentication

All requests require a bearer token:

```
Authorization: Bearer <token>
```

See [authentication.md](authentication.md) for token types, permission bitfields, and OAuth2 flow.

## Documentation

- [**Resources**](resources.md) — Servers, channels, messages with full CRUD operations
- [**Authentication**](authentication.md) — Token formats, permission system, OAuth2 outline
- [**Rate Limiting**](rate-limiting.md) — Per-route buckets, global limits, retry headers

## OpenAPI Specification

Machine-readable OpenAPI 3.1 spec available at [`schemas/openapi.yaml`](../schemas/openapi.yaml). Use it for code generation, type extraction, and request validation.
