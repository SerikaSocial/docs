# Implementation Prompt — Make Imposter Station, Gauntlet Course, and Rope Parkour into real scripted minigames

> Paste this entire file into Z.ai (or any engineering agent) working in the Serika Social
> monorepo. Read `CLAUDE.md` / `AGENTS.md` first. The critical rules there still apply:
> proto-first, no credentials in git, **commit per-repo never from the root**, UDP relay mapping,
> Godot binary is the full path in CLAUDE.md (not on PATH), game UI is built in C# not `.tscn`,
> purple brand hue 262 via `game/UI/Brand.cs`.
>
> **This is not a greenfield.** v1.10.0 already shipped a server-authoritative game API, a HUD
> shell, three public worlds, and three example `.sks` files. The honest status is: **none of
> that is a playable game.** Your job is to make them actual minigames — designed maps, in-world
> affordances, a start UI, and scripts that drive the world — without lying about what the
> sandbox can do.

---

## 0. What is actually true today (read before writing code)

Do not "continue" the v1.10.0 work as if the games exist. They do not.

### What shipped

| Layer | What exists | What it actually does |
|---|---|---|
| API | `server/api/src/gamemode.ts` (pure, 30 tests) + `server/api/src/routes/games.ts` | Roles, kills, votes, placement, win conditions. Live in production (`POST /v1/games/:id/start` returns 401 unauth, which means the route is mounted). |
| Client session | `game/Game/GameSession.cs` | Polls `/state` + `/me` every 1.5s. Exposes `TryStart` / `TryKill` / `TryReport` / `TryVote` / `TryCompleteTask` / `TryFinish`. **Nothing in the game calls any of these except `TryVote`.** |
| HUD | `game/Game/GameHud.cs` | Wrist readout (role / tasks / cooldown) + voting panel. Renders only after a session exists and phase ≠ Lobby. |
| World opt-in | `game/World/WorldLoader.cs` | Manifest `gameMode: "imposter" \| "gauntlet"` stands up a GameSession. `rope.length` stands up `Player/RopeTeam.cs`. |
| Scripts | `tools/serikascript/examples/{imposter_station,gauntlet_course,rope_parkour}.sks` | Log a line and `emit()` advisory numbers. They never complete a task, never kill, never start a round, never move a hazard. |
| Maps | `tools/build_worlds.py` → `build_imposter_station` / `build_gauntlet_course` / `build_rope_parkour` | Primitive Blender boxes/cylinders. Not designed levels. |
| Script runtime | `game/Script/` VM + `ScriptWorld.Interact(playerIndex)` | `on_interact` exists in the VM. **No `IInteractable` ever calls it.** Zone enter/exit works. |

### What is missing (this is the job)

1. **No game-start UI.** Instance host has no button that calls `GameSession.TryStart`. A lobby of 10 people in Imposter Station can stand around forever.
2. **No in-world bindings** for kill / report / task / finish / meeting. The HUD cannot trigger a kill. There is no `[E] Kill`, `[E] Report`, `[E] Do task`, `[E] Call meeting`.
3. **Scripts do not drive gameplay.** They are commentators. Moving gates, task-console feedback, scoreboards, vent covers, finish-line lights — none of it happens.
4. **Maps are grey boxes.** A hidden-role game on six open pads is not Among Us. A 120 m strip of boxes is not Fall Guys. A spiral of cubes is not a parkour course.
5. **`on_interact` is unwired.** `ScriptWorld.Interact` is never reached from `World/Interactor.cs`.
6. **Task completion is not location-gated in UX.** `POST /v1/games/:id/task` just increments. The player never has to stand at a console.
7. **Gauntlet hazards are static.** The comment in `build_gauntlet_course` even admits the "rotating beams" are frozen.
8. **Vents do not teleport.** Scripts **cannot** write a player transform (`PLAYER_POS` is read-only; `PLAYER_ATTACH` rides the player). If vents are real, they are a **native** interactable.

### Production identities (do not recreate these worlds)

| World | Capacity | `gameMode` / extras | Notes |
|---|---|---|---|
| Imposter Station | 10 | `"imposter"` | Public. Six task rooms + hub. ID was `aa921bcc-3665-498a-99b2-1bf3ef11ed34` at publish — re-read live `/v1/worlds` before overwriting. |
| Gauntlet Course | 16 | `"gauntlet"` | Public. ID was `1c7e9d14-7621-4c7f-b063-6f90ce145958`. |
| Rope Parkour | 4 | `rope.length`, no gameMode | Public. Native `RopeTeam` + script coordinator. Look up the live id. |

