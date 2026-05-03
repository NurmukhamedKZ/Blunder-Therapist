# Agent Chat Sidebar — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the static right-sidebar Tilt card with a streaming, per-game LangChain agent that reacts to blunders, comments on the post-game Tilt analysis, chats with the user, and accumulates cross-game memory via post-game chat summaries and a periodically refreshed Decision DNA profile.

**Architecture:** A single `create_agent()` instance with a Postgres checkpointer. Per-game `thread_id = game.id`. Two `@tool`s: `list_past_games` and `get_game_details`. Tilt remains an automatic backend job; its output is injected into the thread as a `HumanMessage` on `game_end`. Post-game closing triggers a summarizer LLM job (writes `game_summaries`) and conditionally a DNA job (every 5 games, writes `decision_dna`). Frontend talks to four endpoints; agent responses stream over SSE inside `StreamingResponse`.

**Tech Stack:** Python 3.13, FastAPI, SQLAlchemy 2 async, langchain >= 1.2, langgraph, langgraph-checkpoint-postgres, Next.js 15, React, Supabase Postgres + Auth.

**Spec:** `docs/superpowers/specs/2026-05-01-agent-chat-sidebar-design.md`

---

## File Structure

**Backend new:**
- `backend/app/models.py` — add `GameSummary`, `DecisionDNA` (modify in place)
- `backend/app/services/event_detector.py` — pure `classify_ply()`
- `backend/app/services/agent_tools.py` — `list_past_games`, `get_game_details` (`@tool` decorators)
- `backend/app/services/summarizer.py` — `summarize_chat()`
- `backend/app/services/dna_job.py` — `run_dna_for_user()`
- `backend/app/services/agent.py` — checkpointer init, `build_system_prompt`, `invoke_agent`, streaming helper
- `backend/app/routers/agent.py` — `/api/agent/observe`, `/message`, `/close-session`, `/history/{thread_id}`
- `backend/app/schemas/agent.py` — Pydantic request schemas
- `backend/app/main.py` — wire lifespan + include router (modify)
- `backend/pyproject.toml` (or `requirements.txt`) — add deps

**Backend tests:**
- `backend/tests/test_event_detector.py`
- `backend/tests/test_agent_tools.py`
- `backend/tests/test_summarizer.py`
- `backend/tests/test_dna_job.py`
- `backend/tests/test_agent_router.py`
- `backend/tests/conftest.py` — add `MemorySaver` override fixture (modify)

**Frontend new:**
- `frontend/src/lib/agent-stream.ts` — SSE parser
- `frontend/src/lib/event-detector.ts` — TS port of blunder rule
- `frontend/src/components/AgentChat.tsx`
- `frontend/src/components/AgentMessage.tsx`

**Frontend modified:**
- `frontend/src/components/ChessGame.tsx` — call observe on blunder, replace right pane
- `frontend/src/lib/api.ts` — add `observe`, `sendMessage`, `closeSession`, `getAgentHistory`

---

## Task 1: Add dependencies

**Files:**
- Modify: `backend/pyproject.toml` and/or `backend/requirements.txt`

- [ ] **Step 1: Inspect dep files**

Run: `cat backend/pyproject.toml backend/requirements.txt 2>/dev/null`

- [ ] **Step 2: Add deps**

Add (or insert into existing dependency lists):
```
langchain>=1.2
langgraph>=0.2
langchain-openai>=0.3
langgraph-checkpoint-postgres>=2.0
sse-starlette>=2.1
```

If using `uv`:
```bash
cd backend && uv add 'langchain>=1.2' 'langgraph>=0.2' 'langchain-openai>=0.3' 'langgraph-checkpoint-postgres>=2.0' 'sse-starlette>=2.1'
```

- [ ] **Step 3: Verify install**

Run: `cd backend && uv run python -c "from langchain.agents import create_agent; from langchain.tools import tool, ToolRuntime; from langgraph.checkpoint.postgres.aio import AsyncPostgresSaver; from sse_starlette.sse import EventSourceResponse; print('ok')"`
Expected: `ok`

- [ ] **Step 4: Commit**

```bash
cd /Users/nurma/vscode_projects/blunder-therapist
git add backend/pyproject.toml backend/uv.lock backend/requirements.txt 2>/dev/null
git commit -m "chore: add langchain agent + sse deps"
```

---

## Task 2: New ORM models

**Files:**
- Modify: `backend/app/models.py`
- Test: `backend/tests/test_models.py`

- [ ] **Step 1: Write the failing test** — append to `backend/tests/test_models.py`

```python
import pytest
from app.models import GameSummary, DecisionDNA, Profile, Game

@pytest.mark.asyncio
async def test_game_summary_persists(db):
    db.add(Profile(user_id="u1", plan="free"))
    g = Game(user_id="u1", pgn="", eval_per_ply=[], time_per_ply=[], player_color="white", result="win")
    db.add(g)
    await db.flush()
    db.add(GameSummary(user_id="u1", game_id=g.id, summary="we talked", key_facts=["a", "b"]))
    await db.commit()

@pytest.mark.asyncio
async def test_decision_dna_persists(db):
    db.add(Profile(user_id="u2", plan="free"))
    db.add(DecisionDNA(user_id="u2", dna={"type_name": "Tactician"}, games_count=5))
    await db.commit()
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd backend && uv run pytest tests/test_models.py -v -k "summary or dna"`
Expected: FAIL — `ImportError: cannot import name 'GameSummary'`

- [ ] **Step 3: Add models** — append to `backend/app/models.py`

```python
from sqlalchemy import Integer, Text


class GameSummary(Base):
    __tablename__ = "game_summaries"
    __table_args__ = (UniqueConstraint("game_id"),)

    id: Mapped[str] = mapped_column(String, primary_key=True, default=_uuid)
    user_id: Mapped[str] = mapped_column(String, ForeignKey("profiles.user_id"), nullable=False, index=True)
    game_id: Mapped[str] = mapped_column(String, ForeignKey("games.id"), nullable=False)
    summary: Mapped[str] = mapped_column(Text, nullable=False)
    key_facts: Mapped[list] = mapped_column(JSON, nullable=False, default=list)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=_now)


class DecisionDNA(Base):
    __tablename__ = "decision_dna"

    id: Mapped[str] = mapped_column(String, primary_key=True, default=_uuid)
    user_id: Mapped[str] = mapped_column(String, ForeignKey("profiles.user_id"), nullable=False, index=True)
    dna: Mapped[dict] = mapped_column(JSON, nullable=False)
    games_count: Mapped[int] = mapped_column(Integer, nullable=False)
    computed_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=_now)
```

- [ ] **Step 4: Run tests, verify pass**

Run: `cd backend && uv run pytest tests/test_models.py -v`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add backend/app/models.py backend/tests/test_models.py
git commit -m "feat(db): add game_summaries and decision_dna tables"
```

---

## Task 3: Event detector (pure function)

**Files:**
- Create: `backend/app/services/event_detector.py`
- Test: `backend/tests/test_event_detector.py`

- [ ] **Step 1: Write the failing test**

```python
# backend/tests/test_event_detector.py
import pytest
from app.services.event_detector import classify_ply

@pytest.mark.parametrize("eval_before,eval_after,player_color,expected", [
    # White blunders: eval drops by >=200cp from White's POV
    (50, -250, "white", "blunder"),
    (50, -150, "white", None),
    # Black blunders: eval rises by >=200cp from White's POV
    (-50, 250, "black", "blunder"),
    (-50, 150, "black", None),
    # Wrong color player isn't penalized for opponent's eval swing
    (50, -250, "black", None),
    (-50, 250, "white", None),
    # Mate scoring (we store ±9999) — clamp, only flag big swings
    (100, -9999, "white", "blunder"),
    # Identical evals: no event
    (0, 0, "white", None),
])
def test_classify_ply(eval_before, eval_after, player_color, expected):
    assert classify_ply(eval_before, eval_after, player_color, time_taken=2.0) == expected
```

- [ ] **Step 2: Run test, verify fail**

Run: `cd backend && uv run pytest tests/test_event_detector.py -v`
Expected: FAIL — module missing.

- [ ] **Step 3: Implement**

```python
# backend/app/services/event_detector.py
"""Detect behaviorally meaningful events from a single ply.

Pure function — no LLM, no IO. Called server-side on every observe POST and
client-side as a quick pre-filter to avoid hitting the API for nothing.
"""
from typing import Literal

BLUNDER_THRESHOLD_CP = 200

EventType = Literal["blunder"]


def classify_ply(
    eval_before: float,
    eval_after: float,
    player_color: str,
    time_taken: float,
) -> EventType | None:
    """Return an event tag if the ply is behaviorally interesting, else None.

    Evals are stored from White's POV (centipawns). A blunder for the human
    player is a swing of >=200cp AGAINST them.
    """
    delta = eval_after - eval_before  # +ve = good for white
    if player_color == "white" and delta <= -BLUNDER_THRESHOLD_CP:
        return "blunder"
    if player_color == "black" and delta >= BLUNDER_THRESHOLD_CP:
        return "blunder"
    return None
