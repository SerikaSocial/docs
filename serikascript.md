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
version    u8  = 1
flags      u8  reserved (0)
budgetTick u16 declared per-tick instruction budget
budgetMem  u16 declared allocation ceiling (KiB)
nHostCalls u16 count of host-call ids referenced
hostCalls  nHostCalls * u16   each on the ALLOWED_HOST_CALLS allowlist
codeLen    u32
code       codeLen bytes      opcode stream
```

## 4. Execution model

- **Event-driven.** Hooks: `on_ready`, `on_tick(dt)`, `on_interact(player)`, `on_enter_zone`,
  `on_exit_zone`, `on_message(name, payload)`. No free-running threads.
- **Fixed opcode set.** Arithmetic, comparison, locals, structured jumps (relative, bounds-checked
  into the code section), `HOST_CALL`, `RET`/`HALT`. No indirect calls, no dynamic dispatch.
- **Bounded.** Every tick runs under a per-tick instruction budget (`budgetTick`), an allocation
  ceiling (`budgetMem`), and a wall-clock watchdog. Overrunning any bound **hard-kills** the
  offending script instance and removes it — the instance/frame keeps running.
- **Isolated state.** Each script has its own value stack, locals, and script-local variable
  store. No cross-script reach. No engine globals.

## 5. Host API — the allowlist (the only bridge)

Deny-by-default. Every id below is on `ALLOWED_HOST_CALLS`; anything else fails validation.

| Group | Calls | Scope |
|---|---|---|
| util | `LOG`, `TIME`, `RANDOM` (deterministic PRNG) | none |
| own nodes | `NODE_MOVE`, `NODE_ROTATE`, `NODE_SET_VISIBLE`, `NODE_PLAY_ANIM` | only nodes the script **declared**; never lookup by arbitrary path |
| media | `SOUND_PLAY`, `SCREEN_SET_TEXT` | whitelisted clips / this script's screens |
| players | `PLAYER_COUNT`, `PLAYER_POS` | read-only transform, within the world |
| state | `VAR_GET`, `VAR_SET` | script-local only |
| net | `NET_EMIT` | **only** the relay's rate-limited script channel |

Explicitly **absent** (and must stay absent): filesystem, `ResourceLoader`/`res://`/`user://`,
reflection, process/OS, arbitrary node lookup, cross-script access, engine singletons, raw sockets.

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

## 8. Rollout gates (do not skip)

1. This document signed off.
2. Server validator + C# VM agree on allowlists (golden test + CI grep invariant).
3. Two-person review of the entire sandbox boundary (`game/Script/`, `serikascript.ts`, the
   bundle detector in `review.ts`).
4. External pentest of the sandbox.
5. Only then: open scripted worlds to non-top-rank submission → review → public.

Until gate 5, scripted worlds route through manual admin review (below rank 8) or spot audit
(rank 8) exactly as System B/D already enforce.
