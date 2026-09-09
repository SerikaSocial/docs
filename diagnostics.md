# Diagnostics

The game ships a catalogue of headless (and a few rendered) diagnostic harnesses. They
verify behaviour **numerically**, never by screenshot. Run them from the `game`
directory with the Godot mono binary (`$GODOT`).

> Pass a real `.ska` where shown. Many avatars carry only the 22 body roles and no
> finger bones, so finger checks report SKIPPED, which is not a pass. Prefer a
> VRM-sourced `.ska` (`sourceFormat=vrm0`) for VR/finger tests — PMX avatars have wrong
> animations and some carry no finger bones.

## Headless diagnostics

```bash
$GODOT --headless --path game -- --serika-aimtest  --ska <path> [--clip Walk]   # head aim + nod/shake
$GODOT --headless --path game -- --serika-vrtest   --ska <path>                 # VrPlayer, no HMD needed
$GODOT --headless --path game -- --serika-fptest   --ska <path>                 # first-person eye probe
$GODOT --headless --path game -- --serika-animtest --clip <Clip> --ska <path>   # retargeting + crouch regression
$GODOT --headless --path game -- --serika-phystest --ska <path>                  # secondary physics
$GODOT --headless --path game -- --serika-keytest                               # keybinds: rebind, conflict, persist
$GODOT --headless --path game -- --serika-voicetest                             # voice: resample, pitch, VAD, codec
$GODOT --headless --path game -- --serika-discordtest [--wait 8]                 # Discord SDK lifecycle
```

### `--serika-vrtest` (11 checks)

ROOT, SPRINT, JUMP, STICK, WALL, plus ORIENT (hand-relative movement follows the
controller, not gaze), DASH (right stick teleports, diagonals turn), GEST (seven
gesture shapes + dead-space hold), FINGER (a curl folds the fingertip toward the palm),
and PANEL (HUD chrome must not make the menu panel — and so the laser pointer —
appear). It instantiates a real `VrPlayer` and injects headset poses and button states.
It does **not** cover bindings, comfort, or tracking quality — those need a Quest
build on a real headset.

### `--serika-animtest`

Retargeting; also asserts the full-pipeline crouch: the pelvis must **stay dropped**
through `Animate(crouching)` (ground leg-IK regression; runs from a physics frame
because the IK raycasts only do anything inside one).

### `--serika-discordtest`

Verifies library load, marshal/free, full client lifecycle, and — where a real Discord
client is logged in — a live presence push.

## VR testing without a headset

Two complementary harnesses, and they are not redundant:

```bash
# Logic net, headless, fast. Poses VrPlayer's nodes directly, injects buttons via VrTestInput.
$GODOT --headless --path game -- --serika-vrtest --ska <path>

# Rendered, driven by a SIMULATED OPENXR DEVICE. Needs a display (DISPLAY=:1 or xvfb-run).
$GODOT --path game --windowed --audio-driver Dummy -- --serika-vrsim --ska <path> --out /tmp/vr
```

`--serika-vrsim` (`Player/VrSimDiagnostic.cs` + `Player/VrSimDevice.cs`) registers real
`XRServer` trackers under the real OpenXR names (`head`, `left_hand`, `right_hand`,
`/user/hand_tracker/{left,right}`), so `XRCamera3D` binds to the head tracker,
`XRController3D` reads inputs by action name, and `VrHandTracking` finds actual
`XRHandTracker`s. **Everything downstream of tracking is the shipping code path, and
`VrPlayer` contains no test-only branch for it.** Because the headset camera is the
scene's active camera in mono, every phase writes a PNG from the player's own viewpoint
plus an observer shot from across the room.

44 assertions across: head tracking, idle drift, walk/strafe/snap-turn, grab and
release, the full VRChat button layout, proportional arm IK, all eight controller
gestures, optical hand tracking engaging and disengaging, and the menu panel's
visibility/ray/click routing.

Use a **VRM-sourced `.ska`, not a PMX one**. Check `sourceFormat` in the `.ska` header
(`vrm0` good, `pmx` not).

## Mirror tests (need a real display)

Mirrors render nothing headless — use `DISPLAY=:1` or `xvfb-run`:

```bash
$GODOT --path game --windowed --audio-driver Dummy -- --serika-mirrortest  --ska <path> --out /tmp/m
$GODOT --path game --windowed --audio-driver Dummy -- --serika-mirrorworld --world <bundle> --ska <path>
```

## Headless smoke test

Against a live relay (needs a valid join ticket — see
[`tools/mint-ticket.ts`](https://github.com/SerikaSocial/tools)):

```bash
$GODOT --headless -- --serika-smoke --endpoint <host:port> --ticket <token>
```

## Codec tests

```bash
# game
dotnet test Net/Codec/Tests    # C# golden tests (must match Rust) + transport liveness
# server
cargo test -p serika-proto    # Rust golden tests
cargo test -p instanced       # relay integration tests (needs Redis)
```

## Gotchas

- **OpenXR startup modal is disabled at the source** (`project.godot`:
  `xr/openxr/startup_alert=false`). On a machine with no HMD the runtime can't be
  reached every launch, and `--headless` doesn't suppress the blocking `zenity` alert —
  the run hangs at 0% CPU with no output. Do not re-enable it.
- **Redirect to a file, don't pipe to `grep`** when debugging a hang — killing a pipe
  swallows the buffer and you see nothing.
- **Unsetting `DISPLAY`** was the old workaround for the OpenXR alert; it's no longer
  required and rendered diagnostics can't use it anyway.