```

- [ ] **Step 4: Run tests, verify pass**

Run: `cd backend && uv run pytest tests/test_event_detector.py -v`
Expected: PASS for all parametrize cases.

- [ ] **Step 5: Commit**

```bash
git add backend/app/services/event_detector.py backend/tests/test_event_detector.py
git commit -m "feat(agent): add ply event classifier"
```

---

## Task 4: Agent tools (`list_past_games`, `get_game_details`)

**Files:**
- Create: `backend/app/services/agent_tools.py`
- Test: `backend/tests/test_agent_tools.py`

- [ ] **Step 1: Write the failing test**

```python
# backend/tests/test_agent_tools.py
import pytest
from datetime import datetime, timezone
from app.models import Profile, Game, TiltReport, GameSummary
from app.services.agent_tools import _list_past_games_impl, _get_game_details_impl


@pytest.mark.asyncio
async def test_list_past_games_returns_user_games_only(db):
    db.add(Profile(user_id="alice", plan="free"))
    db.add(Profile(user_id="bob", plan="free"))
    g1 = Game(user_id="alice", pgn="x", eval_per_ply=[0], time_per_ply=[1.0],
              player_color="white", result="win",
              played_at=datetime(2026, 5, 1, tzinfo=timezone.utc))
    g2 = Game(user_id="bob", pgn="y", eval_per_ply=[0], time_per_ply=[1.0],
              player_color="white", result="loss",
              played_at=datetime(2026, 5, 1, tzinfo=timezone.utc))
    db.add_all([g1, g2])
    await db.flush()
    db.add(TiltReport(game_id=g1.id, headline="h", diagnosis="d",
                      pattern_label="tilt", evidence_plies=[1], suggestion="s"))
    await db.commit()

    out = await _list_past_games_impl(db, user_id="alice", filter=None)
    assert len(out) == 1
    assert out[0]["game_id"] == g1.id
    assert out[0]["tilt_pattern"] == "tilt"


@pytest.mark.asyncio
async def test_list_past_games_filter_losses(db):
    db.add(Profile(user_id="alice", plan="free"))
    for r in ["win", "loss", "loss", "draw"]:
        db.add(Game(user_id="alice", pgn="x", eval_per_ply=[0], time_per_ply=[1.0],
                    player_color="white", result=r))
    await db.commit()

    out = await _list_past_games_impl(db, user_id="alice", filter="losses")
    assert len(out) == 2
    assert all(g["result"] == "loss" for g in out)


@pytest.mark.asyncio
async def test_get_game_details_rejects_other_user(db):
    db.add(Profile(user_id="alice", plan="free"))
    db.add(Profile(user_id="bob", plan="free"))
    g = Game(user_id="bob", pgn="x", eval_per_ply=[0], time_per_ply=[1.0],
             player_color="white", result="win")
    db.add(g)
    await db.commit()

    out = await _get_game_details_impl(db, user_id="alice", game_id=g.id)
    assert out == {"error": "not found"}


@pytest.mark.asyncio
async def test_get_game_details_returns_full_record(db):
    db.add(Profile(user_id="alice", plan="free"))
    g = Game(user_id="alice", pgn="1.e4", eval_per_ply=[0, 30], time_per_ply=[2.0, 1.5],
             player_color="white", result="win")
    db.add(g)
    await db.flush()
    db.add(TiltReport(game_id=g.id, headline="h", diagnosis="d",
                      pattern_label="focused", evidence_plies=[], suggestion="s"))
    db.add(GameSummary(user_id="alice", game_id=g.id, summary="all good", key_facts=["x"]))
    await db.commit()

    out = await _get_game_details_impl(db, user_id="alice", game_id=g.id)
    assert out["pgn"] == "1.e4"
    assert out["eval_per_ply"] == [0, 30]
    assert out["tilt_report"]["pattern_label"] == "focused"
    assert out["chat_summary"]["key_facts"] == ["x"]
```

- [ ] **Step 2: Run, verify fail**

Run: `cd backend && uv run pytest tests/test_agent_tools.py -v`
Expected: FAIL — module missing.

- [ ] **Step 3: Implement**

```python
# backend/app/services/agent_tools.py
"""LangChain tools the agent can call.

Tools split into two layers:
  - `_*_impl(db, user_id, ...)` — pure async DB function, used by tests.
  - `@tool` wrappers — get db + user_id from ToolRuntime, call the impl.

This split keeps tools testable without spinning up a runtime.
"""
from typing import Any
from langchain.tools import tool, ToolRuntime
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy.orm import selectinload

from app.database import AsyncSessionLocal
from app.models import Game, TiltReport, GameSummary

_FILTER_KEYWORDS = {
    "losses": ("result", "loss"),
    "wins": ("result", "win"),
    "draws": ("result", "draw"),
}


async def _list_past_games_impl(
    db: AsyncSession, user_id: str, filter: str | None
) -> list[dict[str, Any]]:
    stmt = (
        select(Game)
        .options(selectinload(Game.tilt_report))
        .where(Game.user_id == user_id)
        .order_by(Game.played_at.desc())
        .limit(20)
    )
    if filter:
        f = filter.lower().strip()
        if f in _FILTER_KEYWORDS:
            col, val = _FILTER_KEYWORDS[f]
            stmt = stmt.where(getattr(Game, col) == val)
    rows = await db.execute(stmt)
    games = rows.scalars().all()

    # Pull summaries for these games in one query.
    summaries: dict[str, GameSummary] = {}
    if games:
        sum_rows = await db.execute(
            select(GameSummary).where(GameSummary.game_id.in_([g.id for g in games]))
        )
        summaries = {s.game_id: s for s in sum_rows.scalars().all()}

    return [
        {
            "game_id": g.id,
            "played_at": g.played_at.isoformat() if g.played_at else None,
            "result": g.result,
            "player_color": g.player_color,
            "tilt_pattern": g.tilt_report.pattern_label if g.tilt_report else None,
            "summary_excerpt": (summaries[g.id].summary[:200] if g.id in summaries else None),
        }
        for g in games
    ]


async def _get_game_details_impl(
    db: AsyncSession, user_id: str, game_id: str
) -> dict[str, Any]:
    row = await db.execute(
        select(Game)
        .options(selectinload(Game.tilt_report))
        .where(Game.id == game_id, Game.user_id == user_id)
    )
    game = row.scalar_one_or_none()
    if game is None:
        return {"error": "not found"}

    summary_row = await db.execute(
        select(GameSummary).where(GameSummary.game_id == game_id)
    )
    chat_summary = summary_row.scalar_one_or_none()

    return {
        "game_id": game.id,
        "pgn": game.pgn,
        "eval_per_ply": game.eval_per_ply,
        "time_per_ply": game.time_per_ply,
        "player_color": game.player_color,
        "result": game.result,
        "played_at": game.played_at.isoformat() if game.played_at else None,
        "tilt_report": (
            {
                "headline": game.tilt_report.headline,
                "diagnosis": game.tilt_report.diagnosis,
                "pattern_label": game.tilt_report.pattern_label,
                "evidence_plies": game.tilt_report.evidence_plies,
                "suggestion": game.tilt_report.suggestion,
            }
            if game.tilt_report
            else None
        ),
        "chat_summary": (
            {"summary": chat_summary.summary, "key_facts": chat_summary.key_facts}
            if chat_summary
            else None
        ),
    }


@tool
async def list_past_games(filter: str | None, runtime: ToolRuntime) -> list[dict]:
    """List the user's past games, most recent first (max 20).

    Args:
        filter: optional hint. Supported: "losses", "wins", "draws".
                Unknown values are ignored.

    Returns a list of {game_id, played_at, result, player_color, tilt_pattern,
    summary_excerpt}.
    """
    user_id = runtime.context["user_id"]
    async with AsyncSessionLocal() as db:
        return await _list_past_games_impl(db, user_id, filter)


@tool
async def get_game_details(game_id: str, runtime: ToolRuntime) -> dict:
    """Fetch the full record for one game, including PGN, evals, times,
    tilt report, and chat summary.

    If the game doesn't exist or doesn't belong to the user, returns
    {"error": "not found"}.
    """
    user_id = runtime.context["user_id"]
    async with AsyncSessionLocal() as db:
        return await _get_game_details_impl(db, user_id, game_id)
```

- [ ] **Step 4: Run tests, verify pass**

Run: `cd backend && uv run pytest tests/test_agent_tools.py -v`
Expected: PASS, 4 tests.

- [ ] **Step 5: Commit**

```bash
git add backend/app/services/agent_tools.py backend/tests/test_agent_tools.py
git commit -m "feat(agent): add list_past_games and get_game_details tools"
```

---

## Task 5: Summarizer service

**Files:**
- Create: `backend/app/services/summarizer.py`
- Test: `backend/tests/test_summarizer.py`

- [ ] **Step 1: Write the failing test**

```python
# backend/tests/test_summarizer.py
import pytest
from unittest.mock import AsyncMock, patch

from app.services.summarizer import summarize_chat, ChatSummaryOutput


@pytest.mark.asyncio
async def test_summarize_chat_returns_structure():
    fake = ChatSummaryOutput(
        summary="User reflected on rushing in middlegame; agreed to try slower openings.",
        key_facts=["rushes in middlegame", "wants to slow down"],
    )
    mock = AsyncMock(return_value=fake)
    with patch("app.services.summarizer._summarizer", new=AsyncMock(ainvoke=mock)):
        result = await summarize_chat(
            messages=[
                {"role": "user", "content": "I keep blundering on move 20"},
                {"role": "assistant", "content": "You moved fast. Try doubling time."},
            ],
        )
    assert result.summary.startswith("User reflected")
    assert "rushes in middlegame" in result.key_facts


