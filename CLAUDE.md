# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Blunder Therapist is a chess platform that analyzes **behavioral patterns** (not just moves) by treating chess games as psychological datasets. It detects tilt, generates player personality profiles, and provides a memory-aware AI coach.

## Running the project

### Backend (FastAPI + Python 3.13+)

```bash
cd backend
uv sync          # preferred — installs from uv.lock
# or: pip install -r requirements.txt
cp .env.example .env  # fill in required vars
uv run uvicorn app.main:app --reload --port 8000
```

API docs at `http://localhost:8000/docs`.

Run the feature extraction smoke test:
```bash
cd backend
python test_features.py
```

Run tests:
```bash
cd backend
pytest tests/ -v
```

### Frontend (Next.js 15 + Node 18+)

```bash
cd frontend
npm install
cp .env.example .env.local   # fill in Supabase keys
cp node_modules/stockfish.js/stockfish.js public/stockfish.js
npm run dev
```

App at `http://localhost:3000`.

```bash
npm run lint    # ESLint via next lint
npm run build   # Production build
```

## Architecture

### Backend (`backend/app/`)

**Auth + DB layer (added):**

- **`database.py`** — Async SQLAlchemy engine (`create_async_engine`) + `AsyncSessionLocal` + `Base`. `get_db()` is the FastAPI dependency injected into every endpoint.
- **`models.py`** — Five ORM models: `Profile` (user_id PK, plan), `Game` (pgn + evals + times per ply), `TiltReport` (1-to-1 with Game), `GameSummary` (post-session chat summary, 1-to-1 with Game), `DecisionDNA` (per-user style snapshot, recomputed every 5 games). All UUID PKs stored as `String`. Tables are created via `Base.metadata.create_all` in the app lifespan on startup.
- **`dependencies.py`** — `get_current_user` verifies the Supabase JWT (HS256 via `python-jose`) and returns `CurrentUser(user_id, plan)`. Auto-creates a `free` Profile on first login. `require_pro` raises 403 if plan ≠ "pro".
- **`routers/games.py`** — `GET /api/games` (paginated history), `GET /api/games/{id}` (detail), `POST /api/games/{id}/report` (re-run tilt detector on stored game).

**AI pipeline:**

