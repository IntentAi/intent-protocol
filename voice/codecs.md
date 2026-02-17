# Voice Codecs

Audio codec requirements and RTP configuration for Intent voice channels.

## Opus (Required)

Opus is the only supported audio codec for voice in Intent. All clients must support Opus encoding and decoding.

**Configuration:**

| Parameter | Value | Notes |
|-----------|-------|-------|
| Sample rate | 48 kHz | Fixed, Opus always negotiates 48kHz in SDP |
| Channels | 2 (stereo) | Stereo negotiated in SDP, content can be mono or stereo |
| Frame size | 20 ms | Balance between latency and compression efficiency |
| Bitrate | 32 - 128 kbps | Adaptive, see profiles below |
| FEC | Enabled | Forward error correction for packet loss resilience |
| DTX | Enabled | Discontinuous transmission, saves bandwidth during silence |
| Payload type | 111 | Standard dynamic payload type for Opus |

**Stereo handling:**

The SFU forwards Opus packets as-is without decoding or re-encoding. If a client sends stereo audio (music, screen share with audio), all receivers get stereo. The SFU does not downmix to mono — it has no reason to touch the audio content.

For voice-only use, set `sprop-stereo=0` in the SDP offer so the encoder produces mono frames. Keep `stereo=1` so receivers stay prepared if another participant sends stereo. Mono within a stereo-negotiated channel saves bandwidth.

## SDP Codec Negotiation

For the MVP, Intent voice channels only accept Opus. The SFU rejects any SDP offer that does not include Opus.

**Expected m= line in SDP offer:**

```
m=audio 9 UDP/TLS/RTP/SAVPF 111
a=rtpmap:111 opus/48000/2
a=fmtp:111 minptime=10;useinbandfec=1;usedtx=1
```

If a client offers additional codecs alongside Opus, the SFU answers with only Opus selected. If Opus is absent from the offer, the SFU rejects the session.

## RTP Header Extensions

Three RTP header extensions are required:

| Extension | URI | Purpose |
|-----------|-----|---------|
| ssrc-audio-level | `urn:ietf:params:rtp-hdrext:ssrc-audio-level` | Per-packet audio level (RFC 6464). Used by the SFU for speaking detection. |
| transport-wide-cc | `http://www.ietf.org/id/draft-holmer-rmcat-transport-wide-cc-extensions-01` | Transport-wide sequence numbers for congestion control. |
| sdes:mid | `urn:ietf:params:rtp-hdrext:sdes:mid` | Media identification for bundle. Required when multiple media sections share a transport. |

Clients must include all three extensions in their SDP offer. The SFU will not negotiate a session without `ssrc-audio-level` and `transport-wide-cc`.

**SDP extension mapping example:**

```
a=extmap:1 urn:ietf:params:rtp-hdrext:ssrc-audio-level
a=extmap:2 http://www.ietf.org/id/draft-holmer-rmcat-transport-wide-cc-extensions-01
a=extmap:3 urn:ietf:params:rtp-hdrext:sdes:mid
```

## RTCP Feedback

The SFU uses these RTCP feedback mechanisms:

| Mechanism | SDP attribute | Purpose |
|-----------|--------------|---------|
| NACK | `a=rtcp-fb:111 nack` | Request retransmission of lost packets |
| Transport-CC | `a=rtcp-fb:111 transport-cc` | Bandwidth estimation via transport-wide feedback |
| REMB | `a=rtcp-fb:111 goog-remb` | Receiver-side bitrate estimation, SFU uses this to signal senders to adjust |

**NACK** allows the SFU to request retransmission of lost packets from the sender. The SFU maintains a short jitter buffer (up to 200ms) and can retransmit packets to receivers that report loss.

**Transport-CC** provides per-packet arrival time feedback used by the TWCC algorithm to estimate available bandwidth. This drives the SFU's congestion control decisions.

**REMB** is sent by the SFU to individual senders when a receiver is congested. The sender should reduce its Opus encoder bitrate accordingly within the 32-128 kbps range.

## Opus Profiles

Recommended encoder configurations. Clients default to voice mode.

| Profile | Bitrate | Opus application | Use case |
|---------|---------|-----------------|----------|
| Voice | 64 kbps | `voip` | Default for voice chat. Optimized for speech, lowest latency. |
| Music | 128 kbps | `audio` | Music bots, screen share with audio, DJ channels. Preserves full frequency range. |
| Low bandwidth | 32 kbps | `voip` | Mobile on cellular, congested networks. Intelligible speech at minimum bitrate. |

**Profile switching:**

Clients can switch profiles at any time by reconfiguring the Opus encoder. No renegotiation is required — the SFU is codec-agnostic and forwards whatever bitrate the sender produces. The change takes effect on the next encoded frame.

When the SFU sends a REMB indicating congestion, clients should drop to the low bandwidth profile until conditions improve.

## Future Codecs

These codecs are not supported in the current spec but are planned for video and screen share:

| Codec | Use case | Priority |
|-------|----------|----------|
| VP8 | Video calls (baseline) | High |
| VP9 | Video calls (improved compression) | Medium |
| AV1 | Screen share, high-quality video | Low (hardware support still limited) |

Video will use the same SDP negotiation pattern as audio, with simulcast layers so the SFU can forward appropriate quality per receiver without transcoding. Full video codec spec comes when video support lands.