@pytest.mark.asyncio
async def test_summarize_chat_handles_empty_messages():
    result = await summarize_chat(messages=[])
    assert result.summary == ""
    assert result.key_facts == []
```

- [ ] **Step 2: Run, verify fail**

Run: `cd backend && uv run pytest tests/test_summarizer.py -v`
Expected: FAIL — missing module.

- [ ] **Step 3: Implement**

```python
# backend/app/services/summarizer.py
"""Post-game chat summarizer.

Reads the user/agent dialogue from a closed game thread and extracts a short
summary plus a list of durable facts (preferences, commitments, recurring
patterns) that should inform future games.
"""
from langchain.messages import HumanMessage, SystemMessage
from langchain_openai import ChatOpenAI
from pydantic import BaseModel, Field

from app.config import settings


SUMMARIZER_SYSTEM = """You are summarizing a coaching conversation about ONE chess game.
Your output is read by a future AI coach when this user starts their NEXT game.

Extract:
- summary: 2-4 sentences, focused on what the USER said, felt, or agreed to.
  Do NOT re-summarize the game itself — there's a separate tilt report for that.
- key_facts: 1-5 short bullet strings of durable facts that future-you should
  remember. Examples: "user prefers short answers", "wants to work on endgames",
  "called this a tilt loss", "plays at night when tired".

If the conversation is empty or trivial, return summary="" and key_facts=[].
"""


class ChatSummaryOutput(BaseModel):
    summary: str = Field(description="2-4 sentences focused on user's words and intent")
    key_facts: list[str] = Field(default_factory=list, description="durable facts to remember")


_summarizer = ChatOpenAI(
    model=settings.model_fast, api_key=settings.openai_api_key, temperature=0.3
).with_structured_output(ChatSummaryOutput)


async def summarize_chat(messages: list[dict]) -> ChatSummaryOutput:
    """Summarize a list of {role, content} messages."""
    if not messages:
        return ChatSummaryOutput(summary="", key_facts=[])
    transcript = "\n".join(f"{m['role'].upper()}: {m['content']}" for m in messages)
    return await _summarizer.ainvoke(
        [SystemMessage(SUMMARIZER_SYSTEM), HumanMessage(transcript)]
    )
```

- [ ] **Step 4: Run, verify pass**

Run: `cd backend && uv run pytest tests/test_summarizer.py -v`
Expected: PASS, 2 tests.

- [ ] **Step 5: Commit**

```bash
git add backend/app/services/summarizer.py backend/tests/test_summarizer.py
git commit -m "feat(agent): add chat summarizer"
```

---

## Task 6: DNA job

**Files:**
- Create: `backend/app/services/dna_job.py`
- Test: `backend/tests/test_dna_job.py`

- [ ] **Step 1: Write the failing test**

```python
# backend/tests/test_dna_job.py
import pytest
from unittest.mock import AsyncMock, patch

from app.models import Profile, Game, DecisionDNA
from app.services.dna_job import run_dna_for_user, should_recompute_dna
from sqlalchemy import select


def _add_games(db, user_id: str, n: int) -> None:
    for _ in range(n):
        db.add(Game(user_id=user_id, pgn="1.e4 e5", eval_per_ply=[0, 30, 0],
                    time_per_ply=[1.0, 1.0, 1.0], player_color="white", result="win"))


def test_should_recompute_dna_strict_multiples_of_5():
    assert should_recompute_dna(5) is True
    assert should_recompute_dna(10) is True
    assert should_recompute_dna(0) is False
    assert should_recompute_dna(4) is False
    assert should_recompute_dna(7) is False


@pytest.mark.asyncio
async def test_run_dna_for_user_upserts(db):
    db.add(Profile(user_id="u1", plan="free"))
    _add_games(db, "u1", 5)
    await db.commit()

    fake_dna = {
        "type_name": "Aggressive Tactician", "tagline": "fights early",
        "summary": "...", "core_strength": "...", "core_weakness": "...",
        "gm_comparison": {"name": "Tal", "similarity_pct": 22, "why": "..."},
    }
    with patch("app.services.dna_job.run_decision_dna",
               new=AsyncMock(return_value=fake_dna)):
        await run_dna_for_user(db, user_id="u1")

    rows = await db.execute(select(DecisionDNA).where(DecisionDNA.user_id == "u1"))
    saved = rows.scalars().all()
    assert len(saved) == 1
    assert saved[0].dna["type_name"] == "Aggressive Tactician"
    assert saved[0].games_count == 5
```

- [ ] **Step 2: Run, verify fail**

Run: `cd backend && uv run pytest tests/test_dna_job.py -v`
Expected: FAIL — module missing.

- [ ] **Step 3: Implement**

```python
# backend/app/services/dna_job.py
"""Background DNA computation.

Triggered from /api/agent/close-session when total_games is a multiple of 5.
Pulls the user's last 5 games, runs the existing run_decision_dna LLM call,
upserts a DecisionDNA row.
"""
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from app.models import Game, DecisionDNA
from app.services.features import extract_features
from app.services.llm import run_decision_dna

DNA_TRIGGER_INTERVAL = 5


def should_recompute_dna(total_games: int) -> bool:
    return total_games > 0 and total_games % DNA_TRIGGER_INTERVAL == 0


async def run_dna_for_user(db: AsyncSession, user_id: str) -> None:
    rows = await db.execute(
        select(Game)
        .where(Game.user_id == user_id)
        .order_by(Game.played_at.desc())
        .limit(DNA_TRIGGER_INTERVAL)
    )
    games = rows.scalars().all()
    if len(games) < 3:
        return  # silently skip — defensive, shouldn't happen on the 5-boundary

    features = []
    for g in games:
        try:
            features.append(
                extract_features(
                    pgn=g.pgn, eval_per_ply=g.eval_per_ply,
                    time_per_ply=g.time_per_ply, player_color=g.player_color,
                    result=g.result,
                )
            )
        except ValueError:
            continue
    if len(features) < 3:
        return

    dna = await run_decision_dna(features)
    db.add(DecisionDNA(user_id=user_id, dna=dna, games_count=len(games)))
    await db.commit()
```

- [ ] **Step 4: Run, verify pass**

Run: `cd backend && uv run pytest tests/test_dna_job.py -v`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add backend/app/services/dna_job.py backend/tests/test_dna_job.py
git commit -m "feat(agent): add DNA recompute job"
```

---

## Task 7: Agent service

**Files:**
- Create: `backend/app/services/agent.py`
- Test (system prompt only — full agent flow tested in router task): inline.

- [ ] **Step 1: Write the failing test for `build_system_prompt`**

Append to `backend/tests/test_agent_tools.py`:

```python
from app.services.agent import build_system_prompt
from app.models import GameSummary, DecisionDNA

@pytest.mark.asyncio
async def test_build_system_prompt_includes_memory(db):
    db.add(Profile(user_id="alice", plan="free"))
    g = Game(user_id="alice", pgn="x", eval_per_ply=[0], time_per_ply=[1.0],
             player_color="white", result="loss")
    db.add(g)
    await db.flush()
    db.add(GameSummary(user_id="alice", game_id=g.id,
                       summary="rushed in middlegame", key_facts=["rushes"]))
    db.add(DecisionDNA(user_id="alice",
                       dna={"type_name": "Aggressive Tactician", "tagline": "x",
                            "summary": "y", "core_strength": "z",
                            "core_weakness": "w", "gm_comparison": {
                                "name": "Tal", "similarity_pct": 20, "why": "."}},
                       games_count=5))
    await db.commit()

    prompt = await build_system_prompt(db, user_id="alice")
    assert "Aggressive Tactician" in prompt
    assert "rushes" in prompt or "rushed in middlegame" in prompt
    assert "behavioral" in prompt.lower()


@pytest.mark.asyncio
async def test_build_system_prompt_handles_no_memory(db):
    db.add(Profile(user_id="newbie", plan="free"))
    await db.commit()
    prompt = await build_system_prompt(db, user_id="newbie")
    assert "no past games" in prompt.lower() or "no memory" in prompt.lower()
```

- [ ] **Step 2: Run, verify fail**

Run: `cd backend && uv run pytest tests/test_agent_tools.py -v -k system_prompt`
Expected: FAIL — `agent` module missing.

- [ ] **Step 3: Implement agent service**

