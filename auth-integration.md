# Identity integration with serika-accounts

Serika Social does not store credentials. `serika-accounts` is the OAuth2/OIDC provider for
the whole Serika ecosystem; we are a client, and we mirror the user record into our own
Postgres keyed by `accounts_id`.

Everything below was **verified empirically** on 2026-08-21 by running `serika-accounts`
against a throwaway MongoDB and executing the real flow — not inferred from reading code.

## The flow

```
Godot client → browser → serika-accounts /api/oauth/authorize?…&code_challenge=…
            ← loopback  http://127.0.0.1:34517/callback?code=…
            → api  POST /v1/session/exchange {code, verifier}
api         → serika-accounts POST /api/oauth/token        → opaque access_token
api         → serika-accounts POST /internal/verify-oauth  → profile + ban check
api         → upsert `users`, mint our own session JWT + instance tickets
```

## Three things that constrain the design

### 1. `/internal/verify` cannot validate OAuth tokens

It calls `jwt.verify(token)` and then looks the token up in the `Session` collection.
OAuth access tokens are **opaque 43-character random strings**, not JWTs, so verification
always fails:

```
POST /internal/verify  {"token": "<opaque oauth token>"}
→ {"error":"Invalid token","valid":false}
```

That endpoint is for session JWTs issued by `/api/auth/login`. It is not usable here.

### 2. `/api/oauth/userinfo` does not check `isBanned`

This is the important one. `userinfo` is the only *stock* endpoint that validates opaque
OAuth tokens — but it never reads `user.isBanned`. A platform-banned user gets a complete,
successful profile response:

```
(user.isBanned = true)
GET /api/oauth/userinfo  Authorization: Bearer <token>
→ 200 {"sub":"…","preferred_username":"spiketester","email":"spike@example.com", …}
```

**Any service that authenticates via `userinfo` alone is admitting banned users.** That is
not just our problem — it applies to every existing OAuth client of serika-accounts.

### 3. `POST /internal/verify-oauth` — added to close the gap

Added to `serika-accounts/src/index.ts`. It resolves an OAuth access token, applies the
same ban policy as `/internal/verify` (including auto-lifting expired bans), and returns
the same `user` shape so callers can treat the two endpoints uniformly. It additionally
returns `scopes` and `clientId`.

Verified behaviour:

| Case | Result |
|---|---|
| Valid token, active user | `valid: true` + user, scopes, clientId |
| Banned user | `valid: false`, `code: ACCOUNT_BANNED`, ban details |
| Ban whose `expiresAt` has passed | auto-lifted, `valid: true`, `isBanned` cleared in DB |
| Revoked token | `valid: false` |
| Wrong `x-service-key` | `valid: false` |

### 4. Email/password login issues a **session JWT**, not an OAuth token

`POST /api/auth/login` returns `{ success, token }` where `token` is a **session JWT**
(`jwt.sign(...)` + a row in the `Session` collection) — **not** an opaque OAuth access
token. It therefore must be verified with `/internal/verify` (which runs `jwt.verify()` and
looks the token up in `Session`), **not** `/internal/verify-oauth` (which only resolves rows
in `OAuthAccessToken`).

Getting this wrong is what made in-game email/password login fail with `verify_failed`
while the browser PKCE flow worked: the PKCE `/exchange` path produces a real OAuth access
token (→ `verifyOAuth`), but the `/login` path produces a session JWT and was being sent to
`verifyOAuth`, which never matched. Fixed in `server/api`:

- `accounts.ts` → `verifyAccountsSession()` calls `/internal/verify` (JWT + ban check, same
  `{ valid, code, user }` shape as `verifyOAuth`).
- `routes/session.ts` `/login` uses `verifyAccountsSession(login.token)`.
- `loginWithEmail()` now forwards `two_factor_code` and surfaces the accounts error codes
  (`EMAIL_NOT_VERIFIED`, `TWO_FACTOR_REQUIRED`, `TWO_FACTOR_INVALID`, `AGE_RESTRICTION`)
  instead of a blanket `invalid_credentials`, so the client can react (e.g. prompt for 2FA).

| Path | accounts endpoint | token type | verify with |
|---|---|---|---|
| Browser PKCE (`/v1/session/exchange`) | `/api/oauth/token` | opaque OAuth access token | `verifyOAuth` → `/internal/verify-oauth` |
| Email/password (`/v1/session/login`) | `/api/auth/login` | session JWT | `verifyAccountsSession` → `/internal/verify` |

## Client registration

Two entries were added to `SERIKA_PRODUCTS` in `serika-accounts/src/config.ts`:

| id | callback | notes |
|---|---|---|
| `serika-social` | `https://api-social.ado.ink/v1/session/callback` | the web/API client |
| `serika-social-game` | `http://127.0.0.1:34517/callback` | the Godot client |

`ado.ink` was added to `ALLOWED_DOMAINS` for CORS.

**The loopback port is fixed at 34517 and cannot be randomized.** `/api/oauth/authorize`
requires an exact match against the client's registered `redirectUris`
(`src/routes/oauth.ts:58`), so the usual "bind port 0 and register any loopback port"
pattern for native apps does not work here.

> Note: OAuth client records also resolve through **serika.moe**
> (`src/services/oauthClients.ts`), which falls back to the local DB. Production
> registration of `serika-social-game` may need to happen in that third repo. The spike
> seeded the client directly into Mongo, which exercised the local-DB fallback path.

## PKCE, and a caveat about client secrets

The game is a public client — it ships on user machines and cannot hold a secret. Verified
that a token exchange with `code_verifier` and **no `client_secret`** succeeds.

> 🔒 `src/routes/oauth.ts:267` reads `if (client_secret && hashSecret(client_secret) !== …)`
> — the secret is only checked *when supplied*. Omitting it entirely skips validation, so
> any client_id can be exchanged against without proving identity. PKCE protects our flow
> (an attacker without the verifier can't use a stolen code), but confidential clients in
> the ecosystem are not getting the protection they think they are. Worth fixing
> separately in serika-accounts: reject a missing secret when the client is not marked
> public.

## Reproducing the spike

```bash
docker run -d --name accounts-spike-mongo -p 37017:27017 mongo:7
cd ../serika-accounts
ACCOUNTS_PORT=3699 \
ACCOUNTS_MONGO_URI=mongodb://localhost:37017/serika-accounts-spike \
JWT_SECRET=spike-jwt-secret \
AUTH_SERVICE_INTERNAL_KEY=spike-internal-key \
bun run src/index.ts
```

Register a user, set `isVerified: true` in Mongo, insert an `oauthclients` document with
`redirectUris: ["http://127.0.0.1:34517/callback"]`, then drive the flow.

Two gotchas that cost time:
- `POST /api/oauth/authorize` authenticates via the **`auth_token` cookie**, not an
  `Authorization` header. A Bearer token returns `{"error":"unauthorized"}`.
- Login requires `isVerified`, which normally needs an email round-trip. Flip it directly
  in Mongo.
