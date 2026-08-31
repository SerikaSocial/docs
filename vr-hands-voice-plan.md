# Next-Gen VR Hand Tracking, 1:1 Body Movement, and Full Spatial Voice Chat

Deliver a complete overhaul of:
1. **Articulated Hand Tracking & Movement System**: 26 OpenXR joints, 3D finger splay (spreading fingers), 3D thumb opposition, 1-Euro jitter filtering, anatomical knuckle convergence, and smooth gesture dynamics.
2. **True 1:1 VR Immersion**: Anatomical cervical neck model, proportional play-space calibration, multi-segment spine & pelvic articulation.
3. **Full Spatial Voice Chat (Desktop + Quest/Android)**: Engine mic input configuration, `AudioStreamMicrophone` capture pipeline, Voice Activity Detection (VAD), Android `RECORD_AUDIO` runtime permissions, and spatial 3D mouth emitters.

---

## User Review Required

> [!IMPORTANT]
> - **Godot Audio Input Setting**: `project.godot` will be updated to include `audio/driver/enable_input=true`. This enables Godot's audio driver microphone capture subsystem.
> - **Wire Codec Compatibility**: Voice packets continue to use the established `VoiceFrame` contract (`0x06 [Seq][Rms][LenHi,LenLo][PCM16]`), ensuring 100% byte-compatibility with the Rust relay (`server/instanced`) and golden test vectors.
> - **Android / Quest Permissions**: On Android/Quest, `OS.RequestPermission("RECORD_AUDIO")` will prompt the user on startup/join so mic access is cleanly granted.

---

## Proposed Changes

### 1. Project Configuration & Permissions

#### [MODIFY] [project.godot](file:///media/pikachubolk/63d7930c-4cfb-4c68-96a6-879048200e36/Documents/GitHub/Godot-SerikaSocial/game/project.godot)
- Add `[audio]` section with `driver/enable_input=true`.

---

### 2. Voice Chat System

#### [MODIFY] [VoiceManager.cs](file:///media/pikachubolk/63d7930c-4cfb-4c68-96a6-879048200e36/Documents/GitHub/Godot-SerikaSocial/game/Player/VoiceManager.cs)
- Instantiate and hold an `AudioStreamPlayer` streaming `AudioStreamMicrophone` connected to the `"Record"` bus.
- Configure `"Record"` audio bus with `AudioEffectCapture` and route send properly to prevent self-echo.
- Implement Voice Activity Detection (VAD) / Noise Gate with adaptive noise floor and ~200ms speech hangover to prevent network flooding during silence.
- Add Android/Quest `OS.RequestPermission("RECORD_AUDIO")` check and request on startup.
- Implement jitter buffering and audio smoothing in `AudioStreamGeneratorPlayback`.

#### [MODIFY] [Main.cs](file:///media/pikachubolk/63d7930c-4cfb-4c68-96a6-879048200e36/Documents/GitHub/Godot-SerikaSocial/game/Main.cs)
- Update `OnVoiceReceived` to position the remote `AudioStreamPlayer3D` at the avatar's mouth/head position (`Height * 0.92f`) with realistic 3D spatial attenuation.
- Ensure microphone mute toggles (Desktop `T`/Mute, VR Left `X`) seamlessly control voice capture and HUD/wrist feedback.

---

### 3. Hand System & Optical Hand Tracking

#### [MODIFY] [VrHandTracking.cs](file:///media/pikachubolk/63d7930c-4cfb-4c68-96a6-879048200e36/Documents/GitHub/Godot-SerikaSocial/game/Player/VrHandTracking.cs)
- Track all 26 standard OpenXR hand joints (wrist, palm, metacarpals, proximals, intermediates, distals, tips).
- Compute per-finger **Splay** (lateral spread / adduction & abduction) in addition to **Curl**.
- Compute full **Thumb 3D Rotation** (opposition across palm + flexion).
- Implement a **1-Euro Low-Pass Filter** (adaptive cutoff frequency based on movement speed) to eliminate micro-jitter while preserving sub-millisecond response for fast motions.
- Support multi-finger pinch detection (Thumb-Index for UI pointing, Thumb-Middle for precision grip, full hand for power grip).

#### [MODIFY] [HandPoser.cs](file:///media/pikachubolk/63d7930c-4cfb-4c68-96a6-879048200e36/Documents/GitHub/Godot-SerikaSocial/game/Avatar/HandPoser.cs)
- Extend `HandPoser` to accept full finger pose parameters (Flexion per phalanx, Splay / abduction per finger, Thumb 3D rotation).
- Implement anatomical knuckle convergence: fingers naturally converge toward the palm center during flexion instead of bending along parallel tracks.
- Preserve rest pose calibration and ensure smooth bone rotation transforms.

#### [MODIFY] [HandGesture.cs](file:///media/pikachubolk/63d7930c-4cfb-4c68-96a6-879048200e36/Documents/GitHub/Godot-SerikaSocial/game/Player/HandGesture.cs)
- Update gesture curl generators and classification to support natural splay and thumb opposition.
- Implement smooth cubic/hermite gesture morphing with realistic finger mass dynamics.

---

### 4. VR Immersion & 1:1 Scale

#### [MODIFY] [VrPlayer.cs](file:///media/pikachubolk/63d7930c-4cfb-4c68-96a6-879048200e36/Documents/GitHub/Godot-SerikaSocial/game/Player/VrPlayer.cs)
- Integrate full 26-joint hand tracking with splay and 1-Euro filtered curls into `UpdateHands`.
- Enhance play-space height and reach calibration to match player physical proportions 1:1 with avatar dimensions.
- Connect wrist, splay, and gesture state to `_poser` and remote pose broadcasting.

#### [MODIFY] [VrAvatarIk.cs](file:///media/pikachubolk/63d7930c-4cfb-4c68-96a6-879048200e36/Documents/GitHub/Godot-SerikaSocial/game/Player/VrAvatarIk.cs)
- Incorporate anatomical cervical spine neck model (offset from eye point to C1-C7 pivot).
- Implement distributed multi-segment spine bending across spine and chest bones.

---

## Verification Plan

### Automated Diagnostics & Codec Suites
1. `dotnet build` in `game/` - Verify 0 errors and 0 warnings.
2. `dotnet test Net/Codec/Tests` - Golden codec tests (must pass 21/21 byte-identically).
3. `$GODOT --headless --path game -- --serika-vrtest --ska "$SKA"` - Headless VR suite across all 4 cached avatar rigs (verify all checks pass).
4. `$GODOT --path game --windowed --audio-driver Dummy -- --serika-vrsim --ska "$SKA" --out /tmp/vrsim_run` - Rendered OpenXR simulation harness (verify all 47+ assertions pass).
5. Headless retargeting & physics suites: `--serika-aimtest`, `--serika-fptest`, `--serika-animtest`, `--serika-phystest`, `--serika-discordtest`.

### Voice & Hand System Verification
- Verify `AudioStreamMicrophone` and `AudioEffectCapture` start cleanly without exception.
- Verify Voice Activity Detection triggers when audio is supplied and settles into silence.
- Verify `AudioStreamGeneratorPlayback` plays back received voice frames smoothly with spatial positioning at the avatar's mouth.
