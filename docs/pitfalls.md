# Pitfalls — the research-integrity checklist

The ways quant research lies to you. Most failed strategies don't fail because the
idea was bad — they fail because one of these crept in unnoticed and made a
mediocre idea look great. Treat this as a checklist: before you trust a result,
confirm you've ruled out each relevant item, or flag the ones you couldn't.

**Rule of the house:** *never proceed silently past a pitfall.* Either confirm it's
handled, or say explicitly "I could not rule this out" in your verdict.

The five below are the ones that kill strategies most often — know them cold. The
rest are grouped after.

---

## The big five

### 1. Lookahead bias
Using information at bar *t* that wasn't actually available until later. The
deadliest because it inflates results enormously and silently.

- Every feature/signal at *t* uses only data with timestamp ≤ *t*'s close.
- Trade on the *next* bar's open/price, not the close you're deciding from.
- No future-derived normalization: don't scale a feature by the full-sample mean/
  std (that leaks the future into the past) — use rolling/expanding windows.
- Labels/targets must be strictly forward (return from *t* to *t+1*), never overlapping into the feature window.
- Indicators with warmup (MAs, RSI) must have enough prior bars; don't backfill.
- **Data availability** — a timestamp isn't when you could *act* on a value. Some
  inputs (revisions, settlement, on-chain confirmation, delayed feeds) land later
  than the bar they're stamped to. If an input arrives with material delay, model
  that lag in the backtest rather than assuming it's there at the bar's close.

### 2. Overfitting
Tuning the strategy until it fits the historical noise rather than a real pattern.

- Complexity must earn its place: every added parameter/feature must improve
  *out-of-sample*, not just in-sample. Start with a baseline and beat it.
- The more parameters you fit, the more the in-sample result is curve-fitting.
- A huge gap between in-sample and OOS performance is the signature — expect OOS
  to be meaningfully worse; if IS is spectacular and OOS collapses, it's overfit.
- Single-feature *and* hundred-feature strategies are both fragile (one is luck,
  the other is noise-fitting) — favor a few features with a clear rationale.
- **Tripwire:** OOS Sharpe > 3, win-rate > 80%, or perfect-looking equity → almost
  certainly overfit (or leaking). See `docs/validation.md`.

### 3. Data snooping (multiple testing)
The more ideas/variants you test, the more likely one looks good *by chance*. The
20th coin flip that comes up heads 8 times isn't skill.

- Count your trials. Testing many variants inflates the best one's apparent
  significance — the more you try, the higher the bar each result must clear.
- Don't HARK (Hypothesize After Results are Known): if you find the pattern first
  and invent the rationale second, you've fit noise. Thesis comes first.
- Each pass over the OOS holdout consumes it. After a few, it's no longer out-of-
  sample — treat further results as in-sample and say so.
- A permutation / reality-check p-value that accounts for the number of trials is
  worth more than a raw t-stat.

### 4. Regime sensitivity
A strategy that only works in one market environment will break when the regime
changes — and crypto regimes change hard and fast.

- Test across **bull, bear, and chop** — and across high- and low-volatility
  periods. A strategy validated only in a 2020–21 bull run is untested.
- Attribute performance by regime. If ~all the PnL comes from one regime, the
  Sharpe is misleading — say so explicitly.
- Watch for structural breaks (a market mechanic changed, a venue launched, a
  major liquidation event). An edge can genuinely exist before a break and vanish
  after.
- Prefer edges with a mechanism that should hold across regimes; be suspicious of
  ones that need a specific environment with no reason why.

### 5. Transaction-cost realism
The fastest way a paper edge evaporates. High-turnover strategies are especially
exposed.

- Always model **fees + slippage**. For perpetuals, model **funding** — it can
  dominate PnL on a carry/hold strategy.
- Test survival at **2× your cost assumption**. If the edge dies, it's too thin.
- Account for **market impact** — you can't fill size at the mid; large orders move
  the price. Don't assume infinite liquidity.
- Respect **minimum order sizes / lot sizes**; drop or round sub-minimum orders.
- Report **cost drag as a % of gross PnL**. If costs eat most of the gross, the
  "edge" is a cost illusion.
- High turnover + thin per-trade edge = death by a thousand fees. Watch turnover.

---

## More pitfalls, grouped

Read these when the relevant phase touches them.

### Data integrity
- **Survivorship bias** — excluding dead/delisted symbols overstates returns.
- **Point-in-time** — use the data as it was *then*, not later-revised values.
- **Timezone / alignment** — one UTC index for all series; flag gaps > 1 bar.
- **Exchange differences** — price/volume differ across venues; don't blend
  naively. Be explicit about which venue's data you're using.
- **Outliers** — don't silently clip real extreme moves (crypto has 30% days);
  but do catch bad ticks / data errors. Know which is which.
- **Sufficient history** — too few bars → no statistical power. Minimum trade
  count matters more than minimum days.

### Feature engineering
- **Normalization leakage** — any global mean/std/min/max leaks the future. Use
  rolling or expanding windows that only see the past.
- **Rolling-window contamination** — a window that straddles the train/test
  boundary leaks. Purge/embargo around splits.
- **Multicollinearity** — highly correlated features double-count the same signal
  and destabilize fits.
- **Feature warmup** — drop the leading bars where the feature isn't yet valid.

### Statistical
- **Multiple testing** — see Data snooping above; correct for it.
- **Autocorrelation** — overlapping returns and serial correlation inflate t-stats.
  Use HAC/Newey-West standard errors or block bootstrap, not naive ones.
- **Effective sample size** — with autocorrelation, your *effective* N is far below
  your bar count. 10,000 1-minute bars is not 10,000 independent observations.
- **Test-then-fit ordering** — pre-register the hypothesis, then test. Not the
  reverse.

### Time-series specific
- **No random k-fold.** Shuffling time-series rows leaks future into past. Always
  use forward-chaining / walk-forward splits.
- **Embargo / purge** between train and test to stop leakage across the boundary.
- **Annualization factor** — crypto trades 24/7/365. Annualize with **365** days
  (not 252) and the correct bars-per-year for your resolution, or your Sharpe is
  wrong.
- **Seasonality / vol clustering** — returns cluster; volatility is persistent.
  Don't assume i.i.d.

### Risk & tail
- **Drawdown is path-dependent** — the *sequence* of losses matters, not just the
  max. A strategy can have a fine max-DD and still be unholdable.
- **Tail correlation** — diversification vanishes in crashes; correlations → 1.
- **Liquidation cascades** — in crypto, forced liquidations create reflexive moves
  that backtests on calm data won't show.
- **Kelly / sizing estimation error** — sizing off an overfit edge estimate
  over-bets. Be conservative.

### Reproducibility
- **Fix random seeds** — any stochastic step (model init, sampling) must be seeded.
- **Cache + version data** — record the exact window and source so the run
  reproduces.
- **Determinism** — the same inputs must give the same metrics. A result you can't
  reproduce isn't a result.

---

## Overfit tripwires (auto-flag)

If any of these fire, stop and treat the strategy as suspect until you've actively
disproven the concern. Don't present it as a win first.

- OOS Sharpe **> 3** (post-cost)
- Win rate **> 80%**
- **< 30** trades (and especially < 100)
- A **single** backtest window with no walk-forward / no OOS
- Zero losing months, or a near-perfectly smooth equity curve
- Information coefficient **> 0.3** with no structural explanation
- All positive performance concentrated in **one regime**
- In-sample result spectacular, OOS collapses

These thresholds are deliberately strict — when in doubt, flag. The tests behind
them and the decision guide are in `docs/validation.md`.
