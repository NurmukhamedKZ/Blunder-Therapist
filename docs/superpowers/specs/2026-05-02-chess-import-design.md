# Chess.com & Lichess Game Import — Design Spec

**Date:** 2026-05-02  
**Status:** Approved

## Overview

Allow users to import their past games from Chess.com and Lichess into Blunder Therapist so the AI coach can analyze behavioral patterns across their full game history — not just games played on the platform.

## Constraints & Decisions

- **Auth model:** username-only, public API only (no OAuth in this iteration)
- **Period:** user selects 30 or 90 days
- **Evals:** Stockfish runs on the backend (Python `stockfish` package, depth 15) for all games lacking pre-computed evals
- **Architecture:** background asyncio task + polling (no Celery, no SSE)

## External APIs

### Chess.com

Endpoint: `GET https://api.chess.com/pub/player/{username}/games/{year}/{month}`

- Returns games per calendar month; call for each month in the selected period
- PGN includes clock annotations: `{[%clk 0:04:58]}` after each move
- Extract `time_per_ply` by computing per-move clock deltas
- No engine evaluations — Stockfish required for all Chess.com games
- Skip `time_class == "daily"` (correspondence chess): no real clock data
- Determine `player_color` from `white.username` / `black.username`
- Map `white.result` / `black.result` ("win", "resigned", "checkmated", "agreed", "stalemate", etc.) → "win" | "loss" | "draw"

### Lichess

Endpoint: `GET https://lichess.org/api/games/user/{username}?since={ms}&until={ms}&evals=true&clocks=true&pgnInJson=false`

- Returns NDJSON stream of games
- `clocks=true` adds `{[%clk ...]}` annotations
- `evals=true` adds `{[%eval ...]}` annotations for games Lichess has already analyzed (~50–70% of games)
- Use Lichess eval where present; fall back to Stockfish for the rest
- Same color/result mapping as Chess.com

## Backend Architecture

### New DB Model: `ImportJob`

```python
class ImportJob(Base):
    __tablename__ = "import_jobs"

    id: str              # PK UUID
    user_id: str         # FK → profiles
    platform: str        # "chess.com" | "lichess"
    username: str
    period_days: int     # 30 | 90
    status: str          # "pending" | "running" | "done" | "failed"
    total_games: int     # games found from platform
    processed_games: int # games saved + analyzed so far
    error: str | None    # error message if status == "failed"
    created_at: datetime
    finished_at: datetime | None
```

### New Services

**`services/chess_import.py`**

Two public async functions:

```python
async def fetch_chesscom_games(username: str, since: datetime, until: datetime) -> list[RawGame]
async def fetch_lichess_games(username: str, since: datetime, until: datetime) -> list[RawGame]
```

`RawGame` dataclass:
```python
@dataclass
class RawGame:
    pgn: str
    platform: str
    platform_game_id: str
    player_color: Literal["white", "black"]
    result: Literal["win", "loss", "draw"]
    time_per_ply: list[float]       # extracted from [%clk] annotations
    eval_per_ply: list[int] | None  # from Lichess [%eval], None for Chess.com
```

Clock parsing: walk PGN comments, collect `[%clk H:MM:SS]` values, convert to seconds, compute per-ply delta as `prev_clock - current_clock`.

**`services/stockfish_service.py`**

```python
async def analyze_pgn(pgn: str, depth: int = 15) -> list[int]
```

- Uses `stockfish` Python package (wraps the binary)
- Returns centipawn evals from White's perspective after each ply
- Runs in a thread pool executor to avoid blocking the event loop
- Raises `StockfishError` on failure

**`services/import_job.py`**

```python
async def run_import(job_id: str, database_url: str) -> None
```