1. **`services/features.py`** — `extract_features()` takes PGN + Stockfish evals (centipawns from White's POV) + per-ply thinking times → returns a `GameFeatures` dataclass. Key metric: `blunder_speed_ratio` (<0.5 = tilt, >1.5 = analysis paralysis). `features_to_llm_summary()` dumps the **full move list** (every ply, both sides) with eval and time — blunders marked with `<<<`, mistakes `<<`, inaccuracies `<`. No cap — the LLM sees the whole game.
2. **`services/tilt_detector.py`** — `run_tilt_detector(features)` — single-game behavioral analysis, structured output via `gpt-4o-mini`. Used by `POST /api/analyze-game`.
3. **`services/dna_decision.py`** — `run_decision_dna(games)` — builds a Decision DNA profile from N games' aggregate stats. Uses `gpt-4o-mini` with `json_object` mode.
4. **`services/coach.py`** — `run_coach_chat(message, history, game_memory)` — stateless coach chat backed by `gpt-4o`. History and memory are passed in full on every call.
5. **`services/event_detector.py`** — `classify_ply(eval_before, eval_after, player_color, time_taken) → "blunder" | None`. Pure Python, no LLM. Called server-side on every `observe` POST and mirrored client-side in `lib/event-detector.ts`.
6. **`services/summarizer.py`** — `summarize_chat(messages) → ChatSummaryOutput`. Uses `gpt-4o-mini` structured output. Called from `close-session` to persist `GameSummary`. Receives the **full thread** (including `[OBSERVATION]` blunder messages and the injected tilt report JSON — these are already HumanMessages so no extra filtering needed). Output schema: `summary` (3-5 sentences), `key_facts` (3-8 durable player facts), and `game_analysis: GameAnalysis` — a nested model with: `opening` (detected opening name), `game_result` (win/loss/draw/unknown), `blunders: list[BlunderNote]` (ply, san, eval_before, eval_after, time_taken, `blunder_type` — one of "tactical oversight" | "time pressure" | "calculation error" | "positional mistake" | "panic blunder" | "pattern blindness" — and a `note`), `main_mistakes`, `loss_reasons`, `behavioral_patterns`, `improvement_areas`. The LLM reads eval numbers directly from `[OBSERVATION]` lines — it must NOT invent them.
7. **`services/dna_job.py`** — `should_recompute_dna(total)` returns True every 5th game; `run_dna_for_user(db, user_id)` pulls the last 5 games, calls `run_decision_dna`, upserts `DecisionDNA`. Triggered from `/api/agent/close-session`.
8. **`main.py`** — Top-level endpoints: `POST /api/analyze-game` (saves game + tilt report), `POST /api/decision-dna` (pro), `POST /api/coach` (pro). DNA/Coach pull games from DB — no resend from frontend.

**LangGraph agent (in-game coach):**

The behavioral coach that runs alongside a live game is a LangGraph agent with persistent thread state. Uses **LangChain 1.2.x** (`create_agent` from `langchain.agents`).

- **`services/agent_tools.py`** — Defines `AgentContext` (`@dataclass` with `user_id: str`, `jwt: str`) used as `context_schema` throughout. Two `@tool`-decorated functions: `list_past_games(filter, runtime: ToolRuntime[AgentContext])` and `get_game_details(game_id, runtime: ToolRuntime[AgentContext])`. Tools access `user_id` via typed `runtime.context.user_id` (not a dict). Each opens a fresh `AsyncSessionLocal` (do NOT reuse the request's session). Each tool has a paired `_*_impl(db, user_id, ...)` for direct testing.
- **`services/agent.py`** — `AgentService` class (module-level singleton `agent_service = AgentService()`). Holds `_agent`, `_checkpointer`, `_pg_cm`, and `_prompt_cache: dict[str, str]` (thread_id → full system prompt). Key methods: `init(database_url, *, in_memory)` — called in lifespan, creates checkpointer and builds the agent with `@dynamic_prompt` middleware; `shutdown()` — closes `AsyncPostgresSaver`; `stream(thread_id, context, messages)` → async generator of SSE strings; `has_state(thread_id)` → bool; `get_messages(thread_id)` → list of stored `BaseMessage`. The `@dynamic_prompt` middleware (defined as a closure inside `init`) checks `_prompt_cache[thread_id]` — on cache miss it opens `AsyncSessionLocal`, queries last 5 `GameSummary` rows + latest `DecisionDNA`, builds the full prompt (BASE + MEMORY + DNA), caches it, and returns it. Cache hit is immediate with no DB query. This guarantees exactly one DB query per game thread, full prompt on every LLM call. `build_system_prompt(db, user_id)` remains exported for tests. Key imports: `from langchain.agents.middleware.types import dynamic_prompt, ModelRequest`; `from langgraph.prebuilt.tool_node import ToolRuntime`. `_format_summaries()` now also surfaces `game_analysis` fields (opening, result, per-blunder type+note, loss reasons, behavioral patterns, improvement areas) into the future system prompt so the in-game coach has full cross-game memory of concrete mistakes.
- **`routers/agent.py`** — SSE streaming endpoints under `/api/agent`. Imports only `agent_service` and `AgentContext` — no checkpointer or seeding logic. Builds `ctx = AgentContext(user_id=user.user_id, jwt=jwt_token)` per request.
  - `POST /observe` — `game_start` returns an empty stream immediately. `blunder` calls `_format_observation_message` which renders the full move list from the last observe point to the current ply (`moves_since_last_observe`), then appends `"Respond in 1-2 sentences max."` so the agent stays brief. `game_end` loads `TiltReport` from DB (404 if missing) and injects it as a `HumanMessage`.
  - `POST /message` — calls `agent_service.stream(thread_id, ctx, [HumanMessage(text)])`.
  - `POST /close-session` — idempotent (checks `GameSummary` by `game_id`). Pulls messages via `agent_service.get_messages(thread_id)`, filters out system-type messages (HumanMessages including `[OBSERVATION]` ones are kept), summarises with `summarize_chat`, persists `GameSummary` (including `game_analysis` via `.model_dump()`), triggers `run_dna_for_user` if `total_games % 5 == 0`. Returns 204.
  - `GET /history/{thread_id}` — returns non-system messages via `agent_service.get_messages(thread_id)`.
- **Thread ID convention:** `thread_id == game.id`. Frontend mints the UUID before the game starts and passes it as `client_game_id` in `POST /api/analyze-game` so the backend uses it as `Game.id`.
- **Tilt is NOT an agent tool.** Injected as a `HumanMessage` on `game_end` observe event.

**Logging:**

- **`logging_setup.py`** — `setup_logging()` configures `structlog` once at startup (called first in `main.py`, before any other imports). Format toggled by `LOG_FORMAT` env var: `console` (dev, human-readable) or `json` (prod). Level set by `LOG_LEVEL`. Routes stdlib logging through structlog so asyncpg/httpx/uvicorn land in the same sink. `ToolCallLogger(component)` is a LangChain `BaseCallbackHandler` that logs every agent tool call (`tool_call_start`, `tool_call_end`, `tool_call_error`) with `duration_ms` — passed as callback in `AgentService.stream()`.
- **`main.py`** — `RequestLoggingMiddleware` (Starlette `BaseHTTPMiddleware`) logs `request_start` / `request_end` (status, duration_ms) for every HTTP request. Calls `clear_contextvars()` at request start to prevent context leakage between async tasks, then binds `request_id` (UUID4) via structlog contextvars. All modules use `log = structlog.get_logger()` at module level and bind contextual fields (e.g. `thread_id`, `user_id`) per call.
- **`dependencies.py`** — after successful JWT decode, calls `structlog.contextvars.bind_contextvars(user_id=...)` so `user_id` appears automatically in all subsequent logs for that request.

Config is in `config.py` via pydantic-settings. CORS origins default to `localhost:3000`.

### Frontend (`frontend/src/`)

**Auth layer (added):**

- **`middleware.ts`** — Next.js edge middleware. Runs on every request, refreshes the Supabase session cookie, redirects unauthenticated users to `/login`.
- **`lib/supabase/client.ts`** — Browser Supabase client (singleton via `createBrowserClient`).
- **`lib/supabase/server.ts`** — Server-side Supabase client (cookie-aware, via `createServerClient`).
- **`app/login/page.tsx`** — Login page: Google OAuth button + email/password form.
- **`app/auth/callback/route.ts`** — OAuth code → session exchange handler.
- **`components/NavBar.tsx`** — Client component with user email display and sign-out button.

**Components:**

- **`app/page.tsx`** — Single-page layout using `NavBar` + `ChessGame`.
- **`components/ChessGame.tsx`** — Orchestrates: react-chessboard, chess.js, Stockfish Web Worker, per-ply timing. On each ply calls `classifyPly()` (client-side) and fires `agentApi.observe("blunder", ...)` if eval drops ≥ 200cp. The blunder payload includes `moves_since_last_observe` — a slice of all moves (ply, san, eval_after, time_sec) from `lastObservePlyRef` up to and including the blunder ply; `lastObservePlyRef` advances after each observe so the next one starts where this one ended. On game end sends data to `POST /api/analyze-game` (passing `client_game_id` = the local UUID) and sets `analysis` state. The right sidebar is now `<AgentChat threadId={gameId} tiltReport={analysis} />`. On "New game" fires `agentApi.closeSession(gameId)`, resets `lastObservePlyRef`, and mints a fresh `gameId`.
- **`components/AgentChat.tsx`** — Per-game chat sidebar. Loads history on mount via `GET /api/agent/history/{threadId}`. When `tiltReport` prop becomes non-null, pushes a `tilt_card` message inline and fires `observe("game_end")` to trigger agent's conversational reaction. User messages go to `POST /api/agent/message`; responses stream via `parseAgentStream`. On unmount + `beforeunload` fires `closeSession` (keepalive POST).
- **`components/AgentMessage.tsx`** — Renders one chat message. Switches on `kind`: `"user"` (right-aligned bubble), `"agent"` (left-aligned bubble with animated cursor while `pending`), `"tilt_card"` (renders `<TiltReport>` inline in the chat flow).
- **`components/TiltReport.tsx`** — Displays the structured Tilt Detector response. Rendered inside `AgentMessage` as a `tilt_card` — no longer used as a standalone sidebar widget.
- **`lib/stockfish.ts`** — `StockfishEngine` wrapper around the stockfish.js Web Worker.
- **`lib/api.ts`** — Typed fetch client. `api.*` methods inject `Authorization: Bearer` and return parsed JSON. `agentApi.*` methods (`observe`, `message`, `closeSession`, `history`) call the agent endpoints; `observe`/`message` return a raw `Response` for streaming. `postStream` helper returns `Response` without consuming the body.
- **`lib/agent-stream.ts`** — `parseAgentStream(response): AsyncIterable<AgentStreamEvent>`. Parses `text/event-stream` format into typed events: `{type: "token", text}`, `{type: "done"}`, `{type: "error", message}`.
- **`lib/event-detector.ts`** — `classifyPly(evalBefore, evalAfter, playerColor) → "blunder" | null`. Client-side mirror of `services/event_detector.py`. Called on every ply in `ChessGame` to avoid a server round-trip for non-events. Threshold: 200cp swing against the human player.

### Key data flow

1. User logs in → Supabase sets `sb-*` httpOnly cookies → Next.js middleware allows through
2. **Game start** → frontend mints `gameId` (UUID), fires `POST /api/agent/observe {event: "game_start"}` to seed the agent thread with system prompt (last 5 summaries + DNA).
3. **During game** → on each player ply: `classifyPly()` client-side; if `"blunder"` → `POST /api/agent/observe {event: "blunder", moves_since_last_observe: [...], ...}` → backend formats the move list into an `[OBSERVATION]` HumanMessage with a 1-2 sentence response constraint → agent streams a brief reaction.
4. **Game end** → `POST /api/analyze-game {client_game_id, pgn, evals, times, ...}` → backend saves `Game` (using `client_game_id` as PK) → runs Tilt → saves `TiltReport` → returns report. Frontend then fires `POST /api/agent/observe {event: "game_end"}` → backend loads TiltReport, injects as HumanMessage → agent streams conversational reaction. Tilt card renders inline in AgentChat.
5. **User chat** → `POST /api/agent/message {thread_id, text}` → agent streams reply. Tools (`list_past_games`, `get_game_details`) available throughout.
6. **Session close** (New game click or page unload) → `POST /api/agent/close-session {thread_id}` → backend summarizes chat via LLM → saves `GameSummary` → if `total_games % 5 == 0` → runs DNA job → upserts `DecisionDNA`.
7. Next game's system prompt includes the just-saved summaries and DNA — closing the cross-game memory loop.

Stockfish runs **in the browser** (WASM via Web Worker) — not on the backend. Evals are always stored from **White's perspective** (centipawns).

### Database schema

```
profiles    — user_id (PK String), plan ("free"|"pro"), created_at
games       — id (PK String UUID), user_id (FK), pgn, eval_per_ply (JSON),
              time_per_ply (JSON), player_color, result, played_at,
              platform_game_id (String index)
tilt_reports — id (PK String UUID), game_id (FK unique), headline, diagnosis,
               pattern_label, evidence_plies (JSON), suggestion
import_jobs  — id (PK String UUID), user_id (FK), platform, username,
               period_days, status, total_games, processed_games, error,
               created_at, finished_at
game_summaries — id, user_id (FK, indexed), game_id (FK unique), summary (Text),
                 key_facts (JSON list), game_analysis (JSON nullable), created_at
                 game_analysis shape: {opening, game_result, blunders: [{ply, san,
                 eval_before, eval_after, time_taken, blunder_type, note}],
                 main_mistakes, loss_reasons, behavioral_patterns, improvement_areas}
                 NOTE: add column on live Postgres with:
                   ALTER TABLE game_summaries ADD COLUMN game_analysis JSONB;
                   ALTER TABLE games ADD COLUMN platform_game_id VARCHAR;
decision_dna  — id, user_id (FK, indexed), dna (JSON), games_count, computed_at
                (no unique constraint — new row per recompute, latest by computed_at)
```

Tables use `String` for all UUIDs (not Postgres UUID type) so they work with both SQLite (tests) and Postgres (prod).

### Environment variables

**Backend `.env`:**
- `OPENAI_API_KEY` — required
- `DATABASE_URL` — use Supabase pooler URL: `postgresql+asyncpg://postgres.{ref}:{password}@aws-0-{region}.pooler.supabase.com:5432/postgres`
- `SUPABASE_URL` — e.g. `https://zfudugmxwdbzkpkfagsd.supabase.co`
- `SUPABASE_JWT_SECRET` — from Supabase dashboard → Settings → API → JWT Secret
- `CORS_ORIGINS` — defaults to `["http://localhost:3000"]`
- `LOG_LEVEL` — `DEBUG` / `INFO` / `WARNING` / `ERROR`, defaults to `INFO`
- `LOG_FORMAT` — `console` (dev, human-readable) or `json` (prod), defaults to `console`

**Frontend `.env.local`:**
- `NEXT_PUBLIC_API_BASE` — defaults to `http://localhost:8000`
- `NEXT_PUBLIC_SUPABASE_URL` — same as backend `SUPABASE_URL`
- `NEXT_PUBLIC_SUPABASE_ANON_KEY` — from Supabase dashboard → Settings → API → anon key

### Supabase connection note

Use the **pooler URL** (Supavisor Session mode, port 5432), not the direct DB host. The direct host (`db.*.supabase.co:5432`) is often blocked by local network firewalls. Username format for pooler: `postgres.{project-ref}`.

### Testing

Tests live in `backend/tests/`. `conftest.py` sets `TESTING=1` and all required env vars before any app imports, then creates an in-memory SQLite engine with `StaticPool` (so all test connections share the same DB). The lifespan skips `create_all` when `TESTING=1` — the `create_tables` session-scoped fixture handles table creation instead. An `_agent_in_memory` autouse fixture calls `init_agent(database_url="", in_memory=True)` so agent tests use `MemorySaver` instead of Postgres.

```bash
cd backend
pytest tests/ -v
```

Key fixtures: `db` (AsyncSession), `client` (unauthenticated AsyncClient), `authed_client` (free plan), `pro_client` (pro plan).

LLM calls in tests are always mocked (`unittest.mock.AsyncMock` + `patch`). Agent invocations use `patch.object(agent_service, "stream", new=fake_stream)` and `patch.object(agent_service, "get_messages", new=AsyncMock(...))` in `test_agent_router.py`. Tool impls are tested via `_*_impl` helpers against the real SQLite session in `test_agent_tools.py`. The `_agent_in_memory` autouse fixture calls `agent_service.init("", in_memory=True)` and resets `agent_service._agent`, `_checkpointer`, `_prompt_cache` after each test.

### Async SQLAlchemy gotcha

Never access relationship attributes after a plain `await session.refresh(obj)` — lazy loading is not supported in async mode. Always use `selectinload` in the query:
```python
select(Game).options(selectinload(Game.tilt_report)).where(...)
```
