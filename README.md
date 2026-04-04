# Dugout — Fantasy Baseball RPG Platform

> Full-stack fantasy baseball with RPG equipment, a player-driven coin economy, and in-house ML on real MLB data. **Live production** app behind Docker, PostgreSQL, and Cloudflare (Tunnel + edge TLS).

**[Live app](https://dugoutfantasy.com)** · Private source · [Screenshots](#screenshots) · [Overview](#overview) · [Security](#security--production-hardening) · [Architecture](#architecture) · [ML pipeline](#ml-pipeline) · [System design](#system-design-highlights) · [Economy](#economy--balance-engineering) · [Ops](#operations) · [Testing](#testing)

---

## Screenshots

| Landing | In-app guide ("How to Play") |
|---------|------------------------------|
| ![Landing](screenshots/landing.png) | ![Guide](screenshots/guide.png) |

| Equipment (8 slots + soulbound) | Weekly matchup & scoring breakdown |
|----------------------------------|-----------------------------------|
| ![Equipment](screenshots/equipment.png) | ![Matchup](screenshots/matchup.png) |

| Shop & marketplace | Gear locker |
|--------------------|-------------|
| ![Shop](screenshots/shop.png) | ![Locker](screenshots/locker.png) |

| MLB research (schedules + roster context) | AI insights & lineup optimizer |
|-------------------------------------------|--------------------------------|
| ![Research](screenshots/research.png) | ![AI Insights](screenshots/ai-insights.png) |

---

## Overview

Dugout layers **RPG progression** on head-to-head fantasy baseball:

- **8 equipment slots** per player (hat, shades, chain, jersey, glove, bat, wristband, cleats) with **6 rarity tiers** (Common → Mythic).
- **Soulbound loot** from in-game performance; **marketplace** resales with tax sinks.
- **Auction draft** with an **AI advisor** panel (real-time bid/let-go recommendations, natural-language reasoning) and **bot opponents** that bid with distinct personalities.
- **5 bot personality archetypes** (Stars & Scrubs, Balanced, Value Hunter, Position Scarcity, Late Surge) — persisted per bot and used all season: draft bidding, waivers, trades, marketplace, lineup optimization, and gear equipping.
- **Flexible league sizes** — commissioners fill empty slots with AI managers; play with 3 friends and 7 bots, or any mix.
- **Switch Mode** moves between Solo and League contexts with independent rosters, gear, scores, and matchups.
- **Daily waiver processing** (8 AM ET) with 2-day claim windows; bot teams evaluate and submit claims based on roster need and personality.
- **Research** pulls MLB schedules and rosters with fantasy ownership context; pitcher roles (SP / relievers / closers) align with MLB depth-chart data where available.
- **AI Insights** exposes projection models and a lineup optimizer (non-generative ML); in-product copy distinguishes proprietary models from LLM hype.

---

## Security & production hardening

Ship-ready controls added for a public URL and real users:

| Area | Approach |
|------|----------|
| **API reads** | Data routes (`/leagues/...`, `/schedule`, marketplace, leaderboard, etc.) require **`Authorization: Bearer`** by default. Optional `DUGOUT_ALLOW_PUBLIC_READS` for debugging only. |
| **SSE** | Live score / draft streams expect a **JWT** (e.g. `?token=`) unless `DUGOUT_ALLOW_PUBLIC_SSE` is explicitly enabled. |
| **Admin / maintenance** | High-impact routes (`roster sync`, bot pitcher backfill, bot jobs) require **`X-Admin-Key`** matching `DUGOUT_ADMIN_API_KEY`; missing key → **404** in production. |
| **Manual scoring** | Gated behind env (`DUGOUT_ALLOW_MANUAL_SCORING`) outside dev to avoid stat injection abuse. |
| **Secrets in Compose** | Production compose passes admin key and feature flags from `.env` into the app container (not only JWT/DB). |
| **Password reset** | Email-based 6-digit code flow via Resend; rate-limited (3 req/min send, 5/min reset); silent success on unknown emails to prevent enumeration. |
| **Social / PWA** | Open Graph + Twitter Card meta, compressed `og-image`, PNG favicons + manifest icons. |

---

## Architecture

```
┌──────────────────────────────────────────────────────────────┐
│              Cloudflare (TLS, Tunnel, optional cache)         │
└──────────────────┬───────────────────────────────────────────┘
                   │
┌──────────────────▼───────────────────────────────────────────┐
│  Docker Compose                                               │
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  app (Gunicorn + Uvicorn workers)                       │  │
│  │  ┌──────────┐  ┌───────────┐  ┌──────────────────────┐   │  │
│  │  │ FastAPI  │  │ React SPA │  │ APScheduler          │   │  │
│  │  │ REST API │  │ (Vite)    │  │ (jobs, scoring, ML)  │   │  │
│  │  └────┬─────┘  └───────────┘  └──────────┬───────────┘   │  │
│  │       │  ┌──────────────────┐  ┌─────────▼──────────┐   │  │
│  │       └─►│ SQLModel / PG    │  │ ML stack           │   │  │
│  │          └──────────────────┘  │ GBR, GMM, draft    │   │  │
│  │                                │ agent, optimizer   │   │  │
│  └────────────────────────────────┴──────────────────────┘   │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  PostgreSQL 16 (volume)                                  │  │
│  └─────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

| Layer | Stack |
|-------|-------|
| **Frontend** | React 19, TypeScript, Vite, Zustand |
| **Backend** | FastAPI, SQLModel, Pydantic v2, Uvicorn/Gunicorn |
| **Database** | PostgreSQL 16 |
| **ML** | scikit-learn (GBR, GMM), NumPy, pandas |
| **Real-time** | SSE (`sse-starlette`) — draft room & live scores |
| **Data** | MLB Stats API (rosters, depth charts, game logs, person stats), Statcast |
| **Auth** | JWT + bcrypt, email verification + password reset (Resend) |
| **Infra** | Multi-stage Docker image, cloud VPS + Cloudflare Tunnel |

---

## ML pipeline

### 1. Player projections (GBR)

Per-game fantasy points via **gradient boosted regression** on rolling game-log features (separate hitter/pitcher feature sets, rolling 5g/15g horizons), tier-aware floors, Statcast context, and scheduled retraining as new games land.

### 2. Tier classification (GMM + production rules)

- **Pools:** Hitters, starters (SP), and relievers (RP + depth-chart CP closers + inferred P arms where games-started ratio indicates relief). Closers compete in the same reliever pool as setup arms — elite stats surface naturally.
- **Features:** Blended prior + current season from PlayerGameLog (rate stats blended linearly; counting stats blended per-game when both seasons exist). Minimum game log threshold before full GMM classification; model fallback for sparse history with sanity gates so tiny samples don't pollute stars.
- **Scoring:** Standardized feature vectors per pool, weighted composite score, then percentile cutoffs (e.g. top ~10% → Star) — avoids "one cluster = 25% stars" artifacts from raw GMM cluster-to-tier mapping.
- **Fallback:** Players without enough data use `projected_pts_per_game` ranking; STAR is reserved for the main pipeline — fallback tier is capped so small-sample flukes don't show as franchise players.
- Tiers feed loot rarity tuning and draft auction valuations.

### 3. Auction pricing engine

Position-aware **piecewise linear curves** map `projected_pts_per_game` to draft dollars, calibrated against the production database's actual ppg distribution:

- **Hitter curve** (weekly = ppg × 6.5): four segments from bench ($1) through generational ($60+) with a 35% budget cap.
- **Pitcher curve** (weekly = ppg × 2.0): four segments tuned so ace SPs ($25–30) never outbid elite everyday hitters.
- **Closer premium:** relievers (low ppg due to 1 IP/game) receive a tier-based floor so elite closers compete at $10–18 despite the SP-centric curve.
- **Multiplier cap** (2.0×) prevents any single bot modifier chain (personality × variance × need × early-star) from producing runaway bids.

Designed so: Ohtani settles ~$55–90, star hitters ~$25–55, ace SPs ~$20–40, mid-rotation ~$5–15, bench ~$1–5. Variance + personality differences keep drafts from feeling scripted.

### 4. Draft advisor (AI panel)

Real-time **bid / let-go** recommendations shown in a collapsible panel during the auction:

- Combines **auction value** (from the pricing engine), **positional scarcity**, **roster need**, and **budget advantage** into a composite score.
- Four action tiers: **Must Have**, **Bid Aggressively**, **Bid Moderately**, **Let Go** — with natural-language reasoning explaining the recommendation.
- Local bid tracking: when the price rises past the recommended ceiling, the panel updates instantly (no API round-trip) to signal "let go."
- Uses the same `projected_pts_per_game` and curve as bot valuations, so advisor recommendations align with the market bots create.

### 5. Lineup optimizer

Assigns starters under **positional eligibility** and **roster lock** rules; respects gear modifiers in effective scoring. Greedy assignment — fast enough for interactive use vs. exponential exact search.

Product messaging: **in-house statistical ML**, not generative AI.

---

## System design highlights

### Real-time draft room

In-memory **auction state machine** (nomination → bidding → timers) with **SSE** fan-out. Draft persistence is intentional trade-off: sub-second bid latency vs. surviving process restarts.

**Bot bidding engine:** 5 personality archetypes control tier multipliers, bid increment ranges, timing profiles, and drain nomination probability (bots sometimes nominate stars they don't need to force opponents to overspend). Highest-valuation bot bids first; personality-aware increments; "fight" behavior where bots near their ceiling occasionally push past it. Need-boost ramps in gradually (prevents uniform inflation when all rosters are empty early).

**AFK assistance:** when a human's turn times out, AI nominates; for bidding, incremental bids (not huge jumps) with a "desperate roster" path so late-draft $1 fills still happen when roster slots outpace budget headroom.

**Pitcher pacing:** enforces filling 9 pitching lineup slots (Yahoo-style) so teams don't end draft-heavy on hitters. Bots use a composite scoring engine for nominations (projection value × scarcity × need). Bots always auto-nominate on their turn even if the commissioner disables global auto-nominate for humans — so drafts don't stall.

### Live scoring

Poll MLB → score completed games for **starters** → apply **gear modifier engine** (caps, diminishing "all" stacks, penalties after cap) → loot rolls → coin grants (per-game cap) → persist logs for ML.

### Waiver system

Daily processing at **8 AM ET**. Dropped players sit on waivers for **2 days** before becoming free agents. Bot teams evaluate claims based on roster need, positional scarcity, and personality profile. Once approved, pickups earn fantasy points immediately.

### Client routing & mode switch

Dashboard `Outlet` is **keyed** on active fantasy team / league so **Switch Mode** remounts pages and refetches cleanly without hard refresh.

---

## Economy & balance engineering

| Flow | Mechanism |
|------|-----------|
| **Earn** | Game performance (capped), draft surplus conversion, matchup rewards |
| **Spend** | Rotating weekly shop, peer marketplace |
| **Sink** | Marketplace tax, pricing tiers |
| **Anti-abuse** | Currency caps, duplicate shop guards, price bounds, server-side validation |

**Loot:** performance triggers → rarity gates scaled by tier → bonus rolls when players beat projections (sleeper reward).

**Anti-snowball:** modifier cap, diminishing returns on stacked "all" buffs, soulbound drops, marketplace-only gear circulation, inverse tier drop bias for bench players.

---

## Operations

- **Backups:** `pg_dump`-based script inside Compose context, retention pruning, cron on the host; dumps live under the app directory (copy off-box for DR).
- **Health:** `/api/health` for load balancers and compose healthchecks.
- **Deploy:** pull → rebuild app image → Alembic migrations → rolling container restart; env-driven feature flags.
- **Scheduled jobs:** nightly roster sync (teams, names, pitcher roles from depth charts), stat recompute, and classification / training hooks as configured.

---

## Testing

**70+** automated tests (pytest) across auth, scoring, balance, matchup scheduling, projections, classifier, lineup optimizer, **draft pricing** (46 curve/bot/economy sanity tests), trades, and integration-style API flows against SQLite fixtures.

---

## Deployment

```yaml
# Simplified compose topology
services:
  db:       # PostgreSQL 16 + healthcheck
  app:      # Node build stage → Python runtime, API + static SPA
  tunnel:   # cloudflared (optional; or TLS-only reverse proxy)
```

Multi-stage **Dockerfile**: build frontend, copy `dist` into API image; **Gunicorn** fronts **Uvicorn** workers.

---

## Tech decisions & trade-offs

| Decision | Rationale |
|----------|-----------|
| **SSE vs WebSocket** | One-way push for scores/draft; mutations stay on REST |
| **In-memory draft** | Latency and timer accuracy; acceptable ephemeral state |
| **GMM + percentile tiers** | Clusters + standardized scores + cutoffs vs. naive "one cluster = one tier" |
| **Piecewise auction curves** | Position-aware, calibrated to real ppg distribution; 2.0× multiplier cap prevents runaway bids while personality variance keeps drafts non-deterministic |
| **Greedy lineup optimizer** | Fast enough for interactive use vs. exponential exact search |
| **JWT-gated reads** | Shrinks anonymous scraping / abuse surface at scale |
| **Admin key for dangerous routes** | Maintenance without exposing "hidden" power endpoints |
| **Bot personality archetypes** | 5 deterministic profiles vs. random behavior; makes AI opponents feel distinct and drafts replayable |
| **Flexible league sizes** | Commissioner-driven bot fill vs. forced 10-player roster; lowers barrier to starting a league |
| **Need-boost ramp** | Dampens early-draft inflation (when all bots need everything) so stars settle in realistic ranges instead of uniformly inflated prices |
| **MLB depth charts for pitcher roles** | API doesn't label every reliever "RP" on active rosters; depth chart gives SP/CP; generic P inferred from games-started ratio |

---

*Built with Python, TypeScript, and live MLB data. Production deployment with Docker, PostgreSQL, and Cloudflare.*
