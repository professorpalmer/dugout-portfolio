# Dugout — Fantasy Baseball RPG Platform

> A full-stack fantasy baseball application that layers RPG-style equipment progression and a player-driven coin economy on top of real MLB data. Built as a live service deployed via Docker + Cloudflare Tunnel.

**Live app** · Private source · [Architecture](#architecture) · [ML Pipeline](#ml-pipeline) · [System Design](#system-design-highlights) · [Economy Balance](#economy--balance-engineering) · [Testing](#testing)

---

## Overview

Dugout combines traditional head-to-head fantasy baseball with RPG mechanics: 8 equipment slots per player, 6 rarity tiers (Common → Mythic), soulbound loot drops, a rotating coin shop, and a player-driven marketplace with tax-based coin sinks.

The platform serves real-time MLB game data, ML-powered projections, an AI-assisted auction draft system, and automated weekly matchup scoring — all self-hosted on a home server behind Cloudflare Tunnel.

### Key Stats

| Metric | Value |
|--------|-------|
| Backend LoC | ~8,000 (Python) |
| Frontend LoC | ~6,500 (TypeScript/React) |
| Test coverage | 75 tests (unit + integration) |
| ML models | 3 (projections, classifier, draft agent) |
| API endpoints | 50+ |
| Real-time feeds | SSE-based draft room + live scores |

---

## Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                     Cloudflare Tunnel                        │
│                    (zero-trust ingress)                       │
└──────────────────┬───────────────────────────────────────────┘
                   │
┌──────────────────▼───────────────────────────────────────────┐
│  Docker Compose                                               │
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  app (Gunicorn + Uvicorn workers)                       │  │
│  │                                                         │  │
│  │  ┌──────────┐  ┌───────────┐  ┌──────────────────────┐ │  │
│  │  │ FastAPI   │  │ React SPA │  │ APScheduler          │ │  │
│  │  │ REST API  │  │ (static)  │  │ (background jobs)    │ │  │
│  │  └──────┬───┘  └───────────┘  └──────────┬───────────┘ │  │
│  │         │                                 │             │  │
│  │  ┌──────▼──────────────────┐  ┌───────────▼──────────┐ │  │
│  │  │ SQLModel ORM            │  │ ML Services           │ │  │
│  │  │ (SQLAlchemy core)       │  │ • Projections (GBR)   │ │  │
│  │  └──────┬──────────────────┘  │ • Classifier (GMM)    │ │  │
│  │         │                     │ • Draft Agent          │ │  │
│  │         │                     │ • Lineup Optimizer     │ │  │
│  │         │                     └────────────────────────┘ │  │
│  └─────────┼───────────────────────────────────────────────┘  │
│            │                                                  │
│  ┌─────────▼──────────┐                                       │
│  │  PostgreSQL 16      │                                       │
│  │  (persistent vol)   │                                       │
│  └────────────────────┘                                       │
└──────────────────────────────────────────────────────────────┘
```

| Layer | Stack |
|-------|-------|
| **Frontend** | React 19, TypeScript, Vite 8, Zustand, TanStack Query |
| **Backend** | FastAPI, SQLModel, Pydantic v2, Uvicorn |
| **Database** | PostgreSQL 16 (Docker volume) |
| **ML/AI** | scikit-learn (GBR, GMM), NumPy, pandas |
| **Real-time** | Server-Sent Events (SSE) via sse-starlette |
| **Data** | MLB Stats API, pybaseball (Statcast) |
| **Auth** | JWT + bcrypt, email verification (Resend) |
| **Infra** | Docker multi-stage build, Cloudflare Tunnel, Gunicorn |

---

## ML Pipeline

### 1. Player Projection Model

Predicts per-game fantasy points using **Gradient Boosted Regression Trees** trained on historical game logs.

**Feature engineering** (17 hitter features, 10 pitcher features):

```python
HITTER_FEATURES = [
    "avg_hits_5g", "avg_hr_5g", "avg_rbi_5g", "avg_runs_5g",
    "avg_sb_5g", "avg_walks_5g", "avg_doubles_5g",
    "avg_hits_15g", "avg_hr_15g", "avg_rbi_15g",
    "hit_streak", "avg_fantasy_pts_5g", "avg_fantasy_pts_15g",
    "season_avg", "season_obp", "season_slg",
    "games_played",  # capped at 50 to prevent extrapolation
]
```

**Key design decisions:**
- Dual models (hitter/pitcher) with distinct feature sets
- Rolling window features at 5-game and 15-game horizons capture both hot streaks and sustained trends
- `games_played` capped at 50 to prevent early-season extrapolation artifacts
- Prediction floor: blends model output with tier-based baseline (50/50) to prevent zero-point projections for active players
- Retrains automatically via APScheduler on new game data ingestion

### 2. Player Tier Classification (GMM)

Unsupervised clustering using **Gaussian Mixture Models** to dynamically classify players into tiers: Star, Starter, Platoon, Bench.

```python
def _map_clusters_to_tiers(centroids, quality_weights):
    """
    Map GMM cluster indices to tiers by scoring each centroid
    on a weighted composite quality metric, then ranking.
    """
    scores = centroids @ np.array(quality_weights)
    ranked = np.argsort(scores)[::-1]
    return {int(idx): TIER_ORDER[min(rank, 3)] for rank, idx in enumerate(ranked)}
```

**Why GMM over K-Means?**
- Soft cluster assignments handle players on the boundary between tiers
- Covariance modeling captures correlations (e.g., high OPS correlates with high HR rate but not necessarily SB)
- Separate GMMs for hitters and pitchers with domain-specific quality weights

**Feature sets:**
- Hitters: OPS, HR rate, RBI rate, SB rate, AVG, games played
- Pitchers: inverse ERA, inverse WHIP, K rate, win rate, save rate, games played

Tiers drive the equipment drop system — Star players find rarer gear more frequently.

### 3. Draft Recommendation Agent

An agentic system that synthesizes multiple analysis "tools" into auction draft advice:

```python
# Tool-calling pattern: gather context, then synthesize
@dataclass
class DraftRecommendation:
    action: str            # "bid" | "nominate" | "pass"
    max_bid: int
    reasoning: str
    confidence: float
    value_score: float     # projection-based value over replacement
    scarcity_score: float  # positional scarcity in league
    need_score: float      # how badly user needs this position
```

**Agent reasoning chain:**
1. **Value analysis** — project player's season points, compute value over replacement level (position-adjusted)
2. **Scarcity analysis** — scan all league rosters for positional supply/demand
3. **Budget analysis** — model remaining budget against unfilled roster slots
4. **Need analysis** — evaluate user's current roster gaps
5. **Synthesis** — weighted combination produces bid recommendation with natural-language reasoning

The agent also handles **AFK detection** — if a player misses 3 consecutive picks, a bot takes over their bidding using the same recommendation engine, maintaining auction competitiveness.

---

## System Design Highlights

### Real-Time Draft Room

The auction draft runs as an **in-memory state machine** with Server-Sent Events for real-time updates to all connected clients.

```
DraftRoom (in-memory)
├── State: NOMINATING | BIDDING | PAUSED | COMPLETE
├── Members[]: { user_id, budget, roster, is_afk, last_active_pick }
├── Current auction: { player, highest_bid, bidder, deadline }
├── Timer thread: manages deadlines, triggers bot actions
└── SSE broadcast: pushes events to all connected clients
```

**Design rationale:** In-memory over database for the draft because:
- Auction state changes every 1-3 seconds during bidding
- Consistency requirements are per-session (draft doesn't survive server restart, which is acceptable)
- Timer precision matters — database polling introduces unacceptable latency for bid deadlines

### Live Scoring Pipeline

```
MLB Stats API (poll every 60s)
    │
    ▼
score_completed_games()
    ├── Filter: only process rostered starters (bench/IL excluded)
    ├── Score: apply fantasy points formula + equipped gear modifiers
    ├── Loot: evaluate trigger conditions → probabilistic drop gate
    ├── Currency: calculate earnings (capped at 75/game)
    └── Persist: game log (for ML training) + user stats + gear instances
```

### Equipment Modifier Engine

The scoring engine applies gear modifiers with multiple balance layers to prevent power concentration:

```python
MAX_GEAR_BOOST_PCT = 0.40
ALL_DIMINISHING_STEPS = [1.0, 0.75, 0.50, 0.35, 0.25, 0.20, 0.15, 0.10]

def apply_gear_modifiers(breakdown, gear_templates):
    # 1. Separate 'all' vs single-stat modifiers vs penalties
    # 2. Apply 'all' modifiers with diminishing returns per stack
    # 3. Apply single-stat modifiers at full value
    # 4. Hard-cap total positive boost at 40% of raw points
    # 5. Apply penalties AFTER cap (risk stays uncapped)
```

This prevents any single player from becoming unbeatable regardless of gear investment, while keeping the risk/reward of high-rarity equipment meaningful.

---

## Economy & Balance Engineering

The coin economy is designed around **healthy circulation** with multiple sinks:

| Flow | Mechanism |
|------|-----------|
| **Earn** | Per-game player performance (capped 75/game), draft surplus (1.5x conversion), weekly matchup bonuses |
| **Spend** | Rotating shop (8 items/week, up to Rare), player marketplace |
| **Sink** | 10% marketplace tax, equipment pricing tiers |
| **Anti-abuse** | Per-game currency cap, shop duplicate prevention, marketplace price bounds (5–50k), server-side tier validation |

### Loot Drop System

Equipment acquisition uses a **two-phase probabilistic system**:

**Phase 1: Performance Triggers**
- Specific in-game achievements unlock themed equipment candidates (e.g., stolen base → speed cleats)
- Each candidate passes through a **rarity-gated probability check** scaled by player tier
- Star players: lower drop rates (they're already valuable). Bench players: higher rates (catch-up mechanic)

**Phase 2: Rarity Roll**
- Players who exceed their ML-projected points by 20%+ get a bonus rarity roll
- The exceed ratio shifts the rarity weight distribution — bigger outperformance = higher chance of rare+ drops
- This creates a natural reward for "sleeper" players who overperform projections

### Anti-Snowball Mechanics

| Mechanism | Purpose |
|-----------|---------|
| 40% modifier cap per player | Prevents gear stacking from creating untouchable players |
| Diminishing returns on "all" modifiers | 2nd stacks at 75%, 3rd at 50%, etc. |
| Soulbound equipment | Looted gear locks to the earning player — can't freely redistribute |
| Per-game currency cap | Prevents rich-get-richer runaway from star player performance |
| Marketplace-only gear exchange | Forces coin circulation; no backdoor trades |
| Inverse tier drop rates | Bench/Platoon players find better gear, creating catch-up opportunities |

---

## Testing

75 automated tests across the full stack:

| Suite | Tests | Covers |
|-------|-------|--------|
| `test_auth` | 8 | Registration, login, JWT validation, edge cases |
| `test_scoring` | 7 | Fantasy point calculation, gear modifiers, penalties |
| `test_balance` | 15 | Modifier cap, diminishing returns, currency cap, duplicate prevention, trade rules, IL mechanics, marketplace bounds, score dedup |
| `test_draft_agent` | 5 | Value-over-replacement, roster needs analysis, bid recommendations |
| `test_lineup_optimizer` | 7 | Position assignment, multi-eligibility, slot optimization |
| `test_matchup` | 11 | Round-robin scheduling, currency rewards, edge cases |
| `test_player_classifier` | 11 | Feature extraction, GMM clustering, tier mapping |
| `test_projections` | 9 | Rolling averages, heuristic fallbacks, feature engineering |
| `test_trades` | 2 | Status transitions, model integrity |

Integration tests use FastAPI's `TestClient` with fresh SQLite databases per test (via pytest fixtures), testing the full request → business logic → database → response cycle.

---

## Deployment

Self-hosted on a home server behind Cloudflare Tunnel (zero-trust ingress, no port forwarding):

```yaml
services:
  db:      # PostgreSQL 16 with health checks
  app:     # Multi-stage Docker build (Node → Python)
  tunnel:  # cloudflare/cloudflared container
```

The multi-stage Dockerfile builds the React frontend in a Node 20 Alpine container, then copies the static assets into a Python 3.12-slim runtime that serves both the API and SPA via Gunicorn with Uvicorn workers.

---

## Tech Decisions & Trade-offs

| Decision | Rationale |
|----------|-----------|
| **SQLModel over raw SQLAlchemy** | Pydantic v2 integration gives type-safe serialization with zero boilerplate; trade-off is less control over complex queries |
| **SSE over WebSocket** | Simpler protocol for one-way server→client updates (draft, scores); bidirectional not needed since mutations go through REST |
| **In-memory draft room** | Sub-second state transitions during auctions; acceptable that draft state doesn't survive restarts |
| **GMM over K-Means for tiers** | Soft assignments + covariance modeling better capture player stat correlations |
| **Greedy optimizer over branch-and-bound** | O(n·m) vs exponential; accuracy loss <2% on tested rosters, but completes in <100ms vs potentially minutes |
| **Per-game currency cap over weekly** | Simpler to implement without schema changes; provides 90% of the anti-hoarding benefit |
| **Soulbound loot + marketplace-only exchange** | Forces coin circulation and prevents gear hoarding through trades |

---

*Built with Python, TypeScript, and real MLB data. Deployed as a live service for family and friends.*
