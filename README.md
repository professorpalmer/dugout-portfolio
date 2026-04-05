# Dugout — Fantasy Baseball RPG Platform

> Full-stack fantasy baseball with RPG equipment, a player-driven coin economy, and in-house ML on real MLB data. **Live production** app behind Docker, PostgreSQL, and Cloudflare (Tunnel + edge TLS).

**[Live app](https://dugoutfantasy.com)** · Private source · [Screenshots](SCREENSHOTS.md) · [Overview](#overview) · [Security](#security--production-hardening) · [Architecture](#architecture) · [ML pipeline](#ml-pipeline) · [System design](#system-design-highlights) · [Economy](#economy--balance-engineering) · [Ops](#operations) · [Testing](#testing)

---

## Screenshots

| My Team | Live Matchup | Equipment | Shop |
|---------|-------------|-----------|------|
| ![My Team](screenshots/my-team.png) | ![Matchup](screenshots/matchup-breakdown.png) | ![Equipment](screenshots/equipment.png) | ![Shop](screenshots/shop-detail.png) |

**[View all 24 screenshots →](SCREENSHOTS.md)** — landing, live scores, box scores, matchups, scoring breakdowns, league standings, gear locker, shop, catalogue, waivers, trades, projections, lineup optimizer, research, player deep-dive, notifications, and more.

---

## Overview

Dugout layers **RPG progression** on head-to-head fantasy baseball:

- **8 equipment slots** per player (hat, shades, chain, jersey, glove, bat, wristband, cleats) with **6 rarity tiers** (Common → Mythic).
- **~200 gear items** with challenge-based unlock triggers, performance-gated rarity rolls, and soulbound loot; **marketplace** resales with tax sinks.
- **Auction draft** with an **AI advisor panel** (real-time bid/let-go recommendations, natural-language reasoning) and **bot opponents** that bid with distinct personalities.
- **5 bot personality archetypes** (Stars & Scrubs, Balanced, Value Hunter, Position Scarcity, Late Surge) — **persisted per bot** with **16 tunable knobs** per archetype, used **all season**: draft bidding, waivers, trades, marketplace, lineup optimization, and gear equipping.
- **Flexible league sizes** — commissioners can **fill empty slots with AI managers** in pre-season. Play with 3 friends and 7 bots, or any mix.
- **Switch Mode** moves between Solo and League contexts with **independent rosters, gear, scores, and matchups**.
- **Live scoring** with 60-second box score polling, in-game fantasy point accrual with **gear snapshot freezing**, push notifications for player events (HRs, steals, K milestones, gear proximity hints).
- **Play-by-play enrichment** — fielding credits (OF catches, double plays), ABS challenge tracking, trailing/go-ahead HR detection, foul ball counting from Statcast.
- **Unified transactions** for waivers and trades; bots participate in both with personality-driven decision making and season-phase-aware trade throttling.
- **Research** with MLB schedules, rosters, and **Statcast analytics** (barrel%, xBA, xSLG, xwOBA, platoon splits, park factors) in player deep-dives.
- **Quantile regression projections** (p10/p50/p90 ranges) with **32 hitter / 20 pitcher features**, Statcast integration, matchup context, and confidence scoring.
- **Gear catalogue** tracks collection progress across all ~200 items with challenge conditions and rarity-grouped browsing.

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
│  │          └──────────────────┘  │ GBR quantile,      │   │  │
│  │                                │ GMM, draft agent,  │   │  │
│  │                                │ lineup optimizer   │   │  │
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
| **ML** | scikit-learn (`HistGradientBoostingRegressor`, GMM), NumPy, pandas, joblib |
| **Real-time** | SSE (`sse-starlette`) — draft room & live scores; Web Push (VAPID) for player event notifications |
| **Data** | MLB Stats API (rosters, depth charts, game logs, play-by-play, transactions), Statcast (pybaseball) |
| **Auth** | JWT + bcrypt, email verification + password reset (Resend) |
| **Infra** | Multi-stage Docker image, **cloud VPS** + Cloudflare Tunnel |

---

## ML pipeline

### 1. Player projections — quantile gradient boosting

Three **`HistGradientBoostingRegressor`** models per player type (hitter/pitcher) produce **p10, p50, p90** projection ranges — not just a point estimate.

**Hitter feature set (32 features):**
- **17 box-score rolling:** 5-game and 15-game rolling averages for hits, HR, RBI, runs, SB, walks, doubles, fantasy points; season AVG/OBP/SLG; hit streak length; games played
- **11 Statcast:** xBA, xSLG, xwOBA, BA/SLG/wOBA differentials (expected vs actual), avg exit velocity, barrel%, hard-hit%, sweet-spot%, avg launch angle
- **4 matchup context:** park factor, platoon advantage flag, opposing pitcher xERA, opposing pitcher xwOBA

**Pitcher feature set (20 features):**
- **12 box-score rolling:** 5g/15g rolling IP, K, ER, hits allowed, fantasy pts; season ERA/WHIP; games played
- **7 Statcast:** xBA/xSLG/xwOBA against, xERA, avg exit velo against, barrel% against, hard-hit% against
- **1 matchup:** park factor

**Key design decisions:**
- **No-leak training:** Sliding window ensures features only come from games *before* the target game
- **Tier baseline floor:** Stars can't project below 40% of their stats-based estimate — prevents model from sandbagging elite players with small recent samples
- **League scoring ratio:** Rescales predictions by (league-weights / default-weights) so custom scoring leagues get adjusted projections automatically
- **Heuristic fallback:** When <5 game logs or no trained model, blends tier baseline + season stats + recent trend
- **Confidence scoring:** Composite of tier bonus, Statcast data availability, stat depth, game log count, and prediction consistency (inverse variance across quantiles)
- **Weekly scaling:** Multiplies per-game projection by estimated weekly appearances (hitters: team scheduled games; SP: ~1.2/week; RP: ~60% of team games)
- **Matchup resolver:** Looks up today's opposing pitcher, computes platoon advantage (+1.0 opposite hand, −0.5 same, +0.3 switch hitter)
- **Auto-retraining:** Models retrain via joblib persistence after every batch of completed games

### 2. Tier classification

Role-aware **absolute thresholds** on `projected_pts_per_game` with separate cutoffs per pool:

| Role | Star | Starter | Platoon | Bench |
|------|------|---------|---------|-------|
| Hitter | ≥ 7.0 | ≥ 4.5 | ≥ 2.5 | < 2.5 |
| SP | ≥ 14.0 | ≥ 8.0 | ≥ 4.0 | < 4.0 |
| RP | ≥ 6.0 | ≥ 3.5 | ≥ 2.0 | < 2.0 |

**Guards:** Small-sample cap (HR<5, RBI<15 → can't be Star/Starter); everyday floor (120+ games → can't fall below Platoon). Tiers feed loot rarity gates, draft auction valuations, and projection confidence.

### 3. Auction pricing engine

Position-aware **piecewise linear curves** map `projected_pts_per_game` to draft dollars, calibrated against the production database's actual ppg distribution:

- **Hitter curve** (weekly = ppg × 6.5): four segments from bench ($1) through generational ($60+) with a 35% budget cap
- **Pitcher curve** (weekly = ppg × 2.0): four segments tuned so ace SPs ($25–30) never outbid elite everyday hitters
- **Closer premium:** relievers receive a tier-based floor ($10–18) despite low ppg from 1 IP/game
- **Multiplier cap** (2.0×) prevents any single bot modifier chain from producing runaway bids

Designed so: Ohtani settles ~$55–90, star hitters ~$25–55, ace SPs ~$20–40, mid-rotation ~$5–15, bench ~$1–5.

### 4. Draft AI advisor

Real-time **bid / let-go** recommendations during the auction:

- Combines **auction value**, **positional scarcity**, **roster need**, and **budget advantage** into a composite score
- Four action tiers: **Must Have**, **Bid Aggressively**, **Bid Moderately**, **Let Go** — with natural-language reasoning
- Local bid tracking: when the price rises past the recommended ceiling, the panel updates instantly (no API round-trip)
- Uses the same pricing curves and projections as bot valuations, so advisor recommendations align with the market bots create

### 5. Lineup optimizer

**Constrained greedy assignment** (two-pass): fill most-constrained positional slots first (fewest eligible candidates), then flex/utility with best remaining. Uses projection engine for (per-game pts × games this week × gear boost fraction). Same diminishing-returns and positive-cap pipeline as live scoring for gear. Provides natural-language START/BENCH reasoning per player. Respects roster locks (won't move players whose MLB games have started).

Product messaging: **in-house statistical ML**, not generative AI.

---

## System design highlights

### Real-time draft room

In-memory **auction state machine** (nomination → bidding → timers) with **SSE** fan-out. Draft persistence is intentional trade-off: sub-second bid latency vs. surviving process restarts.

**Bot bidding engine:** 5 personality archetypes with per-archetype knobs (tier multipliers, bid increments, timing profiles, drain nomination probability). Highest-valuation bot bids first; personality-aware increments; "fight" behavior where bots near their ceiling occasionally push 5–20% beyond. **Pass chance** (0–50%) creates natural "steal" windows. **Position-diversity nomination** tracks a sliding window of last 5 picks — if 4+ pitchers, forces a hitter and vice versa.

**AFK assistance:** after 10 picks without activity, AI nominates via the draft agent; bidding uses incremental raises with a "desperate roster" path so late-draft $1 fills still happen when roster slots outpace budget headroom.

**Pitcher pacing:** enforces filling 9 pitching lineup slots (Yahoo-style) so teams don't end draft-heavy on hitters. Bots always auto-nominate on their turn even if the commissioner disables global auto-nominate for humans — so drafts don't stall.

### Live scoring pipeline

60-second **MLB Stats API polling** → detect in-progress games → fetch box scores → compute fantasy points with **gear modifier engine** (caps, diminishing "all" stacks, penalties after cap) → update `FantasyLiveAccrual` rows with delta tracking → **SSE broadcast** to connected clients.

**Gear snapshots** are frozen at first accrual so mid-game equipment swaps don't retroactively change scoring. When games go final: reverse all live accruals → apply authoritative final scoring with **Statcast enrichment** (foul balls, barrel data, pitch speeds) → **play-by-play parsing** (fielding credits, ABS challenges, trailing HR detection) → loot rolls → coin grants → persist `PlayerGameLog` for ML retraining.

**Stale accrual reconciliation** runs every tick — cleans up phantom rows from postponed/cancelled games, zeroes orphaned accruals from interrupted deploys.

### Live push notifications

Event detection runs inside the accrual loop: HRs (including multi-HR games), triples, multi-hit games (3/4/5+), stolen bases, RBI milestones (3+/5+), K milestones (8/10/12+), wins, saves, quality starts. **Gear proximity hints** tell users they're close to unlocking specific items ("1 more K for Nolan Ryan Rancher"). Rate-limited to 1 notification per user+player+game per 5 minutes.

### Waiver system

Daily processing at **8 AM ET**. Dropped players sit on waivers for **2 days** before becoming free agents. Bot waiver engine runs in phases: replace injured starters → fill empty slots by positional need → positional rebalancing → upgrade worst bench player. Tier-aware drop protection (Stars never droppable; Starters require 3× projection margin). Value Hunter bots stash IL-eligible Stars/Starters.

### Bot trade engine

**League-wide rolling 7-day cap** scales with the season: 1 proposal/week early (>10 weeks to playoffs) → 2 mid-season → 3 near playoffs (≤4 weeks). Per-bot attempt chances are low (6–14%) so proposals trickle in organically. Trade matching finds same-position-bucket players within the personality's max projection gap; 60/40 bias toward best trade vs. random. Incoming trade acceptance uses personality-specific `accept_min_ratio` and `accept_min_surplus` thresholds with injury discount. Late Surge bots ramp via a multiplier (1.0× → 1.6×) as playoffs approach.

### MLB transaction monitoring

Every 15 minutes: poll MLB transactions API for injuries, DFA, trades, activations. Auto-updates player `injury_status` and pushes notifications to roster owners ("Justin Verlander placed on 15-day IL").

### Client routing & mode switch

Dashboard **`Outlet`** is **keyed** on active fantasy team / league so **Switch Mode** remounts pages and refetches cleanly without hard refresh.

---

## Economy & balance engineering

| Flow | Mechanism |
|------|-----------|
| **Earn** | Game performance (capped at 50/game, 400/week), draft surplus conversion, matchup rewards |
| **Spend** | Rotating weekly shop (8 items), peer marketplace |
| **Sink** | Marketplace tax, pricing tiers, salvage exchange rates |
| **Anti-abuse** | Currency caps, duplicate shop guards, price bounds, server-side validation |

### Loot & gear drops

**~200 gear templates** across 8 slots and 6 rarity tiers. Two drop paths:

1. **Challenge triggers:** specific stat lines unlock specific items (e.g. "3+ hits, 0 walks" → Molitor's Iron Band). ABS challenges, trailing HRs, fielding plays, day/night games, and doubleheaders all factor into trigger conditions.
2. **Rarity rolls:** when a player exceeds their projection (threshold varies by tier: bench 1.15×, star 1.30×), a weighted random roll picks a rarity. **Tier-weighted rarity** gives bench/platoon players lower thresholds and an uncommon floor (never roll Common from projection beats).

**Streak triggers** span multiple games: 10+ hit streak, 7/15/20 consecutive starts, 5+ consecutive SB without CS, 3+ weekly saves, 5-game HR streak, 3+ consecutive matchup wins.

**Mythic items** (0.5% base weight): Babe Ruth's Called Shot Bat (490+ ft HR), Billy Hamilton's 1890 Cleats (4+ SB), Jeter's Flip Glove (SS, 5+ assists in a win), Gibson's 1968 Visor (8+ IP, 0 ER, 10+ K), and more. **Event-awarded gear** can be admin-granted for historic real-world moments (soulbound, non-tradeable).

### Anti-snowball mechanics

- **60% positive boost cap** per player; penalties uncapped (risk stays real)
- **Diminishing "all" stacking:** successive all-category modifiers apply at [1.0, 0.75, 0.50, 0.35, 0.25, 0.20, 0.15, 0.10] effectiveness
- **Gear fatigue:** each player can only produce 5 challenge drops per season per user/league
- **Season rarity ceiling:** days 0–14 max Rare; 14–42 max Epic; 42–70 max Legendary; 70+ Mythic unlocked
- **Diminishing daily drops:** 1st drop today = 100%, 2nd = 50%, 3rd = 25%, 4th+ = 10%
- **Daily-use gear lock:** prevents swapping gear between players on the same calendar day
- **Soulbound loot** from performance; marketplace-only circulation for purchased gear; inverse tier drop bias for bench players
- **Foul ball scoring** exists only via gear — base weight is 0.0, but specific equipment activates the category (bridges Statcast enrichment and gear value)

---

## Statcast integration

**Season-level aggregates** (refreshed nightly via pybaseball):
- **Hitter:** xBA, xSLG, xwOBA, BA/SLG/wOBA differentials (expected vs actual), avg exit velo, max exit velo, barrel%, hard-hit%, sweet-spot%, avg launch angle, avg distance
- **Pitcher:** xBA/xSLG/xwOBA against, xERA, avg exit velo against, barrel% against, hard-hit% against
- **30 park factors** (FanGraphs consensus) indexed to 1.0 neutral

**Per-game enrichment** during final scoring:
- **Batter:** max exit velo, max HR distance, max pitch speed faced, estimated foul balls (from pitch descriptions), total pitches seen, barrel count, hard-hit count
- **Pitcher:** max/avg pitch speed, total pitches thrown, whiff count, called strike count

Statcast data feeds into projections (11 hitter / 7 pitcher features), the research player deep-dive UI, and gear trigger conditions.

---

## Operations

- **Backups:** `pg_dump`-based script inside Compose context, **retention** pruning, **cron** on the host; dumps live under the app directory (copy off-box for DR).
- **Health:** `/api/health` for load balancers and compose healthchecks.
- **Deploy:** pull → rebuild app image → rolling container restart; env-driven feature flags.
- **Scheduled jobs:**

| Job | Frequency | Purpose |
|-----|-----------|---------|
| MLB poll | 60 seconds | Schedule, live accrual, final scoring, SSE broadcasts |
| Stat corrections | Daily 8 AM ET | Re-fetch yesterday's box scores for MLB corrections |
| Roster sync | Daily 5 AM ET | Refresh player teams/names, Statcast cache, reclassify, retrain |
| Weekly advance | Monday 6 AM ET | Finalize week, generate new matchups, advance playoffs |
| Bot management | Daily 10 AM ET | Auto-lineup, gear management, bot waivers + trades |
| Waiver processing | Daily 8 AM ET | Per-league waiver claims + post-waiver bot lineup optimize |
| Bot economy | 3× daily | Trade inbox resolution + marketplace buy/sell |
| Game reminders | 4× daily | Push notifications for upcoming player games |
| MLB transactions | Every 15 min | Injury/DFA/trade updates, pushes to roster owners |

---

## Testing

**318** automated tests (pytest) across auth, scoring, balance, matchup scheduling, projections, classifier, lineup optimizer, **draft pricing** (46 curve/bot/economy sanity tests), trades, IL management, live accrual reconciliation, gear triggers, and integration-style API flows against SQLite fixtures.

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
| **Quantile regression (p10/p50/p90)** | Projection ranges let the UI show confidence intervals and the optimizer account for variance |
| **32 hitter / 20 pitcher features** | Enough signal from rolling stats + Statcast + matchup context without overfitting on ~150 games of training data |
| **Greedy lineup optimizer** | Fast enough for interactive use vs. exponential exact search |
| **JWT-gated reads** | Shrinks anonymous scraping / abuse surface at scale |
| **Admin key for dangerous routes** | Maintenance without exposing "hidden" power endpoints |
| **Bot personality archetypes** | 5 deterministic profiles (16 knobs each) vs. random behavior; makes AI opponents feel distinct all season |
| **Flexible league sizes** | Commissioner-driven bot fill vs. forced 10-player roster; lowers barrier to starting a league |
| **Gear snapshot at game start** | Prevents mid-game equipment swaps from retroactively changing live accrual; frozen state replayed on final scoring |
| **Rolling weekly trade cap** | Season-phase-aware throttle (1→2→3/week) vs. flat cooldown; bots feel more desperate near playoffs without flooding early |
| **Foul balls as gear-only scoring** | Base weight = 0.0 but gear activates the category; bridges Statcast enrichment and equipment value without inflating base scoring |
| **Season rarity ceiling + gear fatigue** | Prevents early-season Legendary/Mythic stockpiling and excessive drops from one hot player |
| **MLB depth charts for pitcher roles** | API doesn't label every reliever "RP" on active rosters; depth chart gives SP/CP; generic `P` inferred from games-started ratio |

---

*Built with Python, TypeScript, and live MLB data. Production deployment with Docker, PostgreSQL, and Cloudflare.*
