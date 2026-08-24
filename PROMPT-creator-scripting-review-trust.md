# Implementation Prompt — Creator Scripting, World Review Pipeline & Trust Ladder

> **How to use this file.** This is a self-contained implementation prompt for an engineering
> agent (or human) working in the Serika Social monorepo. Read `CLAUDE.md` / `AGENTS.md` first —
> the critical rules there (proto-first, no credentials in git, per-repo commits, UDP relay
> mapping) still apply. Everything below is additive to the systems already shipped. Build it in
> vertical slices, keep each slice green (`dotnet build` in `game/`, `bun`/`tsc` in `server/`),
> and **do not fake the sandbox** — see the security section.

---

## 0. Non-negotiables (read before writing code)

1. **Authentication reuses the existing flow. Do not invent a new one.** All logins — game,
   web, and the Godot SDK — go through `serika-accounts` OAuth2 + PKCE exactly as the game client
   already does:
   - Reference implementation: `game/Auth/PkceFlow.cs` (browser loopback on the **fixed** port
     `34517`, PKCE only — the client is public, no secret), then `POST /v1/session/exchange`
     `{code, code_verifier}` on the API, which mints our session JWT.
   - Constraints are documented and empirically verified in `docs/auth-integration.md`. The
     loopback redirect must match exactly; two token types, two verifiers (`verifyOAuth` for
     browser/opaque tokens, `verifyAccountsSession` for email/password JWTs). **Re-read that
     doc before touching auth.**
   - The SDK publisher (below) must obtain its token via this same browser-PKCE flow — **never**
     a hand-pasted token, never a bespoke password prompt.

2. **A world is never self-published.** Uploading a world creates a *submission*, not a live
   world. Nothing a creator uploads becomes joinable by others until it clears the pipeline in
   §3. This replaces today's `POST /v1/worlds/upload` behavior, which immediately creates a
   `releaseStatus: 2` (public) world.

3. **Code-bearing worlds require manual admin review — unless the author is at the top trust
   rank.** Only the highest rank may publish scripted worlds without human review (and even then
   they are logged and spot-auditable). Everyone else's scripted worlds sit in the review queue
   until an admin approves them in the panel (§4).

4. **The sandbox is the whole ballgame.** Per `CLAUDE.md`: "One `ResourceLoader.Load()` on a user
   path, one un-audited glTF extension, one whitelist gap = RCE on everyone who visits that
   world. No partial credit." Untrusted creator code is the highest-risk surface in the product.
   If you cannot make a capability provably safe, it does not ship. See §6.

---

## 1. Scope

Deliver four interlocking systems:

| # | System | Repo(s) |
|---|--------|---------|
| A | **SerikaScript** — a sandboxed scripting layer so creators can add behavior (minigames, working theme-park rides, interactive props) to avatars and worlds. | `game/` (VM + host API), `godot-sdk/` (authoring), `proto/` only if networked script events are added |
| B | **World review pipeline** — submission → automated validation → (conditional) manual review → publish. No self-publish. | `server/`, `web/` |
| C | **Expanded trust ladder** — many ranks, each unlocking specific creator capabilities; scripted publishing gated to the top rank. | `server/`, `game/`, `web/` |
| D | **Admin review panel** — a web console that *forces* manual review of code-bearing worlds, with the tools to actually judge them. | `web/`, `server/` |

Ship them in the order **C → B → D → A**: the trust ladder and review pipeline gate everything;
the admin panel makes review possible; scripting is the largest and riskiest and comes last, once
there is a queue and a reviewer to catch it.

---

## 2. System C — Trust ladder ("a bunch of trust ranks")

Today `User.trustLevel` is `Int 0..4` and `server/api/src/trust.ts` gates uploads at level 2.
Expand it into a real ladder. Keep it an integer column (no migration risk to the type) but give
each level a name, a source, and a capability set.

### 2.1 Ranks

Extend `trustLevel` to `0..8`. Names are illustrative — pick final copy with the product owner,
but keep the *shape*: standing rises with activity + vouching, and creator power is gated per rank.

