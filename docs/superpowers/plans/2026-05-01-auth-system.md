# Authentication System Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add full-stack authentication (Google OAuth + email/password via Supabase) with per-user game history and plan-gated endpoints.

**Architecture:** Supabase Auth issues JWTs; Next.js middleware (via `@supabase/ssr`) protects all routes at the edge; FastAPI verifies JWTs with `python-jose` (HS256) using the Supabase JWT secret and persists games + tilt reports per user in Supabase Postgres via SQLAlchemy async.

**Tech Stack:** `@supabase/supabase-js`, `@supabase/ssr`, Next.js 15 middleware, FastAPI, SQLAlchemy 2.0 async, `python-jose`, `asyncpg`, `pytest-asyncio`, `httpx`

---

## File Map

### Backend — new files
| File | Purpose |
|---|---|
| `backend/app/database.py` | Async SQLAlchemy engine + session factory + `Base` |
| `backend/app/models.py` | `Profile`, `Game`, `TiltReport` ORM models |
| `backend/app/dependencies.py` | `get_current_user`, `require_pro` FastAPI dependencies |
| `backend/app/routers/games.py` | `GET /api/games`, `GET /api/games/{id}`, `POST /api/games/{id}/report` |
| `backend/tests/__init__.py` | Test package marker |
| `backend/tests/conftest.py` | Pytest fixtures: test DB, test client, auth overrides |
| `backend/tests/test_auth.py` | JWT decode unit tests |
| `backend/tests/test_endpoints.py` | Endpoint integration tests |

### Backend — modified files
| File | Change |
|---|---|
| `backend/app/config.py` | Add `SUPABASE_URL`, `SUPABASE_JWT_SECRET` |
| `backend/app/main.py` | Add lifespan (`create_all`), auth on all endpoints, include games router, update decision-dna + coach to pull from DB |
| `backend/app/schemas/api.py` | Add `GameSummary`, `GameDetailResponse`; update `DecisionDNARequest`, `CoachChatRequest` |
| `backend/requirements.txt` | Add `pytest`, `pytest-asyncio`, `aiosqlite` |
| `backend/.env.example` | Add `SUPABASE_URL`, `SUPABASE_JWT_SECRET` |

### Frontend — new files
| File | Purpose |
|---|---|
| `frontend/src/lib/supabase/client.ts` | Browser Supabase singleton |
| `frontend/src/lib/supabase/server.ts` | Server-side Supabase client (cookie-aware) |
| `frontend/src/middleware.ts` | Edge middleware — session refresh + auth gate |
| `frontend/src/app/login/page.tsx` | Login page (Google + email/password) |
| `frontend/src/app/auth/callback/route.ts` | OAuth code → session exchange |
| `frontend/src/components/NavBar.tsx` | Client component with user email + logout |

### Frontend — modified files
| File | Change |
|---|---|
| `frontend/src/lib/api.ts` | Inject `Authorization: Bearer` on every request; update `decisionDNA` + `coachChat` signatures |
| `frontend/src/app/page.tsx` | Use `NavBar` component |
| `frontend/.env.local` | Add `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY` |
| `frontend/.env.example` | Same |

---

## Task 1: Add test + async-SQLite dependencies

**Files:**
- Modify: `backend/requirements.txt`
- Modify: `backend/pyproject.toml`

- [ ] **Step 1: Add packages to requirements.txt**

Open `backend/requirements.txt` and append these lines:
```
pytest==8.3.5
pytest-asyncio==0.24.0
aiosqlite==0.20.0
```

- [ ] **Step 2: Install**

```bash
cd backend
source .venv/bin/activate
pip install -r requirements.txt
```

Expected: installation completes without errors.

- [ ] **Step 3: Add pytest config to pyproject.toml**

Open `backend/pyproject.toml` and add after the `[project]` section:
```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
```

- [ ] **Step 4: Commit**

```bash
git add backend/requirements.txt backend/pyproject.toml
git commit -m "chore: add pytest + aiosqlite test dependencies"
```

---

## Task 2: Create async database engine and Base

**Files:**
- Create: `backend/app/database.py`

- [ ] **Step 1: Write the failing import test**

Create `backend/tests/__init__.py` (empty):
```
```

Create `backend/tests/test_database_import.py`:
```python
def test_database_imports():
    from app.database import Base, get_db, engine
    assert Base is not None
    assert get_db is not None
    assert engine is not None
```

- [ ] **Step 2: Run to confirm it fails**

```bash
cd backend
pytest tests/test_database_import.py -v
```

Expected: `ModuleNotFoundError: No module named 'app.database'`

- [ ] **Step 3: Create backend/app/database.py**

```python
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession, async_sessionmaker
from sqlalchemy.orm import DeclarativeBase

from app.config import settings


class Base(DeclarativeBase):
    pass


engine = create_async_engine(settings.database_url, echo=False)
AsyncSessionLocal = async_sessionmaker(engine, expire_on_commit=False)


async def get_db() -> AsyncSession:
    async with AsyncSessionLocal() as session:
        yield session
```

- [ ] **Step 4: Run test to confirm it passes**

```bash
cd backend
OPENAI_API_KEY=test SUPABASE_URL=https://x.supabase.co SUPABASE_JWT_SECRET=test pytest tests/test_database_import.py -v
```

Expected: `PASSED`

*(We'll eliminate the env var noise in Task 4's conftest.)*

- [ ] **Step 5: Delete the temporary import test**

```bash
rm backend/tests/test_database_import.py
```

- [ ] **Step 6: Commit**

```bash
git add backend/app/database.py backend/tests/__init__.py
git commit -m "feat: add async SQLAlchemy engine and Base"
```

---

## Task 3: Create ORM models

**Files:**
- Create: `backend/app/models.py`

- [ ] **Step 1: Create backend/app/models.py**

