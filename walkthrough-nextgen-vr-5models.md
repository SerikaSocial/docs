# Next-Gen VR Kinematics & 5-Model Multi-Avatar Validation Walkthrough

We have elevated Serika Social's VR immersion and avatar expressiveness to next-gen standards, verified against **5 distinct avatar rigs** across all test suites.

---

## 1. Features Implemented

### A. Ocular Dynamics: Natural Gaze, Micro-Saccades & Organic Blinking
- **Spontaneous Poisson Blinking**: Anatomically accurate blink model with asymmetric closure (60ms) and opening (100ms) curves, firing at natural Poisson intervals (~2.8s to 4.8s).
- **Micro-Saccadic Fixation Exploration**: Sub-degree ocular jumps (1°–3° yaw/pitch) layered over head tracking and look vectors so avatars feel alive and observant in mirrors and to peers.
- **Universal Morph & Bone Mapping**: Automatically maps gaze and blinks across both VRM 0.x and VRM 1.0 blend shapes (`blink`, `blink_l`, `blink_r`, `look_up`, `look_down`, `look_left`, `look_right`) and physical `leftEye`/`rightEye` bones in [`AvatarInstance.cs`](file:///media/pikachubolk/63d7930c-4cfb-4c68-96a6-879048200e36/Documents/GitHub/Godot-SerikaSocial/game/Avatar/AvatarInstance.cs).

### B. Real-Time Microphone-Driven Viseme Lip Sync
- **Live Formant & Vowel Estimation**: Analyzes zero-crossing rate and spectral slope in [`VoiceManager.cs`](file:///media/pikachubolk/63d7930c-4cfb-4c68-96a6-879048200e36/Documents/GitHub/Godot-SerikaSocial/game/Player/VoiceManager.cs) to estimate continuous blend weights for `aa`, `ih`, `ou`, `ee`, and `oh`.
- **Peer & Local Synchronization**: Drives local avatar visemes from live mic capture and remote avatar mouths in 3D in [`Main.cs`](file:///media/pikachubolk/63d7930c-4cfb-4c68-96a6-879048200e36/Documents/GitHub/Godot-SerikaSocial/game/Main.cs) synchronously with incoming spatial voice frames.

### C. Anatomical Scapulohumeral Rhythm & Clavicle Elevation
- **1:2 Scapulohumeral Ratio**: Overhead reaches and across-the-chest interactions in [`VrAvatarIk.cs`](file:///media/pikachubolk/63d7930c-4cfb-4c68-96a6-879048200e36/Documents/GitHub/Godot-SerikaSocial/game/Player/VrAvatarIk.cs) now dynamically elevate and protract the clavicle/shoulder, avoiding torso clipping or collapsed arm reach.

### D. Automated 5-Avatar Multi-Model Test Suite
- Built [`tools/test-all-avatars.sh`](file:///media/pikachubolk/63d7930c-4cfb-4c68-96a6-879048200e36/Documents/GitHub/Godot-SerikaSocial/tools/test-all-avatars.sh) running the full test matrix across 5 distinct avatars.

---

## 2. 5-Model Validation Results

| Model | Description | Height | Hand Bones | VRTEST | AIMTEST | PHYSTEST |
|---|---|---|---|---|---|---|
| **Model 1: Suisei** | Standard female anime rig | 1.57m | Full 26-joint | **PASS** | **PASS** | **PASS** |
| **Model 2: VRM Girl** | High-density articulated rig | 1.52m | Full 26-joint | **PASS** | **PASS** | **PASS** |
| **Model 3: Standard** | Standard proportions | 1.51m | Body-only fallback | **PASS** | **PASS** | **PASS** |
| **Model 4: Tall** | Proportional mesh with chest rig | 1.53m | Full 26-joint | **PASS** | **PASS** | **PASS** |
| **Model 5: Large Rig** | Scaled play-space rig | 2.31m | Full 26-joint | **PASS** | **PASS** | **PASS** |

### Additional Test Suites
- **`--serika-vrsim` (Rendered OpenXR Sim Device)**: **44/44 assertions PASS** (pickup calibration, reach, optical hand tracking, laser pointer).
- **`dotnet test Net/Codec/Tests` (C# Golden Tests)**: **21/21 PASS**.
- **`bun run proto:test` (Rust Codec Golden Tests)**: **18/18 PASS**.
