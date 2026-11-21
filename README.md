# Lyftr AI LLM Chatbot

## 1. Live URLs
- Frontend:[https://frontend-phi-eight-10.vercel.app/](https://frontend-phi-eight-10.vercel.app/)
- Backend: [https://lyftr-backend-kd36.onrender.com/](https://lyftr-backend-kd36.onrender.com/)
- Repository: add your public GitHub repository URL here.

## 2. Run locally: with Docker and without
### With Docker
Copy `.env.example` to `.env`, add your Gemini keys, then run:

```bash
make up
```

Open `http://localhost:3000`. The API is at `http://localhost:8000` and the SQLite volume is named `lyftr-data`.

```bash
make down
```

### Without Docker
```bash
cd backend
python -m venv .venv
.venv/Scripts/activate   # Windows
pip install -r requirements.txt
set DATABASE_URL=sqlite:///./app.db
set LLM_API_KEYS=key1,key2
set ENABLE_FAILURE_SIMULATION=true
uvicorn main:app --reload --port 8000

cd ../frontend
npm install
set VITE_API_URL=http://localhost:8000
npm run dev -- --port 3000
```

Run backend tests:

```bash
cd backend
pytest -q
```

Or with Docker running:

```bash
docker compose exec backend pytest -q
```

## 3. Design decisions: key fallback, caching, SQLite session storage
FastAPI exposes exactly `/health`, `/chat`, and `/sessions/{session_id}/messages`. LiteLLM is configured with a local built-in cache and calls the Gemini model through each configured API key. The backend tries keys in order and records only provider-failure fallback attempts. An in-process cache additionally retains the reply and original token usage by an exact model-plus-full-message-list hash. SQLite stores user and assistant rows; an optional sessions table holds the session system prompt while the history endpoint excludes it.

## 4. Verification: how to see failover, cache hits and per-message token usage
Use the **API failover demo** selector above the chat to simulate a key failure. Choose **Fail API key 1** to make the backend switch to key 2, or choose **Fail all API keys** to show the 503 error state. Assistant replies display the active API key and fallback count; when a switch occurs, a green **API switched** badge appears.

Send the same message in two new sessions. The first response should be `LIVE_INFERENCE`; the second with identical system prompt and history should be `LLM_CACHE` and contain the same reply. Send these failover requests:

```bash
curl -X POST http://localhost:8000/chat -H "Content-Type: application/json" -H "X-Simulate-Failure: key_1=429" -d '{"session_id":"demo","message":"Hello"}'
curl -X POST http://localhost:8000/chat -H "Content-Type: application/json" -H "X-Simulate-Failure: *=429" -d '{"session_id":"failed","message":"Hello"}'
```

Check history with `GET /sessions/demo/messages`. Input and output tokens are shown beneath every frontend assistant reply.

## 5. Deploy to Render and Vercel

### Deploy the backend to Render
1. Push the repository to GitHub.
2. In Render, choose **New > Blueprint** and select the repository. Render reads [render.yaml](render.yaml).
3. Set the `LLM_API_KEYS` secret in the Render dashboard. These should be real Gemini keys, with each key separated by a comma.
4. Deploy the service and copy its public URL, for example `https://lyftr-backend.onrender.com`.
5. Set `ALLOWED_ORIGINS` to the exact Vercel frontend URL (without a trailing slash), for example `https://lyftr.vercel.app`.

The free blueprint runs SQLite at `/data/app.db`, but the filesystem is ephemeral. Chat history may be lost during redeploys, restarts, or free-service rebuilds. For persistent production storage, add an external database or upgrade to a Render paid persistent disk.

### Deploy the frontend to Vercel
1. Import the same GitHub repository in Vercel.
2. Set **Root Directory** to `frontend`.
3. Set `VITE_API_URL` to the Render backend URL, for example `https://lyftr-backend.onrender.com`.
4. Deploy. Vite uses `VITE_API_URL` at build time.
5. Copy the Vercel URL and add it to Render's `ALLOWED_ORIGINS`, then redeploy or restart the backend.

The frontend can also use [frontend/vercel.json](frontend/vercel.json) for Vite SPA routing. Never put Gemini API keys in `VITE_*` variables or in frontend code: Vite variables are exposed to browsers.

## 6. Setup Used
Built in VS Code on Windows using Git, Docker Desktop, React/Vite, FastAPI, LiteLLM, SQLite, and AI-assisted implementation.