| Level | Name | How it's earned | Notable capabilities unlocked |
|------:|------|-----------------|-------------------------------|
| 0 | **Visitor** | default on first login | join worlds, chat, wear built-in avatars |
| 1 | **Newcomer** | verified email + N minutes in-world | upload avatars (private), favorite worlds |
| 2 | **Member** | some playtime + no active moderation flags | **submit** static (code-free) worlds for review; publish avatars |
| 3 | **Regular** | sustained activity | submit worlds with **more asset budget**; host public instances |
| 4 | **Known** | vouched by ≥K Trusted+ users, or premium | faster review SLA; submit worlds containing **script** (still reviewed) |
| 5 | **Creator** | ≥M published worlds that passed review clean | larger script capability budget; **auto-approve static worlds** (no human review for code-free) |
| 6 | **Trusted** | manual grant by staff, or strong track record | may review *avatars* (not worlds); community-moderator tools |
| 7 | **Partner** | staff grant | higher instance caps, featured placement eligibility |
| 8 | **Verified Creator** | **top rank**, staff grant only | **publish scripted worlds without manual review** (still logged + spot-audited) |

Admins (`User.isAdmin`) are orthogonal — they can always review and publish, and act as level 8
for capability checks.

### 2.2 Server model & helpers

- Rewrite `server/api/src/trust.ts` into the source of truth:
  - `export const TrustRank = { Visitor:0, Newcomer:1, ... VerifiedCreator:8 } as const;`
  - `TRUST_LABELS: Record<number,string>` and `trustLabel(n)`.
  - **Capability thresholds** as named constants, not magic numbers scattered across routes:
    ```ts
    export const Cap = {
      SubmitStaticWorld:   TrustRank.Member,        // 2
      SubmitScriptedWorld: TrustRank.Known,         // 4
      AutoApproveStatic:   TrustRank.Creator,       // 5
      PublishScriptNoReview: TrustRank.VerifiedCreator, // 8
      UploadAvatar:        TrustRank.Newcomer,      // 1
    } as const;
    ```
  - `requireTrust(userId, min)` (already exists) + `hasCap(userId, cap): Promise<boolean>`.
  - Reads the **live** `trustLevel` column so a just-granted rank takes effect without re-login.
- Keep the login floor logic in `server/api/src/users.ts` (admins → 8, premium → a mid rank) but
  **never lower** a manually granted level.
- Add an admin endpoint to set a user's rank: `POST /v1/admin/users/:id/trust {level, reason}`
  → writes `trustLevel` and a `ModerationAction`-style audit row (see §4.3). Guard with
  `adminOnly`.

### 2.3 Client display

- Game: the pause hub already shows a trust chip (`QuickMenu.SetTrust`) from the login
  `trustLevel`. Extend the label map to 0..8 (`Main.TrustLabels`). Optionally add a small trust
  badge to remote name cards (`NameTag3D`) using the `/v1/users/by-id/:id/card` route — extend
  that route to return `trustLevel`.
- Web: show the rank on the profile page and next to authored worlds.

---

## 3. System B — World review pipeline (no self-publish)

### 3.1 States

Replace the current "upload → instantly public" path. Add an explicit review state machine.
Reuse `World.releaseStatus` for the *final* visibility and add a **submission/review** layer on
`WorldVersion` (versions already carry `buildStatus` and a `validatorReport` JSON — extend them).

Add to `WorldVersion` (Prisma migration):

```prisma
/// 0=draft 1=submitted 2=auto_approved 3=in_review 4=changes_requested 5=approved 6=rejected 7=withdrawn
reviewStatus     Int      @default(0) @map("review_status")
/// Set when SerikaScript (or any executable content) is detected in the bundle.
hasScript        Boolean  @default(false) @map("has_script")
/// Automated validator verdict (asset budget, whitelist, script static-analysis).
validatorReport  Json?    @map("validator_report")   // (already exists — populate it)
reviewedById     String?  @map("reviewed_by_id") @db.Uuid
reviewedAt       DateTime? @map("reviewed_at")
reviewNotes      String   @default("") @map("review_notes")
```

Add a `WorldReview` audit table (one row per decision) so the history is durable:

