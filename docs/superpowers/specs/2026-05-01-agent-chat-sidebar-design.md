# Agent Chat Sidebar — Design Spec

**Date:** 2026-05-01
**Status:** Draft, pending review

## Problem

Today the right sidebar shows a static `<TiltReport>` card after the game ends. The three AI features (Tilt Detector, Decision DNA, Memory-aware Coach) are three independent stateless LLM calls glued onto the UI. There's no continuous interaction, no sense of an "assistant watching the game", and no memory between games.

We want the sidebar to feel like a **per-game AI assistant** that:

- watches the game in real time and reacts to behaviorally meaningful events (not every move),
- runs the Tilt analysis at the end of the game,
- lets the user chat with it freely throughout,
- carries useful memory across games via post-game chat summaries and a periodically refreshed Decision DNA profile.

## Non-goals

- A general "Memory-aware Coach" surface that lives outside of a specific game. The coach disappears as a separate concept; everything happens inside the per-game thread.
- An engine-style move advisor. The assistant is explicitly behavioral, not tactical, and refuses requests like "what should I play here?".
- Live commentary on every move. Only blunders and game-end trigger automatic agent reactions.
- A separate long-term memory store managed by the agent itself (no `remember()` tool, no langgraph `BaseStore`). Cross-game memory is owned by deterministic post-game jobs writing to ordinary tables.

## Architecture overview

```
                                +------------------+
   browser  -- POST observe --> |                  |
   browser  -- POST message --> |   FastAPI        | -- AsyncPostgresSaver --> Supabase
   browser  -- POST close   --> |   /api/agent/*   |     (checkpointer)         Postgres
   browser  <-- SSE stream  --- |                  |
                                +--------+---------+
                                         |
                                         | LangChain >=1.2 create_agent()
                                         v
                                +------------------+
                                |   coach_agent    |  tools:
                                |   thread = gid   |   - list_past_games
                                |                  |   - get_game_details
                                +------------------+
```

### Components

**Existing, unchanged:**
- `services/features.py` — feature extraction.
- `services/llm.run_tilt_detector` — Tilt LLM call with structured output.
- `services/llm.run_decision_dna` — DNA LLM call.
- `routers/games.py` — game history endpoints.
- `POST /api/analyze-game` — keeps the same contract: extract features, run Tilt, save `Game` and `TiltReport`. The agent is layered on top, not in front of, this pipeline.

**New:**
- `services/agent.py` — singleton `create_agent(model, tools, checkpointer, prompt)`, dynamic system prompt builder, two tool implementations.
- `services/event_detector.py` — `classify_ply(eval_before, eval_after, time_taken) -> "blunder" | None`. Pure function, no LLM.
- `services/summarizer.py` — `summarize_chat(messages, tilt_report) -> {summary: str, key_facts: list[str]}`. One LLM call, structured output.
- `services/dna_job.py` — wrapper around existing `run_decision_dna` that pulls the user's last 5 games and upserts into `decision_dna`.
- `routers/agent.py` — four endpoints (see below).
- `models.GameSummary` — new ORM model + table.
- `models.DecisionDNA` — new ORM model + table.

**Frontend new:**
- `components/AgentChat.tsx` — sidebar chat UI with SSE stream parser.
- `components/AgentMessage.tsx` — renders one message; switches on `kind` to render text, `<TiltReport>`, or DNA card inline.
- `lib/agent-stream.ts` — `parseSSE(response: Response): AsyncIterable<AgentEvent>`.
- `lib/api.ts` — additions: `observe`, `sendMessage`, `closeSession`, `getHistory`.

**Frontend changes:**
- `components/ChessGame.tsx` — after each ply call `event_detector` client-side (cheap pure function in TS) and POST `/api/agent/observe` if it returns `"blunder"`. Replace the right `<aside>` block with `<AgentChat threadId={gameId} />`. On `beforeunload` and on "New game" click, call `closeSession(threadId)`.
- `components/TiltReport.tsx` — no logic change, but it's now rendered inside `AgentChat` as one message kind, not as the entire sidebar.

## Data flow

### Starting a game

