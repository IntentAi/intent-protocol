# MLS E2EE Specification

End-to-end encryption using Message Layer Security (MLS) protocol.

## Why MLS?

- O(log n) key operations vs O(n) for Signal Protocol
- Scales better for group chats
- Formal security proofs
- IETF standard (RFC 9420)

## Implementation

Using `openmls` crate for Rust implementation.

 Full MLS integration specification in development
