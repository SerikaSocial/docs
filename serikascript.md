# SerikaScript — design & threat model

> **Status: DESIGN / gate.** Per the implementation prompt (§5, §9 A1) the VM must not open to
> the public until this document is signed off, CI invariants are in place, the sandbox boundary
> has two-person review, and an external pentest has run. The server-side validator and bytecode
> container (`server/api/src/serikascript.ts`) and the C# VM skeleton (`game/Script/`) are built
> against this spec; nothing executes untrusted creator code in production yet.

## 1. Why

Creators want behavior in their worlds and avatars: minigames (scores, timers, win states),
theme-park rides (trigger → animated platform → seated players), interactive props (buttons,
doors, spawners, scoreboards). Today all world content is inert geometry. SerikaScript is the
capability-based, deny-by-default layer that adds behavior **without** handing untrusted authors
a path to run arbitrary code on every visitor's machine.

Per `CLAUDE.md`: *"One `ResourceLoader.Load()` on a user path, one un-audited glTF extension, one
whitelist gap = RCE on everyone who visits that world. No partial credit."* The sandbox is the
whole product risk. If a capability cannot be proven safe, it does not ship.

## 2. Trust boundary

```
 SDK (trusted authoring)          Server (trusted)            Client (runs untrusted bytecode)
 ─────────────────────────        ─────────────────────       ────────────────────────────────
 source → compile → bytecode  →   validate bytecode      →    VM executes bytecode in sandbox
                                   (submit + publish)          host API allowlist is the ONLY
                                   route to review/queue       bridge to the engine
```

- **The runtime never parses untrusted source.** The SDK compiles; the client only ever sees
  bytecode. There is no `eval`, no runtime compiler, no dynamic code generation.
- **The server validates the bytecode twice** — on submit (routing) and again at publish (defense
  in depth) — using `validateScriptBytecode`. A single disallowed opcode or host call fails it.
