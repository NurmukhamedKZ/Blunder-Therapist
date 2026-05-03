# Chess.com & Lichess Game Import — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Allow users to import past games from Chess.com and Lichess by username, running full Stockfish + tilt detection analysis in the background.

**Architecture:** `POST /api/import` creates an `ImportJob` row and fires `asyncio.create_task(run_import(job_id))`. The background task fetches PGNs from platform APIs, runs Stockfish for missing evals, then calls the existing `extract_features` + `run_tilt_detector` pipeline. Frontend polls `GET /api/import/{job_id}` every 1 second.

**Tech Stack:** Python `stockfish` package (new dep), `httpx` (already installed), `asyncio.create_task`, Next.js `setInterval` polling.

---

## File Map

**Backend — create:**
- `backend/app/services/chess_import.py` — PGN clock/eval parsing + platform fetch functions
- `backend/app/services/stockfish_service.py` — async Stockfish wrapper
- `backend/app/services/import_job.py` — background job orchestrator
- `backend/app/routers/imports.py` — REST endpoints for job management
- `backend/tests/test_chess_import.py`
- `backend/tests/test_stockfish_service.py`
- `backend/tests/test_import_job.py`
- `backend/tests/test_import_router.py`

**Backend — modify:**
- `backend/app/models.py` — add `platform_game_id` to `Game`, add `ImportJob` model
- `backend/app/schemas/api.py` — add import request/response schemas
- `backend/app/main.py` — register imports router, reset stuck jobs on startup
- `backend/pyproject.toml` — add `stockfish` dependency

**Frontend — create:**
- `frontend/src/components/ImportGames.tsx` — modal form + progress UI

**Frontend — modify:**
- `frontend/src/lib/api.ts` — add `importApi` object
- `frontend/src/app/page.tsx` — add "Import Games" card

---

## Task 1: Add `stockfish` dependency + DB model changes

**Files:**
- Modify: `backend/pyproject.toml`
- Modify: `backend/app/models.py`

- [ ] **Step 1: Add `stockfish` to pyproject.toml**

Open `backend/pyproject.toml`. Add `"stockfish>=3.28.0"` to the `dependencies` list:

```toml
dependencies = [
    "aiosqlite==0.20.0",
    "asyncpg>=0.31.0",
    "chess>=1.11.2",
    "fastapi>=0.136.1",
    "greenlet>=3.5.0",
    "httpx>=0.28.1",
    "langchain>=1.2.17",
    "langchain-openai>=1.2.1",
    "langgraph>=1.1.10",
    "langgraph-checkpoint-postgres>=3.0.5",
    "pydantic-settings>=2.14.0",
    "pytest==8.3.5",
    "pytest-asyncio==0.24.0",
    "python-chess>=1.999",
    "python-jose[cryptography]>=3.5.0",
    "sqlalchemy>=2.0.49",
    "sse-starlette>=3.4.1",
    "stockfish>=3.28.0",
    "structlog>=24.1.0",
    "uvicorn[standard]>=0.46.0",
]
```

- [ ] **Step 2: Install dependency**

```bash
cd backend
uv sync
```

Expected: resolves and installs `stockfish` package. The Stockfish binary itself must be installed separately on the host (`brew install stockfish` on Mac, `apt install stockfish` on Linux).

- [ ] **Step 3: Add `platform_game_id` to `Game` and create `ImportJob` in `models.py`**

In `backend/app/models.py`, add `platform_game_id` field to `Game` (after `result`):

```python
    result: Mapped[str] = mapped_column(String, nullable=False)
    platform_game_id: Mapped[str | None] = mapped_column(String, nullable=True, index=True)
    played_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=_now)
```

Then add the `ImportJob` class at the end of the file:

```python
class ImportJob(Base):
    __tablename__ = "import_jobs"

    id: Mapped[str] = mapped_column(String, primary_key=True, default=_uuid)
    user_id: Mapped[str] = mapped_column(String, ForeignKey("profiles.user_id"), nullable=False, index=True)
    platform: Mapped[str] = mapped_column(String, nullable=False)
    username: Mapped[str] = mapped_column(String, nullable=False)
    period_days: Mapped[int] = mapped_column(Integer, nullable=False)
    status: Mapped[str] = mapped_column(String, nullable=False, default="pending")
    total_games: Mapped[int] = mapped_column(Integer, nullable=False, default=0)
    processed_games: Mapped[int] = mapped_column(Integer, nullable=False, default=0)
    error: Mapped[str | None] = mapped_column(String, nullable=True)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=_now)
    finished_at: Mapped[datetime | None] = mapped_column(DateTime(timezone=True), nullable=True)
```

- [ ] **Step 4: Add production migration note**

For live Postgres, run these manually (SQLite test DB is recreated automatically):

```sql
ALTER TABLE games ADD COLUMN platform_game_id VARCHAR;
CREATE INDEX ix_games_platform_game_id ON games (platform_game_id);

CREATE TABLE import_jobs (
    id VARCHAR PRIMARY KEY,
    user_id VARCHAR NOT NULL REFERENCES profiles(user_id),
    platform VARCHAR NOT NULL,
    username VARCHAR NOT NULL,
    period_days INTEGER NOT NULL,
    status VARCHAR NOT NULL DEFAULT 'pending',
    total_games INTEGER NOT NULL DEFAULT 0,
    processed_games INTEGER NOT NULL DEFAULT 0,
    error VARCHAR,
    created_at TIMESTAMPTZ DEFAULT now(),
    finished_at TIMESTAMPTZ
);
CREATE INDEX ix_import_jobs_user_id ON import_jobs (user_id);
```

- [ ] **Step 5: Commit**

```bash
cd backend
git add app/models.py pyproject.toml uv.lock
git commit -m "feat: add ImportJob model and platform_game_id to Game"
```

---

## Task 2: Add import schemas

**Files:**
- Modify: `backend/app/schemas/api.py`

- [ ] **Step 1: Add schemas to `schemas/api.py`**

Append to the end of `backend/app/schemas/api.py`:

