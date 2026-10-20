# Lighter.xyz Bot — Trading Strategy Specification (v1)

> Companion to [`TRADING_BOT_PLAN.md`](./TRADING_BOT_PLAN.md). All numbers below are **starting parameters** to be validated and tuned in backtests (§8), not promises of returns.
>
> **Assumption:** the weekly target is a **profit (PnL %) target**. A volume-target engine is specified as an optional module (§6) and can be switched on independently.

---

## 1. Strategy philosophy

1. **No single strategy works in all markets.** The bot runs a small *portfolio* of uncorrelated strategies and a **regime filter** decides which ones may trade.
2. **Survive first, profit second.** Every position has an exchange-side stop from the moment it opens. Risk is sized from the stop distance, never from the target.
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
| Base risk | **0.50% of equity** per trade (core markets); 0.25% for SOL |

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
| Base risk | **0.30% of equity** per trade (whole ladder combined) |

Expected profile: win rate around 60–70%, small average wins, and losses capped by the stop and the regime kill.

### S3 · Funding Tilt (overlay, not a standalone strategy)

Lighter pays funding **hourly** (confirm in the docs at build time).

- When a market's 8h average funding is extreme (annualised > ±30%), the side that **receives** funding gets **+20% size** and the side that pays gets **−50% size**.
- Logged as its own PnL component so we can measure whether it adds value.
- A full delta-neutral funding-arbitrage strategy (Lighter against a hedge venue) is deferred to Phase 3.

---

## 5. Layer 3 — Portfolio construction & position sizing

```
qty = (equity × base_risk × regime_ok × target_multiplier × funding_tilt × vol_scaler)
      ───────────────────────────────────────────────────────────────────────────────
                               |entry − stop| × contract_size
```

| Control | Value |
|---|---|
| `vol_scaler` | Targets **20% annualised portfolio volatility**, clamped to [0.5, 1.25] |
| Max effective leverage | **3×** total notional / equity |
| Correlated exposure cap | BTC, ETH and SOL are highly correlated, so the sum of open risk **in the same direction ≤ 1.25% of equity** |
| Max open positions | 4 in total, at most 1 per strategy per market |
| Margin mode | **Isolated** per position: a bad trade cannot drain the whole account |
| Exchange leverage setting | 5× max per market, giving headroom between our stop and the liquidation price |
| Liquidation buffer | Stop must sit ≥ 3 × ATR closer than the liquidation price, otherwise the size is reduced |

---

## 6. Layer 4 — Weekly Target Engine

Inputs:
- `E₀`: equity on Monday 00:00 IST
- `T`: weekly target, % of E₀
- `L`: weekly loss limit, % of E₀
- `pnl`: realised + unrealised PnL this week
- `t`: fraction of the week elapsed, from 0 to 1

```
progress p = pnl / (T × E₀)

target_multiplier m:
  pnl ≤ −L × E₀               → 0     HALT for the week (manual resume only)
  pnl ≤ −0.5 L × E₀           → 0.5   defensive
  day PnL ≤ −1% E₀            → 0     pause until next day 00:00 IST
  p ≥ 1.0 (target hit)        → 0.25  "house-money mode" with profit floor = 0.8 T
      └ pnl falls to floor    → 0     LOCKED for the rest of the week
  p ≥ t + 0.30 (well ahead)   → 0.6
  otherwise (on or behind)    → 1.0   NEVER above 1.0
```

- When the target is hit, open trades keep their stops, which also tighten to a maximum of 1.5 × ATR.
- **House-money mode** lets winning weeks run past the target while guaranteeing at least 80% of the target is kept.
- Default parameters: `T` is calibrated in §8.5, `L` = 3%, daily stop = 1%.

### Optional · Volume Target Module (S4 Maker Engine)

Used only if the weekly goal is **traded volume** (points or tier). It is strongly preferred on a Premium account; on Standard it is limited to slow quoting.

| Item | Rule |
|---|---|
| Quoting | Two-sided post-only quotes around the micro-price |
| Spread | max(2 ticks, k × 1-min realised vol) |
| Inventory skew | Quotes shift against inventory: skew = −γ × inventory / max_inventory |
| Inventory cap | ±0.5% equity notional; above the cap, quotes go on the reducing side only |
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
| Max drawdown | ≤ 15% |
| Positive weeks | ≥ 55% |
| Correlation between S1 and S2 returns | ≤ 0.3 |

### 8.5 Setting the weekly target `T`
`T` = the **median out-of-sample weekly return** of the combined portfolio × 0.8 (a safety haircut). A target above this level would push the system towards over-trading, so the Target Engine config rejects it.

### 8.6 Go-live ladder
Backtest gates → **2 weeks paper/testnet running 24x7** → live at **10% of capital** for 2–4 weeks → scale in steps of +25%, and only if live results stay within the backtest's 95% bands.

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
  base_risk: 0.005

s2_meanrev:
  vwap_bars: 16
  z_entries: [-2.0, -2.5, -3.0]
  ladder: [0.4, 0.3, 0.3]
  z_stop: -4.0
  time_stop_bars: 16
  base_risk: 0.003

portfolio:
  target_vol_annual: 0.20
  max_leverage: 3.0
  max_same_dir_risk: 0.0125
  margin_mode: isolated

target:
  mode: pnl             # pnl | volume | both
  weekly_target_pct: auto   # from §8.5 calibration
  weekly_loss_limit_pct: 3.0
  daily_loss_limit_pct: 1.0
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
