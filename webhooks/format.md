# Webhook Specification

Webhooks let external services send messages to Intent channels without a bot or gateway connection.

## Webhook Object

```typescript
{
  id: string,              // Snowflake ID
  type: number,            // 1 = Incoming (Phase 1 only)
  channel_id: string,      // Target channel
  server_id: string,       // Parent server
  creator_id: string,      // User who created (omitted with token auth)
  name: string,            // Display name (1-80 chars)
  avatar_url: string | null,
  token: string,           // Secret execution token
  url: string,             // Full execution URL (convenience)
  created_at: string       // ISO 8601
}
```

**Max webhooks per channel:** 15

## REST Endpoints

All authenticated endpoints require `MANAGE_WEBHOOKS` permission. Token-authenticated variants (`/webhooks/:id/:token`) don't need an auth header.

### Create Webhook

`POST /channels/:channel_id/webhooks`

```json
{
  "name": "GitHub Notifications",
  "avatar_url": "https://cdn.intent.chat/avatars/github.png"
}
```

**Response:** `201` - Webhook object (includes token and url)

### List Channel Webhooks

`GET /channels/:channel_id/webhooks`

**Response:** `200` - Array of webhook objects

### List Server Webhooks

`GET /servers/:server_id/webhooks`

**Response:** `200` - Array of all webhook objects across the server

### Get Webhook

`GET /webhooks/:webhook_id` (auth required)
`GET /webhooks/:webhook_id/:token` (no auth, omits `creator_id`)

**Response:** `200` - Webhook object

### Update Webhook

`PATCH /webhooks/:webhook_id` (auth required)
`PATCH /webhooks/:webhook_id/:token` (no auth)

```json
{
  "name": "Updated Name",
  "avatar_url": "https://cdn.intent.chat/avatars/new.png",
  "channel_id": "9999999999"
}
```

All fields optional. `channel_id` moves the webhook to a different channel.

**Response:** `200` - Updated webhook object

### Delete Webhook

`DELETE /webhooks/:webhook_id` (auth required)
`DELETE /webhooks/:webhook_id/:token` (no auth)

**Response:** `204` - No content

### Regenerate Token

`POST /webhooks/:webhook_id/regenerate-token`

Invalidates the current token and returns a new one. Auth required.

**Response:** `200` - Updated webhook object with new token and url

## Executing Webhooks

`POST /webhooks/:webhook_id/:token`

No auth header needed. The token authenticates the request.

**Request body:**

```typescript
{
  content?: string,          // Message text (max 2000 chars)
  username?: string,         // Override display name for this message
  avatar_url?: string,       // Override avatar for this message
  embeds?: Embed[]           // Up to 10 embed objects
}
```

At least one of `content` or `embeds` is required.

**Query parameters:**

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `wait` | boolean | false | Return the created Message object |

**Response:**
- `wait=false` (default): `204` No Content
- `wait=true`: `200` Message object

**Example:**

```bash
curl -X POST "https://api.intent.chat/v1/webhooks/123456/abc-token-xyz?wait=true" \
  -H "Content-Type: application/json" \
  -d '{"content": "Build #142 passed", "username": "CI Bot"}'
```

## Webhook Message Management

Messages sent by webhooks can be retrieved, edited, or deleted using the webhook token.

`GET /webhooks/:webhook_id/:token/messages/:message_id`
`PATCH /webhooks/:webhook_id/:token/messages/:message_id`
`DELETE /webhooks/:webhook_id/:token/messages/:message_id`

PATCH accepts the same body as execution (`content`, `embeds`).

## Discord Compatibility

Intent webhook execution accepts Discord-formatted payloads. Change the URL, keep the payload.

**Compatible fields:**
- `content`, `username`, `avatar_url`, `embeds`, `wait`

**Ignored fields (accepted but no-op):**
- `tts` - no text-to-speech
- `allowed_mentions` - Phase 1 doesn't filter mentions
- `components` - no message components
- `flags` - no message flags
- `thread_id` - no threads
- `attachments` - file uploads not yet supported