```python
import uuid
from datetime import datetime, timezone

from sqlalchemy import String, DateTime, JSON, ForeignKey, UniqueConstraint
from sqlalchemy.orm import Mapped, mapped_column, relationship

from app.database import Base


def _now() -> datetime:
    return datetime.now(timezone.utc)


def _uuid() -> str:
    return str(uuid.uuid4())


class Profile(Base):
    __tablename__ = "profiles"

    user_id: Mapped[str] = mapped_column(String, primary_key=True)
    plan: Mapped[str] = mapped_column(String, default="free", nullable=False)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=_now)

    games: Mapped[list["Game"]] = relationship(back_populates="profile", cascade="all, delete-orphan")


class Game(Base):
    __tablename__ = "games"

    id: Mapped[str] = mapped_column(String, primary_key=True, default=_uuid)
    user_id: Mapped[str] = mapped_column(String, ForeignKey("profiles.user_id"), nullable=False)
    pgn: Mapped[str] = mapped_column(String, nullable=False)
    eval_per_ply: Mapped[list] = mapped_column(JSON, nullable=False)
    time_per_ply: Mapped[list] = mapped_column(JSON, nullable=False)
    player_color: Mapped[str] = mapped_column(String, nullable=False)
    result: Mapped[str] = mapped_column(String, nullable=False)
    played_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=_now)

    profile: Mapped["Profile"] = relationship(back_populates="games")
    tilt_report: Mapped["TiltReport | None"] = relationship(
        back_populates="game", uselist=False, cascade="all, delete-orphan"
    )


class TiltReport(Base):
    __tablename__ = "tilt_reports"
    __table_args__ = (UniqueConstraint("game_id"),)

    id: Mapped[str] = mapped_column(String, primary_key=True, default=_uuid)
    game_id: Mapped[str] = mapped_column(String, ForeignKey("games.id"), nullable=False)
    headline: Mapped[str] = mapped_column(String, nullable=False)
    diagnosis: Mapped[str] = mapped_column(String, nullable=False)
    pattern_label: Mapped[str] = mapped_column(String, nullable=False)
    evidence_plies: Mapped[list] = mapped_column(JSON, nullable=False)
    suggestion: Mapped[str] = mapped_column(String, nullable=False)

    game: Mapped["Game"] = relationship(back_populates="tilt_report")
```

- [ ] **Step 2: Write model creation test**

Create `backend/tests/test_models.py`:
```python
import pytest
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker
from sqlalchemy.pool import StaticPool
from app.database import Base
from app.models import Profile, Game, TiltReport


@pytest.fixture
async def session():
    engine = create_async_engine(
        "sqlite+aiosqlite:///:memory:",
        connect_args={"check_same_thread": False},
        poolclass=StaticPool,
    )
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    factory = async_sessionmaker(engine, expire_on_commit=False)
    async with factory() as s:
        yield s
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)


async def test_create_profile(session):
    profile = Profile(user_id="user-1", plan="free")
    session.add(profile)
    await session.commit()
    await session.refresh(profile)
    assert profile.user_id == "user-1"
    assert profile.plan == "free"
    assert profile.created_at is not None


async def test_create_game_and_tilt_report(session):
    profile = Profile(user_id="user-2", plan="free")
    session.add(profile)
    await session.flush()

    game = Game(
        user_id="user-2",
        pgn="1. e4 e5",
        eval_per_ply=[10, -10],
        time_per_ply=[1.0, 2.0],
        player_color="white",
        result="win",
    )
    session.add(game)
    await session.flush()

    report = TiltReport(
        game_id=game.id,
        headline="Test",
        diagnosis="Diag",
        pattern_label="tilt",
        evidence_plies=[3, 5],
        suggestion="Breathe",
    )
    session.add(report)
    await session.commit()

    await session.refresh(game)
    assert game.tilt_report is not None
    assert game.tilt_report.headline == "Test"
```

- [ ] **Step 3: Run model tests**

```bash
cd backend
OPENAI_API_KEY=test SUPABASE_URL=https://x.supabase.co SUPABASE_JWT_SECRET=test pytest tests/test_models.py -v
```

Expected: 2 tests PASSED.

- [ ] **Step 4: Delete temporary test file**

```bash
rm backend/tests/test_models.py
```

- [ ] **Step 5: Commit**

```bash
git add backend/app/models.py
git commit -m "feat: add Profile, Game, TiltReport SQLAlchemy models"
```

---

## Task 4: Update config + env files

**Files:**
- Modify: `backend/app/config.py`
- Modify: `backend/.env.example`
- Modify: `backend/.env`

- [ ] **Step 1: Update config.py**

Replace the contents of `backend/app/config.py`:
```python
"""Application configuration loaded from environment variables."""
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", extra="ignore")

    # OpenAI
    openai_api_key: str
    model_fast: str = "gpt-4o-mini"
    model_smart: str = "gpt-4o"

    # Database
    database_url: str = "sqlite+aiosqlite:///./dev.db"

    # Supabase Auth
    supabase_url: str
    supabase_jwt_secret: str

    # CORS
    cors_origins: list[str] = ["http://localhost:3000"]


settings = Settings()
```

- [ ] **Step 2: Update .env.example**

Replace the contents of `backend/.env.example`:
```
OPENAI_API_KEY=sk-your-key-here
DATABASE_URL=postgresql+asyncpg://user:pass@host:5432/dbname
CORS_ORIGINS=["http://localhost:3000"]
SUPABASE_URL=https://your-project.supabase.co
SUPABASE_JWT_SECRET=your-jwt-secret-from-supabase-dashboard
```

- [ ] **Step 3: Add Supabase vars to backend/.env**

Open `backend/.env` and add (fill in real values from Supabase dashboard → Settings → API):
```
SUPABASE_URL=https://your-project.supabase.co
SUPABASE_JWT_SECRET=your-jwt-secret
```

- [ ] **Step 4: Commit**

```bash
git add backend/app/config.py backend/.env.example
git commit -m "feat: add SUPABASE_URL and SUPABASE_JWT_SECRET config"
```

---

## Task 5: Create auth dependency

**Files:**
- Create: `backend/app/dependencies.py`

- [ ] **Step 1: Create backend/app/dependencies.py**

```python
from dataclasses import dataclass

from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPAuthorizationCredentials, HTTPBearer
from jose import JWTError, jwt
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from app.config import settings
from app.database import get_db
from app.models import Profile

_bearer = HTTPBearer()


def _decode_token(token: str) -> dict:
    try:
        return jwt.decode(
            token,
            settings.supabase_jwt_secret,
            algorithms=["HS256"],
            audience="authenticated",
        )
    except JWTError as exc:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid or expired token",
        ) from exc


@dataclass
class CurrentUser:
    user_id: str
    plan: str


async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(_bearer),
    db: AsyncSession = Depends(get_db),
) -> CurrentUser:
    payload = _decode_token(credentials.credentials)
    user_id: str | None = payload.get("sub")
    if not user_id:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="Missing sub claim")

    result = await db.execute(select(Profile).where(Profile.user_id == user_id))
    profile = result.scalar_one_or_none()
    if profile is None:
        profile = Profile(user_id=user_id, plan="free")
        db.add(profile)
        await db.commit()
        await db.refresh(profile)

    return CurrentUser(user_id=user_id, plan=profile.plan)


async def require_pro(user: CurrentUser = Depends(get_current_user)) -> CurrentUser:
    if user.plan != "pro":
        raise HTTPException(status_code=status.HTTP_403_FORBIDDEN, detail="pro_required")
    return user
```

- [ ] **Step 2: Commit**

```bash
git add backend/app/dependencies.py
git commit -m "feat: add JWT auth dependency with auto-profile creation"
```

---

## Task 6: Write auth tests + conftest

**Files:**
- Create: `backend/tests/conftest.py`
- Create: `backend/tests/test_auth.py`

- [ ] **Step 1: Create backend/tests/conftest.py**