1. User clicks "New game" in `ChessGame`. Frontend generates a client-side UUID `gameId` and uses it as `threadId`.
2. First `POST /api/agent/observe` with `{thread_id, event: "game_start"}` lazily creates the agent's checkpoint row. Server-side, this calls `build_system_prompt(user_id)` which loads:
   - the user's last 5 `game_summaries` (oldest-first),
   - the latest `decision_dna` row (or `null` if `total_games < 5`),
   - and stores them as the agent's first system message via the checkpointer.
3. Sidebar renders empty chat with a placeholder ("I'm watching. Ask me anything.").

### During a game (blunder reaction)

1. After every ply, `ChessGame.afterMove` runs `classify_ply(eval_before, eval_after, time_taken)`. If `blunder` → `POST /api/agent/observe` with `{thread_id, event: "blunder", ply, san, eval_before, eval_after, time_taken}`.
2. Backend appends a `HumanMessage` of the form `[OBSERVATION] Ply 17: blunder, eval -2.8 → +1.5, 4s think.` to the thread and invokes the agent.
3. The agent's system prompt instructs it to react with one short comment (≤2 sentences) and only if the most recent agent message is at least 5 plies old. Otherwise it should return an empty response.
4. Response streams back as SSE; frontend appends to chat.

### During a game (user chat)

1. User types in the input → `POST /api/agent/message` with `{thread_id, text}`.
2. Standard agent invocation; tools `list_past_games` and `get_game_details` are available.
3. Response streams back. The agent is forbidden by system prompt from giving move advice on the current position; if asked, it redirects to behavioral framing.

### Game end

1. `ChessGame` detects game over, sends `POST /api/analyze-game` (existing endpoint). This synchronously runs Tilt and saves `TiltReport`. Returns the report.
2. Frontend receives the Tilt response and immediately sends `POST /api/agent/observe` with `{thread_id, event: "game_end", tilt_report_id}`.
3. Backend loads the `TiltReport` from DB and appends it to the same thread as a `HumanMessage` of the form:
   ```
   The game just ended. Here's the tilt analysis:
   {full TiltReport JSON}
   Please react conversationally — don't repeat the report verbatim, riff on it.
   ```
   This keeps the thread a coherent dialogue: the agent saw the blunders during the game, and now "the user" (in the message stream sense) shares the post-game analysis. The agent's reply is a natural next turn.
4. Agent streams a soft, conversational reaction. Frontend renders the original Tilt card (`<TiltReport>`) as one message AND streams the agent's text as another message after it.

### Session close

1. Triggers (any one):
   - User clicks "New game" in `ChessGame`.
   - `beforeunload` on the page (best-effort, fire-and-forget POST).
   - Cron job (every 30 minutes) finds checkpoints older than 1 hour with no `game_summary` row and closes them.
2. `POST /api/agent/close-session` with `{thread_id}`.
3. Backend loads all messages from the checkpoint, calls `summarize_chat(messages, tilt_report) -> {summary, key_facts}`, saves a new `game_summaries` row.
4. Backend then queries `count(games where user_id = X)`. If `count % 5 == 0`, enqueue/run `dna_job(user_id)` which calls `run_decision_dna` on the last 5 games and upserts `decision_dna`.
5. Endpoint returns 204. Frontend doesn't wait for the result; the next game will pick up the new memory.

### Returning to an old game

1. User opens `/games/{game_id}` (existing route). Frontend calls `GET /api/agent/history/{game_id}`.
2. Backend loads thread messages from the checkpointer, returns them. Chat UI renders the conversation read-only-ish (input still works; agent responds; the thread continues from where it left off).

## Schema additions

