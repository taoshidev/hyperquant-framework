# Validation — how to prove an edge is real

`docs/pitfalls.md` is what *not* to do. This is what you *must do* to earn the
right to call a result real. Avoiding mistakes isn't enough; you have to actively
try to disprove your own strategy and watch it survive.

Run these during Phase 5 (backtest) and Phase 6 (review). They're cheap relative
to the cost of deploying a strategy that was never real.

---

## 1. Pre-registration (before you look)

Written in Phase 1, enforced here:

- The **pass criteria** (metrics + thresholds) were committed *before* the
  backtest. You evaluate against those — you do not move them after seeing results.
- The **OOS holdout cutoff** was fixed before fitting.
- If you change the strategy and want to re-test, that's a *new* trial — count it
  (see Multiple-trials accounting below).

This single discipline kills most p-hacking before it starts.

---

## 2. Out-of-sample & walk-forward

- **Train in-sample, judge out-of-sample.** Fit/tune only on in-sample data;
  the OOS holdout is touched **once**, at the end.
- **Walk-forward over a single fit.** Roll the train→test window forward through
  time and aggregate. A single train/test split can get lucky; a walk-forward
  shows whether the edge persists as time moves.
- **Purge & embargo** around each split boundary so no feature window or label
  straddles train and test.
- **Never random k-fold** on time series — it trains on the future to predict the
  past. Forward-chaining only.
- Expect OOS to be **worse** than in-sample. A small, consistent degradation is
  healthy. A collapse means overfit; OOS *better* than IS means a leak or a fluke —
  investigate, don't celebrate.

---

## 3. The universal subtests

Three cheap, mechanical checks. Each targets a specific way a result can be fake.
Run them as code; a failure is a stop-and-fix, not a warning.

### a. Synthetic harness
Feed the strategy data with **known** properties:
- **Zero-signal data** (pure random walk / shuffled returns) → the strategy must
  produce **no** edge. If it "finds" alpha in noise, the harness itself is leaking.
- **Known-signal data** (you inject a pattern the strategy should catch) → it must
  **recover** it. If it can't, the implementation is broken.

### b. Mechanical invariants
Assert the things that must be true regardless of strategy:
- Features at *t* use only data ≤ *t* (no lookahead).
- Forward returns are *t → t+1*, never overlapping the feature window.
- Feature scaling uses past-only windows (no global normalization).
- Threshold/parameter monotonicity behaves as expected (tighten a filter → fewer
  trades, not more).

### c. Negative controls
Replace the real signal with something that *should* have no edge and confirm it
doesn't:
- **Random noise** signal → should be insignificant (p > 0.20).
- **Shuffled** version of your real signal → should be insignificant.
- **Misaligned returns** — shift the forward returns several bars off the signal so
  any real link is broken; the edge should vanish. If it *persists*, something leaks.
- **Cheating detector:** a signal built from *future* returns → should show a
  *huge* edge (IC near 1). If it doesn't, your pipeline isn't wired to the returns
  you think it is.

If a negative control shows an edge, something leaks. Find it before going on.

---

## 4. Robustness

A real edge isn't a knife-edge. Stress it:

- **Cost doubling** — re-run at 2× fees/slippage/funding. Survives? Good. Dies?
  Too thin to deploy.
- **Parameter neighborhood** — perturb each parameter ±10–20%. Performance should
  degrade *gracefully*, not fall off a cliff. A lone sharp peak in parameter space
  is overfit; you want a broad plateau.
- **Subsample / regime stability** — split by time and by regime (bull/bear/chop,
  high/low vol). The edge should appear in *most* slices, not live entirely in one.
- **Trade-count sufficiency** — enough trades for the result to be statistical, not
  anecdotal. Prefer ≥ 100; be very skeptical below 30.

---

## 5. Multiple-trials accounting

Every variant you test is another lottery ticket. Track it:

- Keep a running count of how many distinct strategies/variants you've evaluated
  against the data.
- The more trials, the higher the significance bar each must clear — a result that
  would be impressive as trial #1 is unremarkable as trial #50.
- When the OOS holdout has been seen a handful of times, **it is no longer
  out-of-sample.** Tell the trader; either obtain fresh data or label further
  results as in-sample.
- Prefer iterations motivated by the *thesis*, not by what the last backtest's
  residuals suggested (that's curve-fitting the noise).

---

## 6. Tripwires → verdict

The overfit signatures (also in `docs/pitfalls.md`). If any fire, the burden of
proof is on the strategy:

| Tripwire | Default reading |
| --- | --- |
| OOS Sharpe > 3 (post-cost) | Overfit or leaking until proven otherwise |
| Win rate > 80% | Check the payoff ratio (small wins / large losses?); else lookahead or regime bias |
| < 30 trades | Not statistically meaningful |
| Single window, no walk-forward | Untested |
| IC > 0.3, no mechanism | Suspiciously high — investigate |
| All PnL in one regime | Misleading Sharpe — caveat heavily |
| IS great, OOS collapses | Classic overfit |

### Decision guide

- **CONFIRMED** — passes pre-committed criteria on OOS, survives the subtests and
  robustness checks, no unresolved tripwire, has an economic rationale. Present
  with its real (modest) numbers and remaining caveats.
- **MARGINAL** — meets some criteria but a tripwire fired, robustness is shaky, or
  trade count is thin. Say so honestly; suggest what would resolve it (more data,
  fewer parameters, a different market). Do **not** dress this up as CONFIRMED.
- **KILLED** — fails OOS, a negative control lit up, the edge vanishes under
  realistic costs, or there's no plausible mechanism. Report it
  clearly — a clean kill is a successful research outcome and saves the trader money.

When unsure between two verdicts, choose the more skeptical one and tell the trader
exactly what you couldn't rule out.
