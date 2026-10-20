# Lighter.xyz Bot — Trading Strategy Specification (v1)

> Companion to [`TRADING_BOT_PLAN.md`](./TRADING_BOT_PLAN.md). All numbers below are **starting parameters** to be validated and tuned in backtests (§8), not promises of returns.
>
> **Assumption:** the weekly target is a **profit (PnL %) target**. A volume-target engine is specified as an optional module (§6) and can be switched on independently.

---

## 1. Strategy philosophy

1. **No single strategy works in all markets.** The bot runs a small *portfolio* of uncorrelated strategies and a **regime filter** decides which ones may trade.
2. **Fully invested, never over-leveraged.** 100% of the fund is put to work at controlled leverage (default 1×). Every position has an exchange-side stop from the moment it opens.
3. **The target controls risk downwards only.** When behind target the bot trades normally. When ahead it trades smaller. It never doubles up.
4. **Maker-first execution.** Post-only orders avoid paying the spread; taker orders are used only when a signal is about to be missed.
5. **The weekly target is set from the data.** It is calibrated from walk-forward results (§8.5), not chosen first and forced onto the market.

---

## 2. Trading universe

| Market | Role | Why |
|---|---|---|
| **BTC-PERP** | Core | Deepest liquidity, tightest spread |
| **ETH-PERP** | Core | High liquidity, slightly higher volatility |
| **SOL-PERP** | Satellite | More trend-prone and more volatile; half the risk of the core markets |

**Eligibility filter, re-checked hourly:** 24h volume ≥ configured minimum, median spread ≤ 3 bps, order-book depth within ±0.5% ≥ 10× our max order size. A market that fails is set to *exit-only*.

Further markets are added only after they pass the same backtest gates.

---

## 3. Layer 1 — Market Regime Detector

This runs per market on closed **1h bars**. Its output is one of four regimes, which turns strategies on or off.

| Input | Definition |
|---|---|
| `ADX` | ADX(14) on 1h |
| `ER` | Kaufman Efficiency Ratio over 20 bars = \|close − close₂₀\| / Σ\|Δclose\| |
| `VOLP` | Percentile of ATR(14)/price against the trailing 30 days |
| `SHOCK` | Last 1h range > 3 × ATR(14) |

| Regime | Rule | Strategies allowed |
|---|---|---|
| **TREND** | ADX > 25 **and** ER > 0.35 | S1 Trend |
| **RANGE** | ADX < 20 **and** ER < 0.25 **and** 20 ≤ VOLP ≤ 80 | S2 Mean-Reversion, S4 Maker (if on) |
| **CHAOS** | VOLP > 95 **or** SHOCK | **None** — no new entries; existing stops tighten to 1.5 × ATR |
| **NEUTRAL** | everything else | Existing positions managed; no new entries in MVP |

**Hysteresis:** a new regime must hold for **2 consecutive bars** before it takes effect. This stops the regime flipping back and forth around a threshold. The exception is CHAOS, which takes effect immediately.

---

## 4. Layer 2 — Alpha strategies

### S1 · Trend Breakout (TREND regime)

| Item | Rule |
|---|---|
| Timeframes | Signal 1h; direction filter 4h |
| Direction filter | Long only if 4h EMA50 > EMA200; short only if EMA50 < EMA200 |
| Entry | 1h close above the 20-bar Donchian high (long) or below the 20-bar low (short) |
| Funding filter | Skip the entry if hourly funding annualises above 50% against the trade direction |
| Initial stop | Entry ∓ **2.0 × ATR(14, 1h)**, placed on the exchange as a reduce-only stop-loss |
| Take-profit | **1/3 of the position at +2R**, after which the stop moves to breakeven |
| Trailing exit | Chandelier stop at 3.0 × ATR from the highest high (long) or lowest low (short), updated on bar close |
| Time stop | Exit if the trade is still under +0.5R after 48 bars |
| Re-entry | None in the same direction until a new Donchian breakout |
| Size | Full-deployment allocation (§5). At 1×, a stopped trade costs ≈ 1–2% of equity |

