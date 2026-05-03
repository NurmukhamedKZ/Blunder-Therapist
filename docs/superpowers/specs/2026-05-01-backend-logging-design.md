# Backend Logging Design

**Date:** 2026-05-01  
**Status:** Approved

## Overview

Add structured logging to the entire FastAPI backend using `structlog`. Supports two formats: human-readable `console` (dev) and `json` (prod), toggled via env var. All logs within a request automatically carry `request_id` and `user_id` via structlog contextvars.

## Infrastructure

### New file: `app/logging_setup.py`

Contains two exports:

**`setup_logging()`** — configures structlog once at startup:
- Shared processors: `merge_contextvars`, `add_log_level`, `TimeStamper(fmt="iso")`
- Renderer: `JSONRenderer` when `LOG_FORMAT=json`, `ConsoleRenderer` otherwise
- Routes stdlib logging through structlog so asyncpg/httpx/uvicorn land in the same sink
- Filter level set from `LOG_LEVEL` setting

**`ToolCallLogger(component)`** — LangChain `BaseCallbackHandler`:
- `on_tool_start`: logs `tool_call_start` with tool name and input
- `on_tool_end`: logs `tool_call_end` with `duration_ms` and first 200 chars of output
- `on_tool_error`: logs `tool_call_error` with `duration_ms` and error string

### Config (`app/config.py`)

Two new fields added to `Settings`:
```
log_level: str = "INFO"    # DEBUG / INFO / WARNING / ERROR
log_format: str = "console"  # "console" (dev) | "json" (prod)
```

### Entry point (`app/main.py`)

`setup_logging()` called first, before any other imports.

## HTTP Middleware

`RequestLoggingMiddleware` (Starlette `BaseHTTPMiddleware`) added in `main.py`:

- On each request: call `clear_contextvars()` first (prevents context leakage between async tasks), then generate `request_id` (UUID4), bind it + path + method via `bind_contextvars`
- Log `request_start`: method, path, client IP
- Log `request_end`: status_code, duration_ms
- Log `request_error` with `exc_info=True` for unhandled exceptions

`user_id` is bound to contextvars inside `get_current_user` in `dependencies.py` after successful auth, so it appears automatically in all subsequent logs for that request.

## Per-layer Logging

### `app/dependencies.py`
- `auth_error`: on JWT decode failure (level WARNING, no exc details leaked)
- `new_user_created`: when a new Profile row is auto-created

### `app/routers/agent.py`

Module-level `log = structlog.get_logger()`, bound per-call with `thread_id`.

| Endpoint | Events |
|---|---|
| `POST /observe` | `observe_event` (event type, thread_id) |
| `POST /message` | `message_received` (thread_id, text length) |
| `POST /close-session` | `close_session_start`, `close_session_skipped` (if idempotent), `close_session_done` (msg count, dna_triggered bool) |
| `GET /history/{thread_id}` | `history_fetched` (message count) |

### `app/routers/games.py`
- `game_list`: page, count returned
- `game_detail`: game_id
- `report_requested`: game_id

### `app/main.py` endpoints
- `analyze_game_start` / `analyze_game_done` (duration_ms, game_id)
- `analyze_game_llm_error`: on 502 path
- `decision_dna_request` / `decision_dna_done` (duration_ms)
- `coach_request` / `coach_done` (duration_ms)

### `app/services/agent.py`
- `agent_init`: checkpointer type (postgres / memory)
- `agent_shutdown`
- `prompt_cache_hit` / `prompt_cache_miss` (thread_id, user_id)
- `stream_start` (thread_id, message count)
- `stream_done` (thread_id, duration_ms, token count approx)
- `stream_error` (thread_id, exc_info=True)

`ToolCallLogger("agent")` passed as callback in `stream()` — covers all tool calls automatically.

### `app/services/tilt_detector.py`
- `tilt_start` / `tilt_done` (duration_ms)
- `tilt_error` (exc_info=True)

### `app/services/dna_decision.py`
- `dna_start` (game count) / `dna_done` (duration_ms)

### `app/services/summarizer.py`
- `summarize_start` (message count) / `summarize_done` (duration_ms)

### `app/services/dna_job.py`
- `dna_job_triggered` (user_id, total_games)
- `dna_job_done` (duration_ms)
- `dna_job_error` (exc_info=True)

### `app/services/coach.py`
- `coach_start` / `coach_done` (duration_ms)

## ToolCallLogger Integration

In `AgentService.stream()`:

```python
from app.logging_setup import ToolCallLogger

callbacks = [ToolCallLogger("agent")]
async for chunk, _meta in self._agent.astream(
    {"messages": messages},
    config={**config, "callbacks": callbacks},
    context=context,
    stream_mode="messages",
):
    ...
```

## Environment Variables

```
LOG_LEVEL=INFO        # DEBUG / INFO / WARNING / ERROR
LOG_FORMAT=console    # console (dev) | json (prod)
```

Add both to `backend/.env.example`.

## Dependencies

Add `structlog` to `backend/pyproject.toml` (and regenerate `uv.lock`).

## Testing

- Existing tests must not break — `setup_logging()` is safe to call in test env
- No new tests required for logging itself (it's infrastructure, not business logic)
- LLM mocks in existing tests remain unchanged; `ToolCallLogger` is a no-op when agent is mocked