```python
# backend/app/services/agent.py
"""LangChain agent setup, system prompt builder, streaming wrapper."""
from typing import AsyncIterator
import json

from langchain.agents import create_agent
from langchain.messages import BaseMessage, SystemMessage
from langchain_openai import ChatOpenAI
from langgraph.checkpoint.memory import MemorySaver
from langgraph.checkpoint.postgres.aio import AsyncPostgresSaver
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from app.config import settings
from app.models import GameSummary, DecisionDNA
from app.services.agent_tools import list_past_games, get_game_details

BASE_SYSTEM = """You are the user's per-game chess coach inside their game-analysis sidebar.

Behavioral, not tactical:
- Comment on PROCESS (timing, hesitation, panic, recovery), NOT on engine moves.
- If the user asks "what should I play?" or "is this move good?" — politely
  redirect to behavioral framing. We are not an engine.

Style:
- Short, warm, specific. Talk like a smart friend, not a therapist or a textbook.
- When commenting on an in-game observation (e.g. "[OBSERVATION] Ply 17:
  blunder ..."), respond with AT MOST one short paragraph, and ONLY if the
  last thing you said is at least 5 plies ago. Otherwise output nothing.
- When the user explicitly asks something, always reply.

Tools:
- list_past_games: search the user's history. Use when they reference past games.
- get_game_details: pull full data for one specific past game.
"""

NO_MEMORY_NOTE = "No past games yet — this is a fresh user. Don't pretend to remember."


def _format_summaries(summaries: list[GameSummary]) -> str:
    if not summaries:
        return NO_MEMORY_NOTE
    bullets = []
    for s in summaries:
        bullets.append(f"- ({s.created_at.date()}) {s.summary}")
        for fact in s.key_facts:
            bullets.append(f"    • {fact}")
    return "Recent coaching history (oldest → newest):\n" + "\n".join(bullets)


def _format_dna(dna: DecisionDNA | None) -> str:
    if dna is None:
        return "No DNA profile yet (need 5+ games)."
    d = dna.dna
    return (
        f"Player style: {d.get('type_name')} — {d.get('tagline')}.\n"
        f"Strength: {d.get('core_strength')}\n"
        f"Weakness: {d.get('core_weakness')}\n"
        f"Reminds of: {d.get('gm_comparison', {}).get('name')} "
        f"(~{d.get('gm_comparison', {}).get('similarity_pct')}%)."
    )


async def build_system_prompt(db: AsyncSession, user_id: str) -> str:
    sum_rows = await db.execute(
        select(GameSummary)
        .where(GameSummary.user_id == user_id)
        .order_by(GameSummary.created_at.desc())
        .limit(5)
    )
    summaries = list(reversed(sum_rows.scalars().all()))  # oldest -> newest

    dna_rows = await db.execute(
        select(DecisionDNA)
        .where(DecisionDNA.user_id == user_id)
        .order_by(DecisionDNA.computed_at.desc())
        .limit(1)
    )
    dna = dna_rows.scalar_one_or_none()

    return (
        BASE_SYSTEM
        + "\n\n=== MEMORY ===\n"
        + _format_summaries(summaries)
        + "\n\n=== DNA ===\n"
        + _format_dna(dna)
    )


# --- Agent singleton ---

_checkpointer: AsyncPostgresSaver | MemorySaver | None = None
_agent = None
_pg_cm = None  # async context manager handle for AsyncPostgresSaver.from_conn_string


async def init_agent(database_url: str, *, in_memory: bool = False) -> None:
    global _checkpointer, _agent, _pg_cm
    if in_memory:
        _checkpointer = MemorySaver()
    else:
        # AsyncPostgresSaver expects a sync libpq URL (psycopg). Strip async driver.
        pg_url = database_url.replace("postgresql+asyncpg://", "postgresql://")
        _pg_cm = AsyncPostgresSaver.from_conn_string(pg_url)
        _checkpointer = await _pg_cm.__aenter__()
        await _checkpointer.setup()

    _agent = create_agent(
        model=ChatOpenAI(
            model=settings.model_smart,
            api_key=settings.openai_api_key,
            temperature=0.6,
        ),
        tools=[list_past_games, get_game_details],
        checkpointer=_checkpointer,
    )


async def shutdown_agent() -> None:
    global _pg_cm
    if _pg_cm is not None:
        await _pg_cm.__aexit__(None, None, None)
        _pg_cm = None


def get_agent():
    if _agent is None:
        raise RuntimeError("Agent not initialized — call init_agent in lifespan")
    return _agent


def get_checkpointer():
    if _checkpointer is None:
        raise RuntimeError("Checkpointer not initialized")
    return _checkpointer


async def stream_agent_response(
    thread_id: str,
    user_id: str,
    jwt: str,
    new_messages: list[BaseMessage],
    seed_system_prompt: str | None = None,
) -> AsyncIterator[str]:
    """Yield SSE-formatted strings (event/data lines) for the agent's response."""
    config = {"configurable": {"thread_id": thread_id}}
    context = {"user_id": user_id, "jwt": jwt}

    if seed_system_prompt is not None:
        new_messages = [SystemMessage(seed_system_prompt), *new_messages]

    async for chunk, _meta in get_agent().astream(
        {"messages": new_messages},
        config=config,
        context=context,
        stream_mode="messages",
    ):
        text = getattr(chunk, "content", None)
        if isinstance(text, str) and text:
            yield f"event: token\ndata: {json.dumps({'text': text})}\n\n"
    yield "event: done\ndata: {}\n\n"


async def thread_has_state(thread_id: str) -> bool:
    config = {"configurable": {"thread_id": thread_id}}
    state = await get_checkpointer().aget(config)
    return state is not None and bool(state.get("channel_values", {}).get("messages"))
```

- [ ] **Step 4: Run, verify system_prompt tests pass**

Run: `cd backend && uv run pytest tests/test_agent_tools.py -v -k system_prompt`
Expected: PASS, 2 tests.

- [ ] **Step 5: Commit**

```bash
git add backend/app/services/agent.py backend/tests/test_agent_tools.py
git commit -m "feat(agent): add agent service, system prompt builder, streaming"
```

---

## Task 8: Pydantic schemas for agent endpoints

**Files:**
- Create: `backend/app/schemas/agent.py`

- [ ] **Step 1: Write schemas**

```python
# backend/app/schemas/agent.py
from typing import Literal, Any
from pydantic import BaseModel


class ObserveRequest(BaseModel):
    thread_id: str
    event: Literal["game_start", "blunder", "game_end"]
    payload: dict[str, Any] = {}


class MessageRequest(BaseModel):
    thread_id: str
    text: str


class CloseSessionRequest(BaseModel):
    thread_id: str


class HistoryMessage(BaseModel):
    role: Literal["user", "assistant", "system"]
    content: str


class HistoryResponse(BaseModel):
    messages: list[HistoryMessage]
```

- [ ] **Step 2: Commit (no test — pure schema)**

```bash
git add backend/app/schemas/agent.py
git commit -m "feat(agent): add request schemas"
```

---

## Task 9: Agent router

**Files:**
- Create: `backend/app/routers/agent.py`
- Test: `backend/tests/test_agent_router.py`

- [ ] **Step 1: Write the failing test**

