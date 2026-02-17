# Discord API Mapping - Objects

Object structure mappings between Intent and Discord.

## User

| Intent Field | Discord Field | Notes |
|--------------|---------------|-------|
| `id` | `id` | Snowflake, identical |
| `username` | `username` | Identical |
| `display_name` | `global_name` | Renamed |
| `avatar_url` | `avatar` | Intent: full URL, Discord: hash |
| `created_at` | N/A | Intent adds this |
| N/A | `discriminator` | Discord legacy, not used |
| N/A | `bot` | Intent uses token prefix |
| N/A | `system` | Not applicable |
| N/A | `banner` | Not implemented |
| N/A | `accent_color` | Not implemented |

## Server (Guild)

| Intent Field | Discord Field | Notes |
|--------------|---------------|-------|
| `id` | `id` | Snowflake, identical |
| `name` | `name` | Identical |
| `icon_url` | `icon` | Intent: full URL, Discord: hash |
| `owner_id` | `owner_id` | Identical |
| `member_count` | `member_count` | Identical |
| `created_at` | N/A | Intent adds this |
| N/A | `splash` | Not implemented |
| N/A | `discovery_splash` | Not implemented |
| N/A | `banner` | Not implemented |
| N/A | `features` | Not implemented |
| N/A | `verification_level` | Not implemented |
| N/A | `nsfw_level` | Not implemented |
| N/A | `premium_tier` | No boosting |

## Channel

| Intent Field | Discord Field | Notes |
|--------------|---------------|-------|
| `id` | `id` | Snowflake, identical |
| `server_id` | `guild_id` | Renamed |
| `name` | `name` | Identical |
| `type` | `type` | Same values (0=text, 1=voice, 2=category) |
| `topic` | `topic` | Identical |
| `position` | `position` | Identical |
| `parent_id` | `parent_id` | Identical |
| `created_at` | N/A | Intent adds this |
| N/A | `nsfw` | Not implemented |
| N/A | `rate_limit_per_user` | Not implemented |
| N/A | `permission_overwrites` | Planned |

**Channel Types:**
- `0` - Text (both)
- `1` - Voice (both)
- `2` - Category (both)
- `4` - Discord: Server Announcement (not supported)
- `5` - Discord: Announcement Thread (not supported)
- `10-12` - Discord: Threads (not supported)
- `13` - Discord: Stage (not supported)
- `14` - Discord: Directory (not supported)
- `15` - Discord: Forum (not supported)

## Message

| Intent Field | Discord Field | Notes |
|--------------|---------------|-------|
| `id` | `id` | Snowflake, identical |
| `channel_id` | `channel_id` | Identical |
| `author` | `author` | Full User object |
| `content` | `content` | Identical |
| `created_at` | `timestamp` | Renamed |
| `edited_at` | `edited_timestamp` | Renamed |
| N/A | `attachments` | Planned |
| N/A | `embeds` | Planned |
| N/A | `reactions` | Planned |
| N/A | `pinned` | Planned |
| N/A | `mentions` | Not implemented |
| N/A | `mention_roles` | Not implemented |
| N/A | `mention_everyone` | Not implemented |
| N/A | `tts` | Not supported |
| N/A | `nonce` | Not implemented |
| N/A | `message_reference` | Not implemented (replies) |
| N/A | `thread` | Not supported |
| N/A | `components` | Not implemented |
| N/A | `stickers` | Not supported |

## Role

| Intent Field | Discord Field | Notes |
|--------------|---------------|-------|
| `id` | `id` | Snowflake, identical |
| `server_id` | `guild_id` | Renamed |
| `name` | `name` | Identical |
| `color` | `color` | Identical (integer) |
| `position` | `position` | Identical |
| `permissions` | `permissions` | Intent: string, Discord: integer |
| `hoist` | `hoist` | Identical |
| `mentionable` | `mentionable` | Identical |
| `created_at` | N/A | Intent adds this |
| N/A | `icon` | Not implemented |
| N/A | `unicode_emoji` | Not implemented |
| N/A | `managed` | Not applicable |

## Member

| Intent Field | Discord Field | Notes |
|--------------|---------------|-------|
| `user` | `user` | Full User object |
| `server_id` | `guild_id` | Renamed |
| `nickname` | `nick` | Renamed |
| `roles` | `roles` | Array of role IDs |
| `joined_at` | `joined_at` | Identical |
| N/A | `premium_since` | No boosting |
| N/A | `pending` | Not implemented |
| N/A | `permissions` | Computed differently |
| N/A | `communication_disabled_until` | Not implemented |
| N/A | `avatar` | Not implemented |

## Key Differences Summary

1. **URLs vs Hashes:** Intent returns full CDN URLs for avatars/icons. Discord returns hashes that need URL construction.

2. **Timestamps:** Intent uses `created_at`/`edited_at`. Discord uses `timestamp`/`edited_timestamp`.

3. **Guild vs Server:** All `guild_id` fields become `server_id`.

4. **Missing Features:** Many Discord fields don't exist in Intent Phase 1 (threads, stickers, components, etc.)

5. **Simpler Objects:** Intent objects are leaner with fewer optional fields.

## Migration Code

```javascript
// Convert Discord object to Intent format
function discordToIntent(discordObj) {
  return {
    ...discordObj,
    server_id: discordObj.guild_id,
    created_at: discordObj.timestamp,
    edited_at: discordObj.edited_timestamp
  }
}
```
