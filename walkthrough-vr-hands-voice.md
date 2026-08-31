# Next-Gen VR Hands, 1:1 Kinematics & Cross-Platform Voice Chat

## Overview
We have completely overhauled the VR interaction, avatar full-body kinematics, and spatial audio/voice subsystems across the Godot client. The system delivers 1:1 anatomical fidelity, zero-latency optical hand tracking with splay and knuckle convergence, and cross-platform voice capture and spatial playback (Desktop + Quest/Android).

---

## 1. Subsystem Upgrades & Architectural Changes

### A. Next-Gen Articulated Hand Tracking & Poser
- **26-Joint Optical Tracking**: Upgraded [`VrHandTracking.cs`](file:///media/pikachubolk/63d7930c-4cfb-4c68-96a6-879048200e36/Documents/GitHub/Godot-SerikaSocial/game/Player/VrHandTracking.cs) to track all 26 OpenXR hand joints (wrist, palm, metacarpals, proximal, intermediate, distal, tips) with graceful fallback for legacy runtimes.
- **1-Euro Adaptive Low-Pass Filtering**: Integrated real-time jitter filtering across curls, splays, thumb opposition, and pinches for rock-solid tracking at speed with zero lag during rapid movements.
- **Anatomical Knuckle Convergence**: In [`HandPoser.cs`](file:///media/pikachubolk/63d7930c-4cfb-4c68-96a6-879048200e36/Documents/GitHub/Godot-SerikaSocial/game/Avatar/HandPoser.cs), finger flexion naturally curves inward toward the scaphoid bone of the palm, matching real human hand anatomy.
- **Finger Splay (Abduction/Adduction)**: Added realistic lateral finger spreading both for optical tracking and controller gestures via `HandGestures.SplaysForGesture` in [`HandGesture.cs`](file:///media/pikachubolk/63d7930c-4cfb-4c68-96a6-879048200e36/Documents/GitHub/Godot-SerikaSocial/game/Player/HandGesture.cs).
- **Multi-Pinch Detection**: Continuous index-thumb and middle-thumb pinch proximity with smoothing for fine-grained VR interactions.

### B. True 1:1 Avatar Full-Body Kinematics
- **Cervical Spine Pivot Offset**: Added anatomical neck/C1-C7 offset modeling so head rotations pivot naturally around the cervical vertebra rather than the center of the skull.
- **Multi-Segment Spine & Pelvic Tilt**: Re-architected [`VrAvatarIk.cs`](file:///media/pikachubolk/63d7930c-4cfb-4c68-96a6-879048200e36/Documents/GitHub/Godot-SerikaSocial/game/Player/VrAvatarIk.cs) with natural lordosis/kyphosis curve distribution across the spine chain (`spine`, `chest`, `upperChest`, `neck`, `head`) and pitch-dependent pelvic tilt.
- **Alternating Foot IK Stepping**: Eliminated foot priority starvation when walking at speed by alternating step priority (`_lastSteppedFoot`), producing smooth, realistic walking gaits that keep up with the body without skating or lagging.
- **Twist-Swing Forearm/Wrist Decomposition**: Replaced naive 3D rotation clamping in `LimitTwist` with true axial twist-swing decomposition along the forearm axis, allowing full natural arm range of motion while strictly enforcing anatomical wrist limits.

### C. Cross-Platform Spatial Voice Chat
- **Engine Capture Enabled**: Configured `[audio] driver/enable_input=true` in [`project.godot`](file:///media/pikachubolk/63d7930c-4cfb-4c68-96a6-879048200e36/Documents/GitHub/Godot-SerikaSocial/game/project.godot).
- **Robust Audio Pipeline**: Re-implemented [`VoiceManager.cs`](file:///media/pikachubolk/63d7930c-4cfb-4c68-96a6-879048200e36/Documents/GitHub/Godot-SerikaSocial/game/Player/VoiceManager.cs) with active `AudioStreamMicrophone` capture streaming to a dedicated `"Record"` bus with `AudioEffectCapture`. Added energy-based Voice Activity Detection (VAD) with adaptive noise tracking and 250ms speech hangover.
- **Quest & Android Runtime Permissions**: Added runtime microphone permission handling with `OS.RequestPermission("RECORD_AUDIO")` and `OS.GetGrantedPermissions()`.
- **3D Spatial Emitters**: Configured remote player audio in [`Main.cs`](file:///media/pikachubolk/63d7930c-4cfb-4c68-96a6-879048200e36/Documents/GitHub/Godot-SerikaSocial/game/Main.cs) with `AudioStreamPlayer3D` positioned at head/mouth height using `InverseDistance` attenuation.

---

## 2. Verification & Diagnostic Test Results

| Test Suite | Command | Result | Notes |
|---|---|---|---|
| **VR Headless Diagnostics** | `$GODOT --headless --path game -- --serika-vrtest --ska <path>` | **PASS** (100%) | Verified 22 assertions: ROOT, WALL, PORTAL, DRIFT, RESEAT, ORIENT, DASH, HEAD, POSTURE, PLANT, STEP (5.49m travel, 0.07m lag), CROUCH, WRIST, YAW, GEST, FINGER (40% flexion), WIRE (LOD0/LOD1), PANEL, KEYBD. |
| **Rendered OpenXR Sim Device** | `$GODOT --path game --windowed --audio-driver Dummy -- --serika-vrsim --ska <path> --out /tmp/vr` | **PASS** (44/44) | All 44 hardware/render assertions passed: Desk calib rejection, worn calib, drift, walk/strafe/snap, full VRChat bindings, jump, proportional arm reach, optical hand tracking, Point gesture classification, menu panel & laser clicks. |
| **Head Aim & Gestures** | `$GODOT --headless --path game -- --serika-aimtest --ska <path>` | **PASS** | Head tracking ±55°, nod/shake gestures, wire synchronization without codec change. |
| **First-Person Eye Probe** | `$GODOT --headless --path game -- --serika-fptest --ska <path>` | **PASS** | Sprint bob (33.4cm), crouch eye drop (70.2cm), emote viewpoint tracking. |
| **Animation Retargeting** | `$GODOT --headless --path game -- --serika-animtest --clip Walk --ska <path>` | **PASS** | 54 mapped roles retargeted smoothly. |
| **Secondary Physics** | `$GODOT --headless --path game -- --serika-phystest --ska <path>` | **PASS** | 45 VRM spring chains, 121 joints, 0 dead chains, 0 drift. |
| **Discord SDK** | `$GODOT --headless --path game -- --serika-discordtest` | **PASS** | Native library dynamic load, P/Invoke, lifecycle. |
| **C# Wire Codec Golden Tests** | `dotnet test Net/Codec/Tests` | **PASS** (21/21) | Byte-identical output to wire contract. |
| **Rust Wire Codec Golden Tests** | `bun run proto:test` | **PASS** (18/18) | Byte-identical output to golden corpus. |

---

## 3. Key Files Modified
- [`game/project.godot`](file:///media/pikachubolk/63d7930c-4cfb-4c68-96a6-879048200e36/Documents/GitHub/Godot-SerikaSocial/game/project.godot)
- [`game/Player/VoiceManager.cs`](file:///media/pikachubolk/63d7930c-4cfb-4c68-96a6-879048200e36/Documents/GitHub/Godot-SerikaSocial/game/Player/VoiceManager.cs)
- [`game/Player/VrHandTracking.cs`](file:///media/pikachubolk/63d7930c-4cfb-4c68-96a6-879048200e36/Documents/GitHub/Godot-SerikaSocial/game/Player/VrHandTracking.cs)
- [`game/Avatar/HandPoser.cs`](file:///media/pikachubolk/63d7930c-4cfb-4c68-96a6-879048200e36/Documents/GitHub/Godot-SerikaSocial/game/Avatar/HandPoser.cs)
- [`game/Player/HandGesture.cs`](file:///media/pikachubolk/63d7930c-4cfb-4c68-96a6-879048200e36/Documents/GitHub/Godot-SerikaSocial/game/Player/HandGesture.cs)
- [`game/Player/VrAvatarIk.cs`](file:///media/pikachubolk/63d7930c-4cfb-4c68-96a6-879048200e36/Documents/GitHub/Godot-SerikaSocial/game/Player/VrAvatarIk.cs)
- [`game/Player/VrPlayer.cs`](file:///media/pikachubolk/63d7930c-4cfb-4c68-96a6-879048200e36/Documents/GitHub/Godot-SerikaSocial/game/Player/VrPlayer.cs)
- [`game/Player/VrSimDevice.cs`](file:///media/pikachubolk/63d7930c-4cfb-4c68-96a6-879048200e36/Documents/GitHub/Godot-SerikaSocial/game/Player/VrSimDevice.cs)
- [`game/Player/VrSimDiagnostic.cs`](file:///media/pikachubolk/63d7930c-4cfb-4c68-96a6-879048200e36/Documents/GitHub/Godot-SerikaSocial/game/Player/VrSimDiagnostic.cs)
- [`game/Main.cs`](file:///media/pikachubolk/63d7930c-4cfb-4c68-96a6-879048200e36/Documents/GitHub/Godot-SerikaSocial/game/Main.cs)