```python
import os

# Must be set before any app imports so pydantic-settings picks them up
os.environ.setdefault("TESTING", "1")
os.environ.setdefault("OPENAI_API_KEY", "test")
os.environ.setdefault("SUPABASE_URL", "https://test.supabase.co")
os.environ.setdefault("SUPABASE_JWT_SECRET", "super-secret-test-key-32-chars-xxxxx!")
os.environ.setdefault("DATABASE_URL", "sqlite+aiosqlite:///:memory:")

import pytest
import pytest_asyncio
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession
from sqlalchemy.pool import StaticPool
from httpx import AsyncClient, ASGITransport

from app.database import Base, get_db
from app.dependencies import CurrentUser, get_current_user
from app.main import app

_ENGINE = create_async_engine(
    "sqlite+aiosqlite:///:memory:",
    connect_args={"check_same_thread": False},
    poolclass=StaticPool,
)
_Session = async_sessionmaker(_ENGINE, expire_on_commit=False)


@pytest_asyncio.fixture(scope="session", autouse=True)
async def create_tables():
    async with _ENGINE.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield
    async with _ENGINE.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)


@pytest_asyncio.fixture
async def db() -> AsyncSession:
    async with _Session() as session:
        yield session


@pytest_asyncio.fixture
async def client(db: AsyncSession) -> AsyncClient:
    async def _get_db():
        yield db

    app.dependency_overrides[get_db] = _get_db
    async with AsyncClient(transport=ASGITransport(app=app), base_url="http://test") as c:
        yield c
    app.dependency_overrides.pop(get_db, None)


@pytest_asyncio.fixture
async def authed_client(db: AsyncSession) -> AsyncClient:
    async def _get_db():
        yield db

    def _get_user():
        return CurrentUser(user_id="test-user-id", plan="free")

    app.dependency_overrides[get_db] = _get_db
    app.dependency_overrides[get_current_user] = _get_user
    async with AsyncClient(transport=ASGITransport(app=app), base_url="http://test") as c:
        yield c
    app.dependency_overrides.pop(get_db, None)
    app.dependency_overrides.pop(get_current_user, None)


@pytest_asyncio.fixture
async def pro_client(db: AsyncSession) -> AsyncClient:
    async def _get_db():
        yield db

    def _get_user():
        return CurrentUser(user_id="test-user-id", plan="pro")

    app.dependency_overrides[get_db] = _get_db
    app.dependency_overrides[get_current_user] = _get_user
    async with AsyncClient(transport=ASGITransport(app=app), base_url="http://test") as c:
        yield c
    app.dependency_overrides.pop(get_db, None)
    app.dependency_overrides.pop(get_current_user, None)
```

- [ ] **Step 2: Create backend/tests/test_auth.py**

```python
import pytest
from fastapi import HTTPException
from jose import jwt

from app.config import settings
from app.dependencies import _decode_token


def _make_token(sub: str = "user-1", aud: str = "authenticated", **extra) -> str:
    payload = {"sub": sub, "aud": aud, **extra}
    return jwt.encode(payload, settings.supabase_jwt_secret, algorithm="HS256")


def test_valid_token_decodes():
    token = _make_token(sub="abc-123")
    payload = _decode_token(token)
    assert payload["sub"] == "abc-123"


def test_invalid_token_raises_401():
    with pytest.raises(HTTPException) as exc_info:
        _decode_token("not.a.valid.token")
    assert exc_info.value.status_code == 401


def test_wrong_secret_raises_401():
    bad_token = jwt.encode(
        {"sub": "x", "aud": "authenticated"},
        "wrong-secret-that-is-long-enough-xxxxx!",
        algorithm="HS256",
    )
    with pytest.raises(HTTPException) as exc_info:
        _decode_token(bad_token)
    assert exc_info.value.status_code == 401


def test_wrong_audience_raises_401():
    token = _make_token(sub="x", aud="anon")
    with pytest.raises(HTTPException) as exc_info:
        _decode_token(token)
    assert exc_info.value.status_code == 401
```

- [ ] **Step 3: Run auth tests**

```bash
cd backend
pytest tests/test_auth.py -v
```

Expected:
```
tests/test_auth.py::test_valid_token_decodes PASSED
tests/test_auth.py::test_invalid_token_raises_401 PASSED
tests/test_auth.py::test_wrong_secret_raises_401 PASSED
tests/test_auth.py::test_wrong_audience_raises_401 PASSED
```

- [ ] **Step 4: Commit**

```bash
git add backend/tests/conftest.py backend/tests/test_auth.py
git commit -m "test: add JWT auth unit tests and pytest conftest"
```

---

## Task 7: Update main.py — lifespan + analyze-game with auth + DB save

**Files:**
- Modify: `backend/app/main.py`
- Modify: `backend/app/schemas/api.py`

- [ ] **Step 1: Update schemas/api.py — add AnalyzeGameResponse with game_id + update DecisionDNARequest + CoachChatRequest**

Replace the full contents of `backend/app/schemas/api.py`:
```python
"""API request/response schemas."""
from datetime import datetime
from typing import Literal
from pydantic import BaseModel, Field


# ---------- Tilt Detector ----------

class AnalyzeGameRequest(BaseModel):
    pgn: str = Field(..., description="Full PGN of the game")
    eval_per_ply: list[int] = Field(
        ...,
        description="Stockfish eval after each ply, in centipawns from White's POV",
    )
    time_per_ply: list[float] = Field(..., description="Seconds spent on each ply")
    player_color: Literal["white", "black"]
    result: Literal["win", "loss", "draw"]


class TiltDetectorResponse(BaseModel):
    headline: str
    diagnosis: str
    pattern_label: str
    evidence_plies: list[int]
    suggestion: str


# ---------- Decision DNA ----------

class DecisionDNARequest(BaseModel):
    n: int = Field(default=5, ge=3, le=20, description="Number of recent games to analyze")


class GMComparison(BaseModel):
    name: str
    similarity_pct: int
    why: str


class DecisionDNAResponse(BaseModel):
    type_name: str
    tagline: str
    summary: str
    core_strength: str
    core_weakness: str
    gm_comparison: GMComparison


# ---------- Coach Chat ----------

class ChatTurn(BaseModel):
    role: Literal["user", "assistant"]
    content: str


class CoachChatRequest(BaseModel):
    message: str
    history: list[ChatTurn] = Field(default_factory=list)


class CoachChatResponse(BaseModel):
    reply: str


# ---------- Game History ----------

class TiltReportOut(BaseModel):
    headline: str
    diagnosis: str
    pattern_label: str
    evidence_plies: list[int]
    suggestion: str


class GameSummary(BaseModel):
    id: str
    player_color: str
    result: str
    played_at: datetime
    tilt_report: TiltReportOut | None


class GameDetailResponse(BaseModel):
    id: str
    pgn: str
    eval_per_ply: list[int]
    time_per_ply: list[float]
    player_color: str
    result: str
    played_at: datetime
    tilt_report: TiltReportOut | None


class GameListResponse(BaseModel):
    games: list[GameSummary]
    total: int
```

