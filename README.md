# Serika Social

A social VR platform — Godot 4.7 client, user-uploaded avatars and worlds, spatial
voice. Small instances run peer-to-peer; large ones get promoted to dedicated Rust
relays.

Identity comes from [`serika-accounts`](https://github.com/serika-dev/serika-accounts)
over OAuth2 + PKCE. This project never stores credentials — only a mirror of the user
record keyed by `accounts_id`. See [`auth-integration.md`](auth-integration.md).

This repo holds the cross-cutting architecture and design docs. Each component repo
also has its own `README.md` and an `AGENTS.md` with operational detail.

## Repositories

| Repo | What | Stack |
|---|---|---|
| [`proto`](https://github.com/SerikaSocial/proto) | **Wire schema + golden corpus** — the C#/Rust contract | — |
| [`server`](https://github.com/SerikaSocial/server) | REST API, gateway, allocator, Rust relay | Bun + Rust |
| [`game`](https://github.com/SerikaSocial/game) | The client | Godot 4.7 C# |
| [`godot-sdk`](https://github.com/SerikaSocial/godot-sdk) | Creator tooling | Godot 4.7 C# |
| [`web`](https://github.com/SerikaSocial/web) | Profiles, browse, creator dashboard, moderation | Next.js 15 |
| [`infra`](https://github.com/SerikaSocial/infra) | Coolify config, migrations, CDN | — |
| [`tools`](https://github.com/SerikaSocial/tools) | Bot load harness, codec fuzzer, soak tests | — |
| [`docs`](https://github.com/SerikaSocial/docs) | This repo | — |

### Expected local layout

Some scripts reach across repos (`server`'s `up` calls `../infra/dev-up.sh`), so check
them out as siblings:

```
Godot-SerikaSocial/
  proto/   server/   game/   godot-sdk/   web/   infra/   tools/   docs/
```

`server` and `game` both pin `proto` as a submodule — clone them with
`--recurse-submodules`.

## Documentation index

### Getting started
- [`getting-started.md`](getting-started.md) — clone, build, and run all repos locally.
- [`repo-guide.md`](repo-guide.md) — the multi-repo layout, pinning, and contribution workflow.

### Architecture
- [`architecture.md`](architecture.md) — the relay/ownership model, components, and data flow.
- [`webrtc-p2p.md`](webrtc-p2p.md) — P2P as a deployment mode of the same netcode.
- [`milestones.md`](milestones.md) — the seven-milestone plan, exit criteria, dependency graph.

### Controls & diagnostics
- [`controls.md`](controls.md) — desktop and VR (VRChat-compatible) control reference.
- [`diagnostics.md`](diagnostics.md) — the headless diagnostic harness catalogue.

### Identity & trust
- [`auth-integration.md`](auth-integration.md) — OAuth2 + PKCE, session JWTs, the loopback listener.
- [`serikascript.md`](serikascript.md) — the sandboxed world scripting VM.
- [`PROMPT-creator-scripting-review-trust.md`](PROMPT-creator-scripting-review-trust.md) — the untrusted-content trust model.

### VR
- [`vr-hands-voice-plan.md`](vr-hands-voice-plan.md) — hand-tracking + voice plan.
- [`walkthrough-vr-hands-voice.md`](walkthrough-vr-hands-voice.md) — walkthrough.
- [`plan-nextgen-vr-5models.md`](plan-nextgen-vr-5models.md) — next-gen VR, 5-model test plan.
- [`walkthrough-nextgen-vr-5models.md`](walkthrough-nextgen-vr-5models.md) — walkthrough.

## The one rule

`proto` is the contract between three languages. Changing it is a breaking-change
review, and the golden-vector corpus must pass in **both** the Rust and C# test suites —
byte-identical output or the build fails. Codec drift between client and server is the
worst bug class in this design and the corpus is the only thing standing in its way.

Push `proto` first, then move the pins in `server` and `game` together. A client and a
relay on different `proto` commits is precisely the desync the corpus exists to prevent.

## Architecture

The server is a **relay**, not an authoritative simulation. Sync works by object
ownership: whoever owns an object broadcasts its state and everyone else trusts it. The
server is authoritative over membership, moderation, rate limits, and asset access —
the things where cheating actually matters in a social app.

The consequence that makes this design work: **a dedicated relay and a P2P host do the
same job.** Same protocol, same ownership rules. P2P is a deployment mode, not a second
netcode stack.

### Key constraints, learned the hard way

- **Untrusted content is the long pole.** A user-uploaded world containing GDScript is
  RCE. Creators upload source; it's compiled server-side in an isolated container with
  scripts stripped against a node allowlist. Behaviour comes from a sandboxed VM.
- **Godot 4.x C# has no web export.** Browser builds would need a second, non-C#
  client.
- **WebRTC on native Godot** needs the third-party `webrtc-native` GDExtension, plus
  STUN and TURN. P2P is therefore late in the plan, not early — it saves money
  proportional to CCU, and CCU is zero until there are users.
- **There is no supported Opus path in Godot C#**, which likely forces a small native
  GDExtension.

## Milestones

Serika Social is built in seven milestones. See [`milestones.md`](milestones.md) for
full details, exit criteria, and dependency graph.

**M1** vertical slice ✅ · **M2** netcode depth · **M3** voice + VR ·
**M4** untrusted content · **M5** SDK · **M6** P2P · **M7** social + scale
