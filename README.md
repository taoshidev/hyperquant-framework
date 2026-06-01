# Hyperquant — Lead Quant Agent (OSS prompt framework)

An open-source **prompt framework** for an AI quant researcher. You describe a
trading strategy idea in plain English; a **Lead Quant Agent** turns it into
runnable strategy code, fetches market data, backtests it with realistic costs,
stress-tests it against the ways quant research fools itself, and iterates with you
— or tells you honestly that the idea doesn't hold up.

This repo is **instructions, not application code** — the set of Markdown files
that shape how the agent reasons, uses its tools, and avoids the classic pitfalls.
Fork it, read it, and make it your own.

---

## How it runs

```
        you (plain-English idea, in chat)
                    │
                    ▼
        ┌──────────────────────────┐     tools:
        │     Lead Quant Agent      │──▶  • web search / read papers
        │     (AGENT.md + docs/)    │     • fetch market & public data
        │                           │     • read / write / run code
        └──────────────────────────┘     • run backtest
                    │
                    ▼
   spec → data → signal → risk → backtest → self-critique → verdict
                    │
                    ▼
        you approve / iterate  →  versioned re-test
```

**`AGENT.md`** is loaded as the agent's system prompt; the agent reads the
**`docs/`** files on demand as it reaches each step.

---

## The files

| File | Role |
| --- | --- |
| **`AGENT.md`** | The agent's constitution — identity, the build loop, the hard rules, the skeptic discipline. Always loaded; kept lean. |
| `docs/workflow.md` | The phase-by-phase playbook: frame → data → signal → risk → backtest → review. |
| `docs/pitfalls.md` | The research-integrity checklist — lookahead, overfitting, data snooping, regime sensitivity, cost realism, and more. |
| `docs/validation.md` | How to *prove* an edge is real: pre-registration, OOS/walk-forward, the universal subtests, robustness, overfit tripwires. |

Read order for a human: this README → `AGENT.md` → `docs/workflow.md`, then the
rest as needed.

---

## Design philosophy

**A disciplined research process, not a strategy generator.** The agent runs the
full pipeline — framing, data, signal, risk, backtest, review — as a sequence of
structured phases, each with a clear output and a gate it must pass. Built in is a
mandatory self-critique: the agent attacks its own result, hunting for the flaw,
before any of it reaches you.

**Research integrity is the product.** The goal is not a pretty equity curve — it's
the truth about whether a repeatable edge exists. The framework is opinionated
about killing bad ideas fast, modeling costs honestly, guarding the out-of-sample
holdout, and never moving the goalposts after seeing results. A clean "this doesn't
work, here's why" is a successful run.

**Lean by design.** A small always-on system prompt points to deeper docs that the
agent reads only when needed, with a bias toward one well-designed backtest over
many hopeful ones.

**Model-agnostic.** Nothing here depends on a specific LLM. The instructions are
principles, not one model's idioms, so the same framework can run on
different models.

---

## Fork & extend

This is your starter repo — edit it.

- **Tune the rules.** Sharpen `docs/pitfalls.md` and the tripwires in
  `docs/validation.md` to your markets and risk appetite.
- **Add data.** Out of the box the agent works from public market data — OHLCV and
  common indicators; nothing stops you wiring in a richer source (on-chain,
  derivatives) and teaching the agent to use it.
- **Keep `AGENT.md` lean.** It's loaded every turn — put depth in `docs/`, pointers
  in `AGENT.md`.