```python
# ---------- Game Import ----------

class ImportRequest(BaseModel):
    platform: Literal["chess.com", "lichess"]
    username: str
    period_days: Literal[30, 90]


class ImportJobStatus(BaseModel):
    job_id: str
    status: Literal["pending", "running", "done", "failed"]
    total_games: int
    processed_games: int
    error: str | None
    finished_at: datetime | None


class ImportJobListResponse(BaseModel):
    jobs: list[ImportJobStatus]
```

- [ ] **Step 2: Commit**

```bash
cd backend
git add app/schemas/api.py
git commit -m "feat: add import request/response schemas"
```

---

## Task 3: Create `services/chess_import.py` — parsing logic (TDD)

**Files:**
- Create: `backend/app/services/chess_import.py`
- Create: `backend/tests/test_chess_import.py`

- [ ] **Step 1: Write failing tests for parsing helpers**

Create `backend/tests/test_chess_import.py`:

```python
"""Tests for chess platform import helpers."""
import pytest
from app.services.chess_import import parse_clock_times, parse_eval_annotations, map_chesscom_result


PGN_WITH_CLOCKS = """[Event "Live Chess"]
[White "alice"]
[Black "bob"]
[TimeControl "600"]

1. e4 {[%clk 0:09:55]} e5 {[%clk 0:09:58]} 2. Nf3 {[%clk 0:09:44]} Nc6 {[%clk 0:09:50]} *
"""

PGN_WITH_EVALS = """[Event "Rated Blitz game"]
[White "alice"]
[Black "bob"]

1. e4 {[%eval 0.20]} e5 {[%eval 0.15]} 2. Nf3 {[%eval 0.45]} Nc6 {[%eval 0.30]} *
"""

PGN_WITH_MATE_EVAL = """[Event "Rated Blitz game"]
[White "alice"]
[Black "bob"]

1. e4 {[%eval #3]} e5 {[%eval #-2]} *
"""

PGN_NO_ANNOTATIONS = """[Event "Live Chess"]
[White "alice"]
[Black "bob"]

1. e4 e5 2. Nf3 Nc6 *
"""


def test_parse_clock_times_returns_per_ply_seconds():
    times = parse_clock_times(PGN_WITH_CLOCKS)
    # ply 0 (white e4): no previous same-color clock → 0.0
    # ply 1 (black e5): no previous same-color clock → 0.0
    # ply 2 (white Nf3): 9:55 - 9:44 = 11 seconds
    # ply 3 (black Nc6): 9:58 - 9:50 = 8 seconds
    assert len(times) == 4
    assert times[0] == 0.0
    assert times[1] == 0.0
    assert times[2] == pytest.approx(11.0)
    assert times[3] == pytest.approx(8.0)


def test_parse_clock_times_no_annotations_returns_zeros():
    times = parse_clock_times(PGN_NO_ANNOTATIONS)
    assert times == []


def test_parse_eval_annotations_centipawns():
    evals = parse_eval_annotations(PGN_WITH_EVALS)
    assert evals is not None
    assert len(evals) == 4
    assert evals[0] == 20   # 0.20 → 20 cp
    assert evals[1] == 15
    assert evals[2] == 45
    assert evals[3] == 30


def test_parse_eval_annotations_mate():
    evals = parse_eval_annotations(PGN_WITH_MATE_EVAL)
    assert evals is not None
    assert evals[0] == 10_000   # #3 = white forced mate → +10000
    assert evals[1] == -10_000  # #-2 = black forced mate → -10000


def test_parse_eval_annotations_no_evals_returns_none():
    evals = parse_eval_annotations(PGN_NO_ANNOTATIONS)
    assert evals is None


def test_map_chesscom_result_win():
    assert map_chesscom_result("win", "white") == ("white", "win")


def test_map_chesscom_result_resigned():
    # player resigned means the player (black) lost
    assert map_chesscom_result("resigned", "black") == ("black", "loss")


def test_map_chesscom_result_checkmated():
    # white was checkmated → white loses
    assert map_chesscom_result("checkmated", "white") == ("white", "loss")


def test_map_chesscom_result_draw():
    assert map_chesscom_result("agreed", "white") == ("white", "draw")
    assert map_chesscom_result("stalemate", "black") == ("black", "draw")
    assert map_chesscom_result("insufficient", "white") == ("white", "draw")
    assert map_chesscom_result("50move", "black") == ("black", "draw")
    assert map_chesscom_result("repetition", "white") == ("white", "draw")
```

- [ ] **Step 2: Run tests to confirm they fail**

```bash
cd backend
pytest tests/test_chess_import.py -v
```

Expected: `ImportError` or `ModuleNotFoundError` — `chess_import` doesn't exist yet.

- [ ] **Step 3: Create `services/chess_import.py` with parsing helpers**

Create `backend/app/services/chess_import.py`:

