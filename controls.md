# Controls

Serika Social's controls are deliberately VRChat-compatible — social VR players arrive
with the bindings in their hands, and every deviation is something to unlearn while
wearing a headset.

## Desktop

| Input | Action |
|---|---|
| WASD | Move |
| Mouse | Look |
| Shift | Sprint |
| Ctrl | Crouch |
| Space | Jump |
| **V** | Toggle first / third person |
| **T** | Chat |
| **Esc** | Pause menu |

The pause menu (`UI/PauseMenu.cs`) hosts world actions: respawn, camera toggle, emotes,
copy invite link, and **Change avatar…** → the in-world avatar selector
(`UI/AvatarSelector.cs`).

## VR (OpenXR)

VRChat-compatible bindings by design.

| Input | Action |
|---|---|
| Left stick | Locomote |
| Right stick | Turn (snap by default) |
| A (right) | Jump |
| X (left) | Mute |
| B / Y (either) | Quick menu on a tap, action menu on a hold (0.35 s) |
| Stick click | Action menu |
| Grip | Pick up |
| Trigger | Use / interact |
| Both menu buttons, held 1 s | Recentre |

`DeviceProfile.Settings.VrMoveOnRightStick` swaps the sticks; it defaults to **false**
(movement on the left), matching VRChat.

### Controller gestures

Controller gestures are a **lookup table, not a measurement.** A controller has three
signals — thumb rest, trigger, grip — and the seven VRChat shapes plus Neutral fill
those eight combinations exactly. Classifying a controller by synthesising finger
curls and measuring them *cannot* reach Peace or Rock'n'Roll (one grip axis moves
middle, ring and little together). Optical hand tracking goes the other way — the
camera sees every finger, so the shape is measured — and both routes then pose the
avatar's fingers from `HandGestures.CurlsForGesture`, blended over ~90 ms.

### Arm scaling

Avatar arms are scaled to the player's reach (`VrAvatarIk.ScaleToArm`,
`DeviceProfile.Settings.VrArmScaling`). A 1.57 m stylised rig measures ~43 cm shoulder
to wrist against an adult's ~60 cm, so feeding raw controller positions to a two-bone
solver leaves the hands permanently ~15 cm short. Direction is preserved exactly and
distance is scaled by the arm-length ratio, so full player extension is full avatar
extension.

### VR chrome vs menus

VR chrome goes to the wrist; menus go to the panel. `VrUiSurface.HasInteractiveUi`
gates both the panel's render and the laser pointer on whether any *layer* on the panel
is visible. A layer that hides an inner control but never hides itself pins a slab and
a laser in the player's face for the whole session — `InteractionPrompt` and
`ChatOverlay` are mounted with `AddUi(..., chrome: true)` to avoid this. Route
persistent readouts to the wrist by rule.

## Mobile

Touch controls live in `UI/TouchControls.cs`. Quest and Mobile Android exports exist
(`export_presets.cfg`).

## Keybinds

Keybinds are rebindable; `--serika-keytest` exercises rebind, conflict detection,
persistence, and `InputMap` push. See [`diagnostics.md`](diagnostics.md).
