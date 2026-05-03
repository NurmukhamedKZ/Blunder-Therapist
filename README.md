# Blunder Therapist 🧠♟️

> **The chess platform that solves your psychology, not just your openings.**

Blunder Therapist is a revolutionary chess platform that treats your games as a **behavioral dataset**. Instead of just telling you that you missed a tactic, we tell you *why* you missed it by analyzing your emotional state, decision-making patterns, and psychological triggers.

---

## 🚀 The Killer Feature: Deep Behavioral Memory
Most chess AIs are stateless—they forget you the moment the game ends. **Blunder Therapist deeply studies you.** 

Our AI identifies patterns across your entire history (e.g., *"You tend to rush and blunder whenever you have a material advantage over +3.0"*). It tracks your "Decision DNA" and uses long-term memory to provide coaching that is actually personalized to your psychological hurdles, helping you break through plateaus that skill alone can't fix.

---

## 💎 Core AI Features

### 1. 🔥 Tilt Detector (Real-time & Post-game)
We analyze move-by-move timing and evaluation deltas to detect emotional patterns in real-time. 
- **Metrics:** Thinking time volatility, "panic" speed ratios, and recovery performance.
- **Diagnosis:** Get a human-readable psychological report after every game (e.g., "Analysis Paralysis" vs "Revenge Tilting").

### 2. 🧬 Decision DNA
After every 5 games, we recompute your **Decision DNA**—a comprehensive personality profile.
- **Player Archetypes:** Are you a "Panic Calculator" or a "Solid Defender"?
- **GM Comparison:** See which Grandmaster's psychological profile matches yours (e.g., 82% similarity to Mikhail Tal).
- **Growth Tracking:** Watch your Decision DNA evolve as you master your emotions.

### 3. 💬 Persistent AI Coach (LangGraph-powered)
An in-game and dashboard companion that remembers everything.
- **Stateful Memory:** Powered by LangGraph, the coach recalls your specific mistakes from games played days ago.
- **Context-Aware:** It knows when you're tilting mid-game and offers brief, psychological nudges to help you stabilize.
- **Cross-Game Insights:** Ask "Why do I keep losing in the endgame?" and get an answer based on your actual behavioral history.

---
## Links

Frontend - https://blunder-therapist-frontend.vercel.app
Backend - https://blunder-therapist-backend-production.up.railway.app

GitHub Backend - 

---

## 🛠️ Tech Stack

### Backend
- **Core:** Python 3.13 + FastAPI
- **Orchestration:** LangChain & LangGraph (Stateful Agent)
- **Database:** PostgreSQL (Supabase) + SQLAlchemy (Async)
- **Intelligence:** OpenAI GPT-4o (Coach) & GPT-4o-mini (Analysis)
- **Package Manager:** `uv`

### Frontend
- **Framework:** Next.js 15 (App Router) + TypeScript
- **Styling:** Tailwind CSS + shadcn/ui
- **Chess Logic:** react-chessboard + chess.js
- **Engine:** Stockfish.js (WASM, runs 100% in-browser)

---

## 🏗️ Architecture

- **Behavioral Pipeline:** Raw PGN + Timing + Evals → `GameFeatures` → `TiltReport` → `GameSummary` → `DecisionDNA`.
- **Memory Loop:** Every session closure triggers a summarization job that feeds back into the Agent's system prompt for the next game.
- **Event-Driven:** SSE (Server-Sent Events) for real-time AI reactions during gameplay.
- **Import System:** Seamlessly import your history from **Chess.com** and **Lichess** to jumpstart your profile.

---

## 🚦 Getting Started

### Backend
```bash
cd backend
uv sync
cp .env.example .env  # Add OpenAI & Supabase keys
uv run uvicorn app.main:app --reload
```

### Frontend
```bash
cd frontend
npm install
cp .env.example .env.local
npm run dev
```

---

## 📈 Why It Matters
- **60%+ of casual players quit** due to frustration/tilt, not lack of skill.
- Existing coaches (Chessy, Noctie, etc.) focus on **what** move to play.
- We focus on **how** you make decisions.

---

## 🎓 Built for nFactorial Incubator 2026
By **Nurmukhamed Ashekey**, AI Engineer.  
Built in record time using **Antigravity** + **Claude Code** + **Claude Design**.