```python
"""Fetch and parse games from Chess.com and Lichess public APIs."""
import json
import re
from dataclasses import dataclass
from datetime import datetime, timezone
from typing import Literal

import httpx
import structlog

log = structlog.get_logger()

_CHESSCOM_WIN_RESULTS = {"win"}
_CHESSCOM_LOSS_RESULTS = {"resigned", "checkmated", "timeout", "abandoned", "bughousepartnerlose"}
_CHESSCOM_DRAW_RESULTS = {"agreed", "stalemate", "insufficient", "50move", "repetition", "timevsinsufficient"}

_CLK_RE = re.compile(r'\[%clk\s+(\d+):(\d+):(\d+)\]')
_EVAL_RE = re.compile(r'\[%eval\s+(#-?\d+|[+-]?\d+\.?\d*)\]')

MATE_CP = 10_000


@dataclass
class RawGame:
    pgn: str
    platform: str
    platform_game_id: str
    player_color: Literal["white", "black"]
    result: Literal["win", "loss", "draw"]
    time_per_ply: list[float]
    eval_per_ply: list[int] | None


def parse_clock_times(pgn: str) -> list[float]:
    """Extract per-ply thinking time (seconds) from [%clk] annotations.

    Returns empty list if no annotations found.
    For the first move of each color there is no prior same-color clock,
    so those entries are 0.0.
    """
    clocks = []
    for m in _CLK_RE.finditer(pgn):
        h, mi, s = int(m.group(1)), int(m.group(2)), int(m.group(3))
        clocks.append(h * 3600 + mi * 60 + s)

    if not clocks:
        return []

    times: list[float] = []
    for i, clk in enumerate(clocks):
        if i < 2:
            times.append(0.0)
        else:
            times.append(float(clocks[i - 2] - clk))

    return times


def parse_eval_annotations(pgn: str) -> list[int] | None:
    """Extract per-ply engine evals (centipawns, White's POV) from [%eval] annotations.

    Returns None if no annotations found (Stockfish will be used instead).
    Forced-mate annotations (#N) map to ±MATE_CP.
    """
    evals: list[int] = []
    for m in _EVAL_RE.finditer(pgn):
        val = m.group(1)
        if val.startswith("#"):
            mate_in = int(val[1:])
            evals.append(MATE_CP if mate_in > 0 else -MATE_CP)
        else:
            evals.append(int(float(val) * 100))

    return evals if evals else None


def map_chesscom_result(
    result_str: str,
    player_color: Literal["white", "black"],
) -> tuple[Literal["white", "black"], Literal["win", "loss", "draw"]]:
    """Map Chess.com result string to (player_color, 'win'|'loss'|'draw')."""
    if result_str in _CHESSCOM_WIN_RESULTS:
        return player_color, "win"
    if result_str in _CHESSCOM_LOSS_RESULTS:
        return player_color, "loss"
    return player_color, "draw"


async def fetch_chesscom_games(
    username: str,
    since: datetime,
    until: datetime,
) -> list[RawGame]:
    """Fetch games from Chess.com public API for the given date range.

    Skips 'daily' (correspondence) games — no real clock data.
    Raises ValueError if username not found.
    """
    # Collect all year/month pairs in the range
    months: list[tuple[int, int]] = []
    cur = since.replace(day=1)
    while cur <= until:
        months.append((cur.year, cur.month))
        if cur.month == 12:
            cur = cur.replace(year=cur.year + 1, month=1)
        else:
            cur = cur.replace(month=cur.month + 1)

    games: list[RawGame] = []
    async with httpx.AsyncClient(timeout=30) as client:
        for year, month in months:
            url = f"https://api.chess.com/pub/player/{username}/games/{year}/{month:02d}"
            resp = await client.get(url)
            if resp.status_code == 404:
                raise ValueError(f"Username not found on chess.com: {username}")
            resp.raise_for_status()
            data = resp.json()

            for g in data.get("games", []):
                if g.get("time_class") == "daily":
                    continue
                end_ts = g.get("end_time", 0)
                end_dt = datetime.fromtimestamp(end_ts, tz=timezone.utc)
                if not (since <= end_dt <= until):
                    continue

                white_name = g["white"]["username"].lower()
                if white_name == username.lower():
                    player_color: Literal["white", "black"] = "white"
                    result_str = g["white"]["result"]
                else:
                    player_color = "black"
                    result_str = g["black"]["result"]

                _, result = map_chesscom_result(result_str, player_color)
                pgn = g["pgn"]
                platform_game_id = "chesscom_" + g["url"].rstrip("/").split("/")[-1]

                games.append(RawGame(
                    pgn=pgn,
                    platform="chess.com",
                    platform_game_id=platform_game_id,
                    player_color=player_color,
                    result=result,
                    time_per_ply=parse_clock_times(pgn),
                    eval_per_ply=None,
                ))

    log.info("chesscom_fetch_done", username=username, count=len(games))
    return games


async def fetch_lichess_games(
    username: str,
    since: datetime,
    until: datetime,
) -> list[RawGame]:
    """Fetch games from Lichess public API for the given date range.

    Uses evals from Lichess analysis when available; otherwise eval_per_ply is None.
    Raises ValueError if username not found.
    """
    since_ms = int(since.timestamp() * 1000)
    until_ms = int(until.timestamp() * 1000)
    url = f"https://lichess.org/api/games/user/{username}"
    params = {
        "since": since_ms,
        "until": until_ms,
        "evals": "true",
        "clocks": "true",
        "perfType": "bullet,blitz,rapid,classical",
    }
    headers = {"Accept": "application/x-ndjson"}

    async with httpx.AsyncClient(timeout=120) as client:
        resp = await client.get(url, params=params, headers=headers)
        if resp.status_code == 404:
            raise ValueError(f"Username not found on lichess: {username}")
        resp.raise_for_status()
        body = resp.text

    games: list[RawGame] = []
    for line in body.strip().split("\n"):
        line = line.strip()
        if not line:
            continue
        game_data = json.loads(line)
        raw = _parse_lichess_game(game_data, username)
        if raw:
            games.append(raw)

    log.info("lichess_fetch_done", username=username, count=len(games))
    return games


def _parse_lichess_game(data: dict, username: str) -> RawGame | None:
    players = data.get("players", {})
    white_name = (players.get("white", {}).get("user", {}).get("name") or "").lower()

    if white_name == username.lower():
        player_color: Literal["white", "black"] = "white"
    else:
        player_color = "black"

    winner = data.get("winner")
    if winner is None:
        result: Literal["win", "loss", "draw"] = "draw"
    elif winner == player_color:
        result = "win"
    else:
        result = "loss"

    pgn = data.get("pgn", "")
    if not pgn:
        return None

    return RawGame(
        pgn=pgn,
        platform="lichess",
        platform_game_id="lichess_" + data["id"],
        player_color=player_color,
        result=result,
        time_per_ply=parse_clock_times(pgn),
        eval_per_ply=parse_eval_annotations(pgn),
    )
```

- [ ] **Step 4: Run tests — all should pass**

```bash
cd backend
pytest tests/test_chess_import.py -v
```

Expected: all 9 tests PASS.

- [ ] **Step 5: Commit**

```bash
cd backend
git add app/services/chess_import.py tests/test_chess_import.py
git commit -m "feat: add chess_import service with clock/eval parsing"
```