```prisma
model WorldReview {
  id            String   @id @default(uuid()) @db.Uuid
  worldVersionId String  @map("world_version_id") @db.Uuid
  reviewerId    String   @map("reviewer_id") @db.Uuid
  decision      Int      // 0=approve 1=reject 2=request_changes
  notes         String   @default("")
  createdAt     DateTime @default(now()) @map("created_at")
  @@map("world_reviews")
}
```

### 3.2 Submission flow (replaces `POST /v1/worlds/upload`)

1. **Auth + trust gate.** `authed`; require `Cap.SubmitStaticWorld` (level 2). Reject below with
   `403 insufficient_trust` (already implemented — keep the shape).
2. **Normalize** the bundle to `.serikaworld` as today (`toSerikaWorld`).
3. **Detect code.** Scan the bundle for SerikaScript assets / executable content (§6.4). Set
   `hasScript`. If scripted, require `Cap.SubmitScriptedWorld` (level 4) to even submit.
4. **Automated validation** (synchronous, fast; the "validator" the SDK already has a stub for):
   - asset budget (poly/material/texture caps scaled by trust rank),
   - glТF extension whitelist,
   - path/URL whitelist (no `res://`/`user://`/absolute loads),
   - **SerikaScript static analysis** (§6.5) — reject on any disallowed opcode/host call.
   Write the verdict into `validatorReport`. A validator failure → `reviewStatus = rejected`
   with the report; never queues a human.
5. **Route by trust + content:**
   - **Code-free** and author ≥ `Cap.AutoApproveStatic` (level 5): `reviewStatus = auto_approved`
     → publish immediately (§3.4).
   - **Code-free** and author < level 5: `reviewStatus = submitted` → enters the human queue.
   - **Scripted** and author == top rank (`Cap.PublishScriptNoReview`, level 8) **or** admin:
     `reviewStatus = auto_approved` → publish, **but** log a `WorldReview` row with
     `reviewer = author (self, top-trust)` and flag it for **spot audit** (§4.4).
   - **Scripted** and author < level 8: `reviewStatus = in_review` → **must** be reviewed by an
     admin in the panel. There is no path to public for scripted content below the top rank
     except an explicit admin approval.
6. Response tells the creator exactly where they landed: `{ submissionId, reviewStatus, hasScript,
   validator: {...}, estimatedReview: "..." }`.

### 3.3 Visibility rules

- A world is joinable by others **only** when it has a `publishedVersionId` pointing at a
  version whose `reviewStatus ∈ {auto_approved, approved}`.
- The author may always launch their **own** submission privately (a "test instance") regardless
  of review state, so they can iterate — but no one else can join, and it never appears in the
  browser. Enforce in the instance-allocation path (`server/api/src/routes/instances.ts`).

### 3.4 Publish

Publishing sets `World.releaseStatus = 2`, `World.publishedVersionId = version.id`, and pushes the
bundle to the CDN (as today). Emit an audit row. Notify the author.

### 3.5 Re-submission / versioning

New upload for an existing world → new `WorldVersion` with `reviewStatus = submitted/in_review`.
The currently-published version stays live until the new one is approved (no downtime, no
accidental unpublish).

---

## 4. System D — Admin review panel (forces manual review)

A world with `reviewStatus = in_review` (scripted, author below top rank) is **blocked** until an
admin acts. Build the console that makes that judgeable, not a rubber stamp.

### 4.1 Server (all `adminOnly`, prefix `/v1/admin/review`)

- `GET /queue?status=in_review|submitted&scripted=true|false` — paginated queue, newest first,
  with author, trust rank, `hasScript`, validator summary, submission time, SLA age.
- `GET /world-version/:id` — full detail: manifest, asset manifest + sizes, **the extracted
  SerikaScript source**, the validator report, the author's prior review history and any
  moderation actions.
- `GET /world-version/:id/preview-ticket` — mint a single-use ticket to launch the submission in
  an **isolated review instance** (sandbox hardened, see §6), so the reviewer can actually play it.
- `POST /world-version/:id/decision { decision: approve|reject|request_changes, notes }` — writes
  `WorldReview`, updates `reviewStatus`, and on approve runs the publish step (§3.4). On
  `request_changes`, notify the author with the notes.