```python
# app/models.py additions

class GameSummary(Base):
    __tablename__ = "game_summaries"
    id: Mapped[str] = mapped_column(String, primary_key=True, default=...)  # UUID
    user_id: Mapped[str] = mapped_column(String, ForeignKey("profiles.user_id"), index=True)
    game_id: Mapped[str] = mapped_column(String, ForeignKey("games.id"), unique=True)
    summary: Mapped[str] = mapped_column(Text)
    key_facts: Mapped[list[str]] = mapped_column(JSON, default=list)
    created_at: Mapped[datetime] = mapped_column(DateTime, default=datetime.utcnow)


class DecisionDNA(Base):
    __tablename__ = "decision_dna"
    id: Mapped[str] = mapped_column(String, primary_key=True, default=...)  # UUID
    user_id: Mapped[str] = mapped_column(String, ForeignKey("profiles.user_id"), index=True)
    dna: Mapped[dict] = mapped_column(JSON)  # {type_name, tagline, summary, core_strength, core_weakness, gm_comparison}
    games_count: Mapped[int] = mapped_column(Integer)  # snapshot of total games at compute time
    computed_at: Mapped[datetime] = mapped_column(DateTime, default=datetime.utcnow)

    # Latest row per user is the "current" DNA. We keep history for the analytics graph later.
```

`tilt_reports`, `games`, `profiles`: unchanged.

`AsyncPostgresSaver` creates and manages its own tables (`checkpoints`, `checkpoint_writes`, etc.) via `await checkpointer.setup()` in the FastAPI lifespan. We do not manage those tables ourselves.

## Agent configuration

```python
# app/services/agent.py

from langchain.agents import create_agent
from langchain_openai import ChatOpenAI
from langgraph.checkpoint.postgres.aio import AsyncPostgresSaver

_checkpointer: AsyncPostgresSaver | None = None
_agent = None

async def init_agent(database_url: str) -> None:
    global _checkpointer, _agent
    _checkpointer = AsyncPostgresSaver.from_conn_string(database_url)
    await _checkpointer.setup()
    _agent = create_agent(
        model=ChatOpenAI(model=settings.model_smart, temperature=0.6),
        tools=[list_past_games_tool, get_game_details_tool],
        checkpointer=_checkpointer,
        # prompt is built dynamically per-call via SystemMessage injection;
        # see invoke_agent() below.
    )

async def invoke_agent(
    thread_id: str, user_id: str, jwt: str, new_message: BaseMessage,
) -> AsyncIterable[str]:
    config = {"configurable": {"thread_id": thread_id}}
    context = {"user_id": user_id, "jwt": jwt}
    state = await _checkpointer.aget(config)
    if state is None or not state.values.get("messages"):
        # First call for this thread: build and inject the dynamic system prompt.
        system = await build_system_prompt(user_id)
        messages = [system, new_message]
    else:
        messages = [new_message]
    async for chunk in _agent.astream(
        {"messages": messages}, config=config, context=context, stream_mode="messages"
    ):
        yield chunk
```

`build_system_prompt(user_id)` composes:

1. Static base prompt (role, rules, "behavioral not tactical", reaction frequency rule).
2. Last 5 `game_summaries` (or fewer) formatted as a bulleted "what we've talked about before" section.
3. Latest `decision_dna` JSON formatted as a "your style profile" paragraph (or "Not enough games yet" if absent).

`tilt_report` is NOT in the system prompt; it's appended as a `HumanMessage` mid-thread on `game_end` so it reads as a natural turn in the conversation (the agent saw the game live, and now sees the post-game analysis as the next message).

Both tools use the `@tool` decorator from `langchain.tools` and receive a `ToolRuntime` parameter — LangChain 1.x's standard mechanism for passing runtime context (state, store, config) into tools. The runtime carries `user_id` (extracted server-side from the Supabase JWT in the FastAPI dependency) and, if needed, the JWT itself for downstream calls. `ToolRuntime` parameters are never exposed in the model-facing tool schema.

```python
from langchain.tools import tool, ToolRuntime

@tool
async def list_past_games(
    filter: str | None,
    runtime: ToolRuntime,
) -> list[dict]:
    user_id = runtime.context["user_id"]
    # ... SQL ...
```

The agent is invoked with `context={"user_id": uid, "jwt": token}` and `config={"configurable": {"thread_id": gid}}`, so every tool call is automatically scoped to the current user.

### Tool: `list_past_games`