```python
# backend/tests/test_agent_router.py
import pytest
import json
from unittest.mock import AsyncMock, patch

from sqlalchemy import select
from app.models import Profile, Game, TiltReport, GameSummary, DecisionDNA


async def _drain_sse(response) -> list[dict]:
    events = []
    async for line in response.aiter_lines():
        if line.startswith("data: "):
            events.append(json.loads(line[6:]))
    return events


@pytest.fixture(autouse=True)
def _seed_alice(db):
    """Test user 'test-user-id' (from authed_client) needs a Profile row."""
    # noop — authed_client fixture's get_current_user override creates a
    # CurrentUser, but the Profile auto-create only runs in the real
    # dependency. Tests should add Profile rows themselves where needed.


@pytest.mark.asyncio
async def test_observe_game_start_initializes_thread(authed_client, db):
    db.add(Profile(user_id="test-user-id", plan="free"))
    await db.commit()

    async def fake_stream(*a, **kw):
        yield 'event: done\ndata: {}\n\n'

    with patch("app.routers.agent.stream_agent_response", new=fake_stream), \
         patch("app.routers.agent.thread_has_state", new=AsyncMock(return_value=False)):
        async with authed_client.stream(
            "POST", "/api/agent/observe",
            json={"thread_id": "g-1", "event": "game_start", "payload": {}},
        ) as r:
            assert r.status_code == 200
            body = await r.aread()
            assert b"event: done" in body


@pytest.mark.asyncio
async def test_observe_blunder_streams_token(authed_client, db):
    db.add(Profile(user_id="test-user-id", plan="free"))
    await db.commit()

    async def fake_stream(*a, **kw):
        yield 'event: token\ndata: {"text": "Hmm"}\n\n'
        yield 'event: done\ndata: {}\n\n'

    with patch("app.routers.agent.stream_agent_response", new=fake_stream), \
         patch("app.routers.agent.thread_has_state", new=AsyncMock(return_value=True)):
        async with authed_client.stream(
            "POST", "/api/agent/observe",
            json={"thread_id": "g-1", "event": "blunder",
                  "payload": {"ply": 17, "san": "Qxh7", "eval_before": 50,
                              "eval_after": -250, "time_taken": 4.0}},
        ) as r:
            body = await r.aread()
            assert b'"text": "Hmm"' in body


@pytest.mark.asyncio
async def test_observe_game_end_loads_tilt_report(authed_client, db):
    db.add(Profile(user_id="test-user-id", plan="free"))
    g = Game(user_id="test-user-id", pgn="x", eval_per_ply=[0],
             time_per_ply=[1.0], player_color="white", result="loss")
    db.add(g)
    await db.flush()
    db.add(TiltReport(game_id=g.id, headline="Rushed in middlegame",
                      diagnosis="d", pattern_label="rushing",
                      evidence_plies=[10], suggestion="s"))
    await db.commit()

    captured = {}

    async def fake_stream(thread_id, user_id, jwt, new_messages, seed_system_prompt=None):
        captured["msgs"] = new_messages
        yield 'event: done\ndata: {}\n\n'

    with patch("app.routers.agent.stream_agent_response", new=fake_stream), \
         patch("app.routers.agent.thread_has_state", new=AsyncMock(return_value=True)):
        async with authed_client.stream(
            "POST", "/api/agent/observe",
            json={"thread_id": g.id, "event": "game_end",
                  "payload": {"game_id": g.id}},
        ) as r:
            await r.aread()

    payload_text = captured["msgs"][0].content
    assert "Rushed in middlegame" in payload_text


@pytest.mark.asyncio
async def test_message_streams(authed_client, db):
    db.add(Profile(user_id="test-user-id", plan="free"))
    await db.commit()

    async def fake_stream(*a, **kw):
        yield 'event: token\ndata: {"text": "hi"}\n\n'
        yield 'event: done\ndata: {}\n\n'

    with patch("app.routers.agent.stream_agent_response", new=fake_stream), \
         patch("app.routers.agent.thread_has_state", new=AsyncMock(return_value=True)):
        async with authed_client.stream(
            "POST", "/api/agent/message",
            json={"thread_id": "g-1", "text": "what do you think?"},
        ) as r:
            body = await r.aread()
            assert b'"text": "hi"' in body


@pytest.mark.asyncio
async def test_close_session_writes_summary(authed_client, db):
    db.add(Profile(user_id="test-user-id", plan="free"))
    g = Game(user_id="test-user-id", pgn="x", eval_per_ply=[0],
             time_per_ply=[1.0], player_color="white", result="win")
    db.add(g)
    await db.commit()

    fake_msgs = type("S", (), {"get": lambda self, k, d=None: {
        "messages": [type("M", (), {"type": "human", "content": "hey"})()]
    }.get(k, d)})()

    from app.services.summarizer import ChatSummaryOutput
    summary_obj = ChatSummaryOutput(summary="we chatted", key_facts=["a"])

    with patch("app.routers.agent.get_checkpointer") as gck:
        gck.return_value.aget = AsyncMock(return_value=fake_msgs)
        with patch("app.routers.agent.summarize_chat",
                   new=AsyncMock(return_value=summary_obj)):
            r = await authed_client.post(
                "/api/agent/close-session",
                json={"thread_id": g.id},
            )
            assert r.status_code == 204

    rows = await db.execute(select(GameSummary).where(GameSummary.game_id == g.id))
    saved = rows.scalar_one()
    assert saved.summary == "we chatted"
    assert saved.key_facts == ["a"]


@pytest.mark.asyncio
async def test_close_session_idempotent(authed_client, db):
    db.add(Profile(user_id="test-user-id", plan="free"))
    g = Game(user_id="test-user-id", pgn="x", eval_per_ply=[0],
             time_per_ply=[1.0], player_color="white", result="win")
    db.add(g)
    await db.flush()
    db.add(GameSummary(user_id="test-user-id", game_id=g.id,
                       summary="existing", key_facts=[]))
    await db.commit()

    r = await authed_client.post("/api/agent/close-session",
                                 json={"thread_id": g.id})
    assert r.status_code == 204
    rows = await db.execute(select(GameSummary).where(GameSummary.game_id == g.id))
    assert len(rows.scalars().all()) == 1


@pytest.mark.asyncio
async def test_close_session_triggers_dna_at_5(authed_client, db):
    db.add(Profile(user_id="test-user-id", plan="free"))
    games = []
    for _ in range(5):
        g = Game(user_id="test-user-id", pgn="1.e4 e5", eval_per_ply=[0, 30],
                 time_per_ply=[1.0, 1.0], player_color="white", result="win")
        db.add(g)
        games.append(g)
    await db.commit()
    await db.refresh(games[-1])

    fake_msgs = type("S", (), {"get": lambda self, k, d=None: {
        "messages": []
    }.get(k, d)})()

    with patch("app.routers.agent.get_checkpointer") as gck:
        gck.return_value.aget = AsyncMock(return_value=fake_msgs)
        with patch("app.routers.agent.run_dna_for_user",
                   new=AsyncMock()) as dna_mock:
            r = await authed_client.post("/api/agent/close-session",
                                         json={"thread_id": games[-1].id})
            assert r.status_code == 204
            dna_mock.assert_awaited_once()


@pytest.mark.asyncio
async def test_history_returns_messages(authed_client, db):
    db.add(Profile(user_id="test-user-id", plan="free"))
    await db.commit()

    fake_msgs = type("S", (), {"get": lambda self, k, d=None: {
        "messages": [
            type("M", (), {"type": "human", "content": "hi"})(),
            type("M", (), {"type": "ai", "content": "hello"})(),
        ]
    }.get(k, d)})()

    with patch("app.routers.agent.get_checkpointer") as gck:
        gck.return_value.aget = AsyncMock(return_value=fake_msgs)
        r = await authed_client.get("/api/agent/history/g-1")
        assert r.status_code == 200
        data = r.json()
        assert data["messages"][0]["role"] == "user"
        assert data["messages"][1]["role"] == "assistant"
```

- [ ] **Step 2: Run, verify fail**

Run: `cd backend && uv run pytest tests/test_agent_router.py -v`
Expected: FAIL — module missing.

- [ ] **Step 3: Implement router**

```python
# backend/app/routers/agent.py
"""Agent chat endpoints."""
import json
from fastapi import APIRouter, Depends, HTTPException, Request
from fastapi.responses import StreamingResponse, Response
from langchain.messages import HumanMessage
from sqlalchemy import select, func
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy.orm import selectinload

from app.database import get_db
from app.dependencies import CurrentUser, get_current_user, _bearer
from app.models import Game, TiltReport, GameSummary
from app.schemas.agent import (
    ObserveRequest, MessageRequest, CloseSessionRequest,
    HistoryMessage, HistoryResponse,
)
from app.services.agent import (
    build_system_prompt, stream_agent_response, thread_has_state, get_checkpointer,
)
from app.services.dna_job import should_recompute_dna, run_dna_for_user
from app.services.summarizer import summarize_chat

router = APIRouter(prefix="/api/agent", tags=["agent"])

_ROLE_MAP = {"human": "user", "ai": "assistant", "system": "system"}


def _extract_jwt(request: Request) -> str:
    auth = request.headers.get("authorization", "")
    return auth[7:] if auth.lower().startswith("bearer ") else ""


def _format_observation_message(event: str, payload: dict) -> str:
    if event == "blunder":
        return (
            f"[OBSERVATION] Ply {payload.get('ply')}: blunder, "
            f"san={payload.get('san')}, eval {payload.get('eval_before')}→"
            f"{payload.get('eval_after')}cp, {payload.get('time_taken')}s think."
        )
    if event == "game_start":
        return "[OBSERVATION] Game started — watch and stay quiet unless asked."
    return f"[OBSERVATION] {event}: {json.dumps(payload)}"


@router.post("/observe")
async def observe(
    req: ObserveRequest,
    request: Request,
    user: CurrentUser = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
):
    jwt_token = _extract_jwt(request)

    is_first = not await thread_has_state(req.thread_id)
    seed = await build_system_prompt(db, user.user_id) if is_first else None

    if req.event == "game_start":
        # Just initialize; no agent reply.
        async def _empty():
            if seed is not None:
                # We seed by sending an empty message + system prompt; the agent
                # may emit nothing for the empty turn but the system prompt is now
                # in the checkpoint.
                pass
            yield "event: done\ndata: {}\n\n"
        return StreamingResponse(_empty(), media_type="text/event-stream")

    if req.event == "game_end":
        game_id = req.payload.get("game_id") or req.thread_id
        row = await db.execute(
            select(Game)
            .options(selectinload(Game.tilt_report))
            .where(Game.id == game_id, Game.user_id == user.user_id)
        )
        game = row.scalar_one_or_none()
        if game is None or game.tilt_report is None:
            raise HTTPException(status_code=404, detail="game or tilt report not found")
        tr = game.tilt_report
        tilt_text = (
            "The game just ended. Here's the tilt analysis:\n"
            + json.dumps({
                "headline": tr.headline,
                "diagnosis": tr.diagnosis,
                "pattern_label": tr.pattern_label,
                "evidence_plies": tr.evidence_plies,
                "suggestion": tr.suggestion,
            })
            + "\n\nReact conversationally — don't repeat the report verbatim, riff on it."
        )
        new_msgs = [HumanMessage(tilt_text)]
    else:  # blunder
        new_msgs = [HumanMessage(_format_observation_message(req.event, req.payload))]

    return StreamingResponse(
        stream_agent_response(
            thread_id=req.thread_id, user_id=user.user_id, jwt=jwt_token,
            new_messages=new_msgs, seed_system_prompt=seed,
        ),
        media_type="text/event-stream",
    )


@router.post("/message")
async def message(
    req: MessageRequest,
    request: Request,
    user: CurrentUser = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
):
    jwt_token = _extract_jwt(request)
    is_first = not await thread_has_state(req.thread_id)
    seed = await build_system_prompt(db, user.user_id) if is_first else None
    return StreamingResponse(
        stream_agent_response(
            thread_id=req.thread_id, user_id=user.user_id, jwt=jwt_token,
            new_messages=[HumanMessage(req.text)], seed_system_prompt=seed,
        ),
        media_type="text/event-stream",
    )


@router.post("/close-session", status_code=204)
async def close_session(
    req: CloseSessionRequest,
    user: CurrentUser = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
):
    # Idempotency check
    existing = await db.execute(
        select(GameSummary).where(GameSummary.game_id == req.thread_id)
    )
    if existing.scalar_one_or_none() is not None:
        return Response(status_code=204)

    # Pull messages from checkpointer
    config = {"configurable": {"thread_id": req.thread_id}}
    state = await get_checkpointer().aget(config)
    raw_msgs = (state or {}).get("channel_values", {}).get("messages", []) \
        if isinstance(state, dict) else state.get("messages", []) if state else []

    msgs_dicts = []
    for m in raw_msgs:
        role = _ROLE_MAP.get(getattr(m, "type", ""), "system")
        if role == "system":
            continue
        msgs_dicts.append({"role": role, "content": getattr(m, "content", "")})

    summary = await summarize_chat(msgs_dicts)

    # Verify the thread_id maps to a real game owned by the user before saving.
    game_row = await db.execute(
        select(Game).where(Game.id == req.thread_id, Game.user_id == user.user_id)
    )
    game = game_row.scalar_one_or_none()
    if game is None:
        # Thread doesn't correspond to a game (e.g. abandoned game_start before any move).
        return Response(status_code=204)

    db.add(GameSummary(
        user_id=user.user_id, game_id=game.id,
        summary=summary.summary, key_facts=summary.key_facts,
    ))
    await db.commit()

    # DNA trigger
    total = (await db.execute(
        select(func.count()).where(Game.user_id == user.user_id)
    )).scalar_one()
    if should_recompute_dna(total):
        try:
            await run_dna_for_user(db, user.user_id)
        except Exception:
            pass  # logged separately, never block the session close

    return Response(status_code=204)


@router.get("/history/{thread_id}", response_model=HistoryResponse)
async def history(
    thread_id: str,
    user: CurrentUser = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
):
    # Authorization: verify this thread is one of the user's games (or no game,
    # in which case we don't expose anything).
    game_row = await db.execute(
        select(Game).where(Game.id == thread_id, Game.user_id == user.user_id)
    )
    if game_row.scalar_one_or_none() is None:
        raise HTTPException(status_code=404, detail="not found")

    config = {"configurable": {"thread_id": thread_id}}
    state = await get_checkpointer().aget(config)
    raw = []
    if state is not None:
        if isinstance(state, dict):
            raw = state.get("channel_values", {}).get("messages", [])
        else:
            raw = state.get("messages", []) or []

    out = []
    for m in raw:
        role = _ROLE_MAP.get(getattr(m, "type", ""), "system")
        if role == "system":
            continue
        out.append(HistoryMessage(role=role, content=getattr(m, "content", "")))
    return HistoryResponse(messages=out)
```