- `POST /users/:id/trust { level, reason }` — the rank-grant endpoint from §2.2.

### 4.2 Web UI (`web/app/admin/review/`)

- **Queue table**: filter by scripted/static, sort by SLA age, badge overdue items. Purple brand
  (hue 262), consistent with the rest of the site.
- **Detail view**: three panes —
  1. **Metadata & author** (rank, history, past rejections, moderation record),
  2. **Static analysis** (validator report rendered readable; script source with the flagged
     lines highlighted; asset budget bars),
  3. **Live preview** (launch the isolated review instance via the preview ticket, or an embedded
     read-only scene inspector).
- **Decision bar**: Approve / Request changes / Reject with a required notes field on the negative
  paths. Keyboard shortcuts. Optimistic UI with server confirmation.
- Gate the whole section behind `isAdmin` (server already has `adminOnly`; the web must also hide
  the nav and 403 gracefully).

### 4.3 Audit

Every decision, trust grant, and top-rank self-publish writes an immutable audit row (extend
`ModerationAction` or add a dedicated `AuditLog`). Admins can browse it. This is how top-rank
self-publishing stays accountable.

### 4.4 Spot-audit queue

Top-rank (level 8) scripted worlds auto-publish but land in a **spot-audit** list. An admin can
review them post-hoc and, if they find abuse, unpublish + demote in one action. Surfacing this is
what keeps rank 8 a privilege, not a blind spot.

---

## 5. System A — SerikaScript (sandboxed creator code)

> **This is a compiler + VM + security-sandbox project.** `CLAUDE.md` warns it is "easy to
> underestimate by 10×." Budget accordingly. Do the design doc and threat model **before** the VM.

### 5.1 Design goals

- Let creators attach behavior to worlds and avatars — minigames (scores, timers, win states),
  working theme-park rides (triggers → animated platforms → seated players), interactive props
  (buttons, doors, spawners, scoreboards).
- **Capability-based, deny-by-default.** Scripts get an explicit host API surface and *nothing*
  else. No filesystem, no network, no `ResourceLoader`, no reflection, no arbitrary node access
  outside the script's own subtree + declared exports.
- **Deterministic-ish and bounded.** Per-frame instruction budget, allocation ceiling, wall-clock
  watchdog. A malicious or buggy script must not hang or OOM the client.

### 5.2 Recommended shape

- A small, statically-analyzable language (or a locked-down subset), compiled by the SDK to a
  **bytecode** artifact that ships inside the `.serikaworld` / `.ska`. **Ship bytecode, not
  source, to the runtime** — the client never compiles untrusted text; the SDK compiles, the
  server validates the bytecode, the client executes it in the VM.
- A hand-written **VM in C#** (`game/Script/`) with:
  - a fixed opcode set (no dynamic code gen, no `eval`),
  - a **host API allowlist** — the only bridge to the engine (see §5.3),
  - per-script isolated state, an instruction budget per tick, and a hard kill on overrun.
- Golden tests for the VM. If script events are networked (multiplayer minigame state), that is a
  **proto change** — proto-first rules apply, both golden suites must pass byte-identically.

### 5.3 Host API surface (illustrative, allowlist)

Event hooks: `on_ready`, `on_tick(dt)`, `on_interact(player)`, `on_enter_zone(player)`,
`on_exit_zone(player)`, `on_message(name, payload)`.
Safe host calls, scoped to the script's own nodes: move/rotate/animate a declared node, play a
whitelisted sound, set a screen/text value, read player transform *within the world*, emit a
network message *through the relay's rate-limited channel*, get/set script-local variables,
timers. **No** node lookup by arbitrary path, **no** cross-script reach, **no** engine globals.

### 5.4 Detecting "code in a world" (for §3.3)

A world "has code" if the bundle contains any SerikaScript bytecode asset, an executable manifest
hook, or any non-whitelisted glTF extension / embedded resource that could execute. The detector
is conservative: **anything it can't prove is inert counts as code** and routes to review.

### 5.5 Static analysis (validator, server-side, blocking)