Update in place with `server/api/src/publish-scripted-world.ts` (scripted) or `publish-world-bundle.ts` (static). Do not insert duplicate catalogue rows.

---

## 1. Non-negotiables

1. **Read `docs/serikascript.md` and `docs/PROMPT-creator-scripting-review-trust.md`.** The sandbox is the product risk. One whitelist gap = RCE on every visitor. If a capability cannot be proven safe, it does not ship.

2. **Do not put hidden-role authority in a script or on a client.** The relay is an ownership-based fan-out, not an authoritative sim. A client that is told who the imposter is has already leaked it. `/v1/games/:id/me` returns **only your role**. `/state` returns phase and counts. **No new endpoint may return the role map while the game is live.** `gamemode.ts` stays pure (no Redis/Prisma/Elysia). Keep the 30 existing tests; add more rather than rewriting.

3. **Do not add a host call that writes a player transform.** `PLAYER_POS` is read-only. `PLAYER_ATTACH` / `PLAYER_DETACH` parent a *declared node* onto a player bone; the attachment rides the player. A rope that physically pulls bodies is native (`game/Player/RopeTeam.cs`). A vent that teleports is native. A kill that ragdolls is native. Scripts move **their own nodes**.

4. **Do not change proto** unless a new networked script event is genuinely required. Advisory `NET_EMIT` already exists. Authoritative game state stays on HTTP (`GameSession` polls the API). That split is deliberate: the script channel is client-originated and untrusted.

5. **Do not expand the host-call allowlist** unless a specific first-party need is unexpressible today, and then you must update **all four** of: `server/api/src/serikascript.ts`, `game/Script/OpCode.cs` + `IHostBridge.cs` + `ScriptHostBridge.cs`, `tools/serikascript/compiler.ts` (it **imports** the server tables — do not fork them), and `game/Script/Tests/AllowlistAgreementTests.cs` must stay green. Prefer expressing the need with existing calls (`NODE_MOVE`, `NODE_ROTATE`, `NODE_SET_VISIBLE`, `SCREEN_SET_TEXT`, `SOUND_PLAY`, zone hooks, `on_interact`).

6. **Game UI is C# in `game/UI/` and `game/Game/`, styled through `Brand.cs`.** No `.tscn` screens. VR rule from CLAUDE.md: persistent readouts go to the wrist (`AddUi(..., chrome: true)`); interactive menus go to the panel; a layer that hides its children but stays visible pins a laser in the player's face. `GameHud` already splits readout vs voting this way — copy it.

7. **Each subdirectory is its own git repo.** Commit in `game`, `server`, `tools`, `docs` separately. Never from the monorepo root.

8. **Licenses for sourced 3D models must be usable in a commercial shipped client.** Prefer CC0 / public domain. CC-BY is fine with an attribution file. **Reject CC-BY-NC, CC-BY-ND, "free for personal use", Sketchfab "download disabled", and anything whose redistribution terms you cannot quote.** Record every asset in a `SOURCES.md` next to the world.

---

## 2. Architecture — three layers, one game

A minigame in this engine is **not** "write it all in SerikaScript." It is a native mode with a scripted coordinator and a designed world.

```
                    ┌─────────────────────────────────────────┐
  Server (truth)    │  /v1/games  — roles, kills, votes,      │
                    │  placement, win conditions in Redis     │
                    │  Client is told only what it may know   │
                    └─────────────────────────────────────────┘
                                      ▲ HTTP poll + POST
                    ┌─────────────────────────────────────────┐
  Native C#         │  GameSession / GameHud / new            │
  (trusted client)  │  IInteractable nodes / lobby button     │
                    │  RopeTeam / vent teleport / kill target │
                    │  Wires E / trigger / VR panel           │
                    └─────────────────────────────────────────┘
                                      ▲ markers + Interact()
                    ┌─────────────────────────────────────────┐
  SerikaScript      │  script.sskb in the .serikaworld        │
  (untrusted VM)    │  Moves declared nodes, lights, doors,   │
                    │  hazards, scoreboards; fires on zone /  │
                    │  interact; NEVER decides who wins       │
                    └─────────────────────────────────────────┘
                                      ▲
                    ┌─────────────────────────────────────────┐
  World GLB         │  Designed map + SERIKA_SNODE / ZONE /   │
                    │  SPAWN / SEAT / MIRROR markers          │
                    └─────────────────────────────────────────┘
```

