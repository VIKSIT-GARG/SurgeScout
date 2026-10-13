# SurgeScout

**AI Deal Flow Agent for Internet Capital Markets**

SurgeScout is an autonomous AI agent built on [OpenClaw](https://github.com/openclaw/openclaw) that discovers promising projects on [Moltbook](https://www.moltbook.com), analyzes their tokenization potential, and helps launch tokens on the [SURGE](https://surge.xyz) launchpad.

> Think of it as an AI-powered venture scout that operates 24/7 on the agent internet.

## How It Works

SurgeScout runs a continuous pipeline:

```
Scout → Analyze → Advise → Launch → Report
  │         │         │        │        │
  │         │         │        │        └─ Post findings to Moltbook
  │         │         │        └─ Deploy token via SURGE launchpad
  │         │         └─ Generate tokenomics recommendations
  │         └─ LLM-powered scoring (5 dimensions, 0-100)
  └─ Monitor Moltbook submolts for project signals
```

### Scoring Model

Each project is evaluated on 5 dimensions:

| Dimension | Weight | What It Measures |
|-----------|--------|-----------------|
| Market Fit | 25% | Real demand, clear users, market size |
| Team Signal | 20% | Reputation, GitHub activity, past wins |
| Technical Quality | 25% | Working code, deployed contracts, clean architecture |
| Tokenomics Feasibility | 20% | Token utility, revenue model, sustainability |
| Traction | 10% | Moltbook engagement, social proof |

**Verdicts:**
- **STRONG_LAUNCH** (75+) — Ready for token deployment
- **PROMISING** (50-74) — Worth watching
- **NEEDS_WORK** (25-49) — Not ready yet
- **SKIP** (<25) — No tokenization potential

## Features

- **Autonomous Scouting** — Monitors 5+ Moltbook submolts for project signals
- **LLM Analysis** — Deep project evaluation via Claude or GPT
- **Tokenomics Engine** — Generates ticker, supply, bonding curve recommendations
- **SURGE Integration** — Prepares and deploys tokens on Base/Solana via SURGE
- **Moltbook Reporting** — Posts analysis reports to m/lablab submolt
- **Web Dashboard** — Real-time visualization of agent activity
- **OpenClaw Skill** — Drop-in skill for any OpenClaw agent
- **Full Pipeline API** — One endpoint runs the complete cycle

## Quick Start

```bash
# Clone
git clone https://github.com/yashjangrra/SurgeScout.git
cd SurgeScout

# Install
pip install -r requirements.txt

# Configure
cp .env.example .env
# Edit .env — add your ANTHROPIC_API_KEY or OPENAI_API_KEY

# Run
python app.py
# Open http://localhost:8899
```

### Install as OpenClaw Skill

```bash
# Copy skill to OpenClaw workspace
cp -r skill/ ~/.openclaw/workspace/skills/surgescout/

# Or tell your OpenClaw agent:
# "Install the SurgeScout skill from /path/to/SurgeScout/skill"
```

## API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/` | GET | Web dashboard |
| `/api/health` | GET | Health check |
| `/api/state` | GET | Full agent state |
| `/api/scout` | POST | Scout Moltbook for projects |
| `/api/analyze` | POST | Analyze project(s) |
| `/api/analyses` | GET | List all analyses |
| `/api/report/{idx}` | GET | View analysis report |
| `/api/launch` | POST | Launch token |
| `/api/launches` | GET | List all launches |
| `/api/moltbook/post` | POST | Post to Moltbook |
| `/api/moltbook/report` | POST | Post report to Moltbook |
| `/api/pipeline` | POST | Run full pipeline |

## Architecture

```
SurgeScout/
├── app.py                      # Entry point (uvicorn server)
├── src/
│   ├── config.py               # Configuration from .env
│   ├── server.py               # FastAPI server + all routes
│   ├── moltbook.py             # Moltbook API client + project extraction
│   ├── analyzer.py             # LLM-powered analysis engine
│   └── launcher.py             # SURGE launchpad + on-chain deployment
├── skill/
│   ├── SKILL.md                # OpenClaw skill definition
│   └── references/
│       ├── surge-integration.md
│       └── scoring-model.md
├── web/
│   └── index.html              # Dashboard UI (Tailwind + Chart.js)
├── data/                       # Persisted agent state
├── requirements.txt
├── .env.example
├── LICENSE
└── README.md
```

## Tech Stack

- **Runtime**: [OpenClaw](https://github.com/openclaw/openclaw) — self-hosted AI agent
- **Backend**: Python + FastAPI
- **AI**: Anthropic Claude / OpenAI GPT (configurable)
- **On-chain**: SURGE launchpad + Web3.py (Base/Solana)
- **Social**: Moltbook API (agent posting)
- **Frontend**: Tailwind CSS, Chart.js, marked.js

## Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `AI_PROVIDER` | `anthropic` | LLM provider |
| `ANTHROPIC_API_KEY` | - | Claude API key |
| `OPENAI_API_KEY` | - | OpenAI API key |
| `MOLTBOOK_API_URL` | `https://www.moltbook.com` | Moltbook API |
| `MOLTBOOK_SUBMOLT` | `lablab` | Default submolt |
| `SURGE_API_URL` | `https://api.surge.xyz` | SURGE API |
| `BASE_RPC_URL` | `https://sepolia.base.org` | Base RPC |
| `WALLET_PRIVATE_KEY` | - | Wallet for deployments |

## Future Roadmap

Development is organized into 4 phases, each building on the last.

### Phase 1 — Foundation & Reliability

Make the core pipeline production-ready.

- **SQLite Database** — Replace the in-memory `_state` dict + JSON file with SQLite via SQLModel for persistent, queryable storage across restarts
- **Async I/O** — Convert all LLM calls, Moltbook fetches, and SURGE requests from blocking `httpx.Client` + `time.sleep()` to non-blocking `httpx.AsyncClient` + `asyncio`
- **Background Scheduler** — Add `APScheduler` so the scout→analyze pipeline runs automatically on a configurable interval instead of requiring manual triggers
- **Moltbook Auth** — Wire up `MOLTBOOK_AUTH_TOKEN` so posting to Moltbook actually authenticates
- **API Key Middleware** — Protect all endpoints behind a simple `X-API-Key` header check
- **Tests** — Add `pytest` integration tests for endpoints, analyzer, and Moltbook client with mocked HTTP/LLM calls

### Phase 2 — Smarter Intelligence

Improve scouting accuracy and analysis quality.

- **GitHub Enrichment** — For each GitHub URL found during scouting, fetch real repo data (stars, forks, last commit, language) via the GitHub API and feed it into the LLM context
- **Multi-Stage Analysis** — Split the single mega-prompt into 3 focused stages: (1) factual extraction, (2) scoring, (3) tokenomics design — reducing hallucination and improving consistency
- **Cross-Source Scouting** — Add GitHub trending and Moltbook user profile enrichment as additional signal sources beyond submolt scanning
- **Scoring Calibration** — Store past analyses and include few-shot examples of known `STRONG_LAUNCH` projects in the system prompt for more consistent scoring
- **Semantic Deduplication** — Replace the MD5 content hash with embedding-based similarity (cosine >0.9 = duplicate) to catch reformatted reposts

### Phase 3 — Real On-Chain Integration

Make the Launch step deploy actual tokens and close the full loop.

- **ERC20 Deployment** — Ship real pre-compiled ERC20 bytecode and deploy via Web3.py on Base Sepolia testnet instead of returning mock results
- **SURGE API** — Implement the actual SURGE launchpad API endpoints with proper wallet-signature authentication
- **Transaction Tracking** — Store `contract_address`, `tx_hash`, and `chain_id` in the database; add `GET /api/launches/{id}/status` for monitoring
- **Multi-Chain** — Add Solana deployment support via `solana-py` alongside Base
- **Launch Notifications** — Auto-post launch announcements to Moltbook with contract address + block explorer link; support configurable webhook URLs

### Phase 4 — Scale, UX & Ecosystem

Turn SurgeScout from a hackathon project into a real product.

- **React Frontend** — Replace the 841-line monolithic `index.html` with a Next.js + shadcn/ui app: project browser, analysis history, launch tracker, settings page
- **WebSocket Updates** — Replace dashboard polling with a `/ws` WebSocket connection streaming real-time pipeline events
- **Watchlist & Alerts** — Let users watch specific projects or submolts and receive email/webhook alerts when `STRONG_LAUNCH` projects appear
- **Analytics Dashboard** — Scout hit rate, score distributions, launch success rate, and top-performing submolts
- **Multi-Tenancy** — Per-user Moltbook accounts, wallets, watched submolts, and private analyses
- **OpenClaw Skill v2** — Publish to the OpenClaw skill marketplace with agentic loop support and `skill.json` manifest

## SURGE x OpenClaw Hackathon

This project was built for the **SURGE x OpenClaw Hackathon** on [lablab.ai](https://lablab.ai/ai-hackathons/surge-moltbook-hackathon).

- **Track**: Internet Capital Markets for Creators/Products
- **Prize Pool**: $50,000 in $SURGE tokens
- **Technologies**: OpenClaw, SURGE, Moltbook

## License

MIT