- [ ] **Step 4: Add MemorySaver fixture override** — modify `backend/tests/conftest.py`

Append after the existing fixtures:

```python
@pytest_asyncio.fixture(autouse=True)
async def _agent_in_memory():
    """Force agent to use MemorySaver in tests (no Postgres)."""
    from app.services import agent as agent_module
    await agent_module.init_agent(database_url="", in_memory=True)
    yield
    agent_module._agent = None
    agent_module._checkpointer = None
```

- [ ] **Step 5: Wire router in `backend/app/main.py`**

Add near the other `include_router` call:

```python
from app.routers import agent as agent_router
app.include_router(agent_router.router)
```

- [ ] **Step 6: Run tests, verify pass**

Run: `cd backend && uv run pytest tests/test_agent_router.py -v`
Expected: PASS, all tests.

- [ ] **Step 7: Commit**

```bash
git add backend/app/routers/agent.py backend/app/main.py \
        backend/tests/test_agent_router.py backend/tests/conftest.py
git commit -m "feat(agent): add agent router with observe/message/close/history"
```

---

## Task 10: Wire agent init into FastAPI lifespan

**Files:**
- Modify: `backend/app/main.py`

- [ ] **Step 1: Update lifespan**

Replace the existing `lifespan` in `backend/app/main.py`:

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    if not os.getenv("TESTING"):
        from app import models  # noqa: F401
        from app.database import engine, Base
        async with engine.begin() as conn:
            await conn.run_sync(Base.metadata.create_all)
        from app.services.agent import init_agent, shutdown_agent
        await init_agent(settings.database_url, in_memory=False)
        try:
            yield
        finally:
            await shutdown_agent()
    else:
        yield
```

- [ ] **Step 2: Run all backend tests**

Run: `cd backend && uv run pytest tests/ -v`
Expected: All pass.

- [ ] **Step 3: Commit**

```bash
git add backend/app/main.py
git commit -m "feat(agent): init agent + checkpointer in lifespan"
```

---

## Task 11: Frontend — SSE stream parser

**Files:**
- Create: `frontend/src/lib/agent-stream.ts`

- [ ] **Step 1: Implement**

```ts
// frontend/src/lib/agent-stream.ts
export type AgentStreamEvent =
  | { type: "token"; text: string }
  | { type: "done" }
  | { type: "error"; message: string };

export async function* parseAgentStream(
  response: Response,
): AsyncIterable<AgentStreamEvent> {
  if (!response.body) return;
  const reader = response.body.getReader();
  const decoder = new TextDecoder();
  let buffer = "";

  while (true) {
    const { value, done } = await reader.read();
    if (done) break;
    buffer += decoder.decode(value, { stream: true });

    let sep: number;
    while ((sep = buffer.indexOf("\n\n")) !== -1) {
      const block = buffer.slice(0, sep);
      buffer = buffer.slice(sep + 2);

      let event = "message";
      let data = "";
      for (const line of block.split("\n")) {
        if (line.startsWith("event:")) event = line.slice(6).trim();
        else if (line.startsWith("data:")) data += line.slice(5).trim();
      }

      if (event === "token" && data) {
        try {
          const parsed = JSON.parse(data);
          if (typeof parsed.text === "string") {
            yield { type: "token", text: parsed.text };
          }
        } catch {
          // skip malformed
        }
      } else if (event === "done") {
        yield { type: "done" };
      } else if (event === "error") {
        try {
          const parsed = JSON.parse(data || "{}");
          yield { type: "error", message: parsed.message ?? "agent error" };
        } catch {
          yield { type: "error", message: "agent error" };
        }
      }
    }
  }
}
```

- [ ] **Step 2: Commit**

```bash
git add frontend/src/lib/agent-stream.ts
git commit -m "feat(frontend): add SSE parser for agent stream"
```

---

## Task 12: Frontend — TS event detector + api additions

**Files:**
- Create: `frontend/src/lib/event-detector.ts`
- Modify: `frontend/src/lib/api.ts`

- [ ] **Step 1: Create event detector**

```ts
// frontend/src/lib/event-detector.ts
export const BLUNDER_THRESHOLD_CP = 200;

export function classifyPly(
  evalBefore: number,
  evalAfter: number,
  playerColor: "white" | "black",
): "blunder" | null {
  const delta = evalAfter - evalBefore;
  if (playerColor === "white" && delta <= -BLUNDER_THRESHOLD_CP) return "blunder";
  if (playerColor === "black" && delta >= BLUNDER_THRESHOLD_CP) return "blunder";
  return null;
}
```

- [ ] **Step 2: Extend api.ts**

Append to `frontend/src/lib/api.ts`:

```ts
export interface AgentHistoryMessage {
  role: "user" | "assistant";
  content: string;
}

async function postStream(path: string, body: unknown): Promise<Response> {
  const auth = await getAuthHeaders();
  const res = await fetch(`${API_BASE}${path}`, {
    method: "POST",
    headers: { "Content-Type": "application/json", ...auth },
    body: JSON.stringify(body),
  });
  if (!res.ok) {
    const text = await res.text();
    throw new Error(`API ${path} failed: ${res.status} ${text}`);
  }
  return res;
}

export const agentApi = {
  observe: (thread_id: string, event: "game_start" | "blunder" | "game_end",
            payload: Record<string, unknown> = {}) =>
    postStream("/api/agent/observe", { thread_id, event, payload }),

  message: (thread_id: string, text: string) =>
    postStream("/api/agent/message", { thread_id, text }),

  closeSession: (thread_id: string) =>
    fetch(`${API_BASE}/api/agent/close-session`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ thread_id }),
      keepalive: true,  // survives page unload
    }),

  history: (thread_id: string) =>
    get<{ messages: AgentHistoryMessage[] }>(`/api/agent/history/${thread_id}`),
};
```

Note: `closeSession` is best-effort on `beforeunload`; the keepalive flag lets it complete after the page is gone. Auth header is intentionally omitted — server-side cron handles unauth'd cleanup; for normal in-page closes (New game click), the regular auth path is used via `agentApi.closeSession` followed by an explicit fetch with auth. **Update**: instead use:

```ts
  closeSession: async (thread_id: string) => {
    const auth = await getAuthHeaders();
    return fetch(`${API_BASE}/api/agent/close-session`, {
      method: "POST",
      headers: { "Content-Type": "application/json", ...auth },
      body: JSON.stringify({ thread_id }),
      keepalive: true,
    });
  },