### What belongs where (use this as a checklist)

| Need | Where | Why |
|---|---|---|
| Assign roles, count kills, tally votes, assign finish place | API `gamemode.ts` | Hidden info / anti-cheat. Already built. |
| Start game button, kill/report/task/finish prompts, nearby-player targeting | Native C# `IInteractable` + QuickMenu | Needs engine input, peer list, `GameSession`. |
| Ghost appearance, meeting freeze, round countdown on HUD | Native C# | Needs avatar / camera / HUD. |
| Vent teleport, kill confirmation, ragdoll/ghost | Native C# | Must write player transform or avatar state. |
| Rope pulling players together | Native `RopeTeam.cs` | Already built. Do not reimplement in script. |
| Task console glow, meeting-button pulse, door swing, moving Gauntlet hazards, finish-line lights, checkpoint chimes, scoreboard text | **SerikaScript** | This is what scripts are *for*. Declared nodes + `on_tick` / `on_interact` / zones. |
| "Someone is at the meeting table" advisory | Script `emit(channel, payload)` | Untrusted hint. API still decides if a meeting starts. |

---

## 3. Phase 0 — find real 3D models (do this before modelling in Blender primitives)

Replace the procedural boxes. Do **not** generate a new greybox and call it a map. Search, download, convert, attribute.

### Where to search (in this order)

1. **Poly Haven** (CC0) — environment kits, textures, props. https://polyhaven.com
2. **Kenney.nl** (CC0) — modular kits, especially space/sci-fi and platformer. https://kenney.nl
3. **Sketchfab** — filter: downloadable, CC0 or CC-BY, triangulated, reasonable polycount. Reject "download disabled".
4. **Quaternius / Quaternius Ultimate** (public domain / CC0 collections) for stylised props.
5. **OpenGameArt** — check the *exact* license per file, not the page banner.

Use web search. For each candidate record: URL, title, author, license, polycount, whether textures are embedded, whether the scale is metres.

### Search queries (run all of them; pick the best *kit*, not a single hero mesh)

**Imposter Station (Among Us-style spaceship interior, cap 10, social-VR scale):**

- `CC0 sci-fi spaceship interior modular kit glTF`
- `Kenney space kit interior rooms`
- `Sketchfab CC-BY "spaceship corridor" downloadable glTF`
- `Poly Haven metal floor / wall / grating textures 4k` (for re-texturing a kit)
- `CC0 sci-fi console prop`, `emergency meeting table`, `vent grate`, `reactor room`
- Avoid anything branded "Among Us", "MIRA HQ", InnerSloth IP. We want the *genre* (compact spaceship with named rooms and vents), not the franchise.

**Gauntlet Course (Fall Guys-style obstacle race, cap 16, 100–150 m):**

- `CC0 colorful obstacle course kit`, `Kenney platformer kit`
- `Sketchfab CC0 spinning bar hazard`, `moving platform glTF`
- `CC0 finish arch`, `start gate`, `slime / foam pit`
- Stylised, saturated, readable at a sprint. Not realistic industrial.
- Avoid Fall Guys / Mediatonic IP (beans, crowns as logos).

**Rope Parkour (co-op climbing tower, cap 4):**

- `CC0 cliff / tower climbing environment`
- `CC0 wooden scaffold / rope bridge kit`
- `Sketchfab CC-BY "climbing tower" OR "spiral staircase tower"`
- Vertical, readable ledges, somewhere to stand as a pair (wide checkpoints). Not a closed cave.

### Scale and VR constraints (fail the asset if it misses these)

- Godot / glTF is metres, Y-up after Blender conversion (`tools/convert_world.py` + `tools/blender_convert.py`).
- Standing eye height ~1.6 m. Doorways ≥ 2.2 m. Corridors ≥ 2.0 m wide (two avatars + a third passing).
- Imposter rooms must be large enough to walk around a console, small enough that "I was in Reactor" is a real alibi (roughly 8–14 m across).
- Gauntlet lane at start ≥ 12 m across so 16 players are not a mosh pit; widen down the course.
- No nanite / 4K-everywhere: budget the station as one GLB with textures ≤ 2048 (`convert_world.py --max-tex 2048`). Atlas where possible.
- Collision must match walkable floors. `WorldLoader` can generate collision from geometry (`manifest.collision = "geometry"`) — overlapping decorative meshes that block walking are a defect.

### Conversion pipeline (already exists — use it)

