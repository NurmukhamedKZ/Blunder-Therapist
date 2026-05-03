# AgentService Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Refactor the LangGraph agent singleton into a proper `AgentService` class using LangChain 1.x API — typed `AgentContext`, `@dynamic_prompt` middleware with prompt cache, and clean service methods.

**Architecture:** `AgentContext` is a `@dataclass` defined in `agent_tools.py` and used as `context_schema` in `create_agent`. A `@dynamic_prompt` middleware (defined inside `AgentService.init()` as a closure) caches the personalised system prompt per `thread_id` — one DB query per game thread, full prompt on every LLM call. The router drops all seeding logic and calls only `agent_service.stream()` / `agent_service.get_messages()`.

**Tech Stack:** LangChain 1.2.x, LangGraph, FastAPI, SQLAlchemy async, pytest-asyncio

---

## File Map

| File | Action | Responsibility |
|------|--------|----------------|
| `backend/app/services/agent_tools.py` | Modify | Add `AgentContext` dataclass; type `ToolRuntime` generics |
| `backend/app/services/agent.py` | Full rewrite | `AgentService` class + `@dynamic_prompt` + `build_system_prompt` |
| `backend/app/routers/agent.py` | Modify | Remove seed logic; use `agent_service.*` |
| `backend/app/main.py` | Modify | Use `agent_service.init` / `agent_service.shutdown` in lifespan |
| `backend/tests/conftest.py` | Modify | Update `_agent_in_memory` fixture |
| `backend/tests/test_agent_router.py` | Modify | Update all mocks to patch `agent_service` |

---

### Task 1: Add `AgentContext` to `agent_tools.py` and type the tools

**Files:**
- Modify: `backend/app/services/agent_tools.py`

- [ ] **Step 1: Run the existing tool tests to confirm they pass before any change**

```bash
cd backend
uv run pytest tests/test_agent_tools.py -v
```
Expected: all 5 tests PASS.

- [ ] **Step 2: Replace the contents of `agent_tools.py`**

```python
"""LangChain tools the agent can call.

Tools split into two layers:
  - `_*_impl(db, user_id, ...)` — pure async DB function, used by tests.
  - `@tool` wrappers — get user_id from ToolRuntime[AgentContext], call the impl.
"""
from dataclasses import dataclass
from typing import Any

from langchain.tools import tool
from langgraph.prebuilt.tool_node import ToolRuntime
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy.orm import selectinload

from app.database import AsyncSessionLocal
from app.models import Game, TiltReport, GameSummary


@dataclass
class AgentContext:
    user_id: str
    jwt: str


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
async def list_past_games(filter: str | None, runtime: ToolRuntime[AgentContext]) -> list[dict]:
    """List the user's past games, most recent first (max 20).

    Args:
        filter: optional hint. Supported: "losses", "wins", "draws".
                Unknown values are ignored.

    Returns a list of {game_id, played_at, result, player_color, tilt_pattern,
    summary_excerpt}.
    """
    user_id = runtime.context.user_id
    async with AsyncSessionLocal() as db:
        return await _list_past_games_impl(db, user_id, filter)


@tool
async def get_game_details(game_id: str, runtime: ToolRuntime[AgentContext]) -> dict:
    """Fetch the full record for one game, including PGN, evals, times,
    tilt report, and chat summary.

    If the game doesn't exist or doesn't belong to the user, returns
    {"error": "not found"}.
    """
    user_id = runtime.context.user_id
    async with AsyncSessionLocal() as db:
        return await _get_game_details_impl(db, user_id, game_id)
```

- [ ] **Step 3: Run tool tests again to verify they still pass**

```bash
cd backend
uv run pytest tests/test_agent_tools.py -v
```
Expected: all 5 tests PASS. (Impl functions are unchanged; tool wrappers now use `runtime.context.user_id` instead of `runtime.context["user_id"]`.)

- [ ] **Step 4: Commit**

```bash
cd backend
git add app/services/agent_tools.py
git commit -m "refactor: add AgentContext dataclass and type ToolRuntime generics"
```

---

### Task 2: Rewrite `agent.py` as `AgentService` + `@dynamic_prompt`

**Files:**
- Modify: `backend/app/services/agent.py`
- Modify: `backend/tests/conftest.py` (update `_agent_in_memory` fixture)

- [ ] **Step 1: Update the `_agent_in_memory` autouse fixture in `conftest.py`**

Find and replace the entire `_agent_in_memory` fixture (lines ~86-93):

