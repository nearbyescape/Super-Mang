# Lighter.xyz 24x7 Auto-Trading Bot — Architecture & Delivery Plan

> Status: **PLAN / for review** — no code yet. Decisions marked **[DECIDE]** need owner input before build starts.

---

## 0. The most important design decision: what "weekly target" means

A trading bot **cannot guarantee** a profit target. Markets decide the outcome; the bot only decides *how much risk* it takes. The single most common way automated accounts blow up is a "target" logic that **increases size or leverage when behind target** (martingale behaviour). This plan explicitly forbids that.

The bot supports two target modes **[DECIDE which one, or both]**:

| Mode | Target example | Can the bot "fulfil" it? | Main risk |
|---|---|---|---|
| **A. Volume target** (e.g. points / tier farming) | "$2M traded volume per week" | Yes, mostly — volume is controllable | Paying spread/slippage to buy volume; cost per $1M volume must be capped |
| **B. Profit (PnL) target** | "+2% equity per week" | **No guarantee** — it is a goal, not a promise | Chasing the target → over-trading and drawdown |

### Target Engine rules (non-negotiable)
1. **Asymmetric pacing** — when *ahead* of pace, risk is **reduced**; when *behind*, risk stays at the **base level, never above it**.
2. **Lock-in** — once the weekly PnL target is hit, the bot switches to *protect mode* (flat or minimum size) until the week resets.
3. **Weekly loss limit** (e.g. −3% equity) → hard stop until next week + human sign-off to resume.
4. **Daily loss limit** (e.g. −1%) → pause for the rest of the UTC/IST day.
5. Volume mode: hourly volume budget = remaining volume ÷ remaining hours, with a **max cost per $1M volume** cap; if cost exceeds cap, the bot slows down instead of crossing the spread.

Week boundary: Monday 00:00 IST (configurable).

---

## 1. Platform facts that shape the design (Lighter)

- Order-book perpetuals (and spot) DEX on a ZK-rollup; transactions are **signed with an API key** (not the wallet key) and sent via `sendTx` / `sendTxBatch`.
- Official SDKs: **Python** (`elliottech/lighter-python`) and **Go** (`elliottech/lighter-go`). REST + WebSocket.
- **Nonce per API key**, must increase by 1 each tx → we need a robust nonce manager (resync on rejection). Multiple API keys per account are allowed (e.g. one key for quoting, one for cancels/stops) to avoid nonce contention.
- **Account tiers** (from Lighter rate-limit docs — re-verify at build time):

| | Standard | Premium |
|---|---|---|
| Trading fees | 0% maker / 0% taker | small fees (≈0.2 bps maker / ≈2–2.8 bps taker; falls with staked LIT) |
| REST | 60 req / min | 24,000 weighted / min |
| `sendTx` bucket | 60 / min | 4,000+ / min |
| Active orders | 250 / acct, 30 / market | 1,500 / acct, 1,000 / market |
| Latency | slower | lower, better cancel/post-only behaviour |

**Implication:** On *Standard* (zero fees) the bot is limited to ~1 tx/second → suits **low-frequency strategies** (minutes–hours). Market-making / high-frequency volume generation realistically needs **Premium**, and then fees enter the PnL maths. **[DECIDE] account tier.**

- Use a **dedicated sub-account** for the bot so capital at risk is isolated from the main wallet.
- Withdrawals are rate-limited (2/min, 3 per 2h) — not a design issue, but noted for treasury ops.

---

## 2. Strategy plan (full spec: [STRATEGY_PLAN.md](./STRATEGY_PLAN.md))

We start with **simple, testable, low-frequency** strategies and only add complexity when data proves it pays.

| # | Strategy | Fits mode | Frequency | Phase |
|---|---|---|---|---|
| S1 | **Trend-following** (EMA/Donchian breakout on 15m–1h, ATR stops, trailing exit) on BTC/ETH/SOL perps | PnL | Low | MVP |
| S2 | **Mean-reversion** (Bollinger/RSI fade) — only active when the regime filter says "ranging" | PnL | Low–Med | MVP |
| S3 | **Regime filter** (ADX + realised volatility) that switches S1 ↔ S2 and stands aside in chaotic markets | both | — | MVP |
| S4 | **Passive maker / inventory-skewed quoting** for volume with tight inventory limits | Volume | High (Premium) | Phase 2 |
| S5 | **Funding-rate harvesting** (delta-neutral: Lighter perp vs. hedge venue) | PnL | Low | Phase 3 |

