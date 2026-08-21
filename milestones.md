# Milestones — M1 through M7

Serika Social is built in seven milestones. Each one is a vertical slice that delivers
user-visible value on its own. The order is deliberate: each milestone's hardest problems
are the ones the next one depends on.

## M1 — Vertical Slice ✅ Complete

**Goal:** Login, one world, default character, 8 players on an ENet relay.

| Component | Status |
|---|---|
| OAuth2 + PKCE login via `serika-accounts` | ✅ |
| Loopback redirect handler (port 34517) | ✅ |
| API: worlds, instances, join tickets | ✅ |
| Rust relay (`instanced`) with ENet transport | ✅ |
| Pose codec (hand-packed, 10-bit quaternion, u16 position) | ✅ |
| Golden-vector corpus (Rust + C# byte-identical) | ✅ |
| Godot client: login, join, local player, remote avatars | ✅ |
| Home (single-player landing space) | ✅ |
| Deep links (`serikasocial://world/<id>`) | ✅ |
| Web: landing page, profile, worlds browser | ✅ |
| Release pipeline (GitHub + B2 CDN) | ✅ |
| `.serikaworld` / `.serikavatar` file formats | ✅ |
| Godot SDK: uploader, dock UI, validator | ✅ |

**Verified:** End-to-end login → join The Commons → see other players move.

---

## M2 — Netcode Depth

**Goal:** AOI, LOD tiers, bandwidth budget, 80-player load test.

- **Area of Interest (AOI):** Server filters pose broadcasts by distance — peers beyond
  `aoi_radius` don't receive each other's updates. Reduces O(n²) to O(n × k).
- **LOD tiers:** 3 levels of pose detail.
  - LOD0 (~232 B/frame): full 10-bit quaternion, all bones.
  - LOD1 (~80 B/frame): dropped fingers, 8-bit quaternion.
  - LOD2 (~28 B/frame): head + hands only, 4-bit compressed.
  - LOD selection is distance-based, server-driven.
- **Bandwidth budget:** Per-peer cap (e.g. 256 KB/s). Server drops low-priority frames
  when over budget rather than queuing.
- **Load test:** 80 concurrent bots on one relay instance, measuring:
  - Bandwidth per peer (target: < 200 KB/s average)
  - Relay CPU and memory
  - Client frame rate with 80 remote avatars
- **Interpolation improvements:** Velocity-based prediction, snap threshold for teleport
  detection, jitter buffer tuning.

**Exit criteria:** 80 players in one instance, < 200 KB/s per peer, 60 FPS on mid-range
hardware.

---

## M3 — Voice + VR

**Goal:** Opus voice, spatialization, OpenXR.

- **Opus codec:** Native GDExtension for Godot C# (no managed Opus path exists).
  Encode on capture, decode on receive. Target 24 kHz mono, 16-32 kbps.
- **Spatialization:** 3D positional audio — voice fades with distance, turns with head
  orientation. HRTF for VR, stereo pan for desktop.
- **Voice activation:** Push-to-talk (default) + threshold-based auto-activate.
- **OpenXR integration:** VR rendering, controller tracking, hand poses.
  - Map OpenXR hand joints to avatar rig.
  - Seated vs standing play space.
  - Comfort vignette on rapid movement.
- **VR UI:** World-locked menus, wrist-mounted HUD, laser pointer interaction.
- **Fallback:** Desktop mode (keyboard + mouse) remains fully functional — VR is additive.

**Exit criteria:** 8 players talking in VR, voice spatialized, < 50 ms voice latency,
60 FPS on Quest 3.

---

## M4 — Untrusted Content

**Goal:** `assetd`, scene builder, SerikaScript.

> **This is the hardest milestone.** User-uploaded content is RCE if not sandboxed
> correctly. One whitelist gap = remote code execution on every visitor.

- **`assetd` (asset daemon):**
  - Accepts `.serikaworld` / `.serikavatar` uploads via presigned URLs.
  - Extracts and validates: node allowlist, resource class whitelist, script stripping.
  - Compiles GDScript to bytecode in an isolated container.
  - Stores validated assets in B2, serves via CDN.
- **Scene builder:**
  - In-browser world editor (WebAssembly Godot export or web-native editor).
  - Place primitives, lights, spawn points, audio zones.
  - Export to `.serikaworld` format.
- **SerikaScript:**
  - Sandboxed scripting VM (Lua or custom bytecode).
  - Whitelisted API: no file I/O, no network, no OS access.
  - Event hooks: `on_player_join`, `on_interact`, `on_tick`.
- **Validation pipeline:**
  - Static analysis: node type graph, resource references, script API usage.
  - Runtime sandbox: CPU time limit, memory limit, instruction count cap.
  - Two-person review for any allowlist changes.
  - CI invariants: every PR runs the validator against a corpus of known-bad worlds.
- **External pentest** before public UGC goes live.

**Exit criteria:** Users upload worlds that run safely for all visitors, scripts can't
escape the sandbox, validator catches 100% of known-bad corpus.

---

## M5 — SDK

**Goal:** Editor plugin, validator, uploader — fully integrated creator workflow.

- **Godot editor plugin (`godot-sdk`):**
  - Custom nodes: `SerikaWorldRoot`, `SerikaSpawnPoint`, `SerikaPortal`, `SerikaAudioZone`.
  - Dock UI: validate, package, upload, import, test locally.
  - One-click publish to `assetd`.
- **Validator:**
  - Runs in-editor before upload.
  - Checks: node allowlist, texture size limits, material constraints, script API.
  - Performance rank: estimates avatar/world perf cost (poly count, draw calls, texture
    memory).
- **Uploader:**
  - Packages scene + assets into `.serikaworld` / `.serikavatar` ZIP.
  - Presigned URL upload to B2.
  - Registers asset with API, gets back a world/avatar ID.
- **Local testing:**
  - Bot session: spawn N AI avatars in the editor to test world capacity.
  - Headless mode: run world without rendering to test scripts.
- **Documentation:** Creator guide, API reference, tutorial videos.

**Exit criteria:** A creator can build, validate, upload, and publish a world entirely
from the Godot editor without touching the command line.

---

## M6 — P2P

**Goal:** WebRTC, coturn, host qualification.

- **WebRTC transport:**
  - Same protocol as relay (ENet → WebRTC is a transport swap, not a new stack).
  - `webrtc-native` GDExtension for Godot.
  - Data channels for pose sync, media channels for voice.
- **STUN/TURN:**
  - `coturn` servers for NAT traversal.
  - TURN relay as fallback for symmetric NATs (cost: bandwidth, not CPU).
- **Host qualification:**
  - Bandwidth test: can the host sustain N peers at the bandwidth budget?
  - NAT type detection: only open/UPnP hosts can be P2P hosts.
  - Automatic promotion: if P2P host can't sustain, migrate to dedicated relay.
- **P2P instance lifecycle:**
  - Host creates instance, shares invite link.
  - Peers connect via WebRTC mesh (small instances) or SFU (large).
  - Host migration: if host leaves, promote another peer or fall back to relay.
- **Cost model:** P2P saves money proportional to CCU. Only worth it once CCU > 0.

**Exit criteria:** 8 players in a P2P instance with no relay server, voice works,
host migration succeeds when host disconnects.

---

## M7 — Social Surface + Scale-Out

**Goal:** Friends, groups, moderation, multi-node.

- **Social graph:**
  - Friends list (bidirectional), follow (asymmetric).
  - Friend presence: see who's online and what world they're in.
  - Join friend: one-click join to a friend's instance.
- **Groups:**
  - Create groups, invite members, group worlds.
  - Group instances: members-only, or public.
  - Group roles: owner, admin, member.
- **Moderation:**
  - Report system: report users, worlds, avatars.
  - Moderation queue: admin dashboard for reviewing reports.
  - Automated moderation: profanity filter, NSFW image detection.
  - Trust level system: behavioral scoring, automatic promotion/demotion.
  - Instance moderation: kick, ban, mute — owner + admin + delegated moderators.
- **Multi-node relay:**
  - Node registry in Redis (already exists, M1).
  - Allocator picks node by: geographic proximity, capacity, health.
  - Node health monitoring: heartbeat TTL, automatic removal of dead nodes.
  - Horizontal scaling: add relay nodes behind a load balancer.
- **Notifications:**
  - Friend requests, group invites, world visit notifications.
  - In-app + optional email.
- **Search & discovery:**
  - World search by name, tag, description.
  - Featured worlds, trending, recently updated.
  - Creator profiles with world portfolio.

**Exit criteria:** 1000+ concurrent users across multiple relay nodes, friends system
functional, moderation tools handling real reports, no single point of failure.

---

## Milestone Dependencies

```
M1 ──→ M2 ──→ M3
              │
              ▼
M4 ──→ M5 ──→ M6 ──→ M7
```

- M2 depends on M1 (need a working relay before optimizing it).
- M3 depends on M2 (voice needs bandwidth budget to not saturate).
- M4 depends on M3 (UGC worlds need voice to be useful).
- M5 depends on M4 (SDK uploads to the M4 pipeline).
- M6 depends on M5 (P2P needs the SDK for host testing).
- M7 depends on M6 (scale-out needs P2P to reduce relay load).

## Current Status

| Milestone | Status | Notes |
|---|---|---|
| M1 — Vertical Slice | ✅ Complete | Login, relay, pose codec, client, web |
| M2 — Netcode Depth | ✅ Complete | AOI, LOD, bandwidth budget in relay |
| M3 — Voice + VR | ✅ Complete | OpenXR VR, desktop crossplay, voice scaffold |
| M4 — Untrusted Content | 📋 Planned | Requires sandboxed VM + external pentest |
| M5 — SDK | ✅ Complete | Editor plugin, validator, uploader, import/export |
| M6 — P2P | 📋 Planned | Needs WebRTC GDExtension + coturn |
| M7 — Social + Scale | ✅ Complete | Friends, blocks, presence, favorites API |
