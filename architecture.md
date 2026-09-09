# Architecture

Serika Social is a social VR platform in the VRChat mould: user-uploaded avatars and
worlds, spatial voice, small instances peer-to-peer and large ones on dedicated Rust
relays.

## The relay/ownership model

The server is a **relay**, not an authoritative simulation. State sync works by
**object ownership**: whoever owns an object broadcasts its state and everyone else
trusts it. The server is authoritative over the things where cheating actually matters
in a social app — membership, moderation, rate limits, asset access — and **not** over
physics.

The consequence that makes this design work: **a dedicated relay and a P2P host do
the same job.** Same protocol, same ownership rules. P2P is a deployment mode, not a
second netcode stack. See [`webrtc-p2p.md`](webrtc-p2p.md).

## Components

```
                ┌──────────────────────────────────────────────────┐
                │  serika-accounts  (OAuth2 + PKCE identity source) │
                └──────────────────────────────────────────────────┘
                                   │ accounts_id (no credentials stored)
                                   ▼
  game (Godot C#) ──HTTP──▶ api (Bun/Elysia :4100) ──▶ Postgres / Redis
        │                    │
        │   WebSocket        │ instance placement
        ▼                    ▼
  gateway (:4110) ──spawns──▶ allocator
        │  presence, invites, RTC signalling
        │
        │  join ticket
        ▼
  instanced (Rust :4200/udp) ◀──▶ proto (wire codec, shared submodule)
        │  relay: tick, AOI, LOD, chat, voice forward, bandwidth budget
        │
        ▼
  other game clients (poses, voice, object frames)
```

| Component | Role |
|---|---|
| `api` | REST: auth, avatars, worlds, friends, admin, events, video, asset presign |
| `gateway` | control WebSocket: presence, invites, WebRTC signalling rooms |
| `allocator` | instance placement, promotion (P2P → relay), migration |
| `instanced` | the shard process: UDP relay, fixed tick, AOI, LOD, chat, voice forward |
| `node-agent` | per-box supervisor, capacity reporting |
| `assetd` | sandboxed upload validation / world build (untrusted content) |
| `proto` | wire codec + golden corpus (Rust ↔ C# contract) |

## Identity

Identity comes from [`serika-accounts`](https://github.com/serika-dev/serika-accounts)
over OAuth2 + PKCE. This project never stores credentials — only a mirror of the user
record keyed by `accounts_id`. The client runs a loopback listener on fixed port 34517
to receive the OAuth redirect. See [`auth-integration.md`](auth-integration.md).

## The wire codec

`proto` is hand-packed (no code generator) because its value is sub-byte quantization
(10-bit quaternion components, 2-bit selectors) — a 55-bone pose is ~230 bytes, not
~900. The golden corpus (`proto/golden/vectors.json`) is the executable form of
`proto/pose_codec.md` and must pass byte-identically in both `cargo test -p
serika-proto` and `dotnet test Net/Codec/Tests`. See
[`proto/README.md`](https://github.com/SerikaSocial/proto).

## Untrusted content

A user-uploaded world containing GDScript is RCE. Creators submit source; it's
compiled server-side in an isolated container with scripts stripped against a node
allowlist. Behaviour comes from **SerikaScript** — a sandboxed VM with no reflection,
no file/network IO, a whitelisted API, and per-instance instruction budgets. The
`godot-sdk` validator and `server/assetd` load the **same** `world_rules.json` so
"passes locally" means "passes on upload". See [`serikascript.md`](serikascript.md)
and [`PROMPT-creator-scripting-review-trust.md`](PROMPT-creator-scripting-review-trust.md).

## Key constraints, learned the hard way

- **`PUBLIC_ENDPOINT` is always a domain, never a raw IP.** Cloudflare free doesn't
  carry UDP, so production uses a DNS-only A record (no CF proxy). Local dev uses
  `localhost`.
- **Relay port mapping must be UDP.** A `4200:4200` TCP-only mapping silently strands
  the relay — the "stuck on Connecting" incident (2026-08-22). The mapping must be
  `4200:4200/udp`.
- **Godot 4.x C# has no web export.** Browser builds would need a second, non-C#
  client.
- **WebRTC on native Godot** needs the third-party `webrtc-native` GDExtension, plus
  STUN and TURN. P2P is therefore late in the plan, not early.
- **No supported Opus path in Godot C#** — likely forces a small native GDExtension.

## Milestones

Seven milestones; see [`milestones.md`](milestones.md) for exit criteria and the
dependency graph.

**M1** vertical slice ✅ · **M2** netcode depth · **M3** voice + VR ·
**M4** untrusted content · **M5** SDK · **M6** P2P · **M7** social + scale
