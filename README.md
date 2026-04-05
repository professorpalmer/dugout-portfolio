# Dugout — Fantasy Baseball RPG Platform

> Full-stack fantasy baseball with RPG equipment, a player-driven coin economy, and in-house ML on real MLB data. **Live production** app behind Docker, PostgreSQL, and Cloudflare (Tunnel + edge TLS).

**[Live app](https://dugoutfantasy.com)** · Private source · [Screenshots](#screenshots) · [Overview](#overview) · [Security](#security--production-hardening) · [Architecture](#architecture) · [ML pipeline](#ml-pipeline) · [System design](#system-design-highlights) · [Economy](#economy--balance-engineering) · [Ops](#operations) · [Testing](#testing)

---

## Screenshots

### Core experience

| Landing | Mode select |
|---------|-------------|
| ![Landing](screenshots/landing.png) | ![Mode Select](screenshots/mode-select.png) |

| My Team (roster + gear summary) | Equipment detail (8 slots) |
|---------------------------------|---------------------------|
| ![My Team](screenshots/my-team.png) | ![Equipment](screenshots/equipment.png) |

### Live scoring & matchups

| Live Scores | Box score detail |
|-------------|-----------------|
| ![Live Scores](screenshots/live-scores.png) | ![Box Score](screenshots/live-boxscore.png) |

| Weekly matchup | Scoring breakdown |
|----------------|-------------------|
| ![Matchup](screenshots/matchup.png) | ![Breakdown](screenshots/matchup-breakdown.png) |

### League & standings

| League standings + season controls |
|------------------------------------|
| ![League](screenshots/league.png) |

### Gear system

| Gear locker | Item detail popup |
|-------------|-------------------|
| ![Locker](screenshots/locker.png) | ![Locker Detail](screenshots/locker-detail.png) |

| Shop & marketplace | Shop item detail |
|--------------------|-----------------|
| ![Shop](screenshots/shop.png) | ![Shop Detail](screenshots/shop-detail.png) |

| Gear catalogue (collection tracker) |
|--------------------------------------|
| ![Catalogue](screenshots/gear-catalogue.png) |

### Transactions

| Waiver wire | Drop modal (full roster) |
|-------------|-------------------------|
| ![Waivers](screenshots/waivers.png) | ![Drop](screenshots/waiver-drop.png) |

| Trade proposal builder | Outgoing trades |
|------------------------|-----------------|
| ![Trade Propose](screenshots/trade-propose.png) | ![Trade Outgoing](screenshots/trade-outgoing.png) |

### Intelligence & research

| ML projections | Lineup optimizer |
|----------------|-----------------|
| ![Projections](screenshots/projections.png) | ![Optimizer](screenshots/lineup-optimizer.png) |

| Research (team browser) | Player deep-dive (Statcast + matchup) |
|-------------------------|---------------------------------------|
| ![Research](screenshots/research.png) | ![Player](screenshots/research-player.png) |

### Settings & info

| Push notifications | How to Play guide |
|--------------------|-------------------|
| ![Notifications](screenshots/notifications.png) | ![Guide](screenshots/guide.png) |

---

## Overview

Dugout layers **RPG progression** on head-to-head fantasy baseball:

- **8 equipment slots** per player (hat, shades, chain, jersey, glove, bat, wristband, cleats) with **6 rarity tiers** (Common → Mythic).
- **Soulbound loot** from in-game performance; **marketplace** resales with tax sinks.
- **Auction draft** with **AI-assisted** nominations and bids; **solo leagues** with bot opponents and **multiplayer** leagues under one account.
- **5 bot personality archetypes** (Stars & Scrubs, Balanced, Value Hunter, Position Scarcity, Late Surge) — **persisted per bot** and used **all season**: draft bidding, waivers, trades, marketplace, lineup optimization, and gear equipping (not draft-only).
- **Flexible league sizes** — commissioners can **fill empty slots with AI managers** in pre-season so you don't need 10 humans. Play with 3 friends and 7 bots, or any mix.
- **Switch Mode** moves between Solo and League contexts with **independent rosters, gear, scores, and matchups**.
- **Live scoring** with real-time box score polling, in-game fantasy point accrual, and push notifications for player events (home runs, steals, milestones).
- **Unified transactions** surface for waivers and trades; **bot** teams participate in waivers and propose/evaluate trades based on personality and roster need.
- **Research** pulls MLB schedules and rosters with fantasy ownership context; **Statcast data** (barrel%, xBA, xSLG, matchup platoon splits) integrated into player deep-dives.
- **AI Insights** exposes **projection models** and a **lineup optimizer** (non-generative ML); in-product copy distinguishes proprietary models from LLM hype.
- **Gear catalogue** tracks collection progress across all 197 items with unlock challenges and rarity-grouped browsing.

---

## Security & production hardening

Ship-ready controls added for a public URL and real users:

| Area | Approach |
|------|----------|
| **API reads** | Data routes (`/leagues/...`, `/schedule`, marketplace, leaderboard, etc.) require **`Authorization: Bearer`** by default. Optional `DUGOUT_ALLOW_PUBLIC_READS` for debugging only. |
| **SSE** | Live score / draft streams expect a **JWT** (e.g. `?token=`) unless `DUGOUT_ALLOW_PUBLIC_SSE` is explicitly enabled. |
| **Admin / maintenance** | High-impact routes (`roster sync`, **bot pitcher backfill**, bot jobs) require **`X-Admin-Key`** matching `DUGOUT_ADMIN_API_KEY`; missing key → **404** in production. |
| **Manual scoring** | Gated behind env (`DUGOUT_ALLOW_MANUAL_SCORING`) outside dev to avoid stat injection abuse. |
| **Secrets in Compose** | Production compose passes admin key and feature flags from `.env` into the app container (not only JWT/DB). |
| **Password reset** | Email-based 6-digit code flow via Resend; rate-limited (3 req/min for send, 5/min for reset); silent success on unknown emails to prevent enumeration. |
| **Social / PWA** | Open Graph + Twitter Card meta, compressed **`og-image`**, PNG favicons + manifest icons, push notification support via VAPID/Web Push. |

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
| **Real-time** | SSE (`sse-starlette`) — draft room & live scores; Web Push (VAPID) for player event notifications |
| **Data** | MLB Stats API (rosters, depth charts, game logs, person stats, transactions), Statcast |
| **Auth** | JWT + bcrypt, email verification + password reset (Resend) |
| **Infra** | Multi-stage Docker image, **cloud VPS** + Cloudflare Tunnel |

---

## ML pipeline

### 1. Player projections (GBR)

Per-game fantasy points via **gradient boosted regression** on rolling game-log features (separate hitter/pitcher feature sets), tier-aware floors, Statcast context, and scheduled retraining as new games land.

### 2. Tier classification (GMM + production rules)

- **Pools:** Hitters, **starters (SP)**, and **relievers** (RP + depth-chart **CP** closers + inferred `P` arms where games-started ratio indicates relief). Closers compete in the same reliever pool as setup arms — elite stats surface naturally.
- **Features:** Blended **prior + current season** from `PlayerGameLog` (rate stats blended linearly; counting stats blended per-game when both seasons exist). **Minimum game log** threshold before full GMM classification; model fallback for sparse history with **sanity gates** so tiny samples don't pollute stars.
- **Scoring:** **Standardized** feature vectors per pool, **weighted composite score**, then **percentile cutoffs** (e.g. top ~10% → Star) — avoids "one cluster = 25% stars" artifacts from raw GMM cluster-to-tier mapping.
- **Fallback:** Players who can't be classified with enough data use `projected_pts_per_game` ranking; **STAR is reserved** for the main pipeline — fallback tier is capped so small-sample flukes don't show as franchise players.
- **Data:** Optional **historical season backfill** of game logs (e.g. MLB game-log API) so early-season classification isn't driven only by a handful of April games.
- Tiers still feed **loot rarity** tuning.

### 3. Draft agent & lineup optimizer

- **Draft recommendations** combine projection value, positional scarcity, roster need, and budget — with natural-language reasoning for the UI. Suggestions sort **available players** by **`projected_pts_per_game`** (and league-scoped "already drafted" filtering), not a brittle hand-tuned stat formula.
- **Bot personality engine** assigns one of **5 archetypes** per bot at draft creation (deterministic / persisted). Each archetype defines tier multipliers, bid increment ranges, timing profiles, and a "drain nomination" probability — bots will sometimes nominate stars they don't need to force opponents to overspend.
- **Dynamic bidding**: highest-valuation bot bids first (not random), personality-aware increments, and "fight" behavior where bots near their ceiling occasionally push past it.
- **AFK assistance**: when a human's turn times out, **AI nominates**; for bidding, **incremental bids** (not huge jumps) with a **"desperate roster"** path so late-draft $1 fills still happen when roster slots outpace budget headroom.
- **Lineup optimizer** assigns starters under eligibility and **roster lock** rules; respects gear modifiers in effective scoring.
- Product messaging: **in-house statistical ML**, not generative AI.

---

## System design highlights

### Real-time draft room

In-memory **auction state machine** (nomination → bidding → timers) with **SSE** fan-out. Draft persistence is intentional trade-off: sub-second bid latency vs. surviving process restarts.

**Solo / bot path:** bot nominations and counter-bids use the same valuation and **roster-cap** rules as humans; **pitcher pacing** enforces filling **9 pitching lineup slots** (Yahoo-style) so teams don't end draft-heavy on hitters. Bots use a **composite scoring engine** for nominations (projection value × scarcity × need). **Bots always auto-nominate on their turn** even if the commissioner disables global auto-nominate for humans — so drafts don't stall.

### Live scoring

60-second **MLB Stats API polling** → detect in-progress games → fetch box scores → compute fantasy points with **gear modifier engine** (caps, diminishing "all" stacks, penalties after cap) → update `FantasyLiveAccrual` rows in real-time → **SSE broadcast** to connected clients. When games go final: authoritative scoring pass → loot rolls → coin grants → persist game logs for ML retraining. **Gear snapshots** frozen at game start so mid-game equipment changes don't affect scoring.

### Waiver system

Daily processing at **8 AM ET**. Dropped players sit on waivers for **2 days** before becoming free agents. Bot teams evaluate claims based on roster need, positional scarcity, and personality profile. **Full roster detection** proactively prompts a drop modal; search results show projected points, injury status, and ownership.

### Bot trade engine

**League-wide rolling weekly cap** scales with the season: 1 proposal/week early → 2 mid-season → 3 near playoffs. Per-bot attempt chances are intentionally low (6–14%) so proposals trickle in organically. Late Surge bots ramp aggression as playoffs approach. Target: ~15 proposals and 4–5 accepted trades per season.

### Client routing & mode switch

Dashboard **`Outlet`** is **keyed** on active fantasy team / league so **Switch Mode** remounts pages and refetches cleanly without hard refresh.

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

- **Backups:** `pg_dump`-based script inside Compose context, **retention** pruning, **cron** on the host; dumps live under the app directory (copy off-box for DR).
- **Health:** `/api/health` for load balancers and compose healthchecks.
- **Deploy:** pull → rebuild app image → **Alembic migrations** → rolling container restart; env-driven feature flags.
- **Scheduled jobs:** 60s MLB poll, nightly roster sync (teams, names, **pitcher roles** from depth charts), stat corrections, bot economy ticks (3×/day), waiver processing, MLB transaction monitoring, game reminders, and classification / training hooks.

---

## Testing

**318** automated tests (pytest) across auth, scoring, balance, matchup scheduling, projections, classifier, lineup optimizer, **draft pricing** (46 curve/bot/economy sanity tests), trades, IL management, live accrual reconciliation, and integration-style API flows against SQLite fixtures.

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
| **Greedy lineup optimizer** | Fast enough for interactive use vs. exponential exact search |
| **JWT-gated reads** | Shrinks anonymous scraping / abuse surface at scale |
| **Admin key for dangerous routes** | Maintenance without exposing "hidden" power endpoints |
| **Bot personality archetypes** | 5 deterministic profiles vs. random behavior; makes AI opponents feel distinct and drafts replayable |
| **Flexible league sizes** | Commissioner-driven bot fill vs. forced 10-player roster; lowers barrier to starting a league |
| **Gear snapshot at game start** | Prevents mid-game equipment swaps from retroactively changing live accrual; frozen state replayed on final scoring |
| **Rolling weekly trade cap** | Season-phase-aware throttle vs. flat cooldown; bots feel more desperate near playoffs without flooding early |
| **MLB depth charts for pitcher roles** | API doesn't label every reliever "RP" on active rosters; depth chart gives SP/CP; generic `P` inferred from games-started ratio |

---

*Built with Python, TypeScript, and live MLB data. Production deployment with Docker, PostgreSQL, and Cloudflare.*
