# Discord API Mapping - Endpoints

REST API endpoint equivalences between Intent and Discord.

## Quick Reference

| Category | Intent Base | Discord Base |
|----------|-------------|--------------|
| API | `api.intent.chat/v1` | `discord.com/api/v10` |
| Gateway | `gateway.intent.chat` | `gateway.discord.gg` |
| CDN | `cdn.intent.chat` | `cdn.discordapp.com` |

## Server (Guild) Operations

| Intent | Discord | Status |
|--------|---------|--------|
| `GET /servers/:id` | `GET /guilds/:id` | Implemented |
| `POST /servers` | `POST /guilds` | Implemented |
| `PATCH /servers/:id` | `PATCH /guilds/:id` | Implemented |
| `DELETE /servers/:id` | `DELETE /guilds/:id` | Implemented |
| `GET /servers/:id/members` | `GET /guilds/:id/members` | Planned |
| `GET /servers/:id/members/:user_id` | `GET /guilds/:id/members/:user_id` | Planned |
| `PATCH /servers/:id/members/:user_id` | `PATCH /guilds/:id/members/:user_id` | Planned |
| `DELETE /servers/:id/members/:user_id` | `DELETE /guilds/:id/members/:user_id` | Planned |

**Note:** Intent uses "server", Discord uses "guild"

## Channel Operations

| Intent | Discord | Status |
|--------|---------|--------|
| `GET /channels/:id` | `GET /channels/:id` | Implemented |
| `POST /servers/:id/channels` | `POST /guilds/:id/channels` | Implemented |
| `PATCH /channels/:id` | `PATCH /channels/:id` | Implemented |
| `DELETE /channels/:id` | `DELETE /channels/:id` | Implemented |

**Note:** Channel creation uses `/servers/:id/channels` (Intent) vs `/guilds/:id/channels` (Discord)

## Message Operations

| Intent | Discord | Status |
|--------|---------|--------|
| `GET /channels/:id/messages` | `GET /channels/:id/messages` | Implemented |
| `GET /channels/:id/messages/:message_id` | `GET /channels/:id/messages/:message_id` | Implemented |
| `POST /channels/:id/messages` | `POST /channels/:id/messages` | Implemented |
| `PATCH /channels/:id/messages/:message_id` | `PATCH /channels/:id/messages/:message_id` | Implemented |
| `DELETE /channels/:id/messages/:message_id` | `DELETE /channels/:id/messages/:message_id` | Implemented |
| `POST /channels/:id/messages/bulk-delete` | `POST /channels/:id/messages/bulk-delete` | Planned |

## User Operations

| Intent | Discord | Status |
|--------|---------|--------|
| `GET /users/@me` | `GET /users/@me` | Planned |
| `PATCH /users/@me` | `PATCH /users/@me` | Planned |
| `GET /users/:id` | `GET /users/:id` | Planned |

## Authentication

| Intent | Discord | Notes |
|--------|---------|-------|
| `POST /auth/register` | N/A | Intent-specific |
| `POST /auth/login` | N/A | Intent-specific |
| `POST /auth/refresh` | N/A | Intent-specific |
| N/A | `POST /oauth2/token` | Discord OAuth |

**Differences:**
- Intent uses direct auth endpoints with JWT tokens
- Discord uses OAuth2 flow
- Bot tokens work similarly (prefix: `usr_*` / `bot_*` vs `Bot TOKEN`)

## Webhooks

| Intent | Discord | Status |
|--------|---------|--------|
| `POST /channels/:id/webhooks` | `POST /channels/:id/webhooks` | Planned |
| `GET /channels/:id/webhooks` | `GET /channels/:id/webhooks` | Planned |
| `GET /servers/:id/webhooks` | `GET /guilds/:id/webhooks` | Planned |
| `GET /webhooks/:id` | `GET /webhooks/:id` | Planned |
| `GET /webhooks/:id/:token` | `GET /webhooks/:id/:token` | Planned |
| `PATCH /webhooks/:id` | `PATCH /webhooks/:id` | Planned |
| `PATCH /webhooks/:id/:token` | `PATCH /webhooks/:id/:token` | Planned |
| `DELETE /webhooks/:id` | `DELETE /webhooks/:id` | Planned |
| `DELETE /webhooks/:id/:token` | `DELETE /webhooks/:id/:token` | Planned |
| `POST /webhooks/:id/:token` | `POST /webhooks/:id/:token` | Planned |
| `GET /webhooks/:id/:token/messages/:msg_id` | `GET /webhooks/:id/:token/messages/:msg_id` | Planned |
| `PATCH /webhooks/:id/:token/messages/:msg_id` | `PATCH /webhooks/:id/:token/messages/:msg_id` | Planned |
| `DELETE /webhooks/:id/:token/messages/:msg_id` | `DELETE /webhooks/:id/:token/messages/:msg_id` | Planned |

## Roles

| Intent | Discord | Status |
|--------|---------|--------|
| `GET /servers/:id/roles` | `GET /guilds/:id/roles` | Planned |
| `POST /servers/:id/roles` | `POST /guilds/:id/roles` | Planned |
| `PATCH /servers/:id/roles/:role_id` | `PATCH /guilds/:id/roles/:role_id` | Planned |
| `DELETE /servers/:id/roles/:role_id` | `DELETE /guilds/:id/roles/:role_id` | Planned |

## Invites

| Intent | Discord | Status |
|--------|---------|--------|
| `POST /channels/:id/invites` | `POST /channels/:id/invites` | Planned |
| `GET /invites/:code` | `GET /invites/:code` | Planned |
| `DELETE /invites/:code` | `DELETE /invites/:code` | Planned |

## Not Supported

These Discord features are not planned for Intent:

| Discord Endpoint | Reason |
|------------------|--------|
| `/guilds/:id/audit-logs` | Different moderation approach |
| `/guilds/:id/integrations` | No third-party integrations |
| `/stickers/*` | Not planned |
| `/stage-instances/*` | No stage channels |
| `/guilds/:id/scheduled-events` | Not planned |
| `/applications/*` | Different bot system |

## Migration Notes

For bot developers migrating from Discord:

1. Replace `guild` with `server` in all paths
2. Update auth from `Bot TOKEN` to `bot_*` JWT format
3. Base URL changes to `api.intent.chat/v1`
4. Rate limit headers are identical format
5. Snowflake IDs work the same way