```python
@tool
async def list_past_games(filter: str | None = None) -> list[dict]:
    """List the user's past games, most recent first.

    Args:
        filter: optional natural-language hint like "losses", "blunder cascades",
                "last week". The tool maps these to SQL filters; unknown filters
                fall through to "no filter".
    Returns:
        List of {game_id, played_at, result, tilt_pattern, summary_excerpt}, max 20.
    """
```

Backed by SQL only. No LLM. The `filter` arg is mapped via a small whitelist of keywords (`losses`, `wins`, `draws`, `cascade`, `tilt`, `panic`, `recent`, `today`, `week`). Unknown values are ignored — the agent learns what works by trying.

### Tool: `get_game_details`

```python
@tool
async def get_game_details(game_id: str) -> dict:
    """Fetch the full record for one game.

    Returns:
        {pgn, eval_per_ply, time_per_ply, player_color, result, played_at,
         tilt_report: {...} | None,
         chat_summary: {summary, key_facts} | None}
    """
```

Backed by SQL only. Authorization: the tool implementation reads `user_id` from the agent's `config.configurable` and refuses any `game_id` that doesn't belong to that user.

## API endpoints

All four endpoints require Supabase JWT (existing `get_current_user` dependency).

### `POST /api/agent/observe`

Request:
```json
{ "thread_id": "uuid", "event": "blunder" | "game_start" | "game_end",
  "payload": { ... event-specific ... } }
```

Returns: `StreamingResponse` with `Content-Type: text/event-stream`. Events:
```
event: token
data: {"text": "Hmm"}

event: token
data: {"text": ", that "}

event: done
data: {}
```

For `game_start` the agent is initialized but doesn't speak; the stream returns immediately with `event: done`. For `game_end` and `blunder` it may stream a reaction or close immediately if the agent decides to stay silent (no message emitted).

### `POST /api/agent/message`

Request: `{thread_id, text}`. Same SSE response shape. Always streams something (the agent always replies to a direct user message).

### `POST /api/agent/close-session`

Request: `{thread_id}`. Synchronously runs the summarizer + the DNA job (if `total_games % 5 == 0`). Returns `204`. Idempotent: if a `game_summary` for this `game_id` already exists, no-op.

### `GET /api/agent/history/{thread_id}`

Returns the thread's messages from the checkpointer as `[{role, content, created_at}, ...]` for rendering when the user revisits an old game. Authorization: the endpoint verifies the thread's `user_id` (stored in checkpoint metadata) matches the caller.

## Frontend

### `AgentChat`

Props: `{threadId: string, gameOver: boolean, tiltReport: TiltReport | null}`.

State:
- `messages: AgentMessage[]` — full transcript including pending streamed message.
- `streaming: boolean`.

On mount: `GET /api/agent/history/{threadId}` and load existing messages (if any).

On `tiltReport` becoming non-null (parent passed it down after `analyze-game` resolves): push a synthetic message `{kind: "tilt_card", report: tiltReport}` into local state AND call `POST /api/agent/observe` with `event: "game_end"` to trigger the agent's reaction.

`POST /api/agent/observe` for blunder events is called from `ChessGame` directly (parent component owns ply state), not from `AgentChat`. Both POSTs share the same `threadId`.

Input box at the bottom is always enabled. Submitting calls `POST /api/agent/message` and parses the SSE response, appending tokens to a pending message until `event: done`.

### Closing the session

Cleanup hook in `AgentChat`:
```ts
useEffect(() => {
  const close = () => {
    navigator.sendBeacon('/api/agent/close-session', JSON.stringify({thread_id: threadId}))
  }
  window.addEventListener('beforeunload', close)
  return () => {
    window.removeEventListener('beforeunload', close)
    close()  // also fires on unmount (e.g. New game)
  }
}, [threadId])
```

`sendBeacon` does not support custom auth headers, so `close-session` accepts a short-lived auth token in the request body (or we accept the limitation that beforeunload-triggered closes are cron-cleaned by the fallback job).

## Error handling