---

## Task 4: Create `services/stockfish_service.py` (TDD)

**Files:**
- Create: `backend/app/services/stockfish_service.py`
- Create: `backend/tests/test_stockfish_service.py`

- [ ] **Step 1: Write failing test**

Create `backend/tests/test_stockfish_service.py`:

```python
"""Tests for stockfish_service — mocks the binary."""
from unittest.mock import patch, MagicMock
import pytest
from app.services.stockfish_service import _analyze_pgn_sync, MATE_CP

SIMPLE_PGN = """[Event "Test"]
[White "a"]
[Black "b"]

1. e4 e5 2. Nf3 Nc6 *
"""


def _make_stockfish_mock(eval_sequence: list[dict]) -> MagicMock:
    """Returns a Stockfish mock whose get_evaluation() cycles through eval_sequence."""
    sf = MagicMock()
    sf.get_evaluation.side_effect = eval_sequence
    return sf


def test_analyze_pgn_sync_centipawn_evals():
    evals_from_sf = [
        {"type": "cp", "value": 20},
        {"type": "cp", "value": 15},
        {"type": "cp", "value": 45},
        {"type": "cp", "value": 30},
    ]
    mock_sf = _make_stockfish_mock(evals_from_sf)

    with patch("app.services.stockfish_service.Stockfish", return_value=mock_sf):
        result = _analyze_pgn_sync(SIMPLE_PGN, depth=10)

    assert result == [20, 15, 45, 30]
    assert mock_sf.set_fen_position.call_count == 4


def test_analyze_pgn_sync_mate_becomes_max_cp():
    evals_from_sf = [
        {"type": "cp", "value": 200},
        {"type": "mate", "value": 3},    # white forces mate
        {"type": "mate", "value": -2},   # black forces mate
        {"type": "cp", "value": -100},
    ]
    mock_sf = _make_stockfish_mock(evals_from_sf)

    with patch("app.services.stockfish_service.Stockfish", return_value=mock_sf):
        result = _analyze_pgn_sync(SIMPLE_PGN, depth=10)

    assert result[1] == MATE_CP
    assert result[2] == -MATE_CP
```

- [ ] **Step 2: Run test — confirm it fails**

```bash
cd backend
pytest tests/test_stockfish_service.py -v
```

Expected: `ImportError` — module doesn't exist yet.

- [ ] **Step 3: Create `services/stockfish_service.py`**

Create `backend/app/services/stockfish_service.py`:

```python
"""Async Stockfish wrapper for backend game evaluation."""
import asyncio
import io

import chess
import chess.pgn
import structlog
from stockfish import Stockfish

log = structlog.get_logger()

MATE_CP = 10_000
_DEFAULT_DEPTH = 15


async def analyze_pgn(pgn: str, depth: int = _DEFAULT_DEPTH) -> list[int]:
    """Return per-ply Stockfish evals (centipawns, White's POV) for a game PGN.

    Runs in a thread pool executor to avoid blocking the event loop.
    Raises StockfishException if the binary is not installed.
    """
    loop = asyncio.get_event_loop()
    return await loop.run_in_executor(None, _analyze_pgn_sync, pgn, depth)


def _analyze_pgn_sync(pgn: str, depth: int) -> list[int]:
    sf = Stockfish(depth=depth)
    game = chess.pgn.read_game(io.StringIO(pgn))
    board = game.board()
    evals: list[int] = []

    for node in game.mainline():
        board.push(node.move)
        sf.set_fen_position(board.fen())
        ev = sf.get_evaluation()
        if ev["type"] == "cp":
            evals.append(ev["value"])
        else:
            evals.append(MATE_CP if ev["value"] > 0 else -MATE_CP)

    log.debug("stockfish_analysis_done", ply_count=len(evals))
    return evals
```

- [ ] **Step 4: Run tests — all should pass**

```bash
cd backend
pytest tests/test_stockfish_service.py -v
```

Expected: 2 tests PASS.

- [ ] **Step 5: Commit**

```bash
cd backend
git add app/services/stockfish_service.py tests/test_stockfish_service.py
git commit -m "feat: add async Stockfish wrapper service"
```

---

## Task 5: Create `services/import_job.py` (TDD)

**Files:**
- Create: `backend/app/services/import_job.py`
- Create: `backend/tests/test_import_job.py`

- [ ] **Step 1: Write failing tests**

Create `backend/tests/test_import_job.py`:

