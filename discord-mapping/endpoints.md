# Discord API Mapping - Endpoints

Intent API → Discord API endpoint equivalences.

## Server (Guild) Operations

| Intent | Discord | Notes |
|--------|---------|-------|
| `GET /servers/:id` | `GET /guilds/:id` | Same concept |
| `POST /servers` | `POST /guilds` | Same |
| `PATCH /servers/:id` | `PATCH /guilds/:id` | Same |
| `DELETE /servers/:id` | `DELETE /guilds/:id` | Same |

## Channel Operations

| Intent | Discord | Notes |
|--------|---------|-------|
| `GET /channels/:id` | `GET /channels/:id` | Identical |
| `POST /channels/:id/messages` | `POST /channels/:id/messages` | Identical |

 Full mapping documentation in development