```bash
# 1. Convert a sourced model to a .serikaworld (needs Blender on PATH)
python3 tools/convert_world.py <input.glb|fbx|obj> work/<world>/<world>.serikaworld \
    --name "Imposter Station" --lighting lit --max-tex 2048

# 2. Open the GLB in Blender and ADD THE MARKERS. Conversion does not invent
#    SERIKA_* nodes. A pretty map with no markers is still an inert lobby.

# 3. Re-pack, compile script, insert script.sskb into the zip:
bun tools/serikascript/sksc.ts tools/serikascript/examples/imposter_station.sks \
    -o work/imposter/script.sskb --rank 8
# zip the GLB + manifest.json + script.sskb + thumbnail.png → .serikaworld

# 4. First-party scripted publish (dry run, then --publish --backup FILE):
cd server/api
bun --env-file=../.env src/publish-scripted-world.ts \
    --world <EXISTING_ID> --bundle ../../work/imposter/imposter.serikaworld \
    --name "Imposter Station" --capacity 10 --tags minigame,imposter
```

Procedural `build_worlds.py` stays as a **fallback** for when no sourced kit fits, but the primary path is sourced kits + handmade marker layout. If you fall back, the result still has to look like a place, not a CSG demo: trim, materials, lights, readable silhouettes.

Write `tools/worlds/<world>/SOURCES.md` with: asset, author, license, URL, what you used it for, any modifications (re-scale, re-texture, deleted interior junk).

---

## 4. Marker contract (the script and the native code both bind to these names)

`WorldLoader.ResolveMarkers` already swaps:

| Marker name | Becomes | Notes |
|---|---|---|
| `SPAWN` | spawn point | Manifest `spawn` wins if present |
| `SERIKA_MIRROR*` | live `Mirror` | `scale.X` = width, `scale.Z` = height |
| `SERIKA_SEAT*` | `SeatNode` | node yaw = `SitYaw` |
| `SERIKA_VIDEO*` | video screen | not needed for these three worlds |
| `SERIKA_SNODE<n>` | **kept** as `Node3D`, script slot `n` | Parent visual geometry **under** it. Script may move/show/attach this node only. |
| `SERIKA_ZONE<n>` | `ScriptZone` | `scale` = full extents. Fires `on_enter_zone` / `on_exit_zone`. |

Add **new native markers** (this is required; scripts cannot do these jobs):

| New marker | Becomes | Worlds |
|---|---|---|
| `SERIKA_TASK<n>` | `GameTaskNode` (`IInteractable`, prompt "Do task") | Imposter. `n` is the task id (10–15 to match current zones, or 0–5 — pick one and make script + native agree). Calls `GameSession.TryCompleteTask()` only when phase is Playing, local role is Crew, alive, and this console has not already been used **by this player this round**. |
| `SERIKA_MEETING` | `GameMeetingNode` (prompt "Call meeting") | Imposter hub. Calls `TryReport()`. |
| `SERIKA_FINISH` | `GameFinishNode` (prompt optional; also auto-trigger on enter) | Gauntlet. Calls `TryFinish()` once. |
| `SERIKA_VENT<n>` | `GameVentNode` (prompt "Use vent") | Imposter. **Native teleport** between paired vents. Imposter-only in Playing; crew sees the grate but `CanInteract` is false. Pair by index (`n` and `n+10` or an exported `PairId`). |
| `SERIKA_KILL` is **not** a marker | Kill is a player-to-player action | See §5. |

Keep existing `SERIKA_ZONE*` for script feedback (glow when occupied, advisory emit). Native interactables and script zones can coexist on the same console: the zone tells the script "someone is here" (lights), the interactable is how a human completes the task.

Parent every script-movable mesh under its `SERIKA_SNODE`. A mesh sitting *next to* the empty is not the node the script moves — `build_rope_parkour` already documents this and then does it wrong (mesh and empty are siblings). Fix that pattern: **empty is parent, mesh is child.**

---

## 5. Native integration (this is the difference between a HUD and a game)

### 5.1 Lobby / start UI

Add a **Start game** control that is visible only when `_gameSession != null`.