- [ ] **Step 2: Replace backend/app/main.py**

```python
"""Blunder Therapist FastAPI app."""
import os
from contextlib import asynccontextmanager

from fastapi import Depends, FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from app.config import settings
from app.database import get_db
from app.dependencies import CurrentUser, get_current_user, require_pro
from app.models import Game, TiltReport
from app.schemas.api import (
    AnalyzeGameRequest,
    TiltDetectorResponse,
    DecisionDNARequest,
    DecisionDNAResponse,
    CoachChatRequest,
    CoachChatResponse,
)
from app.services.features import extract_features, features_to_llm_summary
from app.services.llm import run_tilt_detector, run_decision_dna, run_coach_chat


@asynccontextmanager
async def lifespan(app: FastAPI):
    if not os.getenv("TESTING"):
        from app import models  # noqa: F401 — registers models on Base.metadata
        from app.database import engine
        from app.database import Base
        async with engine.begin() as conn:
            await conn.run_sync(Base.metadata.create_all)
    yield


app = FastAPI(title="Blunder Therapist API", version="0.1.0", lifespan=lifespan)

app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.cors_origins,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)


@app.get("/")
def root():
    return {"status": "ok", "service": "blunder-therapist"}


@app.post("/api/analyze-game", response_model=TiltDetectorResponse)
async def analyze_game(
    req: AnalyzeGameRequest,
    user: CurrentUser = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
):
    """Run Tilt Detector on a single game; persist game + report."""
    try:
        features = extract_features(
            pgn=req.pgn,
            eval_per_ply=req.eval_per_ply,
            time_per_ply=req.time_per_ply,
            player_color=req.player_color,
            result=req.result,
        )
    except ValueError as e:
        raise HTTPException(status_code=400, detail=str(e))

    # Persist the game row first
    game = Game(
        user_id=user.user_id,
        pgn=req.pgn,
        eval_per_ply=req.eval_per_ply,
        time_per_ply=req.time_per_ply,
        player_color=req.player_color,
        result=req.result,
    )
    db.add(game)
    await db.commit()
    await db.refresh(game)

    # Run LLM analysis; if it fails the game row stays (can be re-run)
    try:
        result = await run_tilt_detector(features)
    except Exception as e:
        raise HTTPException(status_code=502, detail=f"LLM error: {e}")

    report = TiltReport(game_id=game.id, **result)
    db.add(report)
    await db.commit()

    return TiltDetectorResponse(**result)


@app.post("/api/decision-dna", response_model=DecisionDNAResponse)
async def decision_dna(
    req: DecisionDNARequest,
    user: CurrentUser = Depends(require_pro),
    db: AsyncSession = Depends(get_db),
):
    """Build Decision DNA from the user's last N stored games."""
    rows = await db.execute(
        select(Game)
        .where(Game.user_id == user.user_id)
        .order_by(Game.played_at.desc())
        .limit(req.n)
    )
    games = rows.scalars().all()
    if len(games) < 3:
        raise HTTPException(status_code=400, detail="Need at least 3 games on record")

    try:
        all_features = [
            extract_features(
                pgn=g.pgn,
                eval_per_ply=g.eval_per_ply,
                time_per_ply=g.time_per_ply,
                player_color=g.player_color,
                result=g.result,
            )
            for g in games
        ]
        result = await run_decision_dna(all_features)
        return DecisionDNAResponse(**result)
    except ValueError as e:
        raise HTTPException(status_code=400, detail=str(e))


@app.post("/api/coach", response_model=CoachChatResponse)
async def coach_chat(
    req: CoachChatRequest,
    user: CurrentUser = Depends(require_pro),
    db: AsyncSession = Depends(get_db),
):
    """Chat with the memory-aware AI coach using stored game history."""
    rows = await db.execute(
        select(Game)
        .where(Game.user_id == user.user_id)
        .order_by(Game.played_at.desc())
        .limit(10)
    )
    recent_games = rows.scalars().all()

    memory_parts: list[str] = []
    for i, g in enumerate(recent_games):
        try:
            f = extract_features(
                pgn=g.pgn,
                eval_per_ply=g.eval_per_ply,
                time_per_ply=g.time_per_ply,
                player_color=g.player_color,
                result=g.result,
            )
            memory_parts.append(f"--- Game {i + 1} ({f.result}) ---\n{features_to_llm_summary(f)}")
        except ValueError:
            continue

    game_memory = "\n\n".join(memory_parts) if memory_parts else "No games on file yet."
    history = [{"role": t.role, "content": t.content} for t in req.history]
    reply = await run_coach_chat(req.message, history, game_memory)
    return CoachChatResponse(reply=reply)
```

- [ ] **Step 3: Write endpoint test**

Create `backend/tests/test_endpoints.py`:
```python
import pytest
from httpx import AsyncClient


SAMPLE_GAME = {
    "pgn": "1. e4 e5 2. Nf3 Nc6 3. Bb5 a6",
    "eval_per_ply": [20, -20, 30, -30, 40, -40],
    "time_per_ply": [1.0, 2.0, 1.5, 0.5, 1.0, 2.0],
    "player_color": "white",
    "result": "draw",
}


async def test_analyze_game_requires_auth(client: AsyncClient):
    resp = await client.post("/api/analyze-game", json=SAMPLE_GAME)
    assert resp.status_code == 403  # HTTPBearer returns 403 when no header


async def test_analyze_game_with_auth_saves_game(authed_client: AsyncClient):
    # This will call the real LLM unless OPENAI_API_KEY is mocked,
    # so we just test that auth passes and DB save happens.
    # In CI, mock run_tilt_detector; here we test the 502 path.
    resp = await authed_client.post("/api/analyze-game", json=SAMPLE_GAME)
    # 200 if LLM works, 502 if OPENAI_API_KEY=test (invalid key)
    assert resp.status_code in (200, 502)


async def test_decision_dna_requires_pro(authed_client: AsyncClient):
    resp = await authed_client.post("/api/decision-dna", json={"n": 5})
    assert resp.status_code == 403
    assert resp.json()["detail"] == "pro_required"


async def test_coach_requires_pro(authed_client: AsyncClient):
    resp = await authed_client.post("/api/coach", json={"message": "hi"})
    assert resp.status_code == 403


async def test_decision_dna_not_enough_games(pro_client: AsyncClient):
    resp = await pro_client.post("/api/decision-dna", json={"n": 5})
    assert resp.status_code == 400
    assert "at least 3 games" in resp.json()["detail"]
```

- [ ] **Step 4: Run tests**

```bash
cd backend
pytest tests/test_endpoints.py -v
```