**URL change:** `discord.com/api/webhooks/` becomes `api.intent.chat/v1/webhooks/`

## Rate Limits

Per webhook ID, shared across all callers:

| Limit | Window | Type |
|-------|--------|------|
| 5 requests | 2 seconds | Burst |
| 30 requests | 60 seconds | Sustained |

Both limits apply simultaneously. Management endpoints follow standard API rate limits.

Exceeding either returns `429` with `Retry-After` header (seconds).

## Signature Verification

When Intent delivers outbound webhooks (server-to-external), payloads are signed with HMAC-SHA256 so recipients can verify authenticity.

### Signature Headers

| Header | Value |
|--------|-------|
| `X-Intent-Signature` | `sha256=<hex digest>` |
| `X-Intent-Timestamp` | Unix timestamp (seconds) |
| `X-Intent-Delivery-Id` | Unique delivery ID for idempotency |

### Computing the Signature

The signed message is `{timestamp}.{raw_request_body}`. The key is your webhook signing secret.

```python
import hmac, hashlib, time

def verify_signature(secret: str, timestamp: str, body: str, signature: str) -> bool:
    # Reject stale requests (replay protection)
    if abs(time.time() - int(timestamp)) > 300:
        return False

    message = f"{timestamp}.{body}".encode()
    expected = hmac.new(secret.encode(), message, hashlib.sha256).hexdigest()
    # Constant-time comparison to prevent timing attacks
    return hmac.compare_digest(f"sha256={expected}", signature)
```

```javascript
const crypto = require('crypto');

function verifySignature(secret, timestamp, body, signature) {
  if (Math.abs(Date.now() / 1000 - Number(timestamp)) > 300) return false;

  const expected = crypto
    .createHmac('sha256', secret)
    .update(`${timestamp}.${body}`)
    .digest('hex');
  return crypto.timingSafeEqual(
    Buffer.from(`sha256=${expected}`),
    Buffer.from(signature)
  );
}
```

### Idempotency

Use `X-Intent-Delivery-Id` to deduplicate deliveries. Store processed IDs for at least 24 hours.

### Key Management

The signing secret is generated when creating an outbound webhook. Regenerate it with `POST /webhooks/:id/regenerate-token`. Old signatures become invalid immediately.

## Retry Policy

For outbound webhook delivery to external endpoints:

| Attempt | Delay | Cumulative |
|---------|-------|------------|
| 1st retry | 1 second | ~1s |
| 2nd retry | 5 seconds | ~6s |
| 3rd retry | 30 seconds | ~36s |
| 4th retry | 2 minutes | ~2.5min |
| 5th retry | 10 minutes | ~12.5min |

Jitter of up to 20% is added to each delay to prevent thundering herd.

After 5 failed attempts, the delivery is dropped and logged.

**Failure conditions:**
- Connection timeout (5 seconds)
- HTTP response >= 400 (except 429, which is retried with `Retry-After`)
- DNS resolution failure
- TLS handshake failure

**Automatic disable:** After 50 consecutive failed deliveries, the webhook is disabled and the creator is notified via system DM. Re-enable via `PATCH /webhooks/:id` with `{"enabled": true}`.

## Webhook Messages

Messages from webhooks look like normal messages with an extra `webhook_id` field.

```typescript
{
  id: string,
  channel_id: string,
  webhook_id: string,        // Present only on webhook messages
  author: {
    id: string,              // Webhook ID (not a real user)
    username: string,        // Webhook name or override
    display_name: string,
    avatar_url: string | null
  },
  content: string,
  created_at: string,
  edited_at: string | null
}
```

These dispatch `MESSAGE_CREATE` like any other message. Clients identify webhook messages by checking for `webhook_id`.

## Lifecycle

- Deleting a channel deletes all its webhooks
- Deleting a server deletes all its webhooks
- Webhooks don't expire, but tokens can be regenerated
- Disabled webhooks (from failed deliveries) keep their configuration