Each strategy is a plug-in implementing one interface (`on_bar`, `on_book`, `on_fill`, `target_position()`); the strategy only proposes a **target position**, it never sends orders directly. Execution + Risk own the order flow.

---

## 3. System architecture

```
                ┌───────────────────────── Lighter ─────────────────────────┐
                │   WebSocket (books, trades, account)     REST / sendTx     │
                └───────────┬───────────────────────────────▲────────────────┘
                            │                               │
                 ┌──────────▼──────────┐          ┌─────────┴──────────┐
                 │  Market Data Svc    │          │  Execution / OMS   │
                 │  books, trades,     │          │  signer, nonce mgr,│
                 │  candles, funding   │          │  rate limiter,     │
                 └──────────┬──────────┘          │  order state mach. │
                            │                     └─────────▲──────────┘
             ┌──────────────▼──────────────┐                │ approved orders
             │       Strategy Engine        │──proposals──►┌─┴──────────────┐
             │  S1..S5 + regime filter      │              │  Risk Manager  │
             └──────────────▲──────────────┘              │  pre-trade gates│
                            │ risk budget                  │  kill switch    │
             ┌──────────────┴──────────────┐              └──▲─────────────┘
             │       Target / Pacing        │◄── PnL/volume ──┘
             │  weekly → daily → hourly     │
             └──────────────────────────────┘
   Persistence: Postgres + TimescaleDB (orders, fills, PnL, candles)   Redis (hot state)
   Ops: Prometheus + Grafana, Telegram alerts & commands, Admin dashboard (FastAPI + Next.js)
```

### Components
1. **Market Data Service** — WS subscriptions, local order book with sequence checks, candle builder, stale-data detector (no update for N seconds → strategies frozen).
2. **Account Service** — positions, balances, margin, open orders from WS; **REST reconciliation every 30–60 s** and on every reconnect (source of truth = exchange).
3. **Execution / OMS** — signing via SDK, nonce manager, token-bucket rate limiter matching the account tier, client order IDs for idempotency, order state machine (`NEW → ACK → PARTIAL → FILLED/CANCELLED/REJECTED`), retry with backoff only for safe errors.
4. **Risk Manager** — every order passes these gates:
   - max leverage (start 2–3x), max notional per market & total
   - risk per trade 0.25–0.5% of equity (position sized from stop distance)
   - liquidation buffer (distance to liq price ≥ X× ATR)
   - price-band check vs. mark/index (fat-finger protection)
   - max orders/min, max consecutive losses → cool-down
   - daily / weekly loss limits from the Target Engine
   - **Exchange-side reduce-only stop-loss on every position**, so positions stay protected even if the bot dies
5. **Target / Pacing Engine** — section 0 logic; outputs a *risk multiplier* in `[0, 1]` (never > 1).
6. **Supervisor / Watchdog** — heartbeat, auto-restart, on fatal error: cancel all → (optionally) flatten → alert. Manual **kill switch** via Telegram `/kill` and dashboard.
7. **Persistence** — every signal, order, fill, and config change is stored (audit trail + analytics).
8. **Ops & UI** — Grafana (equity curve, PnL vs. target, latency, error rates), Telegram (`/status`, `/pause`, `/resume`, `/kill`, `/target`), web dashboard later.

---

## 4. 24x7 reliability checklist

- Run on a cloud VPS close to Lighter's infrastructure (measure latency from 2–3 regions first).
- Docker Compose + `restart: always`; health-check endpoint; process supervisor.
- WebSocket auto-reconnect with exponential backoff + full state reconciliation after reconnect.
- Startup procedure: load state → reconcile with exchange → cancel orphan orders → resume.
- Nonce resync on any nonce error; separate API keys per function.
- Clock sync (chrony), structured JSON logs, log rotation.
- Alerting: disconnect > 60 s, reconciliation mismatch, loss-limit hit, margin ratio warning, error spike.
- Weekly scheduled maintenance window (target engine reset + report).