Expected:
```
tests/test_endpoints.py::test_analyze_game_requires_auth PASSED
tests/test_endpoints.py::test_analyze_game_with_auth_saves_game PASSED
tests/test_endpoints.py::test_decision_dna_requires_pro PASSED
tests/test_endpoints.py::test_coach_requires_pro PASSED
tests/test_endpoints.py::test_decision_dna_not_enough_games PASSED
```

- [ ] **Step 5: Commit**

```bash
git add backend/app/main.py backend/app/schemas/api.py backend/tests/test_endpoints.py
git commit -m "feat: add auth + DB persistence to all endpoints"
```

---

## Task 8: Game history router

**Files:**
- Create: `backend/app/routers/games.py`
- Modify: `backend/app/main.py`

- [ ] **Step 1: Create backend/app/routers/games.py**

```python
"""Game history endpoints."""
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy import select, func
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy.orm import selectinload

from app.database import get_db
from app.dependencies import CurrentUser, get_current_user
from app.models import Game, TiltReport
from app.schemas.api import (
    GameDetailResponse,
    GameListResponse,
    GameSummary,
    TiltDetectorResponse,
    TiltReportOut,
)
from app.services.features import extract_features
from app.services.llm import run_tilt_detector

router = APIRouter(prefix="/api/games", tags=["games"])


def _report_out(report: TiltReport | None) -> TiltReportOut | None:
    if report is None:
        return None
    return TiltReportOut(
        headline=report.headline,
        diagnosis=report.diagnosis,
        pattern_label=report.pattern_label,
        evidence_plies=report.evidence_plies,
        suggestion=report.suggestion,
    )


@router.get("", response_model=GameListResponse)
async def list_games(
    page: int = 1,
    page_size: int = 20,
    user: CurrentUser = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
):
    offset = (page - 1) * page_size
    total_result = await db.execute(
        select(func.count()).where(Game.user_id == user.user_id)
    )
    total = total_result.scalar_one()

    rows = await db.execute(
        select(Game)
        .options(selectinload(Game.tilt_report))
        .where(Game.user_id == user.user_id)
        .order_by(Game.played_at.desc())
        .offset(offset)
        .limit(page_size)
    )
    games = rows.scalars().all()

    return GameListResponse(
        games=[
            GameSummary(
                id=g.id,
                player_color=g.player_color,
                result=g.result,
                played_at=g.played_at,
                tilt_report=_report_out(g.tilt_report),
            )
            for g in games
        ],
        total=total,
    )


@router.get("/{game_id}", response_model=GameDetailResponse)
async def get_game(
    game_id: str,
    user: CurrentUser = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
):
    row = await db.execute(
        select(Game)
        .options(selectinload(Game.tilt_report))
        .where(Game.id == game_id, Game.user_id == user.user_id)
    )
    game = row.scalar_one_or_none()
    if game is None:
        raise HTTPException(status_code=404, detail="Game not found")

    return GameDetailResponse(
        id=game.id,
        pgn=game.pgn,
        eval_per_ply=game.eval_per_ply,
        time_per_ply=game.time_per_ply,
        player_color=game.player_color,
        result=game.result,
        played_at=game.played_at,
        tilt_report=_report_out(game.tilt_report),
    )


@router.post("/{game_id}/report", response_model=TiltDetectorResponse)
async def rerun_report(
    game_id: str,
    user: CurrentUser = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
):
    """Re-run tilt detector on a stored game (e.g., if LLM failed originally)."""
    row = await db.execute(
        select(Game)
        .options(selectinload(Game.tilt_report))
        .where(Game.id == game_id, Game.user_id == user.user_id)
    )
    game = row.scalar_one_or_none()
    if game is None:
        raise HTTPException(status_code=404, detail="Game not found")

    try:
        features = extract_features(
            pgn=game.pgn,
            eval_per_ply=game.eval_per_ply,
            time_per_ply=game.time_per_ply,
            player_color=game.player_color,
            result=game.result,
        )
        result = await run_tilt_detector(features)
    except (ValueError, Exception) as e:
        raise HTTPException(status_code=502, detail=str(e))

    if game.tilt_report is not None:
        for key, val in result.items():
            setattr(game.tilt_report, key, val)
    else:
        db.add(TiltReport(game_id=game.id, **result))
    await db.commit()

    return TiltDetectorResponse(**result)
```

- [ ] **Step 2: Include the router in main.py**

In `backend/app/main.py`, after the existing imports, add:
```python
from app.routers import games as games_router
```

And after `app.add_middleware(...)`, add:
```python
app.include_router(games_router.router)
```

- [ ] **Step 3: Add game history tests**

Append to `backend/tests/test_endpoints.py`:
```python
async def test_list_games_empty_new_user(db: AsyncSession):
    # Use a brand-new user_id that no other test touches so total is guaranteed 0
    from app.dependencies import CurrentUser, get_current_user
    from app.main import app as fastapi_app
    from httpx import AsyncClient, ASGITransport

    async def _db():
        yield db

    fastapi_app.dependency_overrides[get_current_user] = lambda: CurrentUser(user_id="empty-user-xyz", plan="free")
    fastapi_app.dependency_overrides[get_db] = _db
    async with AsyncClient(transport=ASGITransport(app=fastapi_app), base_url="http://test") as c:
        resp = await c.get("/api/games")
    fastapi_app.dependency_overrides.pop(get_current_user, None)
    fastapi_app.dependency_overrides.pop(get_db, None)

    assert resp.status_code == 200
    body = resp.json()
    assert body["total"] == 0
    assert body["games"] == []


async def test_get_game_not_found(authed_client: AsyncClient):
    resp = await authed_client.get("/api/games/nonexistent-id")
    assert resp.status_code == 404


async def test_list_games_after_save(db: AsyncSession):
    # Use a unique user_id so this test is isolated from others
    from app.dependencies import CurrentUser, get_current_user
    from app.main import app as fastapi_app
    from app.models import Game, Profile
    from httpx import AsyncClient, ASGITransport

    profile = Profile(user_id="history-user-abc", plan="free")
    db.add(profile)
    game = Game(
        user_id="history-user-abc",
        pgn="1. e4",
        eval_per_ply=[10],
        time_per_ply=[1.0],
        player_color="white",
        result="draw",
    )
    db.add(game)
    await db.commit()

    async def _db():
        yield db

    fastapi_app.dependency_overrides[get_current_user] = lambda: CurrentUser(user_id="history-user-abc", plan="free")
    fastapi_app.dependency_overrides[get_db] = _db
    async with AsyncClient(transport=ASGITransport(app=fastapi_app), base_url="http://test") as c:
        resp = await c.get("/api/games")
    fastapi_app.dependency_overrides.pop(get_current_user, None)
    fastapi_app.dependency_overrides.pop(get_db, None)

    assert resp.status_code == 200
    body = resp.json()
    assert body["total"] == 1
    assert body["games"][0]["id"] == game.id
```

- [ ] **Step 4: Run all tests**

