# MessagePack Protocol

Intent uses MessagePack binary protocol for WebSocket communication.

## Why MessagePack?

- Smaller payload size than JSON
- Faster serialization/deserialization
- Binary-safe (for future binary data embedding)

## Message Format

```
{
 "op": <opcode>,
 "d": <data>,
 "s": <sequence>, // optional
 "t": <event_type> // for opcode 0
}
```

 Full protocol specification in development