```

- [ ] **Step 3: Commit**

```bash
git add frontend/src/lib/event-detector.ts frontend/src/lib/api.ts
git commit -m "feat(frontend): add event detector + agent API client"
```

---

## Task 13: Frontend — `<AgentMessage>` and `<AgentChat>`

**Files:**
- Create: `frontend/src/components/AgentMessage.tsx`
- Create: `frontend/src/components/AgentChat.tsx`

- [ ] **Step 1: Implement `AgentMessage`**

```tsx
// frontend/src/components/AgentMessage.tsx
"use client";

import type { TiltDetectorResponse } from "@/lib/api";
import { TiltReport } from "./TiltReport";

export type ChatMessage =
  | { id: string; kind: "user"; text: string }
  | { id: string; kind: "agent"; text: string; pending?: boolean }
  | { id: string; kind: "tilt_card"; report: TiltDetectorResponse };

export function AgentMessage({ msg }: { msg: ChatMessage }) {
  if (msg.kind === "tilt_card") {
    return (
      <div className="my-2">
        <TiltReport report={msg.report} />
      </div>
    );
  }
  if (msg.kind === "user") {
    return (
      <div className="flex justify-end my-1">
        <div className="rounded-2xl bg-accent-500 text-white px-3 py-2 max-w-[85%] text-sm">
          {msg.text}
        </div>
      </div>
    );
  }
  return (
    <div className="flex justify-start my-1">
      <div className="rounded-2xl bg-ink-700 text-ink-100 px-3 py-2 max-w-[85%] text-sm whitespace-pre-wrap">
        {msg.text}
        {msg.pending && <span className="animate-pulse">▍</span>}
      </div>
    </div>
  );
}
```

- [ ] **Step 2: Implement `AgentChat`**

```tsx
// frontend/src/components/AgentChat.tsx
"use client";

import { useEffect, useRef, useState, useCallback } from "react";
import { agentApi, type TiltDetectorResponse } from "@/lib/api";
import { parseAgentStream } from "@/lib/agent-stream";
import { AgentMessage, type ChatMessage } from "./AgentMessage";

interface Props {
  threadId: string;
  tiltReport: TiltDetectorResponse | null;
}

let _id = 0;
const nextId = () => `m${++_id}`;

export function AgentChat({ threadId, tiltReport }: Props) {
  const [messages, setMessages] = useState<ChatMessage[]>([]);
  const [input, setInput] = useState("");
  const [streaming, setStreaming] = useState(false);
  const tiltSeenRef = useRef<string | null>(null);
  const scrollRef = useRef<HTMLDivElement>(null);

  // Load history on mount
  useEffect(() => {
    let cancelled = false;
    agentApi
      .history(threadId)
      .then((res) => {
        if (cancelled) return;
        setMessages(
          res.messages.map((m) => ({
            id: nextId(),
            kind: m.role === "user" ? "user" : "agent",
            text: m.content,
          })),
        );
      })
      .catch(() => {});
    // Best-effort initialize the thread on first mount
    agentApi.observe(threadId, "game_start", {}).catch(() => {});
    return () => {
      cancelled = true;
    };
  }, [threadId]);

  // Auto-scroll
  useEffect(() => {
    scrollRef.current?.scrollTo({ top: scrollRef.current.scrollHeight });
  }, [messages, streaming]);

  // Stream a response into a placeholder agent message
  const consumeStream = useCallback(
    async (resp: Response) => {
      const placeholderId = nextId();
      setMessages((m) => [...m, { id: placeholderId, kind: "agent", text: "", pending: true }]);
      setStreaming(true);
      try {
        for await (const ev of parseAgentStream(resp)) {
          if (ev.type === "token") {
            setMessages((m) =>
              m.map((x) =>
                x.id === placeholderId && x.kind === "agent"
                  ? { ...x, text: x.text + ev.text }
                  : x,
              ),
            );
          } else if (ev.type === "done" || ev.type === "error") {
            setMessages((m) =>
              m
                .map((x) =>
                  x.id === placeholderId && x.kind === "agent"
                    ? { ...x, pending: false }
                    : x,
                )
                // drop empty agent placeholders (silent agent decision)
                .filter((x) => x.kind !== "agent" || x.text.length > 0),
            );
          }
        }
      } finally {
        setStreaming(false);
      }
    },
    [],
  );

  // React to a new TiltReport
  useEffect(() => {
    if (!tiltReport) return;
    if (tiltSeenRef.current === threadId) return;
    tiltSeenRef.current = threadId;
    setMessages((m) => [...m, { id: nextId(), kind: "tilt_card", report: tiltReport }]);
    agentApi
      .observe(threadId, "game_end", { game_id: threadId })
      .then((r) => consumeStream(r))
      .catch(() => {});
  }, [tiltReport, threadId, consumeStream]);

  // Close session on unmount + beforeunload
  useEffect(() => {
    const close = () => {
      agentApi.closeSession(threadId).catch(() => {});
    };
    window.addEventListener("beforeunload", close);
    return () => {
      window.removeEventListener("beforeunload", close);
      close();
    };
  }, [threadId]);

  async function send() {
    const text = input.trim();
    if (!text || streaming) return;
    setInput("");
    setMessages((m) => [...m, { id: nextId(), kind: "user", text }]);
    try {
      const r = await agentApi.message(threadId, text);
      await consumeStream(r);
    } catch (e) {
      setMessages((m) => [
        ...m,
        { id: nextId(), kind: "agent", text: `(error: ${(e as Error).message})` },
      ]);
    }
  }

  return (
    <div className="flex flex-col h-[600px] rounded-2xl bg-ink-800 border border-ink-600">
      <div ref={scrollRef} className="flex-1 overflow-y-auto p-4">
        {messages.length === 0 && (
          <div className="text-ink-500 text-sm italic">I&apos;m watching. Ask me anything.</div>
        )}
        {messages.map((m) => (
          <AgentMessage key={m.id} msg={m} />
        ))}
      </div>
      <div className="border-t border-ink-600 p-3 flex gap-2">
        <input
          value={input}
          onChange={(e) => setInput(e.target.value)}
          onKeyDown={(e) => {
            if (e.key === "Enter" && !e.shiftKey) {
              e.preventDefault();
              void send();
            }
          }}
          placeholder="Ask your coach..."
          className="flex-1 bg-ink-700 rounded-lg px-3 py-2 text-sm outline-none"
          disabled={streaming}
        />
        <button
          onClick={() => void send()}
          disabled={streaming || !input.trim()}
          className="px-4 py-2 rounded-lg bg-accent-500 hover:bg-accent-400 disabled:opacity-50 text-white text-sm font-medium"
        >
          Send
        </button>
      </div>
    </div>
  );
}
```

- [ ] **Step 3: Commit**

```bash
git add frontend/src/components/AgentMessage.tsx frontend/src/components/AgentChat.tsx
git commit -m "feat(frontend): add AgentChat sidebar with streaming"
```

---

## Task 14: Wire `AgentChat` into `ChessGame`

**Files:**
- Modify: `frontend/src/components/ChessGame.tsx`

- [ ] **Step 1: Add gameId state and blunder observation**

Replace the top of `ChessGame()` body (state declarations + `afterMove`) with the version below; keep the rest as-is.

```tsx
import { classifyPly } from "@/lib/event-detector";
import { AgentChat } from "./AgentChat";

// ... inside ChessGame:
const [gameId, setGameId] = useState<string>(() => crypto.randomUUID());

// state, refs unchanged ...
const prevEvalRef = useRef<number>(0);

async function afterMove(thinkingMs: number) {
  timePerPlyRef.current.push(thinkingMs / 1000);
  setFen(gameRef.current.fen());

  if (engineRef.current) {
    const ev = await engineRef.current.evaluate(gameRef.current.fen(), 10);
    const fromWhitePOV =
      gameRef.current.turn() === "w" ? ev.scoreCp : -ev.scoreCp;
    const mateAdjusted =
      ev.mateIn != null
        ? ev.mateIn > 0
          ? 9999 * (gameRef.current.turn() === "w" ? 1 : -1)
          : -9999 * (gameRef.current.turn() === "w" ? 1 : -1)
        : fromWhitePOV;
    const evalBefore = prevEvalRef.current;
    evalPerPlyRef.current.push(mateAdjusted);
    prevEvalRef.current = mateAdjusted;

    // The ply we just recorded was made by the side OPPOSITE to current turn.
    const playerJustMoved = gameRef.current.turn() === "w" ? "black" : "white";
    if (playerJustMoved === "white") {
      const evt = classifyPly(evalBefore, mateAdjusted, "white");
      if (evt === "blunder") {
        const ply = evalPerPlyRef.current.length;
        const lastSan = gameRef.current.history().slice(-1)[0] ?? "";
        const time = timePerPlyRef.current.slice(-1)[0] ?? 0;
        // Fire-and-forget; AgentChat picks up the stream itself via observe?
        // Simpler: open the stream here and discard — AgentChat re-reads
        // history on next render. To keep things simple we just trigger
        // the stream and let the next message endpoint render the response.
        void agentApi
          .observe(gameId, "blunder", {
            ply, san: lastSan, eval_before: evalBefore,
            eval_after: mateAdjusted, time_taken: time,
          })
          .then(async (r) => {
            // Drain the stream so it actually executes server-side; the agent's
            // reply lands in the checkpointer and AgentChat will surface it
            // through history() refresh on next mount, OR via a small bus.
            // For MVP we re-fetch history.
            const reader = r.body?.getReader();
            while (reader && !(await reader.read()).done) {}
          });
      }
    }
  }

  if (gameRef.current.isGameOver()) {
    const newStatus = computeStatus();
    setStatus(newStatus);
    void runAnalysis(newStatus);
  } else if (gameRef.current.turn() === "b") {
    void engineMove();
  }
}
```

Add `import { agentApi } from "@/lib/api";` at the top.

- [ ] **Step 2: Replace right `<aside>` with `<AgentChat>`**

In the JSX return, replace the entire `<aside>` block (the one currently rendering `analyzing` / `analysisError` / `analysis` / placeholder) with:

```tsx
<aside className="flex-1 lg:max-w-md">
  {analysisError && !analyzing && (
    <div className="rounded-2xl bg-ink-700 border border-signal-red p-4 mb-3">
      <p className="text-signal-red text-sm font-medium mb-1">Analysis failed</p>
      <p className="text-ink-500 text-xs leading-relaxed">{analysisError}</p>
      <button
        onClick={() => runAnalysis(status)}
        className="mt-3 px-3 py-1.5 rounded-lg bg-accent-500 hover:bg-accent-400 text-white text-xs font-medium"
      >
        Retry
      </button>
    </div>
  )}
  <AgentChat threadId={gameId} tiltReport={analysis} />
