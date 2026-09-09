# Getting started

How to clone, build, and run every Serika Social repo locally. Check them out as
siblings (some scripts reach across repos — `server`'s `up` calls `../infra/dev-up.sh`).

```
Godot-SerikaSocial/
  proto/   server/   game/   godot-sdk/   web/   infra/   tools/   docs/
```

## Prerequisites

- **Godot 4.7 mono** binary (C# support). It is not on PATH on the dev machine — use
  the full binary path. We'll call it `$GODOT` below.
- **.NET SDK** (for the C# client + codec tests).
- **Bun** (server TypeScript) and **Rust** (stable, for `instanced` + `proto`).
- **Docker** (for the dev datastores via `infra/dev-up.sh`).
- **Prisma** (pulled in by `bun install` in `server`).

## 1. proto (wire codec)

```bash
git clone https://github.com/SerikaSocial/proto.git
cd proto/rust
cargo test -p serika-proto        # golden tests
```

`proto` is normally consumed as a submodule of `server` and `game` (below). You only
need a standalone clone to work on the codec itself.

## 2. server

```bash
git clone --recurse-submodules https://github.com/SerikaSocial/server.git
cd server
cp .env.example .env
bun install

bun run up            # Postgres :5490 + Redis :6490 (needs ../infra)
bun run db:generate   # Prisma client
bun run db:migrate    # apply migrations
bun run db:seed       # seed The Commons world

bun run proto:test            # codec golden tests (must match C# byte-for-byte)
cargo test -p instanced       # relay integration tests (needs Redis)

bun run dev:api               # API on :4100
bun run dev:gateway           # gateway WS on :4110
cargo run -p instanced        # relay on :4200/udp
bun run dev:all               # datastores + API + relay together
```

## 3. game (client)

```bash
git clone --recurse-submodules https://github.com/SerikaSocial/game.git
cd game
dotnet build
dotnet test Net/Codec/Tests    # C# golden tests (must match Rust) + transport liveness
```

Run the editor:

```bash
$GODOT --path game
```

Export a platform build:

```bash
$GODOT --headless --export-release "Linux/X11"      ../dist/linux/SerikaSocial.x86_64
$GODOT --headless --export-release "Windows Desktop" ../dist/windows/SerikaSocial.exe
$GODOT --headless --export-release "Android (Quest)"  ../dist/android/SerikaSocial.apk
```

See [`diagnostics.md`](diagnostics.md) for the headless diagnostic harnesses.

## 4. godot-sdk (creator tooling)

```bash
git clone https://github.com/SerikaSocial/godot-sdk.git
cd godot-sdk
godot --headless --path godot-sdk --script res://tests/run_tests.gd   # validator tests
```

Open it as a Godot project to use the demo world and develop the addon.

## 5. web / infra / tools

These are not part of the open-source set; see their own repos if you have access:
- [`web`](https://github.com/SerikaSocial/web) — Next.js 15, `bun install && bun dev`.
- [`infra`](https://github.com/SerikaSocial/infra) — `dev-up.sh` / `dev-all.sh`,
  Coolify config, migrations.
- [`tools`](https://github.com/SerikaSocial/tools) — `mint-ticket.ts`,
  `publish-release.ts`, load harness.

## Common gotchas

- **Missing proto submodule** → builds fail with missing proto. Run
  `git submodule update --init` in `server` and `game`.
- **Relay "stuck on Connecting"** → the port mapping must be `4200:4200/udp`, not
  TCP-only. See `architecture.md`.
- **`PUBLIC_ENDPOINT` must be a domain**, never a raw IP (local dev uses `localhost`).
- **OAuth loopback** uses fixed port 34517 — make sure it's free.
- **OpenXR startup modal is disabled** at the source (`xr/openxr/startup_alert=false`).
  Do not re-enable it.