```python
"""Tests for import_job orchestrator — mocks platform fetch and Stockfish."""
from datetime import datetime, timezone
from unittest.mock import AsyncMock, patch, MagicMock
import pytest

from app.models import ImportJob, Game, TiltReport, Profile
from app.services.chess_import import RawGame
from app.services.import_job import run_import


MINIMAL_PGN = """[Event "Test"]
[White "alice"]
[Black "bob"]
[Result "1-0"]

1. e4 e5 2. Qh5 Nc6 3. Bc4 Nf6 4. Qxf7# 1-0
"""

FAKE_EVALS = [20, -30, 50, -200, 100, 300, 100, 100]
FAKE_TIMES = [5.0, 4.0, 3.0, 2.0, 6.0, 1.0, 4.0, 3.0]

FAKE_RAW_GAME = RawGame(
    pgn=MINIMAL_PGN,
    platform="chess.com",
    platform_game_id="chesscom_999",
    player_color="white",
    result="win",
    time_per_ply=FAKE_TIMES,
    eval_per_ply=FAKE_EVALS,
)

FAKE_TILT = {
    "headline": "Tilt detected",
    "diagnosis": "You rushed",
    "pattern_label": "speed_tilt",
    "evidence_plies": [3],
    "suggestion": "Slow down",
}


@pytest.fixture
async def import_job(db):
    profile = Profile(user_id="test-user-id", plan="free")
    db.add(profile)
    job = ImportJob(
        user_id="test-user-id",
        platform="chess.com",
        username="alice",
        period_days=30,
        status="pending",
        total_games=0,
        processed_games=0,
    )
    db.add(job)
    await db.commit()
    await db.refresh(job)
    return job


@pytest.mark.asyncio
async def test_run_import_saves_game_and_report(db, import_job):
    with (
        patch("app.services.import_job.fetch_chesscom_games", new=AsyncMock(return_value=[FAKE_RAW_GAME])),
        patch("app.services.import_job.run_tilt_detector", new=AsyncMock(return_value=FAKE_TILT)),
        patch("app.services.import_job.AsyncSessionLocal", return_value=_session_ctx(db)),
    ):
        await run_import(import_job.id)

    await db.refresh(import_job)
    assert import_job.status == "done"
    assert import_job.total_games == 1
    assert import_job.processed_games == 1

    from sqlalchemy import select
    games = (await db.execute(select(Game).where(Game.platform_game_id == "chesscom_999"))).scalars().all()
    assert len(games) == 1
    reports = (await db.execute(select(TiltReport).where(TiltReport.game_id == games[0].id))).scalars().all()
    assert len(reports) == 1
    assert reports[0].headline == "Tilt detected"


@pytest.mark.asyncio
async def test_run_import_skips_duplicate(db, import_job):
    with (
        patch("app.services.import_job.fetch_chesscom_games", new=AsyncMock(return_value=[FAKE_RAW_GAME, FAKE_RAW_GAME])),
        patch("app.services.import_job.run_tilt_detector", new=AsyncMock(return_value=FAKE_TILT)),
        patch("app.services.import_job.AsyncSessionLocal", return_value=_session_ctx(db)),
    ):
        await run_import(import_job.id)

    from sqlalchemy import select
    games = (await db.execute(select(Game).where(Game.platform_game_id == "chesscom_999"))).scalars().all()
    assert len(games) == 1  # second import skipped


@pytest.mark.asyncio
async def test_run_import_marks_failed_on_fetch_error(db, import_job):
    with (
        patch("app.services.import_job.fetch_chesscom_games", new=AsyncMock(side_effect=ValueError("Username not found on chess.com: alice"))),
        patch("app.services.import_job.AsyncSessionLocal", return_value=_session_ctx(db)),
    ):
        await run_import(import_job.id)

    await db.refresh(import_job)
    assert import_job.status == "failed"
    assert "Username not found" in import_job.error


def _session_ctx(db):
    """Wrap the test db session as an async context manager."""
    class _Ctx:
        async def __aenter__(self):
            return db
        async def __aexit__(self, *args):
            pass
    return _Ctx()
```

- [ ] **Step 2: Run tests — confirm failure**

```bash
cd backend
pytest tests/test_import_job.py -v
```

Expected: `ImportError` — module doesn't exist yet.

- [ ] **Step 3: Create `services/import_job.py`**

Create `backend/app/services/import_job.py`:

```python
"""Background job: fetch games from Chess.com/Lichess and run full tilt analysis."""
from datetime import datetime, timedelta, timezone

import structlog
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from app.database import AsyncSessionLocal
from app.models import Game, ImportJob, TiltReport
from app.services.chess_import import RawGame, fetch_chesscom_games, fetch_lichess_games
from app.services.features import extract_features
from app.services.stockfish_service import analyze_pgn
from app.services.tilt_detector import run_tilt_detector

log = structlog.get_logger()


async def run_import(job_id: str) -> None:
    """Entry point for the background import task."""
    try:
        async with AsyncSessionLocal() as db:
            await _run_import(db, job_id)
    except Exception as e:
        log.error("import_job_failed", job_id=job_id, exc_info=True)
        # Open a fresh session — the main session may be in a dirty state
        await _fail_job(job_id, str(e))


async def _run_import(db: AsyncSession, job_id: str) -> None:
    result = await db.execute(select(ImportJob).where(ImportJob.id == job_id))
    job = result.scalar_one()
    job.status = "running"
    await db.commit()

    until = datetime.now(timezone.utc)
    since = until - timedelta(days=job.period_days)

    if job.platform == "chess.com":
        games = await fetch_chesscom_games(job.username, since, until)
    else:
        games = await fetch_lichess_games(job.username, since, until)

    job.total_games = len(games)
    await db.commit()

    if not games:
        job.status = "done"
        job.finished_at = datetime.now(timezone.utc)
        await db.commit()
        return

    for raw in games:
        try:
            await _process_game(db, job, raw)
        except Exception as e:
            log.warning("import_game_skipped", platform_game_id=raw.platform_game_id, error=str(e))
            await db.rollback()  # clear any unflushed state before next game

    job.status = "done"
    job.finished_at = datetime.now(timezone.utc)
    await db.commit()


async def _process_game(db: AsyncSession, job: ImportJob, raw: RawGame) -> None:
    existing = await db.execute(
        select(Game).where(Game.platform_game_id == raw.platform_game_id)
    )
    if existing.scalar_one_or_none():
        job.processed_games += 1
        await db.commit()
        return

    eval_per_ply = raw.eval_per_ply if raw.eval_per_ply is not None else await analyze_pgn(raw.pgn)

    features = extract_features(
        pgn=raw.pgn,
        eval_per_ply=eval_per_ply,
        time_per_ply=raw.time_per_ply,
        player_color=raw.player_color,
        result=raw.result,
    )
    tilt_result = await run_tilt_detector(features)

    game = Game(
        user_id=job.user_id,
        pgn=raw.pgn,
        eval_per_ply=eval_per_ply,
        time_per_ply=raw.time_per_ply,
        player_color=raw.player_color,
        result=raw.result,
        platform_game_id=raw.platform_game_id,
    )
    db.add(game)
    await db.flush()
    db.add(TiltReport(game_id=game.id, **tilt_result))

    job.processed_games += 1
    await db.commit()
    log.info("import_game_done", platform_game_id=raw.platform_game_id, job_id=job.id)


async def _fail_job(job_id: str, error: str) -> None:
    try:
        async with AsyncSessionLocal() as db:
            result = await db.execute(select(ImportJob).where(ImportJob.id == job_id))
            job = result.scalar_one_or_none()
            if job:
                job.status = "failed"
                job.error = error
                job.finished_at = datetime.now(timezone.utc)
                await db.commit()
    except Exception:
        log.error("fail_job_update_failed", job_id=job_id, exc_info=True)
```