- Place it on `game/UI/QuickMenu.cs` world-actions row (Esc / B-Y menu) **and** as a persistent in-world host panel in the hub/start-pad so VR players do not have to open the menu.
- Label by mode: "Start Imposter" / "Start Gauntlet". For Gauntlet, optional rounds stepper (1–10, default 3) matching `TryStart(mode, rounds)`.
- Enabled when: session exists, `Phase == Lobby` or `Ended`, local user is instance owner (the API already 403s `not_host` otherwise — still hide the button for non-hosts so they do not eat an error).
- Disabled with a reason when Imposter lobby has `< 4` or Gauntlet `< 2` (constants in `gamemode.ts`: `MIN_PLAYERS_IMPOSTER`, `MIN_PLAYERS_GAUNTLET`). Show `need` / `have` from the error body if the POST fails.
- On success, toast via existing `InWorldHud.Toast`. `GameHud` already reacts to `RoleRevealed`.
- Rope Parkour has no game API — start is "the team is in the world". Do not show a Start button there. Optionally a scripted countdown via `SCREEN_SET_TEXT` on a declared board.

Wire from `Main.cs` (it already owns `_gameSession` and `AddUi`). Do not have QuickMenu call the API itself — pass an `Action`/`Func` like every other QuickMenu button.

### 5.2 Kill (Imposter)

Kill is **not** an interactable on the map. It is an action on a nearby living crewmate.