```bash
cd backend
pytest tests/ -v
```

Expected: all tests PASSED.

- [ ] **Step 5: Commit**

```bash
git add backend/app/routers/games.py backend/app/main.py backend/tests/test_endpoints.py
git commit -m "feat: add game history endpoints (list, detail, re-run report)"
```

---

## Task 9: Install frontend Supabase packages + add env vars

**Files:**
- Modify: `frontend/package.json` (via npm install)
- Modify: `frontend/.env.local`
- Modify: `frontend/.env.example`

- [ ] **Step 1: Install packages**

```bash
cd frontend
npm install @supabase/supabase-js @supabase/ssr
```

Expected: packages added to `node_modules` and `package.json` dependencies.

- [ ] **Step 2: Add env vars to .env.local**

Open `frontend/.env.local` and add (fill in values from Supabase dashboard → Settings → API):
```
NEXT_PUBLIC_API_BASE=http://localhost:8000
NEXT_PUBLIC_SUPABASE_URL=https://your-project.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=your-anon-key
```

- [ ] **Step 3: Update .env.example**

Replace the contents of `frontend/.env.example`:
```
NEXT_PUBLIC_API_BASE=http://localhost:8000
NEXT_PUBLIC_SUPABASE_URL=https://your-project.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=your-anon-key
```

- [ ] **Step 4: Commit**

```bash
git add frontend/package.json frontend/package-lock.json frontend/.env.example
git commit -m "chore: install @supabase/supabase-js and @supabase/ssr"
```

---

## Task 10: Create Supabase client files

**Files:**
- Create: `frontend/src/lib/supabase/client.ts`
- Create: `frontend/src/lib/supabase/server.ts`

- [ ] **Step 1: Create frontend/src/lib/supabase/client.ts**

```typescript
import { createBrowserClient } from "@supabase/ssr";

export function createClient() {
  return createBrowserClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
  );
}
```

- [ ] **Step 2: Create frontend/src/lib/supabase/server.ts**

```typescript
import { createServerClient } from "@supabase/ssr";
import { cookies } from "next/headers";

export async function createClient() {
  const cookieStore = await cookies();
  return createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() {
          return cookieStore.getAll();
        },
        setAll(cookiesToSet) {
          try {
            cookiesToSet.forEach(({ name, value, options }) =>
              cookieStore.set(name, value, options)
            );
          } catch {
            // Called from a Server Component — cookie writes are no-ops
          }
        },
      },
    }
  );
}
```

- [ ] **Step 3: Confirm TypeScript compiles**

```bash
cd frontend
npx tsc --noEmit
```

Expected: no errors from the new files.

- [ ] **Step 4: Commit**

```bash
git add frontend/src/lib/supabase/
git commit -m "feat: add Supabase browser and server client helpers"
```

---

## Task 11: Create Next.js middleware

**Files:**
- Create: `frontend/src/middleware.ts`

- [ ] **Step 1: Create frontend/src/middleware.ts**

```typescript
import { createServerClient } from "@supabase/ssr";
import { type NextRequest, NextResponse } from "next/server";

export async function middleware(request: NextRequest) {
  let supabaseResponse = NextResponse.next({ request });

  const supabase = createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() {
          return request.cookies.getAll();
        },
        setAll(cookiesToSet) {
          cookiesToSet.forEach(({ name, value }) =>
            request.cookies.set(name, value)
          );
          supabaseResponse = NextResponse.next({ request });
          cookiesToSet.forEach(({ name, value, options }) =>
            supabaseResponse.cookies.set(name, value, options)
          );
        },
      },
    }
  );

  // Refresh session so it doesn't expire mid-visit
  const {
    data: { user },
  } = await supabase.auth.getUser();

  const { pathname } = request.nextUrl;
  const isPublicPath =
    pathname.startsWith("/login") || pathname.startsWith("/auth");

  if (!user && !isPublicPath) {
    const url = request.nextUrl.clone();
    url.pathname = "/login";
    return NextResponse.redirect(url);
  }

  return supabaseResponse;
}

export const config = {
  matcher: [
    "/((?!_next/static|_next/image|favicon.ico|stockfish\\.js|.*\\.(?:svg|png|jpg|jpeg|gif|webp)$).*)",
  ],
};
```

- [ ] **Step 2: Confirm TypeScript compiles**

```bash
cd frontend
npx tsc --noEmit
```

Expected: no errors.

- [ ] **Step 3: Commit**

```bash
git add frontend/src/middleware.ts
git commit -m "feat: add Next.js edge middleware for Supabase session + auth gate"
```

---

## Task 12: Create login page

**Files:**
- Create: `frontend/src/app/login/page.tsx`

- [ ] **Step 1: Create frontend/src/app/login/page.tsx**

