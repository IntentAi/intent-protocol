# Authentication & Authorization

This document specifies how authentication and authorization work in the Intent API.

## Token Types

Intent uses bearer tokens for authentication. There are two token types: user tokens and bot tokens.

### User Tokens

User tokens authenticate as a specific user account. These are obtained through OAuth2 flows and represent a user's session.

**Format:**

```
usr_<base64url>
```

User tokens begin with the `usr_` prefix followed by base64url-encoded data. The encoded portion contains the user ID, issuance timestamp, and signature.

Example: `usr_MTIzNDU2Nzg5MC4xNzA1MzIwMDAwLmFiY2RlZmdoaWprbG1ub3BxcnN0dXZ3eHl6`

### Bot Tokens

Bot tokens authenticate as a bot application. These are generated when creating a bot application and never expire unless manually revoked.

**Format:**

```
bot_<base64url>
```

Bot tokens begin with the `bot_` prefix followed by base64url-encoded data containing the bot application ID and secret key.

Example: `bot_OTg3NjU0MzIxMC5hYmNkZWZnaGlqa2xtbm9wcXJzdHV2d3h5ejEyMzQ1Njc4OTA`

## Authentication Header

All authenticated requests MUST include the `Authorization` header with the bearer token:

```
Authorization: Bearer <token>
```

Example:

```http
GET /servers/1234567890 HTTP/1.1
Host: api.intent.chat
Authorization: Bearer bot_OTg3NjU0MzIxMC5hYmNkZWZnaGlqa2xtbm9wcXJzdHV2d3h5ejEyMzQ1Njc4OTA
```

The server MUST reject requests with invalid, expired, or missing tokens with `401 Unauthorized`.

## Obtaining Tokens

### Bot Token Acquisition

1. Create a bot application through the Intent developer portal
2. Navigate to the bot settings page
3. Click "Regenerate Token" to generate a new bot token
4. Copy the token immediately - it will only be shown once

Bot tokens SHOULD be stored securely and never committed to version control or shared publicly.

### User Token Acquisition

User tokens are obtained through the OAuth2 authorization code flow:

1. Redirect user to `https://intent.chat/oauth/authorize` with client credentials
2. User grants permission
3. Receive authorization code via redirect
4. Exchange code for access token at `POST /oauth/token`

User tokens expire after 7 days. Refresh tokens can be used to obtain new access tokens without re-prompting the user.

## Permissions

Permissions control what actions a user or bot can perform. Permissions are represented as a bitfield where each bit corresponds to a specific permission.

### Permission Bitfield

Permissions are 64-bit integers. Multiple permissions are combined using bitwise OR.

**Server Permissions:**

| Permission | Value | Description |
|------------|-------|-------------|
| `MANAGE_SERVER` | `1 << 0` | Modify server settings |
| `MANAGE_ROLES` | `1 << 1` | Create, edit, delete roles |
| `MANAGE_CHANNELS` | `1 << 2` | Create, edit, delete channels |
| `KICK_MEMBERS` | `1 << 3` | Remove members from server |
| `BAN_MEMBERS` | `1 << 4` | Ban members from server |
| `MANAGE_WEBHOOKS` | `1 << 5` | Create, edit, delete webhooks |
| `VIEW_AUDIT_LOG` | `1 << 6` | View server audit log |

**Channel Permissions:**

| Permission | Value | Description |
|------------|-------|-------------|
| `VIEW_CHANNEL` | `1 << 10` | See channel in list |
| `SEND_MESSAGES` | `1 << 11` | Send messages in channel |
| `MANAGE_MESSAGES` | `1 << 12` | Delete messages from others |
| `EMBED_LINKS` | `1 << 13` | Links auto-embed |
| `ATTACH_FILES` | `1 << 14` | Upload files and attachments |
| `READ_MESSAGE_HISTORY` | `1 << 15` | View messages sent before joining |
| `MENTION_EVERYONE` | `1 << 16` | Use @everyone and @here |

**Voice Permissions:**

| Permission | Value | Description |
|------------|-------|-------------|
| `CONNECT` | `1 << 20` | Connect to voice channel |
| `SPEAK` | `1 << 21` | Speak in voice channel |
| `MUTE_MEMBERS` | `1 << 22` | Mute members in voice |
| `DEAFEN_MEMBERS` | `1 << 23` | Deafen members in voice |
| `MOVE_MEMBERS` | `1 << 24` | Move members between channels |

### Permission Resolution

Permissions are resolved in this order:

1. Server owner has all permissions implicitly
2. Check user's roles in the server and compute combined permissions via bitwise OR
3. Apply channel-specific permission overwrites if they exist
4. If permission bit is set, access is granted

### Permission Scopes (OAuth2)

When users authorize an application via OAuth2, they grant specific scopes. Applications receive only the permissions explicitly granted.

**Available Scopes:**

- `identify` - Read user ID, username, avatar
- `servers` - Read user's server list
- `servers.join` - Join servers on user's behalf
- `messages.read` - Read messages in channels user has access to
- `messages.write` - Send messages as the user
- `bot` - Add bot to servers (requires bot application)

Scope strings are space-separated in the OAuth2 flow: `scope=identify servers messages.read`

## Authorization Checks

### REST API

For each REST request, the server MUST:

1. Verify the bearer token is valid and not expired
2. Resolve the user or bot from the token
3. Check if the user/bot has permission to perform the requested operation
4. Return `403 Forbidden` if permission check fails

### Gateway

Gateway connections authenticate during the `IDENTIFY` opcode. The server validates the token and maintains the authenticated session. Permission checks happen when processing operations like sending messages.

## Self-Hosted Deployments

Self-hosted Intent instances MAY implement their own token generation and validation, but SHOULD maintain compatibility with the token format and permission bitfield structure.

Self-hosted instances MUST NOT accept tokens from other instances unless explicitly configured for federation.

## Federation

Federation allows multiple Intent instances to interoperate. Federated instances MUST implement cross-instance token validation.

When a user from instance A accesses resources on instance B:

1. Instance B receives request with token issued by instance A
2. Instance B validates token by querying instance A's public key endpoint
3. Instance A responds with token validity and associated user/bot ID
4. Instance B caches validation result for token lifetime

Federated deployments are optional. Instances MAY choose to operate in isolated mode and reject all foreign tokens.

## Security Considerations

### Token Storage

- Bot tokens MUST be stored securely and never exposed in client-side code
- User tokens SHOULD be stored in secure, httpOnly cookies when used in web contexts
- Tokens MUST be transmitted only over HTTPS

### Token Revocation

Users can revoke tokens at any time through the Intent settings panel. The server MUST immediately invalidate revoked tokens and reject subsequent requests.

Bot developers can regenerate bot tokens, which invalidates the previous token immediately.

### Rate Limiting

Token-based authentication is subject to rate limiting. See rate limiting specification for details. Each token has its own rate limit bucket.