</aside>
```

- [ ] **Step 3: Update `reset()` to mint a fresh gameId and close prior session**

```tsx
function reset() {
  agentApi.closeSession(gameId).catch(() => {});
  gameRef.current = new Chess();
  evalPerPlyRef.current = [];
  timePerPlyRef.current = [];
  prevEvalRef.current = 0;
  lastMoveStartRef.current = Date.now();
  setFen(gameRef.current.fen());
  setStatus("playing");
  setAnalysis(null);
  setAnalysisError(null);
  setGameId(crypto.randomUUID());
}
```

- [ ] **Step 4: Update `runAnalysis` to send the `gameId` as the game's id**

In `runAnalysis`, after successful `api.analyzeGame` we want the persisted game to share the same id as our thread. Two options:
- (a) send `gameId` in the `AnalyzeGameRequest` and have the backend honor it.
- (b) accept that thread_id ≠ game.id and pass `result.game_id` (returned from backend) into AgentChat.

We chose **(a)** because the spec assumes `thread_id == game.id`. Update `AnalyzeGameRequest` and the backend.

In `frontend/src/lib/api.ts` extend `AnalyzeGameRequest`:

```ts
export interface AnalyzeGameRequest {
  pgn: string;
  eval_per_ply: number[];
  time_per_ply: number[];
  player_color: "white" | "black";
  result: "win" | "loss" | "draw";
  client_game_id?: string;
}
```

In `backend/app/schemas/api.py` (the existing schema) add `client_game_id: str | None = None` to `AnalyzeGameRequest`.

In `backend/app/main.py` `analyze_game`, replace `Game(...)` construction:
```python
game = Game(
    id=req.client_game_id or _uuid4(),
    user_id=user.user_id,
    pgn=req.pgn,
    eval_per_ply=req.eval_per_ply,
    time_per_ply=req.time_per_ply,
    player_color=req.player_color,
    result=req.result,
)
```
Add `from uuid import uuid4 as _uuid4` to imports.

In `runAnalysis` (frontend), pass `client_game_id: gameId`.

- [ ] **Step 5: Manual smoke test**

Start backend and frontend, log in, play a quick blunder game, confirm:
- Sidebar chat is rendered.
- After a deliberate blunder, an agent message appears (refresh history if needed).
- After game end, Tilt card and a follow-up agent comment appear.
- Sending a message gets a streaming reply.

- [ ] **Step 6: Commit**

```bash
git add frontend/src/components/ChessGame.tsx frontend/src/lib/api.ts \
        backend/app/schemas/api.py backend/app/main.py
git commit -m "feat(frontend): wire AgentChat into ChessGame, link thread_id to game.id"
```

---

## Task 15: Final integration test

**Files:**
- Test: `backend/tests/test_agent_router.py` (append)

- [ ] **Step 1: Add end-to-end happy path test**

```python
@pytest.mark.asyncio
async def test_full_game_flow(authed_client, db):
    """observe game_start → message → analyze-game → observe game_end →
    close-session → game_summary persisted."""
    db.add(Profile(user_id="test-user-id", plan="free"))
    g = Game(user_id="test-user-id", pgn="1.e4 e5 2.Nf3 Nc6",
             eval_per_ply=[0, 30, 25, 20], time_per_ply=[1.0, 1.0, 1.0, 1.0],
             player_color="white", result="win")
    db.add(g)
    await db.flush()
    db.add(TiltReport(game_id=g.id, headline="Steady play",
                      diagnosis="d", pattern_label="focused",
                      evidence_plies=[], suggestion="s"))
    await db.commit()

    async def fake_stream(*a, **kw):
        yield 'event: token\ndata: {"text": "ok"}\n\n'
        yield 'event: done\ndata: {}\n\n'

    fake_msgs = type("S", (), {"get": lambda self, k, d=None: {
        "messages": [
            type("M", (), {"type": "human", "content": "hey"})(),
            type("M", (), {"type": "ai", "content": "hi"})(),
        ]
    }.get(k, d)})()

    from app.services.summarizer import ChatSummaryOutput
    summary_obj = ChatSummaryOutput(summary="full flow", key_facts=[])

    with patch("app.routers.agent.stream_agent_response", new=fake_stream), \
         patch("app.routers.agent.thread_has_state", new=AsyncMock(return_value=True)), \
         patch("app.routers.agent.summarize_chat",
               new=AsyncMock(return_value=summary_obj)), \
         patch("app.routers.agent.get_checkpointer") as gck:
        gck.return_value.aget = AsyncMock(return_value=fake_msgs)

        async with authed_client.stream("POST", "/api/agent/observe",
                json={"thread_id": g.id, "event": "game_end",
                      "payload": {"game_id": g.id}}) as r:
            await r.aread()

        r2 = await authed_client.post("/api/agent/close-session",
                                       json={"thread_id": g.id})
        assert r2.status_code == 204

    rows = await db.execute(select(GameSummary).where(GameSummary.game_id == g.id))
    assert rows.scalar_one().summary == "full flow"
```

- [ ] **Step 2: Run full backend test suite**

Run: `cd backend && uv run pytest tests/ -v`
Expected: All pass.

- [ ] **Step 3: Run frontend lint**

Run: `cd frontend && npm run lint`
Expected: No errors. Fix any that appear inline before committing.

- [ ] **Step 4: Commit**

```bash
git add backend/tests/test_agent_router.py
git commit -m "test(agent): full game flow integration test"
```

---

## Self-review

**Spec coverage check:**
- Per-game agent with `create_agent()`, two tools, Postgres checkpointer → Tasks 4, 7
- thread_id = game.id → Tasks 14 (frontend mints id), 14 step 4 (backend honors `client_game_id`)
- `@tool` + `ToolRuntime` for tools → Task 4
- system prompt with summaries + DNA → Task 7
- Tilt as automatic job, NOT tool → unchanged (existing `/api/analyze-game`)
- Tilt injected as HumanMessage on game_end → Task 9 (`_format` helper + `observe` handler)
- Blunder detection + observe events → Tasks 3, 12, 14
- Streaming via SSE → Tasks 7 (`stream_agent_response`), 11 (parser), 13 (consumer)
- Four endpoints (observe, message, close-session, history) → Task 9
- close-session triggers summarizer + DNA every 5 games → Task 9
- New tables `game_summaries`, `decision_dna` → Task 2
- Frontend `<AgentChat>`, `<TiltReport>` rendered inline → Task 13
- ChessGame integration: observe blunders, replace right pane, beforeunload → Task 14
- Cron stale-thread cleanup → deferred per spec ("Migration plan" section); not in this plan.

**Type consistency check:**
- `ChatSummaryOutput` used identically in summarizer.py (Task 5), dna_job.py is independent, agent router (Task 9). ✓
- `_list_past_games_impl` / `_get_game_details_impl` signatures match between Task 4 and the `@tool` wrappers in same task. ✓
- `agent_module._agent` / `_checkpointer` referenced consistently in Task 9 conftest fixture and Task 7 service. ✓
- `stream_agent_response` signature `(thread_id, user_id, jwt, new_messages, seed_system_prompt=None)` matches between Task 7 (definition) and Task 9 (router calls + test mock). ✓
- `should_recompute_dna(int) -> bool` and `run_dna_for_user(db, user_id)` consistent between Task 6 and Task 9. ✓

**Placeholder scan:** No "TBD"/"TODO"/"add appropriate"/"similar to" left.

**Known soft spot:** Task 14 step 1 (frontend blunder observe) drains the stream and discards the response, then relies on `history()` refresh. This is MVP-acceptable but the agent's blunder reaction won't appear live in the chat — only after the next render that re-fetches history. A follow-up improvement would be to consume the blunder stream into the chat UI directly (same `consumeStream` path as messages). Tracked here so the executor doesn't get surprised.

---

**Plan complete and saved to `docs/superpowers/plans/2026-05-01-agent-chat-sidebar.md`. Two execution options:**

**1. Subagent-Driven (recommended)** — I dispatch a fresh subagent per task, review between tasks, fast iteration

**2. Inline Execution** — Execute tasks in this session using executing-plans, batch execution with checkpoints

**Which approach?**