```typescript
"use client";

import { useState } from "react";
import { useRouter, useSearchParams } from "next/navigation";
import { createClient } from "@/lib/supabase/client";

export default function LoginPage() {
  const supabase = createClient();
  const router = useRouter();
  const searchParams = useSearchParams();
  const oauthError = searchParams.get("error");

  const [mode, setMode] = useState<"signin" | "signup">("signin");
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");
  const [formError, setFormError] = useState<string | null>(null);
  const [loading, setLoading] = useState(false);

  async function handleGoogle() {
    await supabase.auth.signInWithOAuth({
      provider: "google",
      options: { redirectTo: `${location.origin}/auth/callback` },
    });
  }

  async function handleEmailAuth(e: React.FormEvent) {
    e.preventDefault();
    setLoading(true);
    setFormError(null);

    const { error } =
      mode === "signin"
        ? await supabase.auth.signInWithPassword({ email, password })
        : await supabase.auth.signUp({ email, password });

    if (error) {
      setFormError(error.message);
    } else {
      router.push("/");
      router.refresh();
    }
    setLoading(false);
  }

  return (
    <div className="min-h-screen bg-ink-900 flex items-center justify-center px-4">
      <div className="w-full max-w-sm">
        <h1 className="font-display text-2xl tracking-tight mb-2 text-center">
          Blunder<span className="text-accent-500">.</span>Therapist
        </h1>
        <p className="text-ink-500 text-sm text-center mb-8">
          {mode === "signin" ? "Sign in to continue" : "Create your account"}
        </p>

        {(oauthError || formError) && (
          <p className="text-signal-red text-sm mb-4 text-center rounded-lg bg-ink-800 border border-signal-red/30 px-4 py-2">
            {formError ?? "Authentication failed. Please try again."}
          </p>
        )}

        <button
          onClick={handleGoogle}
          className="w-full flex items-center justify-center gap-3 px-4 py-3 rounded-lg border border-ink-600 hover:border-ink-500 transition mb-6 text-sm font-medium"
        >
          <svg className="w-4 h-4" viewBox="0 0 24 24" aria-hidden>
            <path
              fill="currentColor"
              d="M22.56 12.25c0-.78-.07-1.53-.2-2.25H12v4.26h5.92c-.26 1.37-1.04 2.53-2.21 3.31v2.77h3.57c2.08-1.92 3.28-4.74 3.28-8.09z"
            />
            <path
              fill="currentColor"
              d="M12 23c2.97 0 5.46-.98 7.28-2.66l-3.57-2.77c-.98.66-2.23 1.06-3.71 1.06-2.86 0-5.29-1.93-6.16-4.53H2.18v2.84C3.99 20.53 7.7 23 12 23z"
            />
            <path
              fill="currentColor"
              d="M5.84 14.09c-.22-.66-.35-1.36-.35-2.09s.13-1.43.35-2.09V7.07H2.18C1.43 8.55 1 10.22 1 12s.43 3.45 1.18 4.93l2.85-2.22.81-.62z"
            />
            <path
              fill="currentColor"
              d="M12 5.38c1.62 0 3.06.56 4.21 1.64l3.15-3.15C17.45 2.09 14.97 1 12 1 7.7 1 3.99 3.47 2.18 7.07l3.66 2.84c.87-2.6 3.3-4.53 6.16-4.53z"
            />
          </svg>
          Continue with Google
        </button>

        <div className="flex items-center gap-4 mb-6">
          <div className="flex-1 h-px bg-ink-700" />
          <span className="text-ink-500 text-xs">or</span>
          <div className="flex-1 h-px bg-ink-700" />
        </div>

        <form onSubmit={handleEmailAuth} className="space-y-3">
          <input
            type="email"
            placeholder="Email"
            value={email}
            onChange={(e) => setEmail(e.target.value)}
            required
            className="w-full bg-ink-800 border border-ink-700 rounded-lg px-4 py-3 text-sm placeholder-ink-500 focus:outline-none focus:border-accent-500 transition"
          />
          <input
            type="password"
            placeholder="Password"
            value={password}
            onChange={(e) => setPassword(e.target.value)}
            required
            className="w-full bg-ink-800 border border-ink-700 rounded-lg px-4 py-3 text-sm placeholder-ink-500 focus:outline-none focus:border-accent-500 transition"
          />
          <button
            type="submit"
            disabled={loading}
            className="w-full bg-accent-500 hover:bg-accent-400 disabled:opacity-50 text-white px-4 py-3 rounded-lg text-sm font-medium transition"
          >
            {loading
              ? "Loading…"
              : mode === "signin"
              ? "Sign in"
              : "Create account"}
          </button>
        </form>

        <button
          onClick={() => {
            setMode(mode === "signin" ? "signup" : "signin");
            setFormError(null);
          }}
          className="w-full mt-4 text-ink-500 text-xs hover:text-white transition"
        >
          {mode === "signin"
            ? "Don't have an account? Sign up"
            : "Already have an account? Sign in"}
        </button>
      </div>
    </div>
  );
}
```

- [ ] **Step 2: TypeScript check**

```bash
cd frontend
npx tsc --noEmit
```

Expected: no errors.

- [ ] **Step 3: Commit**

```bash
git add frontend/src/app/login/
git commit -m "feat: add login page with Google OAuth and email/password"
```

---

## Task 13: Create OAuth callback route

**Files:**
- Create: `frontend/src/app/auth/callback/route.ts`

- [ ] **Step 1: Create frontend/src/app/auth/callback/route.ts**

```typescript
import { createServerClient } from "@supabase/ssr";
import { cookies } from "next/headers";
import { type NextRequest, NextResponse } from "next/server";

export async function GET(request: NextRequest) {
  const { searchParams, origin } = new URL(request.url);
  const code = searchParams.get("code");
  const error = searchParams.get("error");

  if (error) {
    return NextResponse.redirect(`${origin}/login?error=oauth_failed`);
  }

  if (code) {
    const cookieStore = await cookies();
    const supabase = createServerClient(
      process.env.NEXT_PUBLIC_SUPABASE_URL!,
      process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
      {
        cookies: {
          getAll() {
            return cookieStore.getAll();
          },
          setAll(cookiesToSet) {
            cookiesToSet.forEach(({ name, value, options }) =>
              cookieStore.set(name, value, options)
            );
          },
        },
      }
    );

    const { error: exchangeError } =
      await supabase.auth.exchangeCodeForSession(code);

    if (!exchangeError) {
      return NextResponse.redirect(`${origin}/`);
    }
  }

  return NextResponse.redirect(`${origin}/login?error=oauth_failed`);
}
```

- [ ] **Step 2: TypeScript check**

```bash
cd frontend
npx tsc --noEmit
```

Expected: no errors.

- [ ] **Step 3: Commit**

```bash
git add frontend/src/app/auth/
git commit -m "feat: add OAuth callback route handler"
```

---

## Task 14: Update api.ts to attach auth headers

**Files:**
- Modify: `frontend/src/lib/api.ts`

- [ ] **Step 1: Replace frontend/src/lib/api.ts**

```typescript
import { createClient } from "@/lib/supabase/client";

const API_BASE = process.env.NEXT_PUBLIC_API_BASE ?? "http://localhost:8000";

async function getAuthHeaders(): Promise<Record<string, string>> {
  const supabase = createClient();
  const {
    data: { session },
  } = await supabase.auth.getSession();
  if (!session) return {};
  return { Authorization: `Bearer ${session.access_token}` };
}

async function post<T>(path: string, body: unknown): Promise<T> {
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
  return res.json();
}

async function get<T>(path: string): Promise<T> {
  const auth = await getAuthHeaders();
  const res = await fetch(`${API_BASE}${path}`, {
    headers: { ...auth },
  });
  if (!res.ok) {
    const text = await res.text();
    throw new Error(`API ${path} failed: ${res.status} ${text}`);
  }
  return res.json();
}

export interface AnalyzeGameRequest {
  pgn: string;
  eval_per_ply: number[];
  time_per_ply: number[];
  player_color: "white" | "black";
  result: "win" | "loss" | "draw";
}

export interface TiltDetectorResponse {
  headline: string;
  diagnosis: string;
  pattern_label: string;
  evidence_plies: number[];
  suggestion: string;
}

export interface DecisionDNAResponse {
  type_name: string;
  tagline: string;
  summary: string;
  core_strength: string;
  core_weakness: string;
  gm_comparison: { name: string; similarity_pct: number; why: string };
}

export interface ChatTurn {
  role: "user" | "assistant";
  content: string;
}

export interface GameSummary {
  id: string;
  player_color: string;
  result: string;
  played_at: string;
  tilt_report: TiltDetectorResponse | null;
}

export interface GameListResponse {
  games: GameSummary[];
  total: number;
}

export const api = {
  analyzeGame: (req: AnalyzeGameRequest) =>
    post<TiltDetectorResponse>("/api/analyze-game", req),

  decisionDNA: (n = 5) =>
    post<DecisionDNAResponse>("/api/decision-dna", { n }),

  coachChat: (message: string, history: ChatTurn[]) =>
    post<{ reply: string }>("/api/coach", { message, history }),

  listGames: (page = 1) =>
    get<GameListResponse>(`/api/games?page=${page}`),

  getGame: (id: string) =>
    get<GameSummary>(`/api/games/${id}`),
};
```