Expected profile: win rate around 35–45%, average win much larger than average loss, and long losing streaks. This is normal for trend strategies.

### S2 · Range Mean-Reversion (RANGE regime only)

| Item | Rule |
|---|---|
| Timeframe | 15m |
| Fair value | Rolling 15m VWAP over the last 4 hours (16 bars) |
| Signal | z = (price − VWAP) / stdev(16). **Long** when z ≤ −2.0 and RSI(14) < 30; **short** when z ≥ +2.0 and RSI(14) > 70 |
| Entry | Ladder of 3 post-only limit orders at z = −2.0 / −2.5 / −3.0 (sized 40 / 30 / 30%) |
| Take-profit | Back to VWAP (z = 0); half the position closes at z = −1.0 |
| Stop | z = −4.0 or 1.5 × ATR(14, 15m) beyond the last ladder level, whichever is closer |
| Time stop | 16 bars (4 hours) |
| Kill condition | Regime leaves RANGE → cancel unfilled ladder orders and exit at breakeven or better within 2 bars, otherwise at market |
| Size | Full-deployment allocation (§5); the 3 ladder orders together equal the full allocation |

Expected profile: win rate around 60–70%, small average wins, and losses capped by the stop and the regime kill.

### S3 · Funding Tilt (overlay, not a standalone strategy)

Lighter pays funding **hourly** (confirm in the docs at build time).

- When a market's 8h average funding is extreme (annualised > ±30%), the side that **pays** funding gets **−50% size**. In full-deployment mode there is no headroom to increase the receiving side.
- Logged as its own PnL component so we can measure whether it adds value.
- A full delta-neutral funding-arbitrage strategy (Lighter against a hedge venue) is deferred to Phase 3.

---

## 5. Layer 3 — Capital deployment & position sizing (FULL DEPLOYMENT MODE)

**Owner decision:** the bot uses **100% of the fund**. That means the whole sub-account balance is put to work, not only a small "risk slice" per trade.

"100% of the fund" is interpreted as **100% of equity deployed as position notional at controlled leverage**. It does **not** mean posting 100% of equity as margin at high leverage. The difference:

| Leverage on the full fund | Loss per stopped trade (≈2% stop) | Drawdown after a normal 8-loss streak | Adverse move that liquidates (approx.) |
|---|---|---|---|
| **1× (default)** | ≈ 2% | ≈ 15% | none in practice (longs ≈ −100%) |
| 2× (hard cap) | ≈ 4% | ≈ 28% | ≈ −45% |
| 3× | ≈ 6% | ≈ 39% | ≈ −30% |
| 5× | ≈ 10% | ≈ 57% | ≈ −17% |
| 10× | ≈ 20% | ≈ 83% | ≈ −7% (BTC has moved this much in hours) |

Losing streaks of 6–8 trades are normal for a trend strategy that wins 35–45% of its trades. Liquidation levels are approximate; they depend on each market's maintenance margin. A stop-loss can slip past its price during a crash, so liquidation distance is the last line of defence.

### Allocation formula

```
deployable  = equity × L × m × vol_scaler          # L = leverage (default 1.0, hard cap 2.0)
                                                   # m = Target Engine multiplier (≤ 1)
                                                   # vol_scaler ∈ [0.5, 1.0] (can only reduce)
w_i         = market weight (BTC 1.0, ETH 1.0, SOL 0.5)
notional_i  = deployable × w_i / Σ w_active        # split across currently active signals
            capped at: BTC/ETH 100%, SOL 50% of deployable
```

- **One signal active** → that position takes the full deployable amount (subject to the market cap).
- **New signal while capital is fully used** → existing positions are trimmed (reduce-only orders) to their new share. Trims happen only if the change is more than 10%, so we don't churn.
- **No signal / CHAOS regime** → the capital sits in USDC as collateral. Being 100% deployed means *available for* signals; the bot never forces a trade just to be invested.
- **Correlation guard:** BTC, ETH and SOL move together. If all three signal in the same direction, total notional is still capped at `deployable`, so we never stack 3× exposure.
- **Margin:** cross margin on the bot sub-account, with exchange leverage set to 3× per market. This leaves headroom above our 1–2× effective use, so a temporary price spike cannot trigger liquidation.
- **Liquidation guard:** the risk manager rejects any order that would leave the liquidation price closer than 3 × the stop distance.