- **The C# VM re-checks at load** and refuses to run anything the validator would reject. The two
  validators (TS `serikascript.ts`, C# `game/Script/`) MUST agree on the opcode and host-call
  allowlists. That agreement is the sandbox boundary; it is a two-person-review invariant.

## 3. Bytecode container (`SSKB`)

Little-endian. Single source of truth: `server/api/src/serikascript.ts` (TS) and
`game/Script/OpCode.cs` / `HostApi.cs` (C#).

```
magic      4   "SSKB" (0x53 0x53 0x4B 0x42)
version    u8  = 2
flags      u8  reserved (0)
budgetTick u16 declared per-tick instruction budget
budgetMem  u16 declared allocation ceiling (KiB)
nHostCalls u16 count of host-call ids referenced
hostCalls  nHostCalls * u16   each on the ALLOWED_HOST_CALLS allowlist
nEntries   u16 entry-point table count (max 16)
entries    nEntries * (u8 hookId, u32 codeOffset)
nStrings   u16 string table count (max 256)
strings    nStrings * (u16 byteLen, byteLen bytes UTF-8)   each ≤ 512 bytes
codeLen    u32
code       codeLen bytes      opcode stream
```

> **v1 had neither table, and that made §4 unimplementable.** With one flat code section execution
> always began at offset 0, so no hook could be dispatched and `on_tick(dt)` had nowhere to receive
> `dt` — the entire event model was unreachable. v1 is rejected outright rather than migrated;
> nothing had shipped, and one supported shape is one less thing to get wrong at a trust boundary.

## 4. Execution model

- **Event-driven.** Hooks: `on_ready`, `on_tick(dt)`, `on_interact(player)`, `on_enter_zone`,
  `on_exit_zone`, `on_message(name, payload)`. No free-running threads. A module declares an entry
  offset per hook; `ScriptVm.RunHook` is the **only** way into a script — there is deliberately no
  "run the whole module" entry point.
- **Hook arguments arrive in locals 0..n-1.** There is no call stack and no parameter mechanism; a
  hook reads its arguments with `LOAD`. Locals are cleared between invocations so one event cannot
  read another's leftovers; the script-local variable store (`VAR_GET`/`VAR_SET`) is the only
  state that persists across hooks, and it is a **separate array** from the locals.
- **Fixed opcode set.** Arithmetic, comparison, locals, structured jumps (relative, bounds-checked
  into the code section), `HOST_CALL`, `RET`/`HALT`. No indirect calls, no dynamic dispatch.
- **Each hook invocation gets its own budget and watchdog**, so a slow `on_tick` cannot starve
  `on_interact`.
- **Bounded.** Every tick runs under a per-tick instruction budget (`budgetTick`), an allocation
  ceiling (`budgetMem`), and a wall-clock watchdog. Overrunning any bound **hard-kills** the
  offending script instance and removes it — the instance/frame keeps running.
- **Isolated state.** Each script has its own value stack, locals, and script-local variable
  store. No cross-script reach. No engine globals.

## 5. Host API — the allowlist (the only bridge)

Deny-by-default on **two** axes: an id must be on `ALLOWED_HOST_CALLS` at all, and the author must
clear the call's trust floor (`HOST_CALL_MIN_TRUST` / `HostCallTrust.MinTrust`). Both axes are
checked by the server validator *and* re-checked by the client at module load.

**Tier 0 — available to every rank:**

| Group | Calls | Scope |
|---|---|---|
| util | `LOG`, `TIME`, `RANDOM` (deterministic PRNG) | none |
| own nodes | `NODE_MOVE`, `NODE_ROTATE`, `NODE_SET_VISIBLE`, `NODE_PLAY_ANIM` | only nodes the script **declared**; never lookup by arbitrary path |
| media | `SOUND_PLAY`, `SCREEN_SET_TEXT` | whitelisted clips / this script's screens |
| players | `PLAYER_COUNT`, `PLAYER_POS(playerIndex, axis)` | read-only transform, within the world. There is deliberately **no** call that writes a player transform |
| state | `VAR_GET`, `VAR_SET` | script-local only — a store distinct from the VM's locals |
| net | `NET_EMIT` | **only** the relay's rate-limited script channel |

**Trust-gated tiers** — same allowlist, higher floor:

| Floor | Calls | Rationale |
|---|---|---|
| ≥5 (Creator) | `SHADER_SET_FLOAT`, `SHADER_SET_COLOR`, `SHADER_SET_VEC4` | shader uniforms on declared material nodes only |
| ≥6 (Trusted) | `PARTICLE_BURST`, `PARTICLE_SET_RATE`, `SOUND_PLAY_SPATIAL` | performance and audio-nuisance surface |
| ≥8 (VerifiedCreator) | `NET_EMIT_STRING`, `NET_SYNC_GET`, `NET_SYNC_SET` | string payloads and shared relay state; the largest abuse surface |

> These nine calls existed **only in the C# VM** for some time, absent from the server allowlist,
> along with the entire trust-floor concept — the exact allowlist drift T7 names. `PLAYER_POS`
> also returned only X, making a player's height unreadable. Both are fixed, and
> `game/Script/Tests/AllowlistAgreementTests.cs` now reads the TypeScript validator as data and
> fails the build on any divergence in either direction.

### 5.1 Player attachments (`PLAYER_ATTACH` / `PLAYER_DETACH`, Trust≥5)

Scripts may parent **their own declared nodes** to a player's bone — rope ends, harnesses, props,
carried objects. This is the only host call that touches a player at all, and it is deliberately
**one-directional**: the attachment rides the player. There is no call that moves, pushes,
teleports or constrains a player, and adding one would give an author control of another person's
body, which is the capability this design exists to withhold.

That boundary has a concrete consequence worth stating plainly: **a rope that physically pulls
players together is not expressible in script.** The constraint solver belongs in trusted native
code; the script owns the anchoring, the round state and the visuals. A world that needs shared
player physics is a native game mode with a scripted coordinator, not a pure script.

Implementations MUST:

- reject node slots the script did not declare (no arbitrary node lookup — the general rule);
- cap simultaneous attachments per script;
- treat an out-of-range player index as a no-op returning 0, never an engine lookup.

Attach points are a fixed enum (0 hips, 1 head, 2 left hand, 3 right hand, 4 chest) — a bone
*name* would be an arbitrary-lookup channel by another name.

Explicitly **absent** (and must stay absent): filesystem, `ResourceLoader`/`res://`/`user://`,
reflection, process/OS, arbitrary node lookup, cross-script access, engine singletons, raw sockets.

## 5.2 The toolchain

```bash
# compile a script (always re-validates its own output before writing)
bun tools/serikascript/sksc.ts world.sks -o world.sskb [--rank N]

# compiler unit tests — every case round-trips through the real server validator
cd tools && bun test serikascript/compiler.test.ts

# regenerate the cross-language golden corpus after any format/compiler change
bun tools/serikascript/gen-golden.ts

# the sandbox boundary suite: allowlist agreement + VM semantics + golden execution
dotnet test game/Script/Tests
```

The compiler (`tools/serikascript/compiler.ts`) **imports the allowlists from the server
validator** rather than restating them. A compiler carrying its own copy of the host-call table
would be a fourth place for the tables to drift, which is precisely how the drift that motivated
this work went unnoticed. It also enforces the trust floor at compile time, so using a call above
your rank is an error with a line number instead of an opaque rejection at submit.

`tools/serikascript/golden/` holds bytecode compiled by the TypeScript compiler and executed by
the C# VM in `GoldenModuleTests`. This is the same reasoning as `proto/golden`: two
self-consistent test suites prove nothing about each other, so one side produces an artifact and
the other consumes it.

## 5.3 How a world gets behaviour

A `.serikaworld` carries its compiled module as **`script.sskb`** beside `world.glb`. The world
declares what the script may touch using two markers, resolved by `WorldLoader.ResolveMarkers`
exactly like mirrors and seats:

| Marker | Becomes | Notes |
|---|---|---|
| `SERIKA_SNODE<n>` | stays a plain `Node3D`, registered as script slot `n` | the only nodes the script may move/show/attach. **Not** replaced — parent your geometry under it |
| `SERIKA_ZONE<n>` | a `ScriptZone` trigger firing `on_enter_zone`/`on_exit_zone` with id `n` | `scale` = full extents |

A script addresses both by integer slot, never by name or path. That is what makes "declared nodes
only" enforceable in `ScriptHostBridge` rather than aspirational: an out-of-range slot resolves to
null and the call is a no-op, and there is no code path from a script to a node lookup.

Startup order is load modules → bind zones → `ScriptWorld.Start()`. `Start()` is deliberately not
`_Ready()`: `_Ready` runs the moment the node enters the tree, which is *before* the loader has
added any module, so `on_ready` silently never ran. It must also come after `BindZone` or a script
reacting to a zone during startup misses events it should have seen.

Behaviour is additive, never a precondition: a world with no `script.sskb` never constructs a
`ScriptWorld`, and a world whose module is rejected still loads as geometry.

Verify the whole path in-engine with `--serika-scripttest` (see §5.2) — it runs the golden module
against real skeletons, real `Area3D` overlap and the real bridge, and exits non-zero on failure.

## 6. Networking

If a script emits multiplayer state (`NET_EMIT`), it rides the relay's existing rate-limited,
AOI/LOD-governed channel — never a new socket. Adding a networked script event is a **proto
change**: proto-first rules apply and both golden suites must pass byte-identically. Server never
trusts script-reported values for authority (same posture as `rms` in the pose codec).

## 7. Threat model

| # | Threat | Mitigation |
|---|--------|-----------|
| T1 | RCE via arbitrary engine call | Host-API allowlist; no `ResourceLoader`; VM has no reflection/FFI. Validated server-side + at VM load. |
| T2 | RCE via malicious glTF extension / embedded resource | glTF extension whitelist; conservative detector treats anything unprovable as "code" → human review. |
| T3 | Source-level compiler exploit on client | Client never compiles source; only runs validated bytecode. |
| T4 | Denial of service (hang / OOM / spin) | Per-tick instruction budget, allocation ceiling, wall-clock watchdog, hard-kill without crashing the instance. |
| T5 | Cross-script / cross-world data theft | Per-script isolated state; nodes must be declared; no global lookup; no cross-script API. |
| T6 | Network abuse (spam, amplification) | `NET_EMIT` only through the rate-limited relay channel; server budgets enforced. |
| T7 | Validator/VM allowlist drift (client accepts what server rejected, or vice-versa) | Single-source allowlists; CI grep invariant; golden tests; two-person review on any allowlist change. |
| T8 | Top-rank self-publish abuse | Auto-published scripted worlds from rank 8 still write an audit row and land in the spot-audit queue for post-hoc takedown + demotion. |
| T9 | Jump-based control-flow escape | Jumps are relative and bounds-checked into the code section at validation and at execution. |
| T10 | Integer/stack under/overflow in the VM | Fixed operand decoding, stack depth cap, checked arithmetic; malformed bytecode is rejected, never fixed up. |
| T11 | Entry-point escape (offset outside code) | Entry offsets are bounds-checked into the code section at validation on both sides, same as jumps (T9). Duplicate hook entries are rejected rather than resolved. |
| T12 | Attachment abuse (vision-blocking / nuisance props on other players) | `PLAYER_ATTACH` is Trust≥5, limited to the script's own declared nodes, capped per script, and fixed to an enum of attach points. It confers **no** control over player motion. §5.1. |
| T13 | Container table exhaustion | String and entry tables are capped (256 strings × 512 B, 16 entries) and checked before allocation, so a declared-but-absent table cannot be a resource attack. |

## 8. Rollout gates (do not skip)

1. This document signed off.
2. ✅ Server validator + C# VM agree on allowlists — enforced by
   `game/Script/Tests/AllowlistAgreementTests.cs` (`dotnet test game/Script/Tests`), which parses
   `server/api/src/serikascript.ts` and compares opcodes, host-call ids **and** trust floors
   against the C# enums. It fails on divergence in either direction, so neither side can gain a
   capability alone. Wire it into CI.
3. Two-person review of the entire sandbox boundary (`game/Script/`, `serikascript.ts`, the
   bundle detector in `review.ts`).
4. External pentest of the sandbox.
5. Only then: open scripted worlds to non-top-rank submission → review → public.

**Gate 5 scopes UNTRUSTED authorship.** A first-party world whose bytecode is built in this repo
is not the T1–T3 threat model — visitors run code the project wrote. Shipping first-party scripted
worlds ahead of gate 5 is therefore acceptable; opening *submission* to creators is not.

Until gate 5, scripted worlds route through manual admin review (below rank 8) or spot audit
(rank 8) exactly as System B/D already enforce.
