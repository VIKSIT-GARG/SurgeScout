# AGENTS.md — SurgeScout

## Project Overview

SurgeScout is an autonomous AI deal-flow agent for Internet Capital Markets (ICM). It discovers promising projects on Moltbook, analyzes their tokenization potential using a 5-dimension LLM scoring model, and helps launch tokens on the SURGE launchpad.

**Pipeline:** `Scout → Analyze → Advise → Launch → Report`

## Architecture

```
SurgeScout/
├── app.py                      # Entry point — starts uvicorn server on PORT 8899
├── src/
│   ├── __init__.py
│   ├── config.py               # All env-var configuration (LLM keys, API URLs, paths)
│   ├── server.py               # FastAPI app — 12 REST endpoints + demo data + state management
│   ├── moltbook.py             # Moltbook API client — fetch posts, search, post updates, extract projects
│   ├── analyzer.py             # LLM-powered analysis — system prompt, scoring, batch analysis, report generation
│   └── launcher.py             # SURGE launchpad client — prepare launch configs, deploy tokens (stub), Web3.py ERC20
├── skill/
│   ├── SKILL.md                # OpenClaw skill manifest (name, description, author, tools)
│   └── references/
│       ├── scoring-model.md    # Detailed scoring rubric (5 dimensions, weights, examples)
│       └── surge-integration.md # SURGE API reference (token params, bonding curves, chains)
├── web/
│   └── index.html              # Single-page dashboard — Tailwind CSS, Chart.js, marked.js (841 lines)
├── scripts/
│   ├── scout.py                # CLI: run Moltbook scouting standalone
│   ├── analyze.py              # CLI: analyze a single project
│   └── pipeline.py             # CLI: run full Scout→Analyze→Report pipeline
├── data/                       # Runtime state directory (gitignored JSON files, future DB)
├── moltbook-posts.md           # Pre-written Moltbook posts for hackathon submission (10 posts)
├── .env.example                # Template for environment variables
├── requirements.txt            # Python dependencies (fastapi, uvicorn, httpx, anthropic, openai, web3)
├── Procfile                    # Railway deployment command
├── railway.json                # Railway deployment config
├── render.yaml                 # Render deployment config
├── runtime.txt                 # Python version for deployment
└── LICENSE                     # MIT
```

## Key Modules

### `src/config.py`
Loads all configuration from environment variables with sensible defaults. Key vars:
- `AI_PROVIDER` / `ANTHROPIC_API_KEY` / `OPENAI_API_KEY` — LLM provider selection
- `MOLTBOOK_API_URL` / `MOLTBOOK_SUBMOLT` — Moltbook connection
- `SURGE_API_URL` / `BASE_RPC_URL` / `WALLET_PRIVATE_KEY` — on-chain deployment
- `HOST` / `PORT` / `SECRET_KEY` — server settings

### `src/server.py`
FastAPI app with all routes. State is an in-memory Python dict (`_state`) persisted to `data/state.json`. Contains hardcoded demo data (`DEMO_PROJECTS`, `DEMO_ANALYSES`) for presentation purposes. Mounts `web/` as static files.

**Important:** All route handlers are `async def` but the underlying calls to `analyzer.py` and `moltbook.py` are synchronous and blocking. The `analyze_batch()` function uses `time.sleep(3)` between API calls which blocks the entire event loop.

### `src/moltbook.py`
HTTP client for Moltbook (the AI agent social network). Two main functions:
1. **Reading:** `fetch_posts()`, `fetch_trending()`, `search_posts()` — all return lists of post dicts
2. **Writing:** `post_update()`, `post_comment()` — currently unauthenticated (will fail in production)

The `scout_moltbook()` function orchestrates a full scouting run: fetches from 10 submolts × 2 sort orders + trending + 13 keyword searches, then extracts project signals via regex (GitHub URLs, `$TICKER` mentions, keywords like "launch", "deploy", "hackathon").

### `src/analyzer.py`
Sends project context to Claude/GPT with a detailed system prompt requesting structured JSON output. The LLM scores projects on 5 weighted dimensions (market_fit, team_signal, technical_quality, tokenomics_feasibility, traction) and produces a verdict.

JSON extraction from LLM response tries 3 strategies: direct parse → code block extraction → first `{...}` match.

`analyze_batch()` processes up to 8 projects with 3-second delays and 1 retry on rate limits.

### `src/launcher.py`
Prepares token launch configurations and attempts deployment. The `deploy_erc20_base()` function is a **stub** — it returns a mock result with deployer address but does not actually deploy a contract (no bytecode is included). The `deploy_token()` method calls the SURGE API, but falls back to "prepared" status on any error.

## Conventions

- **Python 3.12+** required
- **No type stubs** — uses built-in `list[str]`, `dict | None` syntax
- **Logging** — all modules use `logging.getLogger("surgescout.<module>")`
- **Error handling** — most failures are caught and returned as `{"error": "..."}` dicts rather than raising HTTP exceptions
- **State** — global mutable `_state` dict in `server.py`, no database
- **Frontend** — single HTML file, no build step, CDN-loaded Tailwind/Chart.js/marked.js
- **Deployment** — Railway (primary) or Render, configured via `Procfile` / `railway.json` / `render.yaml`

## Running the Project

```bash
# Install dependencies
pip install -r requirements.txt

# Set up environment
cp .env.example .env
# Edit .env — at minimum set ANTHROPIC_API_KEY or OPENAI_API_KEY

# Start server
python app.py
# Dashboard: http://localhost:8899
# Health: http://localhost:8899/api/health
```

## CLI Tools

```bash
# Scout Moltbook for projects
python scripts/scout.py --submolts lablab defi agents

# Analyze a single project
python scripts/analyze.py --title "MyProject" --body "Description..." --github "https://github.com/user/repo"

# Run full pipeline
python scripts/pipeline.py --submolts lablab cryptocurrency --min-score 50
```

## Testing

No tests exist yet. When adding tests:
- Use `pytest` + `httpx.AsyncClient` for API integration tests
- Mock `_call_llm()` in `analyzer.py` to avoid real API calls
- Mock `httpx` responses in `moltbook.py` and `launcher.py`
- Test JSON extraction edge cases in `_extract_json()`

## Known Limitations

1. **Blocking I/O** — `analyze_batch()` uses `time.sleep(3)` which freezes the FastAPI event loop
2. **No persistent database** — state is a global dict saved to JSON; lost if save fails
3. **No authentication** — CORS is `*`, all endpoints are public, Moltbook posts are unauthenticated
4. **SURGE deployment is a stub** — `deploy_erc20_base()` returns mock data, no real on-chain deployment
5. **No scheduler** — the pipeline must be manually triggered; there is no automatic scouting loop
6. **Single-file frontend** — `web/index.html` is 841 lines with all CSS, HTML, and JS inline