- Desktop: when `GameSession.CanAttemptKill`, the existing `Interactor` scan also considers remote players within `KILL_MAX_DISTANCE` (3.0 m, `gamemode.ts`). Prompt `[E] Kill <name>`. Confirming calls `TryKill(targetUserId, distance)`.
- VR: same, driven by the hand interactor already in `Main.SetupVrInteractors`. Trigger is Use; do not steal Grip (that's grab).
- Targeting uses `_peerUserIds` in `Main.cs` + remote avatar positions. Never guess from nametags' role colour — the client does not have other people's roles.
- On success the victim becomes a ghost (see 5.5). The public event is `"died"` with the **victim** id only. Do not toast the killer.
- Cooldown is already on the wrist readout.

### 5.3 Report / meeting / task / finish

Implement the new `IInteractable` nodes in `game/World/` (or `game/Game/`) and resolve them from `WorldLoader.ResolveMarkers` exactly like `SeatNode`. Join `Interactable.Group` so `Interactor` finds them for free.

- Meeting button: anyone alive, phase Playing → `TryReport()`.
- Task console: crew, alive, Playing, standing at *this* console → `TryCompleteTask()`. After success, disable *this* console for this player (local flag is fine; server still clamps to `TASKS_PER_CREW`). Imposters get prompt "Sabotage" only if you also add a sabotage API — **do not fake it**. If sabotage is out of scope, imposters see no task prompt.
- Finish gate: on zone enter **or** interact, once, `TryFinish()`. The server assigns place; duplicates return `{duplicate:true}`.
- All prompts go through `IInteractPrompt` / `VrInteractLabel`. `InteractionPrompt` is chrome in VR (`AddUi(..., chrome: true)` already). Do not put a persistent "Kill" button on the VR panel.

### 5.4 Wire `on_interact` for scripts

When the player interacts with a `SERIKA_SNODE*` (or a small new `SERIKA_BUTTON<n>` marker), call `ScriptWorld.Interact(localPlayerIndex)` **in addition to** any native handler.

This is how a task console can both complete a server task (native) and pulse its light (script). Missing this is why the example scripts cannot react to a press today.

Local player index: `IScriptPlayers` already exists. Use it. Out-of-range is a no-op.

### 5.5 Ghosts, meeting freeze, gauntlet elimination

- **Ghost (Imposter death/eject):** local player can still move and see, but cannot `TryKill` / `TryReport` / `TryVote` / `TryCompleteTask` (server already refuses). Visual: drop opacity or a simple overlay — do **not** hide the whole mesh (mirrors + third person). Remote ghosts: semi-transparent, no kill target, still in the instance.
- **Meeting:** voting UI already exists. Additionally: lock locomotion for the meeting duration or pull a "meeting camera" only if you can do it without breaking VR (VR: do **not** steal the HMD; freeze locomotion only). Desktop may freeze WASD.
- **Gauntlet elimination:** falling off the course is **time loss, not death** (already the map design). Elimination happens when the host (or a timer) calls `POST /:id/end-round`. Add a host "Close round" button once at least one player has finished, and/or auto-close N seconds after the first finish. Losers become spectators for later rounds (`Alive = false`).
- Falling into the Gauntlet gap: native respawn at last checkpoint or start pad. **This is a teleport → native**, not script.

### 5.6 Host-only round controls

QuickMenu (host): Start / Close meeting (after clock) / Close round (Gauntlet) / Abort (maps to abandoned if you add an endpoint — otherwise leave the session to TTL). Do not let a non-host cut a meeting short; the API already enforces that.

---

## 6. Scripts that actually run the world

Rewrite the three examples. They must compile with:

```bash
bun tools/serikascript/sksc.ts tools/serikascript/examples/<name>.sks -o world.sskb --rank 8
bun test tools/serikascript/compiler.test.ts
dotnet test game/Script/Tests
```

Keep them first-party (`--rank 8` is VerifiedCreator). Do not wait on opening untrusted submission (gate 5 in `docs/serikascript.md`).

### Language recap (do not invent syntax)

See `tools/serikascript/examples/*.sks` and `tools/serikascript/compiler.ts`. Hooks: `on ready`, `on tick(dt)`, `on interact(player)`, `on enter_zone(player, zone)`, `on exit_zone`, `on message`. Host calls already in the examples: `log`, `emit`, `attach`, `playerCount`. Also available: `NODE_MOVE` / `NODE_ROTATE` / `NODE_SET_VISIBLE` / `NODE_PLAY_ANIM` / `SCREEN_SET_TEXT` / `SOUND_PLAY` / `TIME` / `RANDOM` / `VAR_GET`/`VAR_SET` (as `var`). Node slots are **literals** — you cannot index `SNODE[i]` in a loop. Unroll.

### Imposter Station script — required behaviour

- `on ready`: hide vent interiors if you have cover meshes; set meeting button idle.
- `on enter_zone` for task zones 10–15: `NODE_SET_VISIBLE` or rotate a "console on" indicator on the matching `SERIKA_SNODE`. `emit` remains advisory.
- `on interact` at meeting node: pulse the hub light (`NODE_SET_VISIBLE` flicker or `SHADER_SET_FLOAT` if the material is declared and rank allows — first-party is rank 8).
- `on tick`: gentle meeting-button bob (`NODE_MOVE` / `NODE_ROTATE`) so the interactable is findable.
- Do **not** try to know roles. Do **not** call anything like kill. If you need "imposter-only vent open", that visibility change must be native (the script cannot be told who is imposter without leaking it to every client — **every visitor runs the same bytecode**).

Critical implication: **any visual that differs for imposter vs crew is native C#, keyed off `GameSession.MyRole`, not the script.** Vent grates that only imposters can use: native `CanInteract`. An imposter-only extra overlay: native. The script shows the same world to everyone.

### Gauntlet Course script — required behaviour

This is the world where script should do the most.

Declare SNODE slots for every moving hazard. In `on tick(dt)`:

- Band 1: rotate slalom gates (`NODE_ROTATE` accumulating yaw).
- Band 2: bob stepping platforms vertically so the gap is a timing challenge, not just a jump.
- Band 3: spin the beams (`NODE_ROTATE` on X or Y).
- Finish: when `on enter_zone` for the finish zone, flash the arch (`NODE_SET_VISIBLE` / colour via shader if available) and `emit(5, elapsed)` as today.
- Optional `SCREEN_SET_TEXT` on a start-line board: elapsed time.

The native finish node still reports `TryFinish()` — the script never assigns place.

If a platform must carry a player (they stand on it and it moves), **script cannot do that** (no player write). Either (a) the platform is visual and collision is static, or (b) you add a **native** moving platform in C# (kinematic body). Prefer (a) for v1 of this prompt unless you already have a kinematic mover; a platform that slides out from under you without carrying you is still a real hazard.

### Rope Parkour script — required behaviour

Keep attach-on-ready (slots 0–3, hips). Fix the parent/child marker bug so the rope-end mesh actually follows the player.

- Checkpoints 1–4: on enter, `NODE_SET_VISIBLE` a beacon on that tier and `emit(1, zone)` as now. Progress must still only move forward.
- Summit: `emit(2, elapsed)` + board text.
- Native `RopeTeam` stays the physics. Do not ask for a host call that pulls players.

### Compiler / golden

If you change `rope_parkour.sks` behaviour, regenerate golden:

```bash
bun tools/serikascript/gen-golden.ts
dotnet test game/Script/Tests
```

The golden corpus is the same idea as proto: one side produces, the other consumes. Do not hand-edit `.sskb`.

---

## 7. Map design requirements (even with sourced kits)

### Imposter Station

A hidden-role game is an **alibi** game. The layout has to make alibis contestable.

- Named rooms, at least 6, fully enclosed with doorways (not open pads facing a hub). Suggested: Reactor, Medbay, Comms, Storage, Galley, Engines, plus a Hub.
- **Two routes** between any two important rooms (ring + at least one cross corridor or vent pair). Without a second route, every alibi is "who was behind me."
- Hub in the visual centre with a meeting table and a clearly-lit meeting button.
- Vents are **visible** to crew (so they can learn routes) but only imposters can use them (native).
- Task consoles are unique silhouettes, glow when idle, sit against a wall so the interact prompt has a focus point.
- Lighting is interior (`manifest.lighting = "lit"` or `"dark"`), not a sun in a void.
- Cap 10, spawn in hub or a clearly marked arrival bay.

### Gauntlet Course

A race that also kills is two punishments for one mistake. Falling costs time.

- 100–150 m, three **different** hazard bands, lanes that widen.
- Start pad wide enough for 16. Finish arch unmissable, zone covering the whole gate.
- Checkpoints (native respawn pads) at the start of each band so a fall is not a full restart.
- Saturated, readable colours (brand purple hue 262 as accent, not the whole palette).
- Moving hazards must be telegraphed (silhouettes, lights) — VR players cannot hear a whoosh they have not been taught.

### Rope Parkour

- Spiral or stacked tower, ~25–35 m high, four checkpoint ledges wide enough for two avatars (~4 m).
- Falls land on a solid plinth, not the void.
- Summit is obvious. Rope-end props start on the start pad and are SNODE 0–3 **parents** of their meshes.
- Cap 4 matches `RopeTeam` + the unrolled attach in the script.

---

## 8. Server tweaks allowed (small, tested)

Stay inside `gamemode.ts` + `routes/games.ts` + `gamemode.test.ts`. Do not move rules into Redis-only code.

Allowed if needed:

- Reject `POST /task` unless the request includes a `taskId` the server recognises, and each living crew can complete each id at most once. This stops "press anything 5 times in spawn." Keep the kill-distance posture: the id is a **sanity** check, not a cryptographic proof of location (positions are still client-reported).
- Gauntlet: auto-`end-round` N seconds after first finish if the host never clicks. Put the delay in `gamemode.ts` as a named constant.
- Imposter: optional emergency-meeting cooldown so one player cannot stunlock the lobby. Named constant + test.

Not allowed:

- An endpoint that returns `{ roles: ... }` while `phase !== Ended`.
- Trusting script `emit()` for kills, votes, or places.
- Moving win evaluation out of `evaluateImposter` / `evaluateGauntlet`.

---

## 9. Verification (do not declare done on a screenshot)

### Automated

```bash
cd server && bun test api/src/gamemode.test.ts          # existing 30 + your new cases
cd game && dotnet test Script/Tests                     # allowlist + VM + golden
cd game && dotnet test Net/Codec/Tests                  # must stay green; you should not have touched proto
cd game && dotnet build
bun tools/serikascript/sksc.ts tools/serikascript/examples/imposter_station.sks -o /tmp/i.sskb --rank 8
bun tools/serikascript/sksc.ts tools/serikascript/examples/gauntlet_course.sks -o /tmp/g.sskb --rank 8
bun tools/serikascript/sksc.ts tools/serikascript/examples/rope_parkour.sks -o /tmp/r.sskb --rank 8

GODOT=/media/pikachubolk/63d7930c-4cfb-4c68-96a6-879048200e36/Documents/Godot/Godot_v4.7.1-stable_mono_linux_x86_64/Godot_v4.7.1-stable_mono_linux.x86_64
$GODOT --headless --path game -- --serika-scripttest    # real bridge, zones, attach
```

### In-engine (needs a display for maps; headless for logic)

- `--serika-worldtest --world <absolute .serikaworld>` for each rebuilt bundle: spawn, marker counts, collision.
- Manual play loop, **this is the acceptance test**:

**Imposter (4+ clients or 1 host + dummy roster only proves the API — you need at least 2 real clients for kill):**

1. Host opens QuickMenu, Start is disabled under 4 players, enabled at 4.
2. Start → every player sees only **their** role toast. Crew HUD shows tasks; imposter HUD shows kill cooldown, never a crewmate's role.
3. Crew walks to a console, `[E] Do task` increments *their* tasks. Doing it in spawn does nothing useful (taskId rejected or no interactable).
4. Imposter walks up to a crewmate, `[E] Kill` → victim ghosts, room gets `"X was found dead"` without naming the killer.
5. Anyone alive `[E] Call meeting` → voting layer appears (the **layer**, not just its children), skip + names, tie ejects nobody, eject reveals whether they were imposter.
6. Parity (equal living imposters and crew) ends ImposterWin; all tasks by living crew ends CrewWin.
7. VR: readout on wrist, voting on panel, no stuck laser when no meeting.

**Gauntlet:**

1. Start with 2+. Run the course. Moving hazards actually move.
2. Cross the gate → place 1. Second player place 2. Duplicate finish does not steal place 1.
3. Host closes round (or auto-timer) → slowest out, next round or winner.

**Rope:**

1. Two players spawn, rope meshes follow hips, taut at `rope.length`.
2. Checkpoint beacons light in order. Summit emit once.

### Playable definition (all must be true)

A stranger can join the public world, be told how to start, perform every verb with `[E]` / trigger, and reach a win screen **without opening a debug console or calling curl**. If they cannot, it is not a game yet.

---

## 10. Suggested implementation order (vertical slices)

Keep each slice green and committable per-repo.

1. **Native verbs + start UI on the existing greybox maps.** Start, kill, task, report, vote, finish all work. This slice alone makes v1.10.0 *playable*. Ship it even if the maps are still boxes.
2. **Wire `on_interact` + rewrite scripts** so consoles glow, gates rotate, checkpoints beacon. Greybox but alive.
3. **Source kits, convert, place markers, replace geometry.** Keep the same marker ids so scripts do not churn.
4. **Vents, ghosts, gauntlet respawn, auto end-round.** Native-only polish.
5. **Rebuild bundles, publish in place, write SOURCES.md, update `docs/serikascript.md` caveats.**

Do not start with a week of modelling. A beautiful unplayable map is how we got here.

---

## 11. Out of scope

- Opening SerikaScript submission to untrusted creators (gate 5).
- New proto messages for game state (HTTP poll stays).
- Player-transform host calls.
- Among Us / Fall Guys IP (art, names, UI clones of their HUD).
- iOS export (preset exists; needs macOS).
- P2P / voice / sabotage minigames / impostor vision / cameras / admin map.
- Rewriting `gamemode.ts` into the relay.
- Committing `game/Script/Tests/bin` or other build artifacts. Nested `bin/` is gitignored for a reason.

---

## 12. Files you will almost certainly touch

**game (repo `SerikaSocial/game`)**

- `Game/GameSession.cs`, `Game/GameHud.cs` — already the client cache + HUD
- `Main.cs` — owns session, `AddUi`, interactors, peer user ids
- `UI/QuickMenu.cs` — start / close-round buttons
- `UI/Brand.cs` — style only, no one-off colours
- `World/WorldLoader.cs` — new markers
- `World/InteractionNodes.cs`, `World/Interactor.cs` — new interactables + player-as-target
- `Script/ScriptWorld.cs` — already has `Interact`; call it
- `Player/RopeTeam.cs` — leave physics; fix marker parenting on the world side

**server (repo `SerikaSocial/server`)**

- `api/src/gamemode.ts`, `gamemode.test.ts`, `api/src/routes/games.ts` — small rule additions only
- `api/src/publish-scripted-world.ts` — use, do not bypass

**tools (repo `SerikaSocial/tools`)**

- `serikascript/examples/*.sks` — rewrite
- `build_worlds.py` — fallback greybox + marker layout documentation
- `convert_world.py` / `blender_convert.py` — conversion
- `worlds/<name>/SOURCES.md` — new

**docs (repo `SerikaSocial/docs`)**

- `serikascript.md` — replace the "scripts are coordinators only" examples with what the scripts now actually do
- this file, if the contract changes

---

## 13. Definition of done

- [ ] A player can start Imposter and Gauntlet from in-world UI without curl.
- [ ] Kill, report, task, vote, finish are all in-world verbs (E / trigger / VR panel).
- [ ] Scripts move hazards / lights / beacons; they still never assign roles or places.
- [ ] Maps are sourced (or a clearly designed non-box fallback) with `SOURCES.md` and legal licenses.
- [ ] Marker parenting is correct (SNODE is parent of its mesh).
- [ ] `gamemode` tests + Script.Tests + codec tests + `--serika-scripttest` pass.
- [ ] No role map on the wire while live. No new player-write host call. No proto change unless you can justify it in the PR.
- [ ] Bundles republished onto the existing world ids. Per-repo commits.

If you skip the sourced maps, say so explicitly and still ship slice 1+2. A playable greybox is a game. A pretty lobby is not.
