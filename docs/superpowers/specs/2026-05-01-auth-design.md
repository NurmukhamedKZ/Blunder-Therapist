# Authentication System Design

**Date:** 2026-05-01  
**Status:** Approved  
**Scope:** Full-stack auth with Supabase Auth (Google OAuth + email/password), JWT verification on FastAPI, and full user profile + game history persistence.

---

## Summary

Every route requires login. Authentication is handled by Supabase Auth (Google OAuth + email/password). The frontend uses `@supabase/ssr` + Next.js middleware for edge-level route protection. The backend verifies Supabase-issued JWTs via JWKS and enforces the free/pro plan on protected endpoints. Games and tilt reports are persisted per user in Supabase Postgres.

---

## Architecture

```
Browser
  │
  ├─ Next.js Middleware (edge) ──── checks Supabase session cookie
  │     └─ unauthenticated → redirect to /login
  │
  ├─ /login page
  │     ├─ "Continue with Google" → Supabase OAuth redirect
  │     └─ Email/password form → Supabase signInWithPassword
  │
  └─ Authenticated pages (/, /play, /dna, /coach)
        └─ fetch calls to FastAPI with Authorization: Bearer <supabase_jwt>

FastAPI Backend
  ├─ auth dependency: verify JWT via Supabase JWKS, extract user_id + plan
  ├─ existing endpoints now require auth + save results to DB
  └─ new endpoints:
        POST /api/games        — save completed game + tilt report
        GET  /api/games        — list user's game history
        GET  /api/games/{id}   — fetch single game + stored tilt report
        POST /api/games/{id}/report — re-run tilt detector on stored game

Supabase (Postgres)
  ├─ auth.users (managed by Supabase Auth — do not touch)
  ├─ public.profiles (user_id FK, plan, created_at)
  ├─ public.games (id, user_id, pgn, eval_per_ply, time_per_ply, player_color, result, played_at)
  └─ public.tilt_reports (id, game_id, headline, diagnosis, pattern_label, evidence_plies, suggestion)
```

The frontend never hits Supabase DB directly — all data flows through FastAPI. Supabase is used only for Auth (JWT issuance) and as the Postgres host.

---

## Frontend

### Packages added
- `@supabase/supabase-js`
- `@supabase/ssr`

### New files
| File | Purpose |
|---|---|
| `src/lib/supabase/client.ts` | Browser Supabase client singleton |
| `src/lib/supabase/server.ts` | Server-side Supabase client (reads cookies) |
| `src/middleware.ts` | Edge middleware — refreshes session, redirects unauthenticated users to `/login` |
| `src/app/login/page.tsx` | Login page with Google OAuth button + email/password form |
| `src/app/auth/callback/route.ts` | OAuth callback handler — exchanges code for session, redirects to `/` |

### Session flow
1. User visits any page → middleware checks for valid Supabase session cookie
2. No session → redirect to `/login`
3. User signs in (Google or email/password) → Supabase sets `sb-*` httpOnly cookies
4. Middleware sees valid session → allows request through
5. Before every FastAPI call, `api.ts` reads session via `supabase.auth.getSession()` and attaches `access_token` as `Authorization: Bearer <token>`
6. `@supabase/ssr` middleware automatically refreshes the JWT on every request before expiry

### Logout
Logout button calls `supabase.auth.signOut()`, clears cookies, middleware redirects to `/login`.

---

## Backend

### New config values (`config.py`)
- `SUPABASE_URL` — used to construct the JWKS endpoint: `{SUPABASE_URL}/auth/v1/.well-known/jwks.json`
- `SUPABASE_JWT_SECRET` — HS256 fallback for local dev

### Auth dependency
`get_current_user` dependency:
1. Extracts `Bearer` token from `Authorization` header
2. Fetches and caches Supabase JWKS keys
3. Verifies and decodes JWT with `python-jose`
4. Looks up `profiles` table by `user_id`; auto-creates a `free` profile if not found (handles race on first login)
5. Returns `CurrentUser(user_id: str, plan: Literal["free", "pro"])`

All endpoints declare this dependency. `require_pro` is a second dependency that raises `HTTP 403 {"detail": "pro_required"}` if `plan != "pro"`.

### Database models (`app/models.py`)

```python
Profile     user_id (UUID PK, FK auth.users), plan ("free"|"pro"), created_at
Game        id (UUID PK), user_id (FK Profile), pgn (text), eval_per_ply (JSON),
            time_per_ply (JSON), player_color ("white"|"black"), result ("win"|"loss"|"draw"), played_at
TiltReport  id (UUID PK), game_id (UUID FK Game, unique), headline, diagnosis,
            pattern_label, evidence_plies (JSON), suggestion
```

### Endpoint changes

| Endpoint | Auth | Plan | Change |
|---|---|---|---|
| `POST /api/analyze-game` | required | free | saves Game + TiltReport to DB, returns same response shape |
| `POST /api/decision-dna` | required | pro | pulls last N games from DB instead of requiring frontend to send them |
| `POST /api/coach` | required | pro | pulls last 10 games from DB for memory context |
| `GET /api/games` | required | free | new — paginated game history with stored tilt reports |
| `GET /api/games/{id}` | required | free | new — single game + tilt report |
| `POST /api/games/{id}/report` | required | free | new — re-run tilt detector on a stored game |

---

## Error Handling

### Frontend
| Scenario | Behavior |
|---|---|
| Expired/invalid JWT | `api.ts` catches `AuthError`, redirects to `/login` |
| Google OAuth cancelled/failed | `/auth/callback` reads `error` query param, redirects to `/login?error=oauth_failed` |
| Wrong email/password | Inline error message on login form, no redirect |

### Backend
| Scenario | Response |
|---|---|
| Missing/malformed `Authorization` header | `HTTP 401` |
| Expired JWT | `HTTP 401` |
| Valid JWT, no profile row | Auto-create free profile, continue |
| Pro endpoint with free plan | `HTTP 403 {"detail": "pro_required"}` |
| OpenAI fails after game saved | Game row persists, tilt report row absent; re-run via `POST /api/games/{id}/report` |
| Duplicate profile upsert | `ON CONFLICT DO NOTHING` |
| User deleted from Supabase | `HTTP 401` (no profile found, not auto-created since JWT is from deleted user) |

### Session edge cases
- Middleware refresh fails (Supabase outage) → `@supabase/ssr` falls back to existing cookie; user stays logged in with stale token until expiry
- Concurrent first-login requests → profile upsert is idempotent via `ON CONFLICT DO NOTHING`
