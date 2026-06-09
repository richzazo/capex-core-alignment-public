# Capex Core Alignment — RETIRED

**This doc is retired as of June 8, 2026.**

It previously described **Capex Scout** and **Portfolio Intelligence** as two separate products sharing one substrate ("Capex Core"), and governed the boundary between them.

**That framing is obsolete.** Capex Scout and Portfolio Intelligence converged into a single product, **The Desk**, on June 8, 2026 — one codebase, one vision. The old "two surfaces, never merge codebases" rule was operational (PI lived on a second Claude account for parallel building), not architectural, and has been retired.

## Where the current truth lives
- **Product vision, convergence, architecture, locked decisions:** `TheDesk_Handoff_unified-vision-and-convergence_q4t8r.md` (in the project repo / project knowledge).
- **Working style / cross-project behavior:** [PROFILE.md](https://raw.githubusercontent.com/richzazo/multi-project-alignment-public/main/PROFILE.md) (v1.21+).

## The Desk in one line
One AI-native operator surface on Richard's real book: **Cockpit** (risk/health view — Sim 1, what-if on actuals) + **Scout** (research/suggestion arm — Sim 2, agent-driven discovery & allocation). Agents = specialist desks reporting in.

## Still-valid substance carried forward (now living in code + the handoff doc)
- Persistence schema (`signals` / `evidence` / `feedback` / `llm_calls`), all access through `db/store.py`; extended with `account_snapshots` / `position_snapshots` / `option_snapshots`.
- Pydantic contracts (`contracts.py`): SourceItem, SignalScores, Signal, ResearchBrief, Basket.
- Verdict canon: **BUY / SELL / WATCH / SKIP** (direction primitives).
- Signal scoring weights: 25/20/20/15/10/10 (momentum/valuation/catalyst/sentiment_accel/balance_sheet/flow).
- Six source-agents + pluggable reasoner (Haiku for theme extraction, Sonnet for briefs).

The full prior version of this doc remains available in this repo's git history.

> A proper living architecture doc for The Desk (rewritten from single-product reality) is a flagged follow-up. Until it exists, the handoff doc above is the source of truth.
