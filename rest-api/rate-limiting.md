# Rate Limiting

Rate limiting specification for the Intent REST API.

All endpoints at `https://api.intent.chat/v1` are subject to rate limiting. Limits are enforced per-token - each user or bot token has independent tracking.

## Rate Limit Strategy

### Per-Route Buckets

Endpoints share rate limit buckets. Requests to the same bucket share a counter that resets after the time window.

| Endpoint Pattern | Limit | Window |
|-----------------|-------|--------|
| `POST /channels/:id/messages` | 5 requests | 5 seconds |
| `PATCH /channels/:id/messages/:message_id` | 5 requests | 5 seconds |
| `DELETE /channels/:id/messages/:message_id` | 5 requests | 5 seconds |
| `GET /channels/:id/messages` | 50 requests | 60 seconds |
| `POST /servers` | 1 request | 600 seconds |
| `PATCH /servers/:id` | 10 requests | 60 seconds |
| `GET /servers/:id` | 100 requests | 60 seconds |
| `POST /servers/:id/channels` | 10 requests | 60 seconds |
| `PATCH /channels/:id` | 10 requests | 60 seconds |
| `GET /channels/:id` | 100 requests | 60 seconds |

**Bucket identification:**

Buckets are identified by route pattern and major parameters. `POST /channels/123/messages` and `POST /channels/456/messages` use different buckets because the channel ID differs.

Major parameters creating separate buckets:
- `:server_id` - Different servers get separate buckets
- `:channel_id` - Different channels get separate buckets
- `:webhook_id` - Different webhooks get separate buckets

Minor parameters like `:message_id` or `:user_id` share the parent resource's bucket.

### Global Rate Limit

All requests also count toward a global limit to prevent resource monopolization.

- User tokens: 50 requests per second
- Bot tokens: 50 requests per second

Global limits use a 1-second sliding window.

### Authentication Endpoints

Authentication endpoints have stricter IP-based limits to prevent brute force attacks:

| Endpoint | Limit | Window |
|----------|-------|--------|
| `POST /auth/login` | 5 requests | 300 seconds |
| `POST /auth/register` | 3 requests | 3600 seconds |
| `POST /oauth/token` | 10 requests | 60 seconds |

## HTTP Headers

Every response includes rate limit headers for client-side tracking:

```http
X-RateLimit-Limit: 5
X-RateLimit-Remaining: 3
X-RateLimit-Reset: 1708012345
X-RateLimit-Bucket: ch:123:msg
X-RateLimit-Global: false
```

**Header definitions:**

- `X-RateLimit-Limit` - Maximum requests allowed in this bucket
- `X-RateLimit-Remaining` - Requests remaining in current window
- `X-RateLimit-Reset` - Unix timestamp (seconds) when bucket resets
- `X-RateLimit-Bucket` - Bucket identifier for tracking
- `X-RateLimit-Global` - Whether global limit applies

Headers appear on ALL responses, not just when approaching limits.

### Bucket IDs

The `X-RateLimit-Bucket` header format: `<scope>:<id>:<operation>`

Examples:
- `ch:123:msg` - Messaging in channel 123
- `sv:456:mod` - Modifying server 456
- `global` - Global rate limit

Clients MUST use the exact bucket ID from the server. Do not construct bucket IDs client-side.

## 429 Rate Limit Exceeded

When a limit is exceeded, the server returns `429 Too Many Requests`:

```http
HTTP/1.1 429 Too Many Requests
Content-Type: application/json
X-RateLimit-Limit: 5
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1708012350
X-RateLimit-Bucket: ch:123:msg
Retry-After: 4.5

{
  "error": "You are being rate limited.",
  "code": "RATE_LIMIT_EXCEEDED",
  "retry_after": 4.5,
  "global": false
}
```

**Response fields:**

- `error` - Human-readable error message
- `code` - Error code for programmatic handling
- `retry_after` - Seconds to wait before retrying (float)
- `global` - Whether this is a global rate limit

### Global Rate Limit Response

When global limit is exceeded:

```json
{
  "error": "You are being rate limited globally.",
  "code": "RATE_LIMIT_GLOBAL",
  "retry_after": 0.8,
  "global": true
}
```

Global rate limits have shorter retry windows (typically under 1 second).

## Client Implementation

### Tracking Rate Limits

Clients should track buckets to avoid hitting limits:

```python
buckets = {}

def update_bucket(response):
    bucket_id = response.headers['X-RateLimit-Bucket']
    buckets[bucket_id] = {
        'limit': int(response.headers['X-RateLimit-Limit']),
        'remaining': int(response.headers['X-RateLimit-Remaining']),
        'reset': int(response.headers['X-RateLimit-Reset'])
    }

def should_wait(bucket_id):
    if bucket_id not in buckets:
        return 0
    bucket = buckets[bucket_id]
    if bucket['remaining'] > 0:
        return 0
    return max(0, bucket['reset'] - time.time())
```

### Handling 429 Responses

When receiving a 429:

1. Read `retry_after` from response body
2. If `global: true`, pause ALL requests for that duration
3. If `global: false`, only pause requests to that bucket
4. Wait the specified time before retrying

```python
def handle_429(response):
    data = response.json()
    time.sleep(data['retry_after'])
    # Then retry the request
```

### Exponential Backoff

If repeated 429s occur despite respecting `retry_after`:

```python
def retry_with_backoff(func, max_retries=5):
    for attempt in range(max_retries):
        try:
            return func()
        except RateLimitError:
            if attempt == max_retries - 1:
                raise
            delay = (2 ** attempt) + random.uniform(0, 1)
            time.sleep(delay)
```

## Webhook Rate Limits

Webhooks have dual limits:

| Operation | Limit | Window |
|-----------|-------|--------|
| `POST /webhooks/:id/:token` | 5 requests | 2 seconds |
| `POST /webhooks/:id/:token` | 30 requests | 60 seconds |

Both limits must be respected. Limits are per webhook ID (shared across all callers).

## Special Cases

### File Uploads

File uploads count toward normal messaging limits with additional restrictions:

- Max file size: 25 MB (user tokens), 100 MB (bot tokens)
- Max 10 files per message
- Use longer timeouts for upload requests

### Burst Capacity

Messaging buckets allow bursts: send 5 messages immediately, then refill at 1/second. This enables natural conversation without artificial delays.

## Error Codes

Rate limiting error codes:

- `RATE_LIMIT_EXCEEDED` - Per-route bucket exceeded
- `RATE_LIMIT_GLOBAL` - Global limit exceeded
- `RATE_LIMIT_AUTH` - Auth endpoint exceeded (IP-based)

Example:

```json
{
  "error": "You are being rate limited.",
  "code": "RATE_LIMIT_EXCEEDED",
  "retry_after": 4.5,
  "global": false
}
```

## Self-Hosted Deployments

Self-hosted instances MAY configure different limits but SHOULD maintain the same header format and response structure for client compatibility. Use environment variables to configure limits.

## Best Practices

- Track rate limits per bucket using server-provided bucket IDs
- Wait pre-emptively when `X-RateLimit-Remaining` reaches 0
- Respect `retry_after` in 429 responses
- Implement exponential backoff for repeated failures
- Handle both per-route and global limits
- Parse headers on every response, not just errors

Following this spec ensures graceful rate limit handling without user-facing errors.