Orchestration loop:
1. Open own `AsyncSession` (not the request session)
2. Load `ImportJob`, set `status = "running"`
3. Call `fetch_*_games()` for the platform
4. Set `job.total_games = len(games)`
5. For each `RawGame`:
   - Skip if `Game` with same `platform_game_id` already exists (idempotent)
   - If `eval_per_ply is None`: call `stockfish_service.analyze_pgn()`
   - Call `extract_features()` → `run_tilt_detector()`
   - Save `Game` + `TiltReport`
   - Increment `job.processed_games`, commit
6. Set `status = "done"`, set `finished_at`
7. On any unrecoverable error: set `status = "failed"`, store `error` message

### New Router: `routers/imports.py`

```
POST /api/import
  Body: { platform: "chess.com"|"lichess", username: str, period_days: 30|90 }
  Creates ImportJob, fires asyncio.create_task(run_import(job_id, db_url))
  Returns: { job_id: str }

GET /api/import/{job_id}
  Returns: { status, total_games, processed_games, error, finished_at }

GET /api/import
  Returns: list of ImportJob records for the current user (latest first)
```

### Deduplication

`Game` model gets a new nullable column `platform_game_id: str | None`. Before saving, check:
```python
existing = await db.execute(select(Game).where(Game.platform_game_id == raw.platform_game_id))
if existing.scalar_one_or_none():
    continue
```
This makes re-imports idempotent.

## Frontend

### New Component: `ImportGames`

Modal form with:
- **Platform toggle:** Chess.com | Lichess
- **Username input:** text field
- **Period radio:** 30 days | 90 days
- **Submit button:** "Import games"

On submit:
1. `POST /api/import` → receive `job_id`
2. Switch to progress view: progress bar + "X / Y games analyzed"
3. Poll `GET /api/import/{job_id}` every 1 second
4. On `status === "done"`: show "Done! X games imported" + "View games" button → navigate to `/games`
5. On `status === "failed"`: show error message

### Placement

"Import games" button added to the games history page (`/games`) — primary action when the list is empty, secondary button when games exist.

## Data Flow

```
User submits form
  → POST /api/import { platform, username, period_days }
  → Backend: create ImportJob(status=pending)
  → asyncio.create_task(run_import(job_id, db_url))
  → return { job_id }

Frontend polls GET /api/import/{job_id} every 1s
  → shows processed_games / total_games progress

run_import (background):
  → fetch games from platform API
  → for each game:
      eval_per_ply = lichess_eval OR stockfish_service.analyze_pgn()
      extract_features() → run_tilt_detector()
      save Game + TiltReport
      job.processed_games += 1
  → job.status = "done"

Frontend: poll returns done
  → "X games imported"
  → user visits /games → sees imported games
  → AI coach can reference them in future sessions via existing GameSummary pipeline
```

## Known Limitations

- **Server restart during import:** if the process restarts while `status="running"`, the job stays stuck. On startup, any jobs with `status="running"` should be reset to `status="failed"` with error "Server restarted during import".
- **progress before fetch completes:** `total_games` is 0 until the platform API responds; frontend shows "Fetching games..." until `total_games > 0`.

## Error Handling

- **Unknown username:** platform returns 404 → `ImportJob.status = "failed"`, error = "Username not found on {platform}"
- **No games in period:** `total_games = 0`, `status = "done"` immediately, frontend shows "No games found in this period"
- **Stockfish failure on single game:** log warning, skip game, continue (don't fail the whole job)
- **Network error mid-import:** catch exception, set `status = "failed"` with message

## Testing

- `services/chess_import.py`: mock `httpx` responses, test clock parsing and result mapping
- `services/stockfish_service.py`: mock `stockfish` binary calls, test eval output parsing
- `services/import_job.py`: mock both fetch and stockfish, verify Game/TiltReport saved correctly
- `routers/imports.py`: integration test via `authed_client`, mock `run_import` task

## Dependencies

New Python packages needed:
- `stockfish` — Python wrapper for Stockfish binary
- `httpx` — async HTTP client for platform API calls (already likely in project or add)

Stockfish binary must be installed on the server (`apt install stockfish` or equivalent).
