# Core Resources

This document specifies the REST API endpoints for the three foundational Intent resources: Servers, Channels, and Messages.

All endpoints use the base URL `https://api.intent.chat/v1` and require authentication via the `Authorization: Bearer <token>` header unless otherwise noted.

## Servers

A server is a collection of channels, members, and roles. Servers are the top-level organizational unit in Intent.

### Get Server

```
GET /servers/{server_id}
```

Returns a server object for the given ID.

**Response: 200 OK**

```json
{
  "id": "1234567890",
  "name": "My Server",
  "owner_id": "9876543210",
  "icon": "a3d2f1c9b8e7d6c5",
  "description": "A cool server",
  "member_count": 42,
  "created_at": "2024-01-15T08:30:00Z"
}
```

**Errors:**
- `401 Unauthorized` - Invalid or missing token
- `403 Forbidden` - User is not a member of this server
- `404 Not Found` - Server does not exist

### Create Server

```
POST /servers
```

Creates a new server. The authenticated user becomes the owner.

**Request Body:**

```json
{
  "name": "My New Server",
  "icon": "base64-encoded-image-data",
  "description": "Optional server description"
}
```

**Fields:**
- `name` (string, required) - Server name, 2-100 characters
- `icon` (string, optional) - Base64-encoded image data
- `description` (string, optional) - Server description, max 500 characters

**Response: 201 Created**

Returns the created server object with `id` and `created_at` populated.

**Errors:**
- `400 Bad Request` - Invalid request body (missing required fields, invalid data)
- `401 Unauthorized` - Invalid or missing token

### Update Server

```
PATCH /servers/{server_id}
```

Updates server properties. Only the server owner or users with `MANAGE_SERVER` permission can update.

**Request Body:**

```json
{
  "name": "Updated Server Name",
  "description": "Updated description"
}
```

All fields are optional. Only provided fields will be updated.

**Response: 200 OK**

Returns the updated server object.

**Errors:**
- `400 Bad Request` - Invalid field values
- `401 Unauthorized` - Invalid or missing token
- `403 Forbidden` - User lacks `MANAGE_SERVER` permission
- `404 Not Found` - Server does not exist

### Delete Server

```
DELETE /servers/{server_id}
```

Deletes a server. Only the server owner can delete.

**Response: 204 No Content**

**Errors:**
- `401 Unauthorized` - Invalid or missing token
- `403 Forbidden` - User is not the server owner
- `404 Not Found` - Server does not exist

## Channels

Channels are containers for messages within a server.

### Get Channel

```
GET /channels/{channel_id}
```

Returns a channel object.

**Response: 200 OK**

```json
{
  "id": "5555555555",
  "server_id": "1234567890",
  "name": "general",
  "type": 0,
  "position": 0,
  "topic": "General discussion",
  "created_at": "2024-01-15T08:35:00Z"
}
```

**Channel Types:**
- `0` - Text channel
- `1` - Voice channel
- `2` - Announcement channel

**Errors:**
- `401 Unauthorized` - Invalid or missing token
- `403 Forbidden` - User cannot view this channel
- `404 Not Found` - Channel does not exist

### Create Channel

```
POST /servers/{server_id}/channels
```

Creates a channel within a server. Requires `MANAGE_CHANNELS` permission.

**Request Body:**

```json
{
  "name": "new-channel",
  "type": 0,
  "topic": "Channel topic",
  "position": 5
}
```

**Fields:**
- `name` (string, required) - Channel name, 2-100 characters
- `type` (integer, required) - Channel type (0 = text, 1 = voice, 2 = announcement)
- `topic` (string, optional) - Channel topic, max 1024 characters
- `position` (integer, optional) - Sorting position, defaults to bottom

**Response: 201 Created**

Returns the created channel object.

**Errors:**
- `400 Bad Request` - Invalid request body
- `401 Unauthorized` - Invalid or missing token
- `403 Forbidden` - User lacks `MANAGE_CHANNELS` permission
- `404 Not Found` - Server does not exist

### Update Channel

```
PATCH /channels/{channel_id}
```

Updates channel properties. Requires `MANAGE_CHANNELS` permission.

**Request Body:**

All fields are optional.

```json
{
  "name": "updated-name",
  "topic": "Updated topic",
  "position": 3
}
```

**Response: 200 OK**

Returns the updated channel object.