```python
@pytest_asyncio.fixture(autouse=True)
async def _agent_in_memory():
    """Force agent to use MemorySaver in tests (no Postgres)."""
    from app.services.agent import agent_service
    await agent_service.init(database_url="", in_memory=True)
    yield
    agent_service._agent = None
    agent_service._checkpointer = None
    agent_service._prompt_cache = {}
```

- [ ] **Step 2: Replace the entire contents of `agent.py`**

```python
"""AgentService — singleton LangGraph agent with typed context and prompt cache."""
from __future__ import annotations

import json
from typing import AsyncIterator

from langchain.agents import create_agent
from langchain.agents.middleware.types import dynamic_prompt, ModelRequest
from langchain_core.messages import BaseMessage
from langchain_openai import ChatOpenAI
from langgraph.checkpoint.memory import MemorySaver
from langgraph.checkpoint.postgres.aio import AsyncPostgresSaver
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from app.config import settings
from app.database import AsyncSessionLocal
from app.models import GameSummary, DecisionDNA
from app.services.agent_tools import AgentContext, list_past_games, get_game_details

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
    """Build the full personalised system prompt for a user. Used by tests and middleware."""
    sum_rows = await db.execute(
        select(GameSummary)
        .where(GameSummary.user_id == user_id)
        .order_by(GameSummary.created_at.desc())
        .limit(5)
    )
    summaries = list(reversed(sum_rows.scalars().all()))

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


async def _build_prompt_for_user(user_id: str) -> str:
    """Open a fresh DB session and build the prompt. Used by the middleware closure."""
    async with AsyncSessionLocal() as db:
        return await build_system_prompt(db, user_id)


class AgentService:
    def __init__(self) -> None:
        self._agent = None
        self._checkpointer: AsyncPostgresSaver | MemorySaver | None = None
        self._pg_cm = None
        self._prompt_cache: dict[str, str] = {}

    async def init(self, database_url: str, *, in_memory: bool = False) -> None:
        if in_memory:
            self._checkpointer = MemorySaver()
        else:
            pg_url = database_url.replace("postgresql+asyncpg://", "postgresql://")
            self._pg_cm = AsyncPostgresSaver.from_conn_string(pg_url)
            self._checkpointer = await self._pg_cm.__aenter__()
            await self._checkpointer.setup()

        @dynamic_prompt
        async def _personalized_prompt(request: ModelRequest[AgentContext]) -> str:
            thread_id = (
                request.runtime.execution_info.thread_id
                if request.runtime and request.runtime.execution_info
                else None
            )
            if thread_id and thread_id in self._prompt_cache:
                return self._prompt_cache[thread_id]
            user_id = request.runtime.context.user_id
            prompt = await _build_prompt_for_user(user_id)
            if thread_id:
                self._prompt_cache[thread_id] = prompt
            return prompt

        self._agent = create_agent(
            model=ChatOpenAI(
                model=settings.model_smart,
                api_key=settings.openai_api_key,
                temperature=0.6,
            ),
            tools=[list_past_games, get_game_details],
            checkpointer=self._checkpointer,
            context_schema=AgentContext,
            middleware=[_personalized_prompt],
        )

    async def shutdown(self) -> None:
        if self._pg_cm is not None:
            await self._pg_cm.__aexit__(None, None, None)
            self._pg_cm = None

    async def stream(
        self,
        thread_id: str,
        context: AgentContext,
        messages: list[BaseMessage],
    ) -> AsyncIterator[str]:
        """Yield SSE-formatted strings (event/data lines) for the agent's response."""
        config = {"configurable": {"thread_id": thread_id}}
        async for chunk, _meta in self._agent.astream(
            {"messages": messages},
            config=config,
            context=context,
            stream_mode="messages",
        ):
            text = getattr(chunk, "content", None)
            if isinstance(text, str) and text:
                yield f"event: token\ndata: {json.dumps({'text': text})}\n\n"
        yield "event: done\ndata: {}\n\n"

    async def has_state(self, thread_id: str) -> bool:
        """Returns True if the checkpointer has messages for this thread."""
        config = {"configurable": {"thread_id": thread_id}}
        state = await self._checkpointer.aget(config)
        return state is not None and bool(state.get("channel_values", {}).get("messages"))

    async def get_messages(self, thread_id: str) -> list[BaseMessage]:
        """Returns all stored messages for the thread (for close-session and history)."""
        config = {"configurable": {"thread_id": thread_id}}
        state = await self._checkpointer.aget(config)
        if state is None:
            return []
        if isinstance(state, dict):
            return state.get("channel_values", {}).get("messages", [])
        return state.get("messages", []) or []


agent_service = AgentService()
```