Before a scripted world can even enter the human queue, the server validates the bytecode:
verify opcodes are in the allowed set, every host call is on the allowlist, budgets are declared
and within caps for the author's rank, and there are no forbidden constructs. Reject with a
precise report otherwise. This runs in §3.2 step 4 and again at publish time (defense in depth).

### 5.6 SDK authoring (`godot-sdk/`)

- Add a script node type + editor UI to attach a script to a world/avatar node.
- Compile to bytecode on save; run the same validator locally so creators get instant feedback.
- The uploader publishes the bytecode inside the bundle.

---

## 6. Security requirements (do not ship without these)

1. **Deny-by-default everywhere.** Host API allowlist, glTF extension whitelist, asset path
   whitelist. New capability = explicit, reviewed addition.
2. **Ship bytecode, validate on the server, execute in the VM.** The runtime never parses
   untrusted source. The server re-validates on submit *and* on publish.
3. **Resource bounds.** Per-tick instruction budget, allocation ceiling, watchdog timer, and a
   hard kill that removes the offending script without crashing the instance.
4. **Isolated review instances.** The admin preview runs the submission with the strictest
   sandbox profile and no other players.
5. **Two-person + external.** Per `CLAUDE.md`: CI-grepped invariants, two-person review on the
   sandbox boundary, and an external pentest before scripted worlds open to the public.
6. **The top-rank exception is logged, not trusted blindly.** Level-8 self-publish still writes an
   audit row and enters the spot-audit queue.

---

## 7. Data-model change summary

- `User.trustLevel`: keep `Int`, extend meaning to `0..8`. No type change.
- `WorldVersion`: add `reviewStatus`, `hasScript`, `reviewedById`, `reviewedAt`, `reviewNotes`;
  populate the existing `validatorReport`.
- New `WorldReview` (per-decision audit), new/extended `AuditLog` for trust grants & self-publish.
- All via Prisma migrations (`bun run db:migrate`). **No credentials in git.**

---

## 8. Acceptance criteria

- [ ] Uploading a world **never** makes it publicly joinable directly; it creates a submission in
      the correct `reviewStatus` for the author's rank and content.
- [ ] A code-free world from a level-5+ creator auto-publishes; from a lower rank it queues.
- [ ] A scripted world from a **non-top-rank** author is **blocked** until an admin approves it in
      the panel. There is no bypass.
- [ ] A scripted world from the **top rank** (or admin) auto-publishes **and** appears in the
      spot-audit queue with an audit row.
- [ ] The admin panel lists the queue, shows script source + validator report + author history,
      launches an isolated preview, and records every decision.
- [ ] All logins (game, web, **SDK**) authenticate through the existing serika-accounts PKCE flow;
      no new auth path, no hand-pasted tokens.
- [ ] SerikaScript runs only validated bytecode in the sandboxed VM; a script cannot touch the
      filesystem, network (except the rate-limited relay channel), `ResourceLoader`, or nodes
      outside its declared scope; a runaway script is killed without taking down the instance.
- [ ] `dotnet build` (game), server typecheck/tests, and — if script networking is added — both
      proto golden suites pass.
- [ ] Trust ranks display in-game (pause hub, name cards) and on the web profile.

---

## 9. Suggested slice order

1. **C1** — expand `trust.ts` ranks + capability constants; admin trust-grant endpoint; client
   label maps. (Small, unblocks gating.)
2. **B1** — submission state machine + `reviewStatus` migration; change upload to create
   submissions; static-vs-scripted routing (script detection can start as "bundle contains a
   `.sscript`/bytecode asset"). No scripting execution yet.
3. **D1** — admin queue + decision endpoints + web review console (static worlds first).
4. **B2/D2** — isolated preview instances; author private test launch; spot-audit queue.
5. **A1** — SerikaScript design doc + threat model (**gate: sign-off before coding the VM**).
6. **A2** — VM + host API allowlist + validator + SDK authoring; wire the review pipeline's
   script path end-to-end.
7. **Hardening** — CI invariants, two-person review, external pentest, then open scripted worlds.

---

*Keep `AGENTS.md` and `CLAUDE.md` in sync if any workflow here changes them. Commit per-repo.
Proto-first if you touch the wire format.*
