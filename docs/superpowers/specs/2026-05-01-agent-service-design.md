# AgentService Design

**Date:** 2026-05-01  
**Scope:** `backend/app/services/agent.py`, `backend/app/services/agent_tools.py`, `backend/app/routers/agent.py`

## Problem

`agent.py` currently uses global variables (`_agent`, `_checkpointer`, `_pg_cm`) and procedural functions. Tools access user context via an untyped dict (`runtime.context["user_id"]`). Routers manually handle system prompt seeding (`build_system_prompt`, `seed_system_prompt`, `thread_has_state`). This needs to be a proper `AgentService` class using the LangChain 1.x API correctly.

## Design

### 1. `AgentContext` dataclass

```python
@dataclass
class AgentContext:
    user_id: str
    jwt: str
```

Passed to `create_agent(context_schema=AgentContext)`. Replaces the untyped `context={"user_id": ..., "jwt": ...}` dict.

### 2. `@dynamic_prompt` middleware + prompt cache

Injected into `create_agent(middleware=[personalized_prompt])`. Runs before every model call. The full system prompt (BASE + MEMORY + DNA) must be available on **every** LLM call so the model always has the user's history and style profile.

`AgentService` holds an in-memory cache:

```python
self._prompt_cache: dict[str, str] = {}  # thread_id → full system prompt
```

Middleware logic:

- If `thread_id` is already in `_prompt_cache` → return cached full prompt immediately (no DB query).
- If not in cache (first call of the thread) → open `AsyncSessionLocal`, query last 5 `GameSummary` rows + latest `DecisionDNA` for `runtime.context.user_id`, build full prompt, store in `_prompt_cache[thread_id]`, return it.

This guarantees **exactly one DB query per thread** (per game) while ensuring every subsequent LLM call still receives the full memory/DNA context. Cache lives in process memory — on server restart a single new DB query is made per active thread. Routers no longer call `build_system_prompt` or pass `seed_system_prompt`.

### 3. `AgentService` class

```python
class AgentService:
    def __init__(self) -> None:
        self._agent = None
        self._checkpointer: AsyncPostgresSaver | MemorySaver | None = None
        self._pg_cm = None
        self._prompt_cache: dict[str, str] = {}  # thread_id → full system prompt

    async def init(self, database_url: str, *, in_memory: bool = False) -> None:
        """Called in FastAPI lifespan on startup."""

    async def shutdown(self) -> None:
        """Called in FastAPI lifespan on shutdown. Closes AsyncPostgresSaver context."""

    async def stream(
        self,
        thread_id: str,
        context: AgentContext,
        messages: list[BaseMessage],
    ) -> AsyncIterator[str]:
        """Yields SSE-formatted strings: event: token / event: done."""

    async def has_state(self, thread_id: str) -> bool:
        """Returns True if the checkpointer has messages for this thread."""

    async def get_messages(self, thread_id: str) -> list[BaseMessage]:
        """Returns all stored messages for a thread (for close-session and history)."""


agent_service = AgentService()  # module-level singleton
```

### 4. Updated tools

```python
@tool
async def list_past_games(filter: str, runtime: ToolRuntime[AgentContext]) -> str:
    user_id = runtime.context.user_id  # typed access, not dict

@tool
async def get_game_details(game_id: str, runtime: ToolRuntime[AgentContext]) -> str:
    user_id = runtime.context.user_id
```

Both tools still open a fresh `AsyncSessionLocal` (same as now — do NOT reuse request session).

### 5. Router changes

`routers/agent.py` imports only `agent_service` and `AgentContext`. Removed imports: `build_system_prompt`, `stream_agent_response`, `thread_has_state`, `get_checkpointer`.

Every endpoint that currently does:
```python
is_first = not await thread_has_state(req.thread_id)
seed = await build_system_prompt(db, user.user_id) if is_first else None
return StreamingResponse(stream_agent_response(..., seed_system_prompt=seed), ...)
```

becomes:
```python
return StreamingResponse(
    agent_service.stream(req.thread_id, AgentContext(user.user_id, jwt_token), messages),
    media_type="text/event-stream",
)
```

`close-session` and `history` replace direct checkpointer access with `agent_service.get_messages(thread_id)`.

## What does NOT change

- `thread_id == game.id` convention — unchanged.
- `BASE_SYSTEM` prompt content — unchanged.
- `summarizer.py`, `dna_job.py`, `features.py`, `tilt_detector.py` — untouched.
- `close-session` logic (idempotency, DNA trigger) — same, just uses `agent_service.get_messages()`.
- Test fixtures — `_agent_in_memory` autouse fixture calls `agent_service.init("", in_memory=True)`.
- SSE format (`event: token` / `event: done`) — unchanged.

## Files changed

| File | Change |
|------|--------|
| `services/agent.py` | Full rewrite: `AgentService` class + `@dynamic_prompt` + `AgentContext` |
| `services/agent_tools.py` | Update tool signatures to `ToolRuntime[AgentContext]` |
| `routers/agent.py` | Remove seed logic, use `agent_service.*` |
| `main.py` | `init_agent()` → `agent_service.init()` in lifespan |
