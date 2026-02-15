# Intent API Schemas

Machine-readable schemas for the Intent API using OpenAPI 3.1.

## Files

- `openapi.yaml` - Complete OpenAPI 3.1 specification with all models and endpoints

## Usage

### Validate the Spec

```bash
npx @redocly/cli lint openapi.yaml
```

### Generate Documentation

```bash
npx @redocly/cli build-docs openapi.yaml -o index.html
```

### Generate TypeScript Types

```bash
npx openapi-typescript openapi.yaml -o types.ts
```

### Generate Python Client

```bash
pip install openapi-python-client
openapi-python-client generate --path openapi.yaml
```

### Generate Go Client

```bash
go install github.com/deepmap/oapi-codegen/cmd/oapi-codegen@latest
oapi-codegen -package intent -generate types,client openapi.yaml > client.go
```

## Models

Core schemas defined:

- **User** - User account
- **Server** - Server/guild
- **Channel** - Text/voice/category channel
- **Message** - Chat message with attachments and embeds
- **Role** - Permission role
- **Member** - Server member (user + server-specific data)
- **Attachment** - File attachment
- **Embed** - Rich embed content
- **Error** - API error response

## Endpoints

REST API paths included:

- `GET /servers/{id}` - Get server
- `POST /servers` - Create server
- `PATCH /servers/{id}` - Update server
- `DELETE /servers/{id}` - Delete server
- `GET /channels/{id}` - Get channel
- `POST /servers/{id}/channels` - Create channel
- `PATCH /channels/{id}` - Update channel
- `DELETE /channels/{id}` - Delete channel
- `GET /channels/{id}/messages` - List messages
- `GET /channels/{id}/messages/{id}` - Get message
- `POST /channels/{id}/messages` - Create message
- `PATCH /channels/{id}/messages/{id}` - Update message
- `DELETE /channels/{id}/messages/{id}` - Delete message

## Integration

SDK developers should use this spec to:

1. Auto-generate TypeScript types for intent.js
2. Auto-generate Python dataclasses for intent.py
3. Validate API requests/responses in tests
4. Generate client libraries
5. Generate API documentation

## Notes

- All IDs are Snowflakes (64-bit integers as strings)
- All timestamps are ISO 8601 UTC format
- Bearer authentication required for all endpoints
- Error responses include `error` (string) and `code` (string) fields
- Rate limiting follows specification in `rest-api/rate-limiting.md`
