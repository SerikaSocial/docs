# Implementation Plan: Insane Next-Gen VR & 5-Model Validation Suite

We are elevating the VR avatar experience to next-generation fidelity (matching and exceeding VRChat / Resonite / Half-Life: Alyx standards), accompanied by an automated multi-model test harness across 5 distinct avatar rigs.

---

## Proposed Changes

### 1. Natural Gaze Synthesis, Micro-Saccades & Organic Blinking
Avatars without gaze motion look lifeless. We will implement realistic ocular dynamics in [`AvatarInstance.cs`](file:///media/pikachubolk/63d7930c-4cfb-4c68-96a6-879048200e36/Documents/GitHub/Godot-SerikaSocial/game/Avatar/AvatarInstance.cs) and [`VrPlayer.cs`](file:///media/pikachubolk/63d7930c-4cfb-4c68-96a6-879048200e36/Documents/GitHub/Godot-SerikaSocial/game/Player/VrPlayer.cs):
- **Poisson Blinking**: Natural spontaneous blinking model (inter-blink intervals averaging 3.5s with realistic blink velocity curve: 60ms closure, 120ms opening).
- **Micro-Saccades & Fixation Drift**: Physiological miniature saccadic eye jumps (1-3° amplitude every 0.8–2.0s) layered onto head motion so eyes naturally fixate and explore rather than staying frozen.
- **Eye Bone & Morph Target Driving**: Automatically maps gaze angles and blinks to `leftEye`/`rightEye` bones and VRM expression morphs (`blink`, `blink_l`, `blink_r`, `look_up`, `look_down`, `look_left`, `look_right`).

---

### 2. Mic-Driven Real-Time Audio Lip Sync (Visemes)
- **Local & Remote Viseme Synthesizer**: Connect `VoiceManager` audio capture RMS and low/mid/high frequency formants in [`VoiceManager.cs`](file:///media/pikachubolk/63d7930c-4cfb-4c68-96a6-879048200e36/Documents/GitHub/Godot-SerikaSocial/game/Player/VoiceManager.cs) to drive vowel mouth shapes (`aa`, `ih`, `ou`, `ee`, `oh`) and jaw bone opening.
- **Remote Peer Sync**: Remote player audio levels drive the remote avatar's mouth in 3D in [`Main.cs`](file:///media/pikachubolk/63d7930c-4cfb-4c68-96a6-879048200e36/Documents/GitHub/Godot-SerikaSocial/game/Main.cs) so everyone's mouth animates accurately when speaking over voice chat.

---

### 3. Anatomical Scapulohumeral Rhythm (Clavicle & Shoulder Elevation)
- In [`VrAvatarIk.cs`](file:///media/pikachubolk/63d7930c-4cfb-4c68-96a6-879048200e36/Documents/GitHub/Godot-SerikaSocial/game/Player/VrAvatarIk.cs), when the hand reaches above the shoulder or crosses the chest, the clavicle/scapula elevates and protracts with a natural 1:2 anatomical ratio.
- Prevents shoulder distortion and allows natural overhead reaching without arm length clipping.

---

### 4. Dynamic Center-of-Mass & Hip Weight Shifting
- In [`VrAvatarIk.cs`](file:///media/pikachubolk/63d7930c-4cfb-4c68-96a6-879048200e36/Documents/GitHub/Godot-SerikaSocial/game/Player/VrAvatarIk.cs), when one foot is stepping, dynamically shift the pelvic root laterally over the supporting planted foot.
- Add subtle spine compression and lateral pelvic tilt during foot strike for a grounded, organic locomotion feel.

---

### 5. Multi-Model Automated Diagnostic Suite (5 Distinct Avatars)
Create an automated test runner script `tools/test-all-avatars.sh` that validates all 5 avatar models across the full diagnostic matrix:
1. Avatar 1: `311c0356...` (Suisei, 1.57m, standard female proportions)
2. Avatar 2: `18e26489...` (VRM anime girl, 1.52m, articulated finger bones)
3. Avatar 3: `014a12ed...` (Standard 1.51m humanoid)
4. Avatar 4: `f461face...` (Tall / distinct proportions)
5. Avatar 5: `a20e89b4...` or `4cdd7254...` (Chibi / distinct proportions)

---

## Verification Plan

### Automated Tests
1. `dotnet build` in `game/`
2. Run `--serika-vrtest` on all 5 distinct avatar models:
   ```bash
   $GODOT --headless --path game -- --serika-vrtest --ska <model_1>
   $GODOT --headless --path game -- --serika-vrtest --ska <model_2>
   $GODOT --headless --path game -- --serika-vrtest --ska <model_3>
   $GODOT --headless --path game -- --serika-vrtest --ska <model_4>
   $GODOT --headless --path game -- --serika-vrtest --ska <model_5>
   ```
3. Run `--serika-vrsim` (rendered OpenXR device simulator):
   ```bash
   $GODOT --path game --windowed --audio-driver Dummy -- --serika-vrsim --ska <model_2> --out /tmp/vr
   ```
4. Run `--serika-phystest`, `--serika-aimtest`, `--serika-fptest`, `--serika-animtest` across the avatars.
5. Codec verification: `dotnet test Net/Codec/Tests` and `bun run proto:test`.
