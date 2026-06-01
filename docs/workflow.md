# Workflow — the build loop

The phase-by-phase playbook for taking an idea to a validated (or killed) strategy.
This mirrors a quant firm's pipeline — data → research → risk → backtest →
independent review — collapsed into six phases you run yourself, in order.

Each phase has: a **goal**, the **output** it produces, and a **gate** (what must
be true before you move on). Don't advance past a failed gate — fix it or kill the
idea.

> The integrity rules referenced here are in `docs/pitfalls.md`; the validation
> tests are in `docs/validation.md`.

---

## Phase 1 — Frame the idea

**Goal:** turn the trader's plain-English idea into a precise, testable spec.

Produce a clear spec with:
- **Thesis** — one sentence: what edge are we capturing?
- **Economic rationale** — *why should this edge exist and persist?* Name the
  mechanism (risk premium, behavioral bias, structural flow, microstructure). "It
  worked in the data" is not a rationale — that's how you find noise. If you can't
  state a mechanism, tell the trader; weak rationale is the strongest predictor of
  a backtest that won't replicate live.
- **Market & timeframe** — instrument(s), bar resolution, holding horizon.
- **Rules** — entry, exit, and risk/exit-stop logic, explicitly.
- **Parameters** — and their plausible ranges (you'll test robustness later).
- **Pre-committed pass criteria** — the metrics and thresholds that will decide
  pass/fail, written *now*, before any backtest. e.g. `OOS Sharpe ≥ 1.0`,
  `max drawdown ≤ 20%`, `≥ 100 trades`, `permutation p < 0.05`.

**Gate:** thesis has a real economic mechanism; pass criteria are written down.
If the idea is vague, ask the trader 1–2 sharpening questions before coding.

---

## Phase 2 — Data

**Goal:** get enough trustworthy, point-in-time history.

- Fetch public market data — OHLCV and common technical indicators. Cache
  everything for reproducibility.
- **Enough history** to cover multiple market regimes (bull, bear, chop) — a
  strategy tested only in a bull market is untested. Aim for several years on
  daily, proportionally more bars on intraday.
- **Point-in-time discipline** — no survivorship (include delisted/dead symbols
  where relevant), correct timezone alignment (use UTC), watch for exchange
  differences. See `docs/pitfalls.md` § Data.
- **Carve out the OOS holdout now.** Pick a cutoff date (e.g. the most recent
  25–30%). Quarantine it. You will not fit, tune, or even look at it until Phase 5.

**Gate:** data spans multiple regimes, is point-in-time clean, OOS split is fixed
and recorded.

---

## Phase 3 — Signal + regime

**Goal:** implement the directional logic, and understand what it depends on.

- Implement the rule or model on the **in-sample** data only.
- **Start simple.** A baseline (a single clear rule, a linear model) before
  anything expressive. Complexity must *earn its place* by beating the baseline
  out-of-sample — see overfitting in `docs/pitfalls.md`.
- **Regime awareness** — note which regime the edge lives in. Does it only work in
  trending markets? High vol? Label the regimes and check whether the signal is
  one-regime-dependent (a major fragility flag).
- **No lookahead** in feature construction — features at bar *t* use data ≤ *t*.
  Lag anything uncertain by one bar.

**Gate:** signal computes without lookahead; you can state which regime(s) it
relies on.

---

## Phase 4 — Risk & sizing

**Goal:** turn the directional view into a sane position.

- **Volatility-aware sizing** — scale exposure down when vol is high; don't bet a
  fixed notional regardless of conditions.
- **Caps** — max position, max leverage, max concentration.
- **Drawdown control** — de-risk after losses; define a stop or throttle.
- This layer can *override* the signal. A great signal with reckless sizing is a
  blow-up waiting to happen.

**Gate:** position sizing is bounded and volatility-aware; worst-case exposure is
something the trader would accept.

---

## Phase 5 — Backtest

**Goal:** measure the strategy honestly, with realistic frictions.

- Run via the backtest tool; it returns the performance metrics (Sharpe, Sortino,
  Calmar, max drawdown, win rate, trade count, turnover, cost drag, per-regime
  attribution).
- **Realistic costs always** — fees + slippage; funding for perps. Then check it
  survives **2× costs** (robustness, not optimism).
- **Walk-forward, not a single fit.** Roll the train/test window forward; never
  use random k-fold on time series (it leaks the future). Purge/embargo around
  splits.
- **Run the validation suite** in `docs/validation.md`: synthetic harness,
  mechanical invariants, and negative controls. These are cheap and catch the
  failures a pretty equity curve hides.
- **Only now** evaluate on the OOS holdout — once. If you tuned and want to retest,
  you need *fresh* OOS, not the same window again.

**Gate:** metrics computed with realistic costs; validation suite run; OOS
evaluated exactly once.

---

## Phase 6 — Review & iterate

**Goal:** an honest verdict, then a controlled next iteration.

1. **Skeptic hat** (from `AGENT.md`): actively try to break the result. Run it
   against the tripwires in `docs/validation.md` (Sharpe > 3, win > 80%, < 30
   trades, single window, IC suspiciously high, one-regime-only).
2. **Audit**: confirm no rule was violated — no lookahead, OOS used once, costs
   modeled, criteria were pre-committed.
3. **Verdict** to the trader — `CONFIRMED` / `MARGINAL` / `KILLED`, with the
   reason and the caveats you couldn't rule out. Show the equity curve and the
   numbers plainly. Don't oversell `MARGINAL` as `CONFIRMED`.
4. **Iterate** — if the trader wants changes, treat it as a **new version** rather
   than editing the last one in place — keep the previous run intact so the trader
   can compare — and re-run from the relevant phase to show the delta. Each new idea
   tested on the OOS is another peek — track how many and tell the trader when the
   holdout is spent.

**Gate:** verdict is honest and reproducible; iteration produces a new version
rather than overwriting the last.

---

## Notes on iterating without fooling yourself

Every time you test a new variant against the same data, you get another chance to
find noise that looks like signal. This is **data snooping** and it's silent.
Defenses (detail in `docs/pitfalls.md` / `docs/validation.md`):

- Count your trials. The more variants you test, the higher the bar each must clear.
- Prefer changes motivated by the *thesis*, not by what the last backtest showed.
- When the OOS holdout has been used a few times, it's no longer out-of-sample —
  say so, and either get fresh data or treat further results as in-sample.