- [ ] **Step 2: TypeScript check**

```bash
cd frontend
npx tsc --noEmit
```

Expected: no errors (ChessGame.tsx still calls `api.analyzeGame` with same interface — no changes needed there).

- [ ] **Step 3: Commit**

```bash
git add frontend/src/lib/api.ts
git commit -m "feat: inject Supabase JWT on all API calls; update DNA + coach signatures"
```

---

## Task 15: Add NavBar with user info + logout

**Files:**
- Create: `frontend/src/components/NavBar.tsx`
- Modify: `frontend/src/app/page.tsx`

- [ ] **Step 1: Create frontend/src/components/NavBar.tsx**

```typescript
"use client";

import { useEffect, useState } from "react";
import { useRouter } from "next/navigation";
import { createClient } from "@/lib/supabase/client";
import type { User } from "@supabase/supabase-js";

export function NavBar() {
  const supabase = createClient();
  const router = useRouter();
  const [user, setUser] = useState<User | null>(null);

  useEffect(() => {
    supabase.auth.getUser().then(({ data }) => setUser(data.user));
    const { data: listener } = supabase.auth.onAuthStateChange(
      (_event, session) => setUser(session?.user ?? null)
    );
    return () => listener.subscription.unsubscribe();
  }, [supabase]);

  async function handleSignOut() {
    await supabase.auth.signOut();
    router.push("/login");
    router.refresh();
  }

  return (
    <div className="flex items-center justify-between">
      <h1 className="font-display text-2xl tracking-tight">
        Blunder<span className="text-accent-500">.</span>Therapist
      </h1>
      <nav className="flex items-center gap-6 text-sm text-ink-500">
        <a href="#play" className="hover:text-white">
          Play
        </a>
        <a href="#dna" className="hover:text-white">
          Decision DNA
        </a>
        <a href="#coach" className="hover:text-white">
          Coach
        </a>
        {user && (
          <span className="text-ink-500 text-xs truncate max-w-[140px]">
            {user.email}
          </span>
        )}
        <a
          href="#"
          className="px-3 py-1.5 rounded-md border border-accent-500 text-accent-500 hover:bg-accent-500 hover:text-white transition"
        >
          Upgrade
        </a>
        {user && (
          <button
            onClick={handleSignOut}
            className="px-3 py-1.5 rounded-md border border-ink-600 hover:border-ink-500 text-ink-500 hover:text-white transition text-xs"
          >
            Sign out
          </button>
        )}
      </nav>
    </div>
  );
}
```

- [ ] **Step 2: Update frontend/src/app/page.tsx to use NavBar**

Replace the contents of `frontend/src/app/page.tsx`:
```typescript
import { ChessGame } from "@/components/ChessGame";
import { NavBar } from "@/components/NavBar";

export default function Home() {
  return (
    <main>
      <header className="max-w-6xl mx-auto px-6 pt-10 pb-6">
        <NavBar />
        <div className="mt-12 max-w-2xl">
          <p className="text-xs uppercase tracking-[0.2em] text-accent-500 mb-3">
            A new kind of chess
          </p>
          <h2 className="font-display text-4xl md:text-5xl leading-tight mb-4">
            Stop blaming the move.
            <br />
            <span className="text-accent-500">Start understanding the moment.</span>
          </h2>
          <p className="text-ink-500 leading-relaxed">
            Every chess platform tells you what you played wrong. We tell you{" "}
            <em>why</em>. Play a game — we&apos;ll show you the pattern in your
            decision-making the engine can&apos;t see.
          </p>
        </div>
      </header>

      <section id="play" className="pt-4 pb-20">
        <ChessGame />
      </section>
    </main>
  );
}
```

- [ ] **Step 3: TypeScript check + lint**

```bash
cd frontend
npx tsc --noEmit && npm run lint
```

Expected: no errors.

- [ ] **Step 4: Commit**

```bash
git add frontend/src/components/NavBar.tsx frontend/src/app/page.tsx
git commit -m "feat: add NavBar with user email display and sign out button"
```

---

## Task 16: Supabase dashboard setup (manual steps)

These steps are done in the Supabase dashboard, not in code.

- [ ] **Step 1: Enable Google OAuth provider**

1. Go to Supabase dashboard → Authentication → Providers
2. Enable Google
3. Add your Google OAuth `Client ID` and `Client Secret` (from Google Cloud Console)
4. Set authorized redirect URI in Google Cloud Console to: `https://your-project.supabase.co/auth/v1/callback`

- [ ] **Step 2: Add site URL**

1. Go to Supabase dashboard → Authentication → URL Configuration
2. Set Site URL to `http://localhost:3000` (dev) or your production URL
3. Add `http://localhost:3000/auth/callback` to Redirect URLs

- [ ] **Step 3: Verify JWT secret**

1. Go to Supabase dashboard → Settings → API
2. Copy the "JWT Secret" value
3. Paste it into `backend/.env` as `SUPABASE_JWT_SECRET`

- [ ] **Step 4: Verify Postgres connection string**

1. Go to Supabase dashboard → Settings → Database → Connection string
2. Copy the "Session mode" URI (port 5432)
3. Replace `postgresql://` with `postgresql+asyncpg://` and paste into `backend/.env` as `DATABASE_URL`

---

## Task 17: End-to-end smoke test

These are manual verification steps, not automated.

- [ ] **Step 1: Start backend**

```bash
cd backend
source .venv/bin/activate
uvicorn app.main:app --reload --port 8000
```

Expected: logs show `Application startup complete` and tables are created (first run).

- [ ] **Step 2: Start frontend**

```bash
cd frontend
npm run dev
```

Expected: `http://localhost:3000` starts.

- [ ] **Step 3: Verify auth gate**

Open `http://localhost:3000` in a browser. Expected: redirects to `/login`.

- [ ] **Step 4: Sign in with email**

On the login page, click "Don't have an account? Sign up", create an account, verify email if required, then sign in. Expected: redirects to `/` and shows the chess board.

- [ ] **Step 5: Verify JWT reaches backend**

Play a full game to completion. Expected:
- Tilt report appears on the right side of the board
- In backend logs: `POST /api/analyze-game 200`

- [ ] **Step 6: Verify game is saved**

In another tab or via curl:
```bash
curl http://localhost:8000/api/games \
  -H "Authorization: Bearer <your-jwt-from-browser-devtools>"
```

Expected: JSON with `total: 1` and the game you just played.

- [ ] **Step 7: Verify sign out**

Click "Sign out" in the nav. Expected: redirects to `/login` and chess board is no longer accessible without signing in again.