- [ ] **Step 4: Run tests — all pass**

```bash
cd backend
pytest tests/test_import_job.py -v
```

Expected: 3 tests PASS.

- [ ] **Step 5: Commit**

```bash
cd backend
git add app/services/import_job.py tests/test_import_job.py
git commit -m "feat: add import_job background orchestrator"
```

---

## Task 6: Create `routers/imports.py` (TDD)

**Files:**
- Create: `backend/app/routers/imports.py`
- Create: `backend/tests/test_import_router.py`

- [ ] **Step 1: Write failing tests**

Create `backend/tests/test_import_router.py`:

```python
"""Tests for /api/import endpoints."""
from unittest.mock import patch, AsyncMock
import pytest

from app.models import Profile, ImportJob


@pytest.fixture(autouse=True)
async def profile(db):
    db.add(Profile(user_id="test-user-id", plan="free"))
    await db.commit()


@pytest.mark.asyncio
async def test_create_import_job_returns_202(authed_client):
    with patch("app.routers.imports.asyncio.create_task"):
        resp = await authed_client.post("/api/import", json={
            "platform": "chess.com",
            "username": "alice",
            "period_days": 30,
        })
    assert resp.status_code == 202
    data = resp.json()
    assert data["status"] == "pending"
    assert data["total_games"] == 0
    assert data["processed_games"] == 0
    assert "job_id" in data


@pytest.mark.asyncio
async def test_get_import_job_status(authed_client, db):
    with patch("app.routers.imports.asyncio.create_task"):
        create_resp = await authed_client.post("/api/import", json={
            "platform": "lichess",
            "username": "bob",
            "period_days": 90,
        })
    job_id = create_resp.json()["job_id"]

    resp = await authed_client.get(f"/api/import/{job_id}")
    assert resp.status_code == 200
    data = resp.json()
    assert data["job_id"] == job_id
    assert data["status"] == "pending"


@pytest.mark.asyncio
async def test_get_import_job_not_found(authed_client):
    resp = await authed_client.get("/api/import/nonexistent-id")
    assert resp.status_code == 404


@pytest.mark.asyncio
async def test_list_import_jobs(authed_client):
    with patch("app.routers.imports.asyncio.create_task"):
        await authed_client.post("/api/import", json={"platform": "chess.com", "username": "alice", "period_days": 30})
        await authed_client.post("/api/import", json={"platform": "lichess", "username": "alice", "period_days": 90})

    resp = await authed_client.get("/api/import")
    assert resp.status_code == 200
    data = resp.json()
    assert len(data["jobs"]) == 2
```

- [ ] **Step 2: Run tests — confirm failure**

```bash
cd backend
pytest tests/test_import_router.py -v
```

Expected: `404 Not Found` for all routes — router not registered yet.

- [ ] **Step 3: Create `routers/imports.py`**

Create `backend/app/routers/imports.py`:

```python
"""Import job endpoints — create and poll background import jobs."""
import asyncio

import structlog
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from app.database import get_db
from app.dependencies import CurrentUser, get_current_user
from app.models import ImportJob
from app.schemas.api import ImportJobListResponse, ImportJobStatus, ImportRequest
from app.services.import_job import run_import

router = APIRouter(prefix="/api/import", tags=["import"])
log = structlog.get_logger()


def _job_to_status(job: ImportJob) -> ImportJobStatus:
    return ImportJobStatus(
        job_id=job.id,
        status=job.status,
        total_games=job.total_games,
        processed_games=job.processed_games,
        error=job.error,
        finished_at=job.finished_at,
    )


@router.post("", response_model=ImportJobStatus, status_code=202)
async def create_import_job(
    req: ImportRequest,
    user: CurrentUser = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
):
    job = ImportJob(
        user_id=user.user_id,
        platform=req.platform,
        username=req.username,
        period_days=req.period_days,
    )
    db.add(job)
    await db.commit()
    await db.refresh(job)
    asyncio.create_task(run_import(job.id))
    log.info("import_job_created", job_id=job.id, platform=req.platform, username=req.username)
    return _job_to_status(job)


@router.get("/{job_id}", response_model=ImportJobStatus)
async def get_import_job(
    job_id: str,
    user: CurrentUser = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
):
    result = await db.execute(
        select(ImportJob).where(ImportJob.id == job_id, ImportJob.user_id == user.user_id)
    )
    job = result.scalar_one_or_none()
    if job is None:
        raise HTTPException(status_code=404, detail="Import job not found")
    return _job_to_status(job)


@router.get("", response_model=ImportJobListResponse)
async def list_import_jobs(
    user: CurrentUser = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
):
    result = await db.execute(
        select(ImportJob)
        .where(ImportJob.user_id == user.user_id)
        .order_by(ImportJob.created_at.desc())
    )
    jobs = result.scalars().all()
    return ImportJobListResponse(jobs=[_job_to_status(j) for j in jobs])
```

- [ ] **Step 4: Run tests — all pass**

```bash
cd backend
pytest tests/test_import_router.py -v
```

Expected: 4 tests PASS.

- [ ] **Step 5: Commit**

```bash
cd backend
git add app/routers/imports.py tests/test_import_router.py
git commit -m "feat: add import router with create/get/list endpoints"
```

---

## Task 7: Register router and reset stuck jobs in `main.py`

**Files:**
- Modify: `backend/app/main.py`

- [ ] **Step 1: Add imports and register router**

In `backend/app/main.py`, add the import alongside the other router imports:

```python
from app.routers import imports as imports_router
```

Add `app.include_router(imports_router.router)` next to the other `include_router` calls:

```python
app.include_router(games_router.router)
app.include_router(agent_router.router)
app.include_router(imports_router.router)
```

- [ ] **Step 2: Reset stuck jobs in lifespan**

