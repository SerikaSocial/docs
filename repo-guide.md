# Repo guide

Serika Social is a **multi-repo monorepo**: each subdirectory is its own git repo under
`github.com/SerikaSocial`. Commit and push per-repo, never from the workspace root (the
root is not a git repo).

## Layout

| Repo | Stack | Open source? |
|---|---|---|
| `proto` | Rust codec | yes |
| `server` | Bun + Elysia + Prisma; Rust relay (`instanced`) | yes |
| `game` | Godot 4.7 C# | yes |
| `godot-sdk` | Godot 4.7 addon | yes |
| `docs` | this repo | yes |
| `web` | Next.js 15 | no |
| `infra` | Docker, Coolify, playit | no |
| `tools` | release + load tooling | no |

## Submodules: `proto`

`proto` is the wire codec and is pinned as a **git submodule** in both `server` and
`game`:

```
server/proto  → SerikaSocial/proto
game/proto    → SerikaSocial/proto
```

Both repos must pin the **same** `proto` commit. Clone with `--recurse-submodules`, or
run `git submodule update --init` after cloning.

### Changing the protocol

The order matters:

1. Commit and push in `proto` first.
2. Bump the pin in `server` **and** `game`, in that order or simultaneously.
3. Both golden suites must pass against the new `proto` commit:
   - `cargo test -p serika-proto` (in `server`)
   - `dotnet test Net/Codec/Tests` (in `game`)

A client and a relay on different `proto` commits is exactly the desync the golden
corpus exists to prevent. See [`proto/README.md`](https://github.com/SerikaSocial/proto)
and `proto/pose_codec.md`.

## Committing

Each repo has its own `AGENTS.md` (mirrored as `CLAUDE.md`) with operational rules.
Match the existing commit style (short imperative subject). Commit per-repo:

```bash
cd server
git add -A && git commit -m "Relay: AvatarChanged (0x0C), so avatar swaps reach the room"
git push
```

Never `git commit` from the workspace root.

## Secrets

`.env` is gitignored in every repo that has one; copy from `.env.example`. This
project never stores passwords — it mirrors `serika-accounts` users keyed by
`accounts_id`. See [`auth-integration.md`](auth-integration.md).

Before making a repo public, scan full git history for secrets (e.g.
[`gitleaks`](https://github.com/gitleaks/gitleaks) `git` scan). Production IPs and
credentials must be redacted — and if they're in history, the history must be rewritten
(`git-filter-repo --replace-text`) before the flip.