- [ ] **Step 3: Run `test_agent_tools.py` to confirm `build_system_prompt` tests still pass**

```bash
cd backend
uv run pytest tests/test_agent_tools.py -v
```
Expected: all 5 tests PASS.

- [ ] **Step 4: Commit**

```bash
cd backend
git add app/services/agent.py tests/conftest.py
git commit -m "refactor: replace agent globals with AgentService class and dynamic_prompt middleware"
```

---

### Task 3: Update `routers/agent.py` and router tests

**Files:**
- Modify: `backend/app/routers/agent.py`
- Modify: `backend/tests/test_agent_router.py`

- [ ] **Step 1: Replace the contents of `routers/agent.py`**

```python
"""Agent chat endpoints."""
import json
from fastapi import APIRouter, Depends, HTTPException, Request
from fastapi.responses import StreamingResponse, Response
from langchain_core.messages import HumanMessage
from sqlalchemy import select, func
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy.orm import selectinload

from app.database import get_db
from app.dependencies import CurrentUser, get_current_user
from app.models import Game, TiltReport, GameSummary
from app.schemas.agent import (
    ObserveRequest, MessageRequest, CloseSessionRequest,
    HistoryMessage, HistoryResponse,
)
from app.services.agent import agent_service
from app.services.agent_tools import AgentContext
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
    ctx = AgentContext(user_id=user.user_id, jwt=jwt_token)

    if req.event == "game_start":
        async def _empty():
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
    else:
        new_msgs = [HumanMessage(_format_observation_message(req.event, req.payload))]

    return StreamingResponse(
        agent_service.stream(req.thread_id, ctx, new_msgs),
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
    ctx = AgentContext(user_id=user.user_id, jwt=jwt_token)
    return StreamingResponse(
        agent_service.stream(req.thread_id, ctx, [HumanMessage(req.text)]),
        media_type="text/event-stream",
    )


@router.post("/close-session", status_code=204)
async def close_session(
    req: CloseSessionRequest,
    user: CurrentUser = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
):
    existing = await db.execute(
        select(GameSummary).where(GameSummary.game_id == req.thread_id)
    )
    if existing.scalar_one_or_none() is not None:
        return Response(status_code=204)

    raw_msgs = await agent_service.get_messages(req.thread_id)
    msgs_dicts = []
    for m in raw_msgs:
        role = _ROLE_MAP.get(getattr(m, "type", ""), "system")
        if role == "system":
            continue
        msgs_dicts.append({"role": role, "content": getattr(m, "content", "")})

    summary = await summarize_chat(msgs_dicts)

    game_row = await db.execute(
        select(Game).where(Game.id == req.thread_id, Game.user_id == user.user_id)
    )
    game = game_row.scalar_one_or_none()
    if game is None:
        return Response(status_code=204)

    db.add(GameSummary(
        user_id=user.user_id, game_id=game.id,
        summary=summary.summary, key_facts=summary.key_facts,
    ))
    await db.commit()

    total = (await db.execute(
        select(func.count()).where(Game.user_id == user.user_id)
    )).scalar_one()
    if should_recompute_dna(total):
        try:
            await run_dna_for_user(db, user.user_id)
        except Exception:
            pass

    return Response(status_code=204)


@router.get("/history/{thread_id}", response_model=HistoryResponse)
async def history(
    thread_id: str,
    user: CurrentUser = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
):
    raw = await agent_service.get_messages(thread_id)
    out = []
    for m in raw:
        role = _ROLE_MAP.get(getattr(m, "type", ""), "system")
        if role == "system":
            continue
        out.append(HistoryMessage(role=role, content=getattr(m, "content", "")))
    return HistoryResponse(messages=out)
```

- [ ] **Step 2: Replace the contents of `tests/test_agent_router.py`**