---

## 6. Layer 4 — Weekly Target Engine

Inputs:
- `E₀`: equity on Monday 00:00 IST
- `T`: weekly target, % of E₀
- `L`: weekly loss limit, % of E₀
- `pnl`: realised + unrealised PnL this week
- `t`: fraction of the week elapsed, from 0 to 1

`m` = **fraction of the fund the bot may deploy**. 1.0 = 100%.

```
progress p = pnl / (T × E₀)

target_multiplier m:
  pnl ≤ −L × E₀               → 0     HALT for the week (manual resume only)
  pnl ≤ −0.5 L × E₀           → 0.5   defensive
  day PnL ≤ −4% E₀            → 0     pause until next day 00:00 IST
  p ≥ 1.0 (target hit)        → 0.25  "house-money mode" with profit floor = 0.8 T
      └ pnl falls to floor    → 0     LOCKED for the rest of the week
  p ≥ t + 0.30 (well ahead)   → 0.6
  otherwise (on or behind)    → 1.0   NEVER above 1.0
```

- When the target is hit, open trades keep their stops, which also tighten to a maximum of 1.5 × ATR.
- **House-money mode** lets winning weeks run past the target while guaranteeing at least 80% of the target is kept.
- Default parameters (full deployment at 1×): `T` is calibrated in §8.5, `L` = 10%, daily stop = 4%. These are re-derived from the Monte-Carlo drawdowns in §8.3.

### Optional · Volume Target Module (S4 Maker Engine)

Used only if the weekly goal is **traded volume** (points or tier). It is strongly preferred on a Premium account; on Standard it is limited to slow quoting.

| Item | Rule |
|---|---|
| Quoting | Two-sided post-only quotes around the micro-price |
| Spread | max(2 ticks, k × 1-min realised vol) |
| Inventory skew | Quotes shift against inventory: skew = −γ × inventory / max_inventory |
| Inventory cap | ±25% of deployable capital; above the cap, quotes go on the reducing side only |
| Toxicity guard | Pause 30 s when trade-flow imbalance > 70% over 10 s or price moves > 0.3 × 1-min ATR |
| Pacing | hourly volume budget = remaining volume / remaining hours; when ahead of budget, quoting slows down |
| Cost cap | Net cost per $1M volume (fees + adverse selection) ≤ configured maximum, else the module pauses |
| Rate budget | Requotes only when the fair price moves ≥ 1 tick, sent as `sendTxBatch` |

---

## 7. Execution playbook

| Situation | Order method |
|---|---|
| S1 breakout entry | **Limit-chase**: post-only at the best price, re-priced up to 3 times over 30 s; then IOC with a slippage cap of 0.15 × ATR if the signal is still valid. Otherwise skip. |
| S2 ladder entry | Post-only, good-till-time (expiry 4 hours) |
| Protection | Every entry sent as a **grouped OTOCO** order (entry + reduce-only stop-loss + take-profit), which counts as one transaction against the quota |
| Trailing stop | Modify the existing stop order; only when the new level moves ≥ 0.25 × ATR, to save rate budget |
| Emergency exit | Cancel all orders (all markets, index 255), then reduce-only IOC market orders |
| Rate-limit budget (Standard tier, 60 tx/min) | 70% strategy, **30% reserved** for cancels and emergencies |

---

## 8. Validation & research plan

### 8.1 Data
- **Lighter** history (candles, trades, funding), used for costs, spreads and recent behaviour. History is short, so:
- **Binance / Bybit perpetuals**: 3+ years of 15m/1h candles plus funding, used for signal research. Final checks are always rerun on Lighter data.
- A **live recorder** stores Lighter order books and trades from day 1, so we build our own high-quality dataset.

