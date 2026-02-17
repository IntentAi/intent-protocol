# MessagePack Binary Protocol

Intent's gateway uses [MessagePack](https://msgpack.org/) binary encoding for all WebSocket frames. Every message sent or received on the gateway is a MessagePack-encoded map, never JSON text.

## Why MessagePack

- ~30% smaller payloads than equivalent JSON
- Faster encode/decode — no string parsing, direct binary reads
- Native binary type for future attachment/media embedding
- Deterministic encoding — same input always produces same bytes

## WebSocket Frame Requirements

| Property | Value |
|----------|-------|
| Frame type | Binary (`0x02`), not Text (`0x01`) |
| Compression | None (MessagePack is already compact) |
| Max frame size | 64 KiB per message |
| Fragmentation | Not supported — each message is a single frame |

Clients that send Text frames will receive close code `4002` (decode error). Payloads exceeding 64 KiB are rejected with close code `4009` (payload too large).

## Message Structure

Every gateway message is a MessagePack map with string keys:

```
{
  "op": <uint>,          // opcode — always present
  "d":  <any | null>,    // data payload — always present (null if empty)
  "t":  <str | null>,    // event name — present on Dispatch (op 0) only
  "s":  <uint | null>    // sequence number — present on Dispatch (op 0) only
}
```

All four keys are always included in server-to-client messages. Client-to-server messages only require `op` and `d`.

## Type Mapping

MessagePack types map to gateway fields as follows:

| Field | MessagePack Type | Notes |
|-------|-----------------|-------|
| `op` | positive fixint (0-12) | Single byte for all current opcodes |
| `d` | map, null, bool, or int | Depends on opcode (see opcodes.md) |
| `t` | fixstr or nil | Event names are always < 32 chars, fit in fixstr |
| `s` | uint 32 | Sequence numbers reset per session |
| Snowflake IDs | fixstr | Always encoded as strings, not integers |
| Timestamps | fixstr | ISO 8601 strings (e.g. `"2024-01-15T12:00:00Z"`) |
| Booleans | bool (true/false) | `0xc3` / `0xc2` |
| Null values | nil | `0xc0` |
| Nested objects | map | String keys, values vary |
| Arrays | array | Used for server lists in Ready, ice_servers, etc. |

### Snowflake IDs Are Strings

Snowflake IDs are 64-bit integers but are always encoded as MessagePack strings, not integers. This avoids precision loss in languages where integers are limited to 53 bits (JavaScript `Number`). Clients must treat IDs as opaque strings for comparison and storage.

## Encoding Examples

### JavaScript / TypeScript

```javascript
import { encode, decode } from '@msgpack/msgpack';

// connect with binary type
const ws = new WebSocket('wss://gateway.intent.chat');
ws.binaryType = 'arraybuffer';

// send Identify
function send(payload) {
  ws.send(encode(payload));
}

send({
  op: 2,
  d: { token: 'usr_MTIzNDU2Nzg5MC4xNzA1MzIwMDAwLmFiY2Q...' }
});

// receive and decode
ws.onmessage = (event) => {
  const msg = decode(new Uint8Array(event.data));
  switch (msg.op) {
    case 0:  // Dispatch
      handleEvent(msg.t, msg.d, msg.s);
      break;
    case 3:  // Ready
      startHeartbeat(msg.d.heartbeat_interval);
      break;
    case 11: // Heartbeat ACK
      ackHeartbeat();
      break;
  }
};
```

Recommended library: `@msgpack/msgpack` (pure JS, no native deps, ~4 KB gzipped).

### Python

```python
import asyncio
import websockets
import msgpack

async def connect():
    async with websockets.connect(
        'wss://gateway.intent.chat',
    ) as ws:
        # send Identify
        await ws.send(msgpack.packb({
            'op': 2,
            'd': {'token': 'usr_MTIzNDU2...'},
        }))

        async for raw in ws:
            msg = msgpack.unpackb(raw, raw=False)
            if msg['op'] == 0:
                handle_event(msg['t'], msg['d'], msg['s'])
            elif msg['op'] == 3:
                start_heartbeat(msg['d']['heartbeat_interval'])
            elif msg['op'] == 11:
                ack_heartbeat()
```

Recommended library: `msgpack` (C extension, fast). Use `raw=False` to decode byte strings as Python `str`.

### Rust

```rust
use serde::{Deserialize, Serialize};

#[derive(Serialize, Deserialize)]
struct GatewayMessage {
    op: u8,
    d: serde_json::Value,  // flexible payload
    #[serde(skip_serializing_if = "Option::is_none")]
    t: Option<String>,
    #[serde(skip_serializing_if = "Option::is_none")]
    s: Option<u32>,
}

// encode
let msg = GatewayMessage { op: 1, d: serde_json::Value::Null, t: None, s: None };
let bytes = rmp_serde::to_vec_named(&msg).unwrap();

// decode
let msg: GatewayMessage = rmp_serde::from_slice(&bytes).unwrap();
```

Recommended library: `rmp-serde`. Use `to_vec_named` (not `to_vec`) to encode map keys as strings rather than integer indices.

## Size Comparison

Heartbeat (op 1) — smallest possible message:

| Format | Bytes | Encoding |
|--------|-------|----------|
| JSON | 19 | `{"op":1,"d":null}` |
| MessagePack | 7 | `82 a2 6f 70 01 a1 64 c0` |
| Savings | 63% | |

Ready payload with 3 servers (typical):

| Format | Approx. Bytes |
|--------|---------------|
| JSON | ~820 |
| MessagePack | ~560 |
| Savings | ~32% |

Savings increase with payload size due to more efficient encoding of repeated structure.

## Wire Format Reference

For implementers building from scratch, here's how the key MessagePack types appear on the wire:

```
fixmap (up to 15 entries):  1000xxxx (0x80-0x8f)
fixstr (up to 31 bytes):    101xxxxx (0xa0-0xbf)
positive fixint (0-127):    0xxxxxxx (0x00-0x7f)
nil:                        11000000 (0xc0)
false:                      11000010 (0xc2)
true:                       11000011 (0xc3)
uint 32:                    11001110 (0xce) + 4 bytes
str 8 (up to 255 bytes):    11011001 (0xd9) + 1 byte len + data
map 16:                     11011110 (0xde) + 2 byte len + entries
array 16:                   11011100 (0xdc) + 2 byte len + entries
```

A Heartbeat message encodes as:

```
82        fixmap, 2 entries
  a2 6f 70    fixstr "op"
  01          fixint 1
  a1 64       fixstr "d"
  c0          nil
```

Total: 7 bytes.

## Error Handling

| Close Code | Meaning | When |
|------------|---------|------|
| `4002` | Decode error | Malformed MessagePack, or Text frame sent instead of Binary |
| `4003` | Not authenticated | First message was not Identify (op 2) |
| `4009` | Payload too large | Message exceeds 64 KiB |

On decode error, the server closes the connection immediately. Clients should not retry without fixing the encoding issue.

## Implementation Notes

- Always set `ws.binaryType = 'arraybuffer'` in browser environments. The default `'blob'` type requires an extra async conversion step.
- MessagePack maps preserve insertion order but clients should not rely on key ordering — always access fields by name.
- Unknown keys in server messages should be ignored, not rejected. The server may add new fields in future versions.
- Clients must handle both fixmap and map16 for the top-level message — large Dispatch payloads (e.g. Ready with many servers) will exceed 15 map entries in `d`.