```python
import pytest
import json
from unittest.mock import AsyncMock, patch

from langchain_core.messages import HumanMessage, AIMessage
from sqlalchemy import select
from app.models import Profile, Game, TiltReport, GameSummary, DecisionDNA
from app.services.agent import agent_service


async def _drain_sse(response) -> list[dict]:
    events = []
    async for line in response.aiter_lines():
        if line.startswith("data: "):
            events.append(json.loads(line[6:]))
    return events


@pytest.mark.asyncio
async def test_observe_game_start_returns_done(authed_client, db):
    db.add(Profile(user_id="test-user-id", plan="free"))
    await db.commit()

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

    async def fake_stream(thread_id, ctx, messages):
        yield 'event: token\ndata: {"text": "Hmm"}\n\n'
        yield 'event: done\ndata: {}\n\n'

    with patch.object(agent_service, "stream", new=fake_stream):
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

    async def fake_stream(thread_id, ctx, messages):
        captured["msgs"] = messages
        yield 'event: done\ndata: {}\n\n'

    with patch.object(agent_service, "stream", new=fake_stream):
        async with authed_client.stream(
            "POST", "/api/agent/observe",
            json={"thread_id": g.id, "event": "game_end",
                  "payload": {"game_id": g.id}},
        ) as r:
            await r.aread()

    assert "Rushed in middlegame" in captured["msgs"][0].content


@pytest.mark.asyncio
async def test_message_streams(authed_client, db):
    db.add(Profile(user_id="test-user-id", plan="free"))
    await db.commit()

    async def fake_stream(thread_id, ctx, messages):
        yield 'event: token\ndata: {"text": "hi"}\n\n'
        yield 'event: done\ndata: {}\n\n'

    with patch.object(agent_service, "stream", new=fake_stream):
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

    fake_msgs = [HumanMessage("hey")]

    from app.services.summarizer import ChatSummaryOutput
    summary_obj = ChatSummaryOutput(summary="we chatted", key_facts=["a"])

    with patch.object(agent_service, "get_messages", new=AsyncMock(return_value=fake_msgs)), \
         patch("app.routers.agent.summarize_chat", new=AsyncMock(return_value=summary_obj)):
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

    with patch.object(agent_service, "get_messages", new=AsyncMock(return_value=[])), \
         patch("app.routers.agent.run_dna_for_user", new=AsyncMock()) as dna_mock:
        r = await authed_client.post("/api/agent/close-session",
                                     json={"thread_id": games[-1].id})
        assert r.status_code == 204
        dna_mock.assert_awaited_once()


@pytest.mark.asyncio
async def test_history_returns_messages(authed_client, db):
    db.add(Profile(user_id="test-user-id", plan="free"))
    await db.commit()

    fake_msgs = [HumanMessage("hi"), AIMessage("hello")]

    with patch.object(agent_service, "get_messages", new=AsyncMock(return_value=fake_msgs)):
        r = await authed_client.get("/api/agent/history/g-1")
        assert r.status_code == 200
        data = r.json()
        assert data["messages"][0]["role"] == "user"
        assert data["messages"][1]["role"] == "assistant"


@pytest.mark.asyncio
async def test_full_game_flow(authed_client, db):
    """observe game_end → close-session → game_summary persisted."""
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

    async def fake_stream(thread_id, ctx, messages):
        yield 'event: token\ndata: {"text": "ok"}\n\n'
        yield 'event: done\ndata: {}\n\n'

    fake_msgs = [HumanMessage("hey"), AIMessage("hi")]

    from app.services.summarizer import ChatSummaryOutput
    summary_obj = ChatSummaryOutput(summary="full flow", key_facts=[])

    with patch.object(agent_service, "stream", new=fake_stream), \
         patch.object(agent_service, "get_messages", new=AsyncMock(return_value=fake_msgs)), \
         patch("app.routers.agent.summarize_chat", new=AsyncMock(return_value=summary_obj)):

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

- [ ] **Step 3: Run all router tests**

```bash
cd backend
uv run pytest tests/test_agent_router.py -v
```
Expected: all 9 tests PASS.

- [ ] **Step 4: Commit**

```bash
cd backend
git add app/routers/agent.py tests/test_agent_router.py
git commit -m "refactor: update agent router to use AgentService; remove seed/checkpointer logic"
```

---

### Task 4: Update `main.py` lifespan

**Files:**
- Modify: `backend/app/main.py`

- [ ] **Step 1: Replace the lifespan block in `main.py`**

Find the current lifespan function and replace it:

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    if not os.getenv("TESTING"):
        from app import models  # noqa: F401
        from app.database import engine, Base
        async with engine.begin() as conn:
            await conn.run_sync(Base.metadata.create_all)
        from app.services.agent import agent_service
        await agent_service.init(settings.database_url, in_memory=False)
        try:
            yield
        finally:
            await agent_service.shutdown()
    else:
        yield
```

- [ ] **Step 2: Run the full test suite**

```bash
cd backend
uv run pytest tests/ -v
```
Expected: all tests PASS. Zero failures.

- [ ] **Step 3: Commit**

```bash
cd backend
git add app/main.py
git commit -m "refactor: use agent_service.init/shutdown in FastAPI lifespan"
```