Inside the `if not os.getenv("TESTING"):` block of the `lifespan` function, add the reset logic right after `create_all`. The full `lifespan` should look like:

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    if not os.getenv("TESTING"):
        from app import models  # noqa: F401
        from app.database import engine, Base, AsyncSessionLocal
        from app.models import ImportJob
        from sqlalchemy import update

        async with engine.begin() as conn:
            await conn.run_sync(Base.metadata.create_all)

        # Reset any jobs that were running when the server last stopped
        async with AsyncSessionLocal() as db:
            await db.execute(
                update(ImportJob)
                .where(ImportJob.status == "running")
                .values(status="failed", error="Server restarted during import")
            )
            await db.commit()

        from app.services.agent import agent_service
        await agent_service.init(settings.database_url, in_memory=False)
        try:
            yield
        finally:
            await agent_service.shutdown()
    else:
        yield
```

- [ ] **Step 3: Run the full test suite to confirm nothing is broken**

```bash
cd backend
pytest tests/ -v
```

Expected: all tests PASS.

- [ ] **Step 4: Commit**

```bash
cd backend
git add app/main.py
git commit -m "feat: register imports router and reset stuck jobs on startup"
```

---

## Task 8: Frontend — add `importApi` to `lib/api.ts`

**Files:**
- Modify: `frontend/src/lib/api.ts`

- [ ] **Step 1: Add import types and `importApi` to `api.ts`**

Append to the end of `frontend/src/lib/api.ts`:

```typescript
// ---------- Import API ----------

export interface ImportJobStatus {
  job_id: string;
  status: "pending" | "running" | "done" | "failed";
  total_games: number;
  processed_games: number;
  error: string | null;
  finished_at: string | null;
}

export const importApi = {
  create: (platform: "chess.com" | "lichess", username: string, period_days: 30 | 90) =>
    post<ImportJobStatus>("/api/import", { platform, username, period_days }),

  status: (job_id: string) =>
    get<ImportJobStatus>(`/api/import/${job_id}`),

  list: () =>
    get<{ jobs: ImportJobStatus[] }>("/api/import"),
};
```

- [ ] **Step 2: Commit**

```bash
cd frontend
git add src/lib/api.ts
git commit -m "feat: add importApi to api client"
```

---

## Task 9: Create `ImportGames` component

**Files:**
- Create: `frontend/src/components/ImportGames.tsx`

- [ ] **Step 1: Create the component**

Create `frontend/src/components/ImportGames.tsx`:

```tsx
"use client";

import { useState, useEffect, useRef } from "react";
import { Upload, X, CheckCircle, AlertCircle, Loader2 } from "lucide-react";
import { importApi, ImportJobStatus } from "@/lib/api";

interface ImportGamesProps {
  onClose: () => void;
  onDone: () => void;
}

type Step = "form" | "progress" | "done" | "error";