**Errors:**
- `400 Bad Request` - Invalid field values
- `401 Unauthorized` - Invalid or missing token
- `403 Forbidden` - User lacks `MANAGE_CHANNELS` permission
- `404 Not Found` - Channel does not exist

### Delete Channel

```
DELETE /channels/{channel_id}
```

Deletes a channel. Requires `MANAGE_CHANNELS` permission.

**Response: 204 No Content**

**Errors:**
- `401 Unauthorized` - Invalid or missing token
- `403 Forbidden` - User lacks `MANAGE_CHANNELS` permission
- `404 Not Found` - Channel does not exist

## Messages

Messages are the content posted in channels.

### Get Message

```
GET /channels/{channel_id}/messages/{message_id}
```

Returns a single message.

**Response: 200 OK**

```json
{
  "id": "7777777777",
  "channel_id": "5555555555",
  "author": {
    "id": "9876543210",
    "username": "alice",
    "avatar": "abc123def456"
  },
  "content": "Hello world!",
  "timestamp": "2024-01-15T12:00:00Z",
  "edited_timestamp": null,
  "attachments": [],
  "embeds": []
}
```

**Errors:**
- `401 Unauthorized` - Invalid or missing token
- `403 Forbidden` - User cannot view this channel
- `404 Not Found` - Message or channel does not exist

### List Messages

```
GET /channels/{channel_id}/messages
```

Returns messages in a channel. Messages are returned in reverse chronological order (newest first).

**Query Parameters:**
- `limit` (integer, optional) - Max number of messages to return (1-100, default 50)
- `before` (snowflake, optional) - Get messages before this message ID
- `after` (snowflake, optional) - Get messages after this message ID

**Response: 200 OK**

Returns an array of message objects.

**Errors:**
- `400 Bad Request` - Invalid query parameters
- `401 Unauthorized` - Invalid or missing token
- `403 Forbidden` - User cannot view this channel
- `404 Not Found` - Channel does not exist

### Create Message

```
POST /channels/{channel_id}/messages
```

Posts a message to a channel. Requires `SEND_MESSAGES` permission.

**Request Body:**

```json
{
  "content": "Message text goes here",
  "embeds": []
}
```

**Fields:**
- `content` (string, required) - Message content, 1-2000 characters
- `embeds` (array, optional) - Array of embed objects (max 10)

At least one of `content` or `embeds` MUST be present.

**Response: 201 Created**

Returns the created message object.

**Errors:**
- `400 Bad Request` - Invalid request body (empty content, content too long)
- `401 Unauthorized` - Invalid or missing token
- `403 Forbidden` - User lacks `SEND_MESSAGES` permission
- `404 Not Found` - Channel does not exist

### Edit Message

```
PATCH /channels/{channel_id}/messages/{message_id}
```

Edits a message. Users can only edit their own messages.

**Request Body:**

```json
{
  "content": "Updated message text"
}
```

**Response: 200 OK**

Returns the updated message object with `edited_timestamp` set.

**Errors:**
- `400 Bad Request` - Invalid request body
- `401 Unauthorized` - Invalid or missing token
- `403 Forbidden` - User is not the message author
- `404 Not Found` - Message or channel does not exist

### Delete Message

```
DELETE /channels/{channel_id}/messages/{message_id}
```

Deletes a message. Users can delete their own messages, or any message if they have `MANAGE_MESSAGES` permission.

**Response: 204 No Content**

**Errors:**
- `401 Unauthorized` - Invalid or missing token
- `403 Forbidden` - User cannot delete this message
- `404 Not Found` - Message or channel does not exist

## Common Patterns

### Snowflake IDs

All resource IDs are snowflakes - 64-bit integers represented as strings in JSON. Snowflakes encode timestamp information and are sortable.

### Timestamps

All timestamps use ISO 8601 format with UTC timezone: `2024-01-15T12:00:00Z`

### Pagination

List endpoints support cursor-based pagination using `before` and `after` parameters with snowflake IDs. Responses do not include pagination metadata; clients determine if more results exist by checking if the result count equals the limit.

### Error Responses

All error responses follow this format:

```json
{
  "code": 50001,
  "message": "Missing Access"
}
```

Common error codes:
- `10001` - Unknown Server
- `10003` - Unknown Channel
- `10008` - Unknown Message
- `50001` - Missing Access
- `50013` - Missing Permissions