- **Stream interruption**: if the SSE stream errors mid-response, the frontend keeps whatever tokens it received and shows a small "reconnect" link that re-POSTs the same message.
- **Agent invocation failure**: server returns `event: error\ndata: {"message": "..."}` and ends the stream. Frontend renders an inline error and a retry button on that message.
- **Tool failure**: tool raises → caught by the agent → it reports "I couldn't pull up that data right now" and continues.
- **Summarizer failure on close**: log, do not block. Cron picks it up later. The user never sees the failure.
- **DNA job failure**: log, do not block. Next eligible game retries.
- **Authorization mismatch in `get_game_details`**: tool returns `{error: "not found"}`. Agent doesn't expose the existence of other users' games.

## Testing

New tests in `backend/tests/`:

- `test_event_detector.py` — pure function, exhaustive boundary cases for `classify_ply`.
- `test_summarizer.py` — feeds canned message lists, asserts structure of `{summary, key_facts}`. LLM calls mocked.
- `test_agent_router.py` — uses `authed_client` fixture, asserts:
   - `POST /observe game_start` creates a checkpoint and seeds the system prompt.
   - `POST /observe blunder` invokes the agent (LLM mocked).
   - `POST /observe game_end` requires a saved `TiltReport` and injects it.
   - `POST /message` streams.
   - `POST /close-session` writes a `GameSummary`, idempotent on re-call.
   - `POST /close-session` triggers DNA job at the 5/10/15 boundary, doesn't at 4/6/7.
   - `GET /history/{thread_id}` rejects threads belonging to other users.
- `test_tools.py` — `list_past_games` and `get_game_details` with the `db` fixture; verify auth scoping.

For the agent itself, mock `ChatOpenAI` to return canned responses. We are NOT testing LLM quality here; we are testing that wiring is correct.

For the checkpointer in tests: use `AsyncPostgresSaver` against the same in-memory SQLite engine the existing test setup uses, OR swap to `MemorySaver` only inside tests via a fixture-scoped override of `init_agent`. Decision: use `MemorySaver` in tests — the checkpointer's persistence layer is langgraph's responsibility, not ours, so testing against a different backend than prod is acceptable here.

## Costs (rough)

Per game (1 game = ~40 plies, ~3 blunders, ~6 user messages, 1 close):
- 3 × blunder reaction (gpt-4o-mini): ~3k tokens each = $0.001
- 6 × user chat (gpt-4o): ~4k tokens each = $0.06
- 1 × game_end reaction (gpt-4o): ~5k = $0.025
- 1 × Tilt detector (gpt-4o-mini, unchanged): ~$0.005
- 1 × summarizer (gpt-4o-mini): ~$0.003
- DNA every 5 games: ~$0.01 amortized = $0.002 per game

Total per game: ~$0.10. Acceptable for a "Pro" feature; might want gpt-4o-mini fallback for chat on free tier later.

## Migration plan

1. Add migrations for `game_summaries` and `decision_dna` tables (alembic, or `Base.metadata.create_all` since the project is currently using that).
2. Install `langgraph`, `langgraph-checkpoint-postgres`, ensure `langchain >= 1.2`.
3. Implement backend in this order: event detector → tools → agent service → router → summarizer → DNA job → cron fallback.
4. Implement frontend: `agent-stream.ts` → `AgentChat` → `AgentMessage` → wire into `ChessGame`.
5. Smoke test: play a full game end-to-end locally.
6. Cron job for stale-thread cleanup is deferred to a follow-up; in MVP, `beforeunload` + manual "New game" click are the only triggers.

## Risks / open questions

- **`AsyncPostgresSaver` with Supabase pooler**: the pooler is in session mode (port 5432) — should be fine. If we hit issues we'll fall back to a direct connection just for the checkpointer.
- **Agent silence rule**: "stay silent if last message <5 plies ago" is a prompt instruction, not a hard constraint. The model might still be chatty. We'll measure post-MVP and convert to a programmatic gate if needed.
- **Message ordering on game_end**: the agent reaction should arrive AFTER the Tilt card is visible in the chat. Frontend enforces this by inserting the synthetic Tilt-card message before sending `observe game_end`.
- **Tool `list_past_games` keyword filter** is intentionally crude. If the agent struggles to find what it wants, we'll switch to a richer filter (date ranges, eval thresholds) before adding embeddings/semantic search.