## 5. Security

- **Wallet (L1) private key never on the server.** Only the Lighter API key lives there (env/secret manager, not in git).
- Dedicated sub-account with only the capital we are willing to risk.
- Treasury sweeps (profit withdrawal) done manually by the owner.
- Telegram commands restricted to whitelisted user IDs; dashboard behind auth + 2FA.
- Dependency pinning, secret scanning in CI.

---

## 6. Tech stack

| Layer | Choice | Why |
|---|---|---|
| Core bot | **Python 3.12 + asyncio (uvloop)** | Official SDK, fast to iterate, strong quant ecosystem |
| Models / config | pydantic, YAML config per environment | Typed, validated configs |
| Storage | PostgreSQL + TimescaleDB, Redis | Time-series fills/candles, hot state |
| Backtesting | own event-driven backtester (same strategy code) + vectorised research in pandas/polars | No code divergence between backtest & live |
| Monitoring | Prometheus + Grafana, Telegram bot | Industry standard |
| Dashboard (Phase 3) | FastAPI + Next.js | Modern UI consistent with NearbyEscape stack direction |
| Deploy | Docker Compose → GitHub Actions CI (lint, type-check, tests) | Simple, reproducible |
| Hot path upgrade (only if needed) | Go (`lighter-go`) execution service | If Python latency becomes the bottleneck for maker strategy |

### Proposed repo layout
```
bot/
  config/          # default.yaml, testnet.yaml, mainnet.yaml
  core/            # events, clock, models
  exchange/        # lighter client wrapper, ws, signer, nonce, rate limiter
  data/            # market data service, candle builder, recorder
  strategies/      # base.py, trend.py, mean_revert.py, regime.py, maker.py
  risk/            # risk manager, limits, kill switch
  target/          # weekly target & pacing engine
  oms/             # order manager, state machine, reconciliation
  storage/         # db models, repositories
  ops/             # telegram, metrics, health
  backtest/        # engine, fill simulator, reports
tests/
docker/
docs/
```

---

## 7. Delivery roadmap (≈ 10–12 weeks to confident mainnet)

| Phase | Weeks | Deliverables | Exit criteria |
|---|---|---|---|
| **0. Decisions** | 0.5 | Target mode, tier, capital, markets, risk limits signed off | This doc approved |
| **1. Foundation** | 2 | Lighter client wrapper, WS data, OMS, nonce/rate limiter, data recorder, Telegram status — on **testnet** | 72 h continuous testnet run, zero orphan orders, clean reconciliations |
| **2. Research & backtest** | 2–3 | Backtester, S1–S3 implemented, walk-forward tests on 1–2 yrs of data incl. fees/slippage/funding | Positive expectancy out-of-sample; max DD within limits; no curve-fitting red flags |
| **3. Risk + Target engine + paper** | 2 | Risk manager, target/pacing engine, kill switch, exchange-side stops, Grafana | 2 weeks paper/testnet 24x7; all failure drills passed (kill WS, kill process, network cut) |
| **4. Mainnet pilot** | 2–4 | Live with **small capital** (e.g. 5–10% of planned size) | Live results within tolerance of paper; no operational incidents |
| **5. Scale & extend** | ongoing | Scale capital stepwise, S4 maker (Premium), S5 funding, dashboard | Monthly review gates |

### Failure drills before going live
Kill the process mid-position · drop the network for 5 min · exchange returns 429s · nonce desync · stale order book · sudden 10% wick · margin call scenario. Each must end in a safe, known state.

---

## 8. KPIs & weekly report

Sent automatically every Monday: PnL (abs & %), vs. target, volume, fees paid, funding paid/received, win rate, profit factor, max drawdown, Sharpe (rolling), number of risk-limit triggers, uptime %, incidents.

---

## 9. Open questions **[DECIDE]**

1. Target type: **volume**, **profit %**, or both? What number per week?
2. Starting capital on the bot sub-account and maximum acceptable weekly loss?
3. Account tier: Standard (zero fee, slow) or Premium (fees, fast)?
4. Markets: BTC/ETH only to start, or include SOL/others?
5. Hosting preference (AWS / GCP / Hetzner) and who receives alerts?
6. Do you already hold a funded Lighter account and API key for testnet?
