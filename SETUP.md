# Setup

## Backend (Python 3.11+)

```bash
cd backend
python -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate
pip install -r requirements.txt

cp .env.example .env
# edit .env: paste your OPENAI_API_KEY

uvicorn app.main:app --reload --port 8000
```

Visit http://localhost:8000/docs to see the API.

## Frontend (Node 18+)

```bash
cd frontend
npm install

cp .env.example .env.local

# Stockfish needs the worker file in /public. Easiest way:
# Download stockfish.js from https://github.com/lichess-org/stockfish.js
# (or `cp node_modules/stockfish.js/stockfish.js public/stockfish.js`)
mkdir -p public
cp node_modules/stockfish.js/stockfish.js public/stockfish.js

npm run dev
```

Visit http://localhost:3000.

## Next steps after baseline works

Day 2 plan (the AI day - your main value-add):
- [ ] Add a /history page that lists saved games
- [ ] Wire `decision-dna` endpoint into UI (button: "Build my Decision DNA" - needs 3+ games)
- [ ] Wire `coach` endpoint into a chat sidebar
- [ ] Add Supabase for persistence so games survive refresh

Day 3 plan (polish):
- [ ] Add Stripe paywall on Decision DNA + Coach (test mode)
- [ ] Make the Tilt Report exportable as a shareable image (html-to-image lib)
- [ ] Record a 90s demo video
- [ ] Deploy: backend to Railway, frontend to Vercel

## Production notes (don't do these for the MVP)

- Move Stockfish eval from frontend to backend (more accurate, but slower)
- Add JWT auth instead of trusting frontend
- Rate-limit the LLM endpoints
- Cache evaluations by FEN