### 8.2 Cost model
Fees by tier (Standard 0%; Premium per the fee table) · slippage = half the spread + impact by size · fill probability of post-only orders allowing for the Standard-tier speed bump · hourly funding.

### 8.3 Method
- **Walk-forward:** 6 months train → 1 month test, rolled across the full history. Only out-of-sample months count.
- **Robustness:** changing each parameter by ±20% must not turn the result negative (we look for flat plateaus, not sharp peaks).
- **Monte-Carlo:** 5,000 reshuffles of the trade sequence → the 95th-percentile drawdown becomes the live drawdown alarm.
- At least **200 trades per strategy** out-of-sample before a strategy can go live.

### 8.4 Acceptance gates (out-of-sample, after costs)

| Metric | Gate |
|---|---|
| Profit factor | ≥ 1.3 |
| Sharpe (annualised) | ≥ 1.0 |
| Max drawdown (at 1× full deployment) | ≤ 20% |
| Positive weeks | ≥ 55% |
| Correlation between S1 and S2 returns | ≤ 0.3 |

### 8.5 Setting the weekly target `T`
`T` = the **median out-of-sample weekly return** of the combined portfolio × 0.8 (a safety haircut). A target above this level would push the system towards over-trading, so the Target Engine config rejects it.

### 8.6 Go-live ladder
Backtest gates → **2 weeks paper/testnet running 24x7** (costs no money) → **3-day live smoke test at the exchange's minimum order size** (proves signing, stops and reconciliation with real funds) → **100% of the fund at 1×**. Moving to 2× requires 4+ live weeks within the backtest's 95% bands.

### 8.7 Live kill criteria (per strategy)
- Drawdown > 1.5 × the Monte-Carlo 95th-percentile drawdown → strategy disabled, owner alerted
- Live slippage > 2 × modelled for 3 days → execution review
- 4 weeks of live Sharpe < 0 while the backtest expected > 1 → strategy goes back to research

---

## 9. Configuration preview

```yaml
universe:
  BTC: { risk_weight: 1.0 }
  ETH: { risk_weight: 1.0 }
  SOL: { risk_weight: 0.5 }

regime:
  adx_trend: 25
  adx_range: 20
  er_trend: 0.35
  er_range: 0.25
  chaos_vol_pct: 95
  hysteresis_bars: 2

s1_trend:
  donchian: 20
  atr_stop: 2.0
  chandelier_atr: 3.0
  partial_tp_r: 2.0
  time_stop_bars: 48

s2_meanrev:
  vwap_bars: 16
  z_entries: [-2.0, -2.5, -3.0]
  ladder: [0.4, 0.3, 0.3]
  z_stop: -4.0
  time_stop_bars: 16

capital:
  deployment: full          # 100% of the fund
  leverage: 1.0             # effective leverage on the full fund
  leverage_hard_cap: 2.0    # config validation rejects anything higher
  exchange_leverage: 3.0    # per-market setting = liquidation headroom
  vol_scaler_range: [0.5, 1.0]
  market_caps: { BTC: 1.0, ETH: 1.0, SOL: 0.5 }
  rebalance_threshold: 0.10
  margin_mode: cross

target:
  mode: pnl             # pnl | volume | both
  weekly_target_pct: auto   # from §8.5 calibration
  weekly_loss_limit_pct: 10.0
  daily_loss_limit_pct: 4.0
  house_money_floor: 0.8
  week_start: "MON 00:00 Asia/Kolkata"
```

---

## 10. Build order (strategy track)

1. Data recorder + historical loaders (Lighter + Binance)
2. Indicators library + regime detector (unit-tested against reference values)
3. Event-driven backtester sharing the live strategy interface
4. S1 → validate → S2 → validate → portfolio combination
5. Target Engine simulation over historical weeks (how often the target is hit, how often locked or halted)
6. Calibrate `T`, freeze v1 parameters, start paper trading 24x7
