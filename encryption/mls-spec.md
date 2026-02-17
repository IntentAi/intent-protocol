# MLS E2EE Specification

End-to-end encryption for DMs and group DMs using the Message Layer Security protocol (RFC 9420).

## Why MLS

- O(log n) key operations vs O(n) for Signal Protocol in groups
- Scales to group chats without per-member re-encryption
- Forward secrecy and post-compromise security built in
- IETF standard with formal security proofs
- Supports async member addition via key packages

Intent uses the `openmls` crate server-side with `openmls_rust_crypto` backend.

## Ciphersuite

**MVP ciphersuite:** `MLS_128_DHKEMX25519_AES128GCM_SHA256_Ed25519` (0x0001)

| Component | Algorithm |
|-----------|-----------|
| KEM | X25519 DHKEM |
| AEAD | AES-128-GCM |
| Hash | SHA-256 |
| Signature | Ed25519 |

All key packages, groups, and messages use this single ciphersuite. The server rejects key packages with any other ciphersuite ID. This keeps the MVP simple — ciphersuite negotiation and post-quantum hybrids are roadmap items (see [Future Enhancements](#future-enhancements)).

## Encrypted Channel Types

Two new channel types extend the existing set (0 = text, 1 = voice, 2 = category):

| Type | Name | Participants | Description |
|------|------|-------------|-------------|
| 3 | DM | Exactly 2 | 1:1 encrypted direct message |
| 4 | Group DM | 3-10 | Encrypted group direct message |

Both types have `server_id: null` — they exist outside servers. All messages in these channels are MLS-encrypted. There is no unencrypted fallback.

DM channels (type 3) are deduplicated per user pair. Calling create with the same recipient returns the existing channel. Group DMs (type 4) are not deduplicated.

**Channel object (DM):**

```typescript
{
  id: string,
  type: 3,
  server_id: null,
  recipients: User[],         // other participants (excludes self)
  mls_initialized: boolean,   // false until MLS group is set up
  created_at: string
}
```

**Channel object (Group DM):**

```typescript
{
  id: string,
  type: 4,
  server_id: null,
  name: string | null,        // optional group name
  icon_url: string | null,    // optional group icon
  owner_id: string,           // user who created the group DM
  recipients: User[],         // other participants (excludes self)
  mls_initialized: boolean,
  created_at: string
}
```

## Terminology

| Term | Meaning |
|------|---------|
| Key package | A pre-key bundle that allows async addition to an MLS group. One-time use. |
| Welcome | MLS message sent to a new member so they can join a group. Contains the group's current key state. |
| Commit | MLS message that advances the group's key state. Used for key rotation, member add/remove. |
| Proposal | MLS message requesting a change. Must be followed by a Commit to take effect. |
| Leaf | A node in the MLS ratchet tree representing one client (device). |
| Client | A single device. One user may have multiple clients, each with its own leaf. |
| Ratchet tree | Binary tree tracking all group members' key material. |

## Key Package Management

Every client (device) uploads key packages to the server. Key packages are consumed once when someone adds that device to a group. Clients should maintain a pool of available key packages and upload more when the count gets low.

### Upload Key Packages

```
POST /users/@me/mls/key-packages
```

Uploads one or more key packages for the authenticated client.

**Request Body:**

```json
{
  "client_id": "device_abc123",
  "key_packages": [
    "<base64 serialized KeyPackage>",
    "<base64 serialized KeyPackage>"
  ]
}
```

**Fields:**
- `client_id` (string, required) - Unique device identifier for this client, 1-64 characters
- `key_packages` (array of strings, required) - Base64-encoded serialized MLS key packages, 1-100 per request

**Response: 204 No Content**

**Errors:**
- `400 Bad Request` - Invalid key package format, too many packages, or ciphersuite mismatch
- `401 Unauthorized` - Invalid or missing token

**Notes:**
- The server validates each key package: correct ciphersuite, valid signature, not expired
- Key packages with unsupported ciphersuites are rejected
- Clients should upload 50-100 key packages at registration and replenish when low
- One key package per device should be marked as a **last resort** key package (MLS `last_resort` extension). Last resort packages are never deleted when claimed — they're reused indefinitely as a fallback when all other packages are exhausted. This prevents a scenario where a device becomes completely unreachable because its pool ran dry.

### Claim Key Packages

```
GET /users/{user_id}/mls/key-packages
```

Claims one key package per active client (device) for the target user. Claimed packages are deleted from the server and cannot be reused.

**Response: 200 OK**

```json
{
  "key_packages": [
    {
      "client_id": "device_abc123",
      "key_package": "<base64 serialized KeyPackage>"
    },
    {
      "client_id": "device_def456",
      "key_package": "<base64 serialized KeyPackage>"
    }
  ]
}
```

Each entry represents one of the target user's devices. The caller receives exactly one key package per device.

**Errors:**
- `401 Unauthorized` - Invalid or missing token
- `404 Not Found` - User does not exist

**Notes:**
- Claiming your own key packages is allowed (needed for multi-device group setup)
- If a device has zero remaining packages, it is omitted from the response and the server dispatches `MLS_KEY_PACKAGES_LOW` to that device
- If no devices have remaining packages, the response contains an empty array: `"key_packages": []`

### Key Package Count

```
GET /users/@me/mls/key-packages/count
```

Returns the number of unused key packages for each of the authenticated user's clients.

**Response: 200 OK**

```json
{
  "counts": {
    "device_abc123": 47,
    "device_def456": 12
  }
}
```

**Errors:**
- `401 Unauthorized` - Invalid or missing token

## DM Channel Creation

### Create DM Channel

```
POST /users/@me/channels
```

Opens a DM or group DM channel. For 1:1 DMs, returns the existing channel if one already exists with that recipient.

**Request Body (1:1 DM):**

```json
{
  "recipient_id": "9876543210"
}
```

**Request Body (Group DM):**

```json
{
  "recipient_ids": ["9876543210", "1111111111", "2222222222"]
}
```

**Fields:**
- `recipient_id` (snowflake, required for DM) - Target user ID
- `recipient_ids` (array of snowflakes, required for group DM) - Target user IDs, 2-9 entries

Provide exactly one of `recipient_id` or `recipient_ids`. Using `recipient_id` creates a type 3 channel. Using `recipient_ids` creates type 4.

**Response: 200 OK** (existing DM) or **201 Created** (new channel)

Returns the channel object. The channel is created with `mls_initialized: false`. The creating client must initialize MLS before messages can be sent.

**Errors:**
- `400 Bad Request` - Invalid recipient IDs, too many recipients, or self-referencing
- `401 Unauthorized` - Invalid or missing token
- `404 Not Found` - One or more recipients do not exist

### Add Member to Group DM

```
PUT /channels/{channel_id}/recipients/{user_id}
```

Adds a user to an existing group DM (type 4 only). Only the group DM owner can add members. Maximum 10 participants.

**Response: 204 No Content**

**Side effects:**
- The calling client must follow up with a Commit via `POST /channels/{channel_id}/mls/commit` that includes the new member's key packages and a Welcome message
- Server will not deliver `MLS_WELCOME` to the new member until the Commit is received

**Errors:**
- `400 Bad Request` - Channel is type 3 (1:1 DMs cannot add members) or max participants reached
- `401 Unauthorized` - Invalid or missing token
- `403 Forbidden` - Not the group DM owner
- `404 Not Found` - Channel or user does not exist

### Remove Member from Group DM

```
DELETE /channels/{channel_id}/recipients/{user_id}
```

Removes a user from a group DM. The owner can remove anyone; non-owners can only remove themselves (leave).

**Response: 204 No Content**

**Side effects:**
- The calling client must follow up with a Commit that removes the member's leaves from the MLS group, triggering key rotation so the removed member cannot decrypt future messages
- If the owner leaves, ownership transfers to the longest-tenured remaining member

**Errors:**
- `400 Bad Request` - Cannot remove the last member (delete the channel instead)
- `401 Unauthorized` - Invalid or missing token
- `403 Forbidden` - Non-owner trying to remove someone else
- `404 Not Found` - Channel or user does not exist

## MLS Group Initialization

After creating a DM channel, the initiating client sets up the MLS group and distributes key material.

### Initialize MLS Group

```
POST /channels/{channel_id}/mls/init
```

Initializes the MLS group for an encrypted channel. Can only be called once per channel, and only by a member of that channel.

**Request Body:**

```json
{
  "commit": "<base64 serialized MLS Commit>",
  "welcome": "<base64 serialized MLS Welcome>",
  "group_info": "<base64 serialized MLS GroupInfo>",
  "ratchet_tree": "<base64 serialized ratchet tree>"
}
```

**Fields:**
- `commit` (string, required) - Serialized MLS Commit message that created the group
- `welcome` (string, required) - Serialized MLS Welcome message for all other members
- `group_info` (string, required) - Serialized MLS GroupInfo for the group's current state
- `ratchet_tree` (string, optional) - Serialized ratchet tree, included if not using ratchet tree extension

**Response: 201 Created**

```json
{
  "channel_id": "5555555555",
  "mls_initialized": true,
  "group_epoch": 1
}
```

**Errors:**
- `400 Bad Request` - Invalid MLS message format or ciphersuite mismatch
- `401 Unauthorized` - Invalid or missing token
- `403 Forbidden` - Not a member of this channel
- `409 Conflict` - MLS group already initialized

**Side effects:**
- Server sets `mls_initialized: true` on the channel
- Server dispatches `MLS_WELCOME` gateway event to all other channel members' devices, delivering the same Welcome blob to each. The MLS Welcome is a single `MLSMessage` of type `welcome` containing a `secrets` array with per-recipient entries encrypted to each device's init key (RFC 9420 Section 12.4.3). Each device decrypts only its own entry.
- Server stores `group_info` and `ratchet_tree` for late-joining devices

### Full DM Creation Flow

Alice wants to DM Bob. Both have devices registered with key packages.

```
Alice                           Server                          Bob
  |                               |                               |
  |  POST /users/@me/channels     |                               |
  |  { "recipient_id": "bob" }    |                               |
  |------------------------------>|                               |
  |  201: channel (type 3)        |                               |
  |<------------------------------|                               |
  |                               |                               |
  |  GET /users/bob/mls/key-pkgs  |                               |
  |------------------------------>|                               |
  |  200: [bob_device1, ...]      |                               |
  |<------------------------------|                               |
  |                               |                               |
  | (create MLS group locally,    |                               |
  |  add Bob's key packages)      |                               |
  |                               |                               |
  |  POST /channels/{id}/mls/init  |                               |
  |  { commit, welcome,           |                               |
  |    group_info }               |                               |
  |------------------------------>|                               |
  |  201: initialized             |  MLS_WELCOME event            |
  |<------------------------------|-------- dispatch ------------>|
  |                               |                               |
  |                               |              (process Welcome, |
  |                               |               join MLS group)  |
  |                               |                               |
  | POST /channels/{id}/messages   |                               |
  | { "mls_ciphertext": "..." }   |                               |
  |------------------------------>|  MESSAGE_CREATE               |
  |                               |  { mls_ciphertext: "..." }    |
  |                               |-------- dispatch ------------>|
  |                               |                               |
  |                               |               (decrypt locally)|
```

For group DMs, the flow is identical except the initiator fetches key packages for all recipients and the Welcome message includes all of them.

## Encrypted Message Format

Encrypted messages flow through the standard message endpoint. The `mls_ciphertext` field replaces `content` for E2EE channels.

### Sending

```
POST /channels/{channel_id}/messages
```

**Request Body (encrypted channel):**

```json
{
  "mls_ciphertext": "<base64 serialized MLS ApplicationMessage>"
}
```

**Fields:**
- `mls_ciphertext` (string, required) - Base64-encoded MLS ApplicationMessage ciphertext

The `content` field is not accepted for encrypted channels. The server stores the ciphertext as an opaque blob.

**Response: 201 Created**

```json
{
  "id": "7777777777",
  "channel_id": "5555555555",
  "author": {
    "id": "9876543210",
    "username": "alice",
    "display_name": "Alice",
    "avatar_url": null,
    "created_at": "2024-01-01T00:00:00Z"
  },
  "content": null,
  "mls_ciphertext": "<base64 ciphertext>",
  "created_at": "2024-01-15T12:00:00Z",
  "edited_at": null
}
```

**Errors:**
- `400 Bad Request` - Missing `mls_ciphertext`, or trying to send `content` in an encrypted channel
- `401 Unauthorized` - Invalid or missing token
- `403 Forbidden` - Not a member of the channel, or MLS group not yet initialized
- `404 Not Found` - Channel does not exist

### Receiving

Encrypted messages arrive via `MESSAGE_CREATE` gateway dispatch:

```json
{
  "op": 0,
  "t": "MESSAGE_CREATE",
  "s": 42,
  "d": {
    "message": {
      "id": "7777777777",
      "channel_id": "5555555555",
      "author": {
        "id": "9876543210",
        "username": "alice",
        "display_name": "Alice",
        "avatar_url": null,
        "created_at": "2024-01-01T00:00:00Z"
      },
      "content": null,
      "mls_ciphertext": "<base64 ciphertext>",
      "created_at": "2024-01-15T12:00:00Z",
      "edited_at": null
    }
  }
}
```

The client decrypts `mls_ciphertext` using its local MLS group state. The `content` field is always null for encrypted messages. Metadata (author, timestamp, channel) remains in plaintext for server-side indexing and delivery.

### Editing

Editing an encrypted message replaces the ciphertext:

```
PATCH /channels/{channel_id}/messages/{message_id}
```

```json
{
  "mls_ciphertext": "<base64 new ciphertext>"
}
```

Only the original author can edit. The server replaces the stored ciphertext and sets `edited_at`.

The server dispatches `MESSAGE_UPDATE` with the full updated message object:

```json
{
  "op": 0,
  "t": "MESSAGE_UPDATE",
  "s": 43,
  "d": {
    "message": {
      "id": "7777777777",
      "channel_id": "5555555555",
      "author": {
        "id": "9876543210",
        "username": "alice",
        "display_name": "Alice",
        "avatar_url": null,
        "created_at": "2024-01-01T00:00:00Z"
      },
      "content": null,
      "mls_ciphertext": "<base64 new ciphertext>",
      "created_at": "2024-01-15T12:00:00Z",
      "edited_at": "2024-01-15T12:05:00Z"
    }
  }
}
```

Clients decrypt the new `mls_ciphertext` to display the updated content.

### Deleting

Deletion works identically to unencrypted messages:

```
DELETE /channels/{channel_id}/messages/{message_id}
```

The server deletes the ciphertext blob. `MESSAGE_DELETE` fires with just the message and channel IDs.

### Message Padding

MLS ciphertext leaks approximate plaintext length. Without padding, an observer can distinguish "yes" from a paragraph. Clients must pad the plaintext before encryption.

**Policy:** Pad all messages to the nearest 256-byte boundary (minimum 256 bytes). OpenMLS supports this via `padding_size` in the group configuration:

```rust
MlsGroup::builder()
    .padding_size(256)
    // ...
```

This adds at most 255 bytes of overhead per message — negligible for a messaging app but enough to obscure whether someone sent a short reply or a longer message.

### Encrypted Attachments

Files, images, and other attachments in encrypted channels use a two-layer approach:

1. **Client encrypts the file** with a random 256-bit AES-GCM key (the "file key")
2. **Client uploads the encrypted blob** to the CDN via the standard attachment upload endpoint
3. **Client sends an MLS-encrypted message** containing the file key, CDN URL, filename, size, and MIME type

The MLS ApplicationMessage plaintext for an attachment:

```json
{
  "type": "attachment",
  "content": "optional caption text",
  "attachments": [
    {
      "filename": "photo.jpg",
      "content_type": "image/jpeg",
      "size": 524288,
      "url": "https://cdn.intent.chat/attachments/enc_abc123",
      "encryption": {
        "key": "<base64 AES-256-GCM key>",
        "iv": "<base64 initialization vector>",
        "digest": "<base64 SHA-256 hash of plaintext file>"
      }
    }
  ]
}
```

The server and CDN only see encrypted blobs. The file key is only available inside the MLS ciphertext, so only group members can decrypt attachments.

**Upload flow:**

```
1. Client generates random AES-256-GCM key and IV
2. Client encrypts the file: AES-256-GCM(key, iv, plaintext_file)
3. Client uploads encrypted blob: POST /channels/{channel_id}/attachments
   Server returns CDN URL
4. Client builds the attachment JSON above
5. Client encrypts the JSON via MLS: group.create_message(json)
6. Client sends: POST /channels/{channel_id}/messages { "mls_ciphertext": "..." }
```

Recipients decrypt the MLS message, extract the file key, download the encrypted blob from the CDN URL, and decrypt it with the file key. The `digest` field lets clients verify file integrity after decryption.

### Message History for New Devices

MLS does not provide access to messages encrypted before a device joined the group. This is by design — it's a consequence of forward secrecy.

When a new device is added to an existing encrypted channel:
- It can only decrypt messages sent **after** it joined the group
- Historical messages remain inaccessible to the new device
- Clients should display a clear marker: "Messages before this point are not available on this device"

An existing device on the same account may optionally re-encrypt and transfer message history to the new device over a separate E2EE session between the two devices. This is a client-side feature, not a protocol requirement, and is a roadmap item.

## Key Rotation and Group State Management

MLS groups evolve through Commits and Proposals. Commits advance the group epoch (key state). Proposals request changes that take effect when included in a Commit.

### Submit Commit

```
POST /channels/{channel_id}/mls/commit
```

Submits an MLS Commit message that advances the group's key state.

**Request Body:**

```json
{
  "commit": "<base64 serialized MLS Commit>",
  "welcome": "<base64 serialized MLS Welcome or null>",
  "group_info": "<base64 serialized updated GroupInfo>"
}
```

**Fields:**
- `commit` (string, required) - Serialized MLS Commit
- `welcome` (string, optional) - Serialized Welcome message if the Commit adds new members
- `group_info` (string, required) - Updated GroupInfo after the Commit

**Response: 200 OK**

```json
{
  "group_epoch": 5
}
```

**Side effects:**
- Server dispatches `MLS_COMMIT` to all other group members
- If `welcome` is present, server dispatches `MLS_WELCOME` to newly added members
- Server updates stored `group_info`

**Errors:**
- `400 Bad Request` - Invalid Commit format
- `401 Unauthorized` - Invalid or missing token
- `403 Forbidden` - Not a member of this channel
- `409 Conflict` - Epoch mismatch (another Commit was processed first; re-fetch group info and retry)

### Submit Proposal

```
POST /channels/{channel_id}/mls/proposal
```

Submits an MLS Proposal. Proposals are advisory until a Commit includes them.

**Request Body:**

```json
{
  "proposal": "<base64 serialized MLS Proposal>"
}
```

**Response: 200 OK**

```json
{
  "proposal_ref": "<base64 proposal reference hash>"
}
```

**Side effects:**
- Server dispatches `MLS_PROPOSAL` to all other group members
- Server stores the proposal until a Commit consumes it or it expires

**Errors:**
- `400 Bad Request` - Invalid Proposal format
- `401 Unauthorized` - Invalid or missing token
- `403 Forbidden` - Not a member of this channel

### Fetch Group Info

```
GET /channels/{channel_id}/mls/group-info
```

Returns the current MLS group state. Used by new devices joining an existing group or clients recovering from state loss.

**Response: 200 OK**

```json
{
  "group_info": "<base64 serialized GroupInfo>",
  "ratchet_tree": "<base64 serialized ratchet tree or null>",
  "group_epoch": 5,
  "pending_proposals": [
    "<base64 serialized Proposal>",
    "<base64 serialized Proposal>"
  ]
}
```

**Errors:**
- `401 Unauthorized` - Invalid or missing token
- `403 Forbidden` - Not a member of this channel
- `404 Not Found` - MLS group not initialized

### When to Commit

Clients should issue Commits for:

- **Key rotation** - Periodically (recommended: every 24 hours or every 100 messages, whichever comes first) to maintain forward secrecy
- **Member removal** - When a user leaves or is removed from the DM, or when a device is deauthorized
- **Member addition** - When adding someone to a group DM, or when a user registers a new device
- **Self-update** - When rotating a client's own leaf key material

## Gateway Events

MLS events are dispatched via opcode 0 (Dispatch) like all other events.

### MLS_WELCOME

Dispatched when you've been added to an encrypted channel (or a new device is added to an existing group).

**Payload:**

```typescript
{
  channel_id: string,
  welcome: string,              // base64 serialized MLS Welcome
  group_info: string,           // base64 serialized GroupInfo
  ratchet_tree: string | null   // base64 serialized ratchet tree
}
```

**Example:**

```json
{
  "op": 0,
  "t": "MLS_WELCOME",
  "s": 50,
  "d": {
    "channel_id": "5555555555",
    "welcome": "base64...",
    "group_info": "base64...",
    "ratchet_tree": null
  }
}
```

**When it fires:**
- You're added to a new DM or group DM
- A new device on your account is added to an existing group

**Client action:** Process the Welcome message to join the MLS group and derive keys.

### MLS_COMMIT

Dispatched when the group's key state advances.

**Payload:**

```typescript
{
  channel_id: string,
  commit: string,            // base64 serialized MLS Commit
  group_epoch: number,       // new epoch after this Commit
  committer_id: string       // user ID of who issued the Commit
}
```

**Example:**

```json
{
  "op": 0,
  "t": "MLS_COMMIT",
  "s": 51,
  "d": {
    "channel_id": "5555555555",
    "commit": "base64...",
    "group_epoch": 5,
    "committer_id": "9876543210"
  }
}
```

**When it fires:**
- Key rotation
- Member added or removed
- A client updated its leaf key

**Client action:** Process the Commit to advance local group state. If processing fails (e.g., missed a prior Commit), fetch group info via `GET /channels/{id}/mls/group-info` and re-sync.

### MLS_PROPOSAL

Dispatched when someone proposes a change to the group.

**Payload:**

```typescript
{
  channel_id: string,
  proposal: string,          // base64 serialized MLS Proposal
  proposal_ref: string,      // base64 proposal reference hash
  proposer_id: string        // user ID of who sent the Proposal
}
```

**Example:**

```json
{
  "op": 0,
  "t": "MLS_PROPOSAL",
  "s": 52,
  "d": {
    "channel_id": "5555555555",
    "proposal": "base64...",
    "proposal_ref": "base64...",
    "proposer_id": "1111111111"
  }
}
```

**When it fires:**
- A member proposes adding or removing someone
- A member proposes updating their own key material

**Client action:** Store the Proposal locally. It takes effect when a Commit references it.

### MLS_KEY_PACKAGES_LOW

Dispatched when the server detects that a device's key package pool is running low.

**Payload:**

```typescript
{
  client_id: string,       // device that needs more key packages
  remaining: number        // how many key packages are left
}
```

**Example:**

```json
{
  "op": 0,
  "t": "MLS_KEY_PACKAGES_LOW",
  "s": 53,
  "d": {
    "client_id": "device_abc123",
    "remaining": 3
  }
}
```

**When it fires:**
- After a key package is claimed and the remaining count drops below 10

**Client action:** Upload a fresh batch via `POST /users/@me/mls/key-packages`.

### DEVICE_ADDED

Dispatched to a user's existing devices when a new device registers key packages on their account.

**Payload:**

```typescript
{
  user_id: string,         // the user who added a device
  client_id: string        // new device's client ID
}
```

**Example:**

```json
{
  "op": 0,
  "t": "DEVICE_ADDED",
  "s": 54,
  "d": {
    "user_id": "9876543210",
    "client_id": "device_ghi789"
  }
}
```

**When it fires:**
- A new device on your account uploads its first batch of key packages

**Client action:** For each active encrypted channel, fetch the new device's key package and issue a Commit to add it to the MLS group.

### DEVICE_REMOVED

Dispatched to a user's remaining devices when a device is deauthorized or its session is revoked.

**Payload:**

```typescript
{
  user_id: string,         // the user who lost a device
  client_id: string        // removed device's client ID
}
```

**Example:**

```json
{
  "op": 0,
  "t": "DEVICE_REMOVED",
  "s": 55,
  "d": {
    "user_id": "9876543210",
    "client_id": "device_abc123"
  }
}
```

**When it fires:**
- A device session is revoked or the user explicitly removes a device

**Client action:** For each active encrypted channel, issue a Commit removing the old device's leaf to trigger key rotation.

## Multi-Device Support

MLS treats each device as an independent leaf in the ratchet tree. A user with a phone and a desktop has two leaves in every group they're in.

### Device Registration

When a user sets up a new device:

1. New device generates a `client_id` (UUID recommended) and stores it locally
2. New device generates a credential with the user's identity
3. New device creates and uploads key packages via `POST /users/@me/mls/key-packages`
4. New device is now available for inclusion in MLS groups

### Adding a New Device to Existing Groups

When a user logs into a new device, their existing devices detect this and add the new device to all active groups:

```
1. New device uploads key packages
2. Server dispatches DEVICE_ADDED event to user's existing devices
3. For each active encrypted channel:
   a. Existing device fetches new device's key package
   b. Existing device issues a Commit adding the new leaf
   c. Server delivers Welcome to the new device via MLS_WELCOME
4. New device processes each Welcome and joins each group
```

Until this process completes for a given channel, the new device cannot decrypt messages in that channel. Clients should indicate this state in the UI.

### Removing a Device

When a device is deauthorized (user logs out, revokes a session, loses a device):

1. One of the user's remaining devices issues a Commit removing the old device's leaf from each group
2. This triggers key rotation so the removed device cannot decrypt future messages
3. If no remaining devices can issue the Commit (all devices lost), the user must re-initialize each group from another member's Commit

### Client ID in Key Packages

Key packages carry the client identity in the MLS credential. The credential format for Intent:

```
<user_id>:<client_id>
```

Example: `9876543210:device_abc123`

This lets other group members associate leaves with specific devices and users.

## Server Responsibilities

### What the Server Does

- **Stores key packages** in a per-device pool. Claimed packages are deleted immediately.
- **Creates and manages DM channels** with deduplication for 1:1 DMs.
- **Routes MLS messages** (Welcome, Commit, Proposal, ciphertext) between clients without inspecting content.
- **Validates channel membership** before accepting any MLS operation or message.
- **Stores message metadata** (author, channel, timestamp) in plaintext for indexing and delivery.
- **Stores message content** as opaque ciphertext blobs.
- **Tracks group epoch** to detect and reject stale Commits (409 Conflict).
- **Stores GroupInfo** and ratchet tree for late-joining devices.
- **Monitors key package counts** and dispatches low-count warnings.

### What the Server Cannot Do

- Read or modify message content without detection
- Forge messages on behalf of users (MLS uses per-sender signing keys)
- Derive group keys or session keys
- Determine anything about message content from the ciphertext

### Metadata Limitations

MLS encrypts message **content** but not **metadata**. The server can observe:

- Who is messaging whom (channel membership is plaintext)
- Message timestamps and frequency
- Approximate message size (mitigated by padding, not eliminated)
- Which devices are active in a group
- When key rotations happen

This is an inherent limitation of a client-server architecture where the server routes messages. Sealed sender (hiding the sender from the server) and private membership (hiding who's in a group) are research-level problems tracked in the roadmap.

### Threat Model

The server operates under an **honest-but-curious** adversary model. It routes messages correctly but may attempt to read content. MLS guarantees:

- **Confidentiality** - Only group members can decrypt messages
- **Integrity** - Ciphertext tampering is detected via AEAD
- **Authentication** - Every message is signed by the sender's Ed25519 key
- **Forward secrecy** - Compromising current keys doesn't reveal past messages
- **Post-compromise security** - After a key rotation, a previously compromised member regains security

### Storage Requirements

The server must persist:

| Data | Retention | Purpose |
|------|-----------|---------|
| Key packages | Until claimed or expired | Async group creation |
| GroupInfo + ratchet tree | Current epoch only | Late-joining devices |
| Pending proposals | Until consumed by Commit or 7-day expiry | Proposal aggregation |
| Message ciphertext | Same as message retention policy | Message history |
| Message metadata | Same as message retention policy | Indexing, delivery |

## Error Handling

### Epoch Mismatch

If two clients issue Commits simultaneously, one will succeed and the other gets `409 Conflict`. The losing client should:

1. Fetch current group info via `GET /channels/{id}/mls/group-info`
2. Process any pending Commits received via `MLS_COMMIT`
3. Re-create the Commit against the new epoch
4. Retry submission

### Decryption Failure

If a client cannot decrypt a message (missed a Commit, corrupted state):

1. Fetch group info and pending proposals
2. Attempt to re-sync by processing any missed Commits
3. If re-sync fails, request a re-add: remove self from group (Proposal) and have another member add you back with a fresh key package
4. Display undecryptable messages as "[encrypted message]" in the UI

### Missing Key Packages

If `GET /users/{user_id}/mls/key-packages` returns a response where some devices are omitted (no remaining packages):

- Those devices are excluded from the MLS group at creation time
- They will be added later via a Commit when they upload new key packages and a `DEVICE_ADDED`-style flow runs
- The group can still be created with the user's remaining devices

## Endpoint Summary

| Method | Endpoint | Description | Status |
|--------|----------|-------------|--------|
| `POST` | `/users/@me/mls/key-packages` | Upload key packages for a device | Planned |
| `GET` | `/users/{user_id}/mls/key-packages` | Claim key packages (one per device) | Planned |
| `GET` | `/users/@me/mls/key-packages/count` | Check remaining key package counts | Planned |
| `POST` | `/users/@me/channels` | Create DM or group DM channel | Planned |
| `PUT` | `/channels/{channel_id}/recipients/{user_id}` | Add member to group DM | Planned |
| `DELETE` | `/channels/{channel_id}/recipients/{user_id}` | Remove member / leave group DM | Planned |
| `POST` | `/channels/{channel_id}/mls/init` | Initialize MLS group on a channel | Planned |
| `POST` | `/channels/{channel_id}/mls/commit` | Submit MLS Commit | Planned |
| `POST` | `/channels/{channel_id}/mls/proposal` | Submit MLS Proposal | Planned |
| `GET` | `/channels/{channel_id}/mls/group-info` | Fetch current MLS group state | Planned |

Endpoints use the `/mls/` path namespace to separate encryption operations from core resource endpoints and avoid collisions with future user or channel sub-resources.

## Event Summary

| Event | Direction | Description | Status |
|-------|-----------|-------------|--------|
| `MLS_WELCOME` | Server -> Client | You've been added to an encrypted channel | Planned |
| `MLS_COMMIT` | Server -> Client | Group key state advanced | Planned |
| `MLS_PROPOSAL` | Server -> Client | Pending change proposed | Planned |
| `MLS_KEY_PACKAGES_LOW` | Server -> Client | Upload more key packages | Planned |
| `DEVICE_ADDED` | Server -> Client | New device registered on your account | Planned |
| `DEVICE_REMOVED` | Server -> Client | Device deauthorized on your account | Planned |

## Implementation Notes

- Server-side MLS validation uses the `openmls` crate with `openmls_rust_crypto` backend
- All serialized MLS messages use TLS serialization format (as defined in RFC 9420), base64-encoded for JSON transport
- Clients must persist their MLS group state, signature keys, and credential across sessions
- If a client loses its local MLS state, it must be re-added to groups via the removal + re-add flow
- Key packages should use a 90-day expiration; clients should rotate before expiry
- The server should reject key packages with unsupported extensions to maintain interoperability

## Future Enhancements

Features tracked for post-MVP phases. None of these block the initial E2EE launch.

| Feature | Description | Phase |
|---------|-------------|-------|
| Post-quantum ciphersuites | Hybrid ML-KEM-768 + X25519 ciphersuite (draft-ietf-mls-pq-ciphersuites) to defend against harvest-now-decrypt-later. OpenMLS already supports X-Wing. Requires ciphersuite negotiation in key packages. | Phase 3 |
| Key verification | Out-of-band identity verification (safety numbers / QR codes) so users can confirm they're talking to the right person, not a MITM. | Phase 3 |
| Message franking | Abuse reporting without breaking E2EE. Sender attaches a frankable tag that a recipient can reveal to the server to prove a message was sent, without exposing the content to the server preemptively. Based on MLS extensions draft. | Phase 3 |
| Sealed sender | Hide the sender's identity from the server so it can't observe who messages whom. Requires an anonymous submission protocol layered on top of MLS. Research-level. | Phase 5 |
| Encrypted server channels | Extend MLS E2EE beyond DMs to opt-in server text channels. Significantly more complex due to larger groups and permission models. | Phase 3 |
| Message history transfer | Allow an existing device to re-encrypt and transfer message history to a new device on the same account over a separate E2EE session. | Phase 4 |
| Ciphersuite negotiation | Support multiple ciphersuites with automatic negotiation based on key package capabilities. Required before adding PQ suites. | Phase 3 |