export default function ImportGames({ onClose, onDone }: ImportGamesProps) {
  const [platform, setPlatform] = useState<"chess.com" | "lichess">("lichess");
  const [username, setUsername] = useState("");
  const [periodDays, setPeriodDays] = useState<30 | 90>(30);
  const [step, setStep] = useState<Step>("form");
  const [job, setJob] = useState<ImportJobStatus | null>(null);
  const [errorMsg, setErrorMsg] = useState("");
  const pollRef = useRef<ReturnType<typeof setInterval> | null>(null);

  const stopPolling = () => {
    if (pollRef.current) {
      clearInterval(pollRef.current);
      pollRef.current = null;
    }
  };

  useEffect(() => () => stopPolling(), []);

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    if (!username.trim()) return;
    try {
      const created = await importApi.create(platform, username.trim(), periodDays);
      setJob(created);
      setStep("progress");
      pollRef.current = setInterval(async () => {
        try {
          const updated = await importApi.status(created.job_id);
          setJob(updated);
          if (updated.status === "done") {
            stopPolling();
            setStep("done");
          } else if (updated.status === "failed") {
            stopPolling();
            setErrorMsg(updated.error ?? "Import failed");
            setStep("error");
          }
        } catch {
          // transient poll error — keep polling
        }
      }, 1000);
    } catch (err: unknown) {
      setErrorMsg(err instanceof Error ? err.message : "Failed to start import");
      setStep("error");
    }
  };

  const progress = job && job.total_games > 0
    ? Math.round((job.processed_games / job.total_games) * 100)
    : 0;

  return (
    <div className="fixed inset-0 bg-black/70 flex items-center justify-center z-50 p-4">
      <div className="bg-ink-800 border border-ink-700 rounded-2xl p-6 w-full max-w-md shadow-2xl">
        <div className="flex items-center justify-between mb-6">
          <div className="flex items-center gap-3">
            <div className="w-10 h-10 rounded-xl bg-accent-500/20 flex items-center justify-center border border-accent-500/30">
              <Upload size={20} className="text-accent-500" />
            </div>
            <h2 className="text-lg font-semibold text-white">Import Games</h2>
          </div>
          <button
            onClick={onClose}
            className="text-ink-400 hover:text-white transition-colors"
          >
            <X size={20} />
          </button>
        </div>

        {step === "form" && (
          <form onSubmit={handleSubmit} className="space-y-5">
            <div>
              <label className="text-ink-400 text-sm mb-2 block">Platform</label>
              <div className="flex gap-2">
                {(["lichess", "chess.com"] as const).map((p) => (
                  <button
                    key={p}
                    type="button"
                    onClick={() => setPlatform(p)}
                    className={`flex-1 py-2 rounded-xl text-sm font-medium transition-colors border ${
                      platform === p
                        ? "bg-accent-500 text-white border-accent-500"
                        : "bg-ink-700 text-ink-300 border-ink-600 hover:bg-ink-600"
                    }`}
                  >
                    {p === "lichess" ? "Lichess" : "Chess.com"}
                  </button>
                ))}
              </div>
            </div>

            <div>
              <label className="text-ink-400 text-sm mb-2 block">Username</label>
              <input
                type="text"
                value={username}
                onChange={(e) => setUsername(e.target.value)}
                placeholder={`Your ${platform} username`}
                className="w-full bg-ink-700 border border-ink-600 rounded-xl px-4 py-2.5 text-white placeholder-ink-500 focus:outline-none focus:border-accent-500 transition-colors"
              />
            </div>

            <div>
              <label className="text-ink-400 text-sm mb-2 block">Period</label>
              <div className="flex gap-2">
                {([30, 90] as const).map((d) => (
                  <button
                    key={d}
                    type="button"
                    onClick={() => setPeriodDays(d)}
                    className={`flex-1 py-2 rounded-xl text-sm font-medium transition-colors border ${
                      periodDays === d
                        ? "bg-accent-500 text-white border-accent-500"
                        : "bg-ink-700 text-ink-300 border-ink-600 hover:bg-ink-600"
                    }`}
                  >
                    {d} days
                  </button>
                ))}
              </div>
            </div>

            <button
              type="submit"
              disabled={!username.trim()}
              className="w-full py-3 rounded-xl bg-accent-500 text-white font-semibold hover:bg-accent-400 disabled:opacity-40 disabled:cursor-not-allowed transition-colors"
            >
              Import games
            </button>
          </form>
        )}

        {step === "progress" && job && (
          <div className="space-y-4">
            <div className="flex items-center gap-3 text-ink-300">
              <Loader2 size={18} className="animate-spin text-accent-500 shrink-0" />
              <span className="text-sm">
                {job.total_games === 0
                  ? "Fetching games…"
                  : `Analyzing ${job.processed_games} / ${job.total_games} games`}
              </span>
            </div>
            {job.total_games > 0 && (
              <div className="w-full bg-ink-700 rounded-full h-2">
                <div
                  className="bg-accent-500 h-2 rounded-full transition-all duration-300"
                  style={{ width: `${progress}%` }}
                />
              </div>
            )}
            <p className="text-ink-500 text-xs">
              Each game is evaluated by Stockfish — this may take a few minutes.
            </p>
          </div>
        )}

        {step === "done" && job && (
          <div className="space-y-4 text-center">
            <CheckCircle size={40} className="text-green-400 mx-auto" />
            <p className="text-white font-semibold text-lg">
              {job.processed_games} game{job.processed_games !== 1 ? "s" : ""} imported!
            </p>
            <p className="text-ink-400 text-sm">
              The AI coach now has access to your {platform} history.
            </p>
            <button
              onClick={onDone}
              className="w-full py-3 rounded-xl bg-accent-500 text-white font-semibold hover:bg-accent-400 transition-colors"
            >
              View games
            </button>
          </div>
        )}

        {step === "error" && (
          <div className="space-y-4 text-center">
            <AlertCircle size={40} className="text-red-400 mx-auto" />
            <p className="text-white font-semibold">Import failed</p>
            <p className="text-ink-400 text-sm">{errorMsg}</p>
            <button
              onClick={() => setStep("form")}
              className="w-full py-3 rounded-xl bg-ink-700 text-white font-semibold hover:bg-ink-600 transition-colors"
            >
              Try again
            </button>
          </div>
        )}
      </div>
    </div>
  );
}
```

- [ ] **Step 2: Commit**

```bash
cd frontend
git add src/components/ImportGames.tsx
git commit -m "feat: add ImportGames modal component with progress polling"
```

---

## Task 10: Add "Import Games" card to main page

**Files:**
- Modify: `frontend/src/app/page.tsx`

- [ ] **Step 1: Add import to the main dashboard page**

In `frontend/src/app/page.tsx`, add the import at the top (after existing imports):

```tsx
import { useState } from "react";  // already imported — add ImportGames
import ImportGames from "@/components/ImportGames";
```

Add state for showing the modal (after the `displayName` line):

```tsx
  const [showImport, setShowImport] = useState(false);
```

Add an "Import Games" card in the existing `<section>` area, after the "Play Now" card. Find the closing `</section>` after the Play Now link and add before it:

```tsx
        <button
          onClick={() => setShowImport(true)}
          className="group mt-3 flex w-full items-center gap-4 bg-ink-800 border border-ink-700 rounded-2xl p-6 hover:bg-ink-700 transition-all hover:border-ink-600 shadow-lg shadow-black/20 text-left"
        >
          <div className="w-12 h-12 rounded-xl bg-ink-700 group-hover:bg-ink-600 flex items-center justify-center transition-colors shrink-0 shadow-inner">
            <Upload size={24} className="text-white" />
          </div>
          <div>
            <h3 className="text-xl font-semibold text-white mb-1">Import Games</h3>
            <p className="text-ink-400 text-sm">Analyze your Chess.com or Lichess history</p>
          </div>
        </button>
```

Add the modal at the end of the returned JSX, just before the closing `</div>`:

```tsx
      {showImport && (
        <ImportGames
          onClose={() => setShowImport(false)}
          onDone={() => setShowImport(false)}
        />
      )}
```

- [ ] **Step 2: Make sure `Upload` is imported from lucide-react**

Check the existing lucide imports line in `page.tsx`. It currently has:
```tsx
import { Swords, RotateCcw, Upload, Crown, User as UserIcon } from "lucide-react";
```
`Upload` is already there — no change needed.

- [ ] **Step 3: Start dev server and smoke test**

```bash
cd frontend
npm run dev
```

Open `http://localhost:3000`. Verify:
1. "Import Games" card is visible below "Play Now"
2. Clicking it opens the modal with platform toggle, username input, period radio
3. Closing with X button works
4. Selecting Chess.com or Lichess toggles correctly
5. Switching 30/90 days toggles correctly
6. Submit button is disabled when username is empty

- [ ] **Step 4: Commit**

```bash
cd frontend
git add src/app/page.tsx
git commit -m "feat: add Import Games entry point to dashboard"
```

---

## Self-Review Checklist (run before declaring done)

- [ ] `pytest tests/ -v` passes with no failures
- [ ] `npm run lint` passes with no errors in frontend
- [ ] `npm run build` succeeds
- [ ] Stockfish binary is installed on the dev machine (`which stockfish`)
- [ ] Manual smoke test: import 1-2 games from Lichess (public account), confirm they appear in the DB and have tilt reports
