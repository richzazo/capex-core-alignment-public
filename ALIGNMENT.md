# Capex Core: Cross-Product Alignment

**Purpose.** This doc aligns two products in development on a shared architecture: **Capex Scout** (mobile-first AI-native equity research) and **Portfolio Intelligence** (desktop-first investment operations dashboard). Both are being built by Richard Zazo across separate Claude accounts/projects. This doc gives any Claude chat — in either project — a consistent read of how they relate and what each is allowed to assume about the other.

This is the canonical reference. If anything in either product's working memory contradicts this doc, this doc wins until Richard explicitly updates it.

Last updated: May 9, 2026 (v1.2 — persistence schema shipped + locked conventions).

---

## 1. The two products in one paragraph

Richard is building two operator surfaces on a shared substrate. **Capex Scout** is a mobile-first signals feed — discovery and conviction-building, organized around themes and second-order beneficiaries. **Portfolio Intelligence** is a desktop-first operations dashboard — taking ideas and positions and producing executable actions (trim, add, alert) through an Action Queue, with eventual IBKR order routing. They are not competitors and not duplicates. They are two stages of one workflow: Discovery → Conviction → Action → Monitoring. Capex Scout owns the first half. PI owns the second half.

## 2. Why this is one product, not two

The agent architecture, signal kinds, verdict pills, theme ontology, per-ticker resolution, and brief structure overlap *structurally*, not just aesthetically. Same six analyst agent types, same signal kinds, same per-ticker BUY/HOLD/WATCH/SELL resolution. The difference is what the operator does after seeing an idea:

- **Capex Scout:** "Here's a signal — watchlist it, action it, skip it." Stops at the verdict.
- **Portfolio Intelligence:** "Here's the signal applied to your $103K book — trim NVDA by 25 shares, here's the impact preview, click to execute." Starts at the position.

Same data, different operator surfaces, different end-of-workflow.

## 3. The architecture

```
                  ┌──────────────────────────────────────────────┐
                  │                CAPEX CORE                     │
                  │       (shared reasoning + data layer)         │
                  ├──────────────────────────────────────────────┤
                  │  Source-agents (News, X, EDGAR, Reddit,       │
                  │    Transcripts, Options Flow, Macro, Quant)   │
                  │  Reasoner interface (Anthropic LLM today)     │
                  │  Theme ontology + canonicalization            │
                  │  Signal scoring (25/20/20/15/10/10 weights)   │
                  │  Per-ticker call resolution                   │
                  │  Brief synthesis                              │
                  │  Basket engine                                │
                  │  Persistence (signals, themes, evidence,      │
                  │    feedback, provenance)                       │
                  └──────────────┬───────────────────┬───────────┘
                                 │                   │
                ┌────────────────┘                   └────────────────┐
                ▼                                                     ▼
   ┌──────────────────────────┐                   ┌────────────────────────────────┐
   │     CAPEX SCOUT           │                   │   PORTFOLIO INTELLIGENCE        │
   │     (mobile surface)       │                   │      (desktop surface)          │
   ├──────────────────────────┤                   ├────────────────────────────────┤
   │ Discovery + conviction    │                   │ Operations + execution          │
   │                           │                   │                                 │
   │ - Signal feed             │                   │ - Today (action queue)          │
   │ - Status buckets          │                   │ - Portfolio (drift, rebalance)  │
   │   New / Watch / Acted /   │                   │ - Research (workbench)          │
   │   Skip                    │                   │ - Positions × signals join      │
   │ - Verdict + horizon pills │                   │ - Order routing (paper → IBKR)  │
   │ - Provenance hover        │                   │ - 130/30 long/short structure   │
   │ - Watchlist promote       │                   │ - Target weights / drift logic  │
   │                           │                   │                                 │
   │  Reads CORE.signals       │                   │  Reads CORE.signals,            │
   │  Writes CORE.feedback     │                   │        CORE.briefs,             │
   │                           │                   │        CORE.baskets             │
   │                           │                   │  Writes positions, orders,      │
   │                           │                   │         action_history          │
   └──────────────────────────┘                   └────────────────────────────────┘
                ▲                                                     ▲
                │                                                     │
        morning scan, train,                                  trading day, desk,
        idea pipeline                                         execute the book
```

## 4. What lives where (the core boundary)

### Lives in Capex Core (shared, single source of truth)

- **Source-agents:** News, EDGAR/Insider, X/social, Reddit, Transcripts/earnings, Options Flow. Macro Lens and Quant Signals to be added (PI has them as concepts, CS has the slot).
- **Agent prompts:** The system prompts that turn raw text into theme assignments and signal extractions.
- **Theme ontology:** The canonical list of themes (AI Power Scarcity, Cooling Bottleneck, Sovereign AI, etc.) with canonical tickers.
- **Signal scoring:** Composite confidence using weights momentum 25 / valuation 20 / catalyst 20 / sentiment_accel 15 / balance_sheet 10 / flow 10.
- **Signal kinds:** emerging_theme, momentum, rel_value, insider, catalyst, social_accel, unusual_flow. PI's "News, Social, Insider, Analyst, Earnings, Momentum, Valuation, Macro" mapping is a presentation layer over these primitives — same underlying data.
- **Per-ticker call resolution:** BUY / HOLD / WATCH / SELL with primary and secondary signal drivers.
- **Brief synthesis:** what the company does, why matters, why now, bull/base/bear, catalysts, headwinds, valuation, momentum, crowding, balance sheet, comparables, related themes.
- **Basket engine:** clusters signals by theme into thematic baskets with confidence-weighted constituents.
- **Persistence schema:** see Section 5b for the shipped table shapes.
- **Reasoner interface:** the pluggable LLM seam.

### Lives in Capex Scout (surface-specific)

- Mobile UI / IBM Plex Mono + Instrument Serif aesthetic
- Status bucket workflow (New / Watchlist / Acted / Skip)
- Watchlist promote and dismiss logic *as user feedback signals* — the data goes back to core as feedback events
- Time horizon as first-class field (TRADE / SWING / HOLD / LONG)
- Verdict pill display
- Provenance hover UI
- Mobile-first interaction patterns (tap to expand, swipe, copy ticker)

### Lives in Portfolio Intelligence (surface-specific)

- Desktop UI / Geist + JetBrains Mono aesthetic, neutral palette
- Action Queue with impact preview math
- Hierarchical Portfolio table (baskets → positions, target/current/drift columns)
- Per-position quick Add / Trim panel with realized vs unrealized P&L
- Robinhood-style analyst ratings widget with firm-level breakdown
- Theme universe drill-down with sub-themes
- Positions table (53 holdings, today's-mover sort)
- Hero strip (portfolio value, today's P&L, YTD)
- Notes per position (free-text)
- Promote-to-active-basket flow with basket picker
- 130/30 long/short structure (planned)
- Target weights, drift thresholds, rebalance triggers (planned)
- Aggression dial / model-driven targets (planned)
- Options strategist module (planned)
- IBKR order routing (planned, v2.0)

## 5. Data primitives (Pydantic contracts that should be shared)

The following Pydantic models are defined in Capex Scout's `contracts.py`. They are the wire format for any cross-product communication. Any future API service exposing core to PI will return these shapes.

```python
class SourceItem:        # raw text from any source
    id: str
    source: str          # 'reuters', 'edgar_form4', 'x', 'reddit', 'transcript', 'options_flow'
    published_at: datetime
    text: str
    tickers_mentioned: list[str]

class SignalScores:      # composite confidence components
    momentum: float
    valuation: float
    catalyst: float
    sentiment_accel: float
    balance_sheet: float
    flow: float
    # composite() applies 25/20/20/15/10/10 weights

class Signal:            # a structured investable observation
    id: str
    type: SignalType     # see signal kinds above
    theme: str
    primary_ticker: str
    second_order_tickers: list[str]
    what_changed: str
    why_now: str
    scores: SignalScores
    confidence: float
    evidence: list[SourceItem]
    # status field (new/watchlist/acted/skip) lives in the persistence layer

class ResearchBrief:     # the analyst-grade thesis on a signal
    signal_id: str
    primary_ticker: str
    what_company_does: str
    why_matters: str
    why_now: str
    bull: str
    base: str
    bear: str
    catalysts: list[str]
    headwinds: list[str]
    valuation_read: str
    momentum_read: str
    crowding_score: float
    balance_sheet_health: str
    comparables: list[str]
    related_themes: list[str]

class Basket:            # tradeable expression of a theme
    name: str
    theme: str
    tickers: list[str]
    weights: dict[str, float]
    confidence: float
    rationale: str
    signal_ids: list[str]
```

PI's existing data structures (positions, target weights, agent ideas, analyst ratings) are not in core — they are PI-specific.

The **bridge** between core and PI is `Signal.id` and `ResearchBrief.signal_id`. Whenever PI shows an "agent idea" that originated from core, it carries the underlying `signal_id`, which lets PI fetch the full brief, evidence trail, and provenance.

## 5b. Persistence schema (shipped May 9, 2026)

CS shipped a SQLite-backed persistence layer. The table shapes below are the canonical structure for any future PI-side data layer that needs to read or write through core. When core extracts to a service in v2.0, these tables become the Postgres schema (or are surfaced as REST/GraphQL endpoints derived from them).

PI's surface should not directly read CS's SQLite file — instead, plan for an API in front of these tables. But the *shapes* are binding.

```sql
signals (
  id TEXT PRIMARY KEY,                 -- short uuid
  type TEXT NOT NULL,                  -- SignalType literal
  theme TEXT NOT NULL,
  primary_ticker TEXT NOT NULL,
  second_order_tickers TEXT,           -- JSON array
  what_changed TEXT,
  why_now TEXT,
  scores_json TEXT,                    -- full SignalScores as JSON
  confidence REAL NOT NULL,
  status TEXT NOT NULL DEFAULT 'new',  -- new | watchlist | acted | skip
  first_seen_at TIMESTAMP NOT NULL,
  last_seen_at TIMESTAMP NOT NULL,
  run_count INTEGER NOT NULL DEFAULT 1
)

evidence (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  signal_id TEXT NOT NULL REFERENCES signals(id) ON DELETE CASCADE,
  source_item_id TEXT NOT NULL,
  source TEXT NOT NULL,
  published_at TIMESTAMP,
  text TEXT NOT NULL,
  tickers_mentioned TEXT,              -- JSON array
  UNIQUE(signal_id, source_item_id)
)

feedback (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  signal_id TEXT NOT NULL REFERENCES signals(id) ON DELETE CASCADE,
  from_status TEXT,
  to_status TEXT NOT NULL,
  note TEXT,
  created_at TIMESTAMP NOT NULL
)

llm_calls (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  model TEXT NOT NULL,
  purpose TEXT NOT NULL,               -- 'extract_themes' | 'synthesize_brief'
  input_hash TEXT NOT NULL,            -- sha256 for dedup
  input_tokens INTEGER,
  output_tokens INTEGER,
  cost_usd REAL,
  latency_ms INTEGER,
  cached BOOLEAN DEFAULT 0,
  created_at TIMESTAMP NOT NULL
)
```

### Conventions locked with this schema

**Logical-key dedup on signals.** Upsert dedup is by `(type, theme, primary_ticker)`, NOT by surrogate uuid. Reason: agents mint a fresh uuid per run, so id-based dedup would never match. Implication: if the same theme fires on the same ticker from multiple sources on the same day, they merge into one `signals` row with stronger evidence (which is correct behavior — multiple corroborating sources = higher credibility, not duplicate signals). For per-fire-event granularity (e.g., "this theme has fired 47 times this month"), count rows in the `evidence` table, not the `signals` table.

**UTC for all "today's data" comparisons.** SQLite's `CURRENT_TIMESTAMP` writes UTC. Python's `date.today()` is local. Anywhere the system asks "what happened today," the date string must be computed via `datetime.now(timezone.utc).date().isoformat()`. Both products inherit this. Don't mix local-tz dates with UTC timestamps.

**Status preservation across runs.** When the pipeline re-runs, `upsert_signal` updates `last_seen_at`, `run_count`, and content fields, but *does not* reset `status`. A signal a user moved to `watchlist` stays in `watchlist` even after fresh data ratifies it again. Same applies for `acted` and `skip`.

**Persistence boundary.** All DB operations go through `db/store.py` functions in CS. Pydantic models stay framework-agnostic; the store layer knows how to convert `Signal` → row. PI's eventual data layer should mirror this discipline — no raw SQL in surface code, all DB access through a thin store module.

## 6. Naming reconciliation

PI and CS have diverged on some names. These are the canonical names going forward (use these in core; surfaces can rename for display):

| Concept | Canonical (core) | PI's older name | CS's older name |
|---|---|---|---|
| Operator decision | Verdict | Action / Recommendation | Verdict |
| Buy/sell direction | BUY / SELL / WATCH / SKIP | BUY / SELL / HOLD / WATCH | BUY / SELL / WATCH / SKIP |
| Time-to-thesis | Horizon | (not yet first-class) | Horizon (Trade/Swing/Hold/Long) |
| Idea source agent | Source agent | Newsroom / X Pulse / Insider Monitor / Earnings Tracker / Macro Lens / Quant Signals | NewsAnalyst / EdgarAnalyst / SocialAnalyst / RedditAnalyst / TranscriptAnalyst / OptionsFlowAnalyst |
| Signal evidence trail | Provenance | "What's driving the call" | Provenance hover |

Reconciliation rule: PI's display names ("Newsroom Agent" etc.) are friendlier to operators and should win on the surface. The internal class/identifier names in core should stay short and code-friendly (`NewsAnalyst`). Keep both, map between them at the surface layer.

**Verdict canonical: BUY / SELL / WATCH / SKIP.** Decided May 9, 2026.

Rationale: these are *direction primitives*, not position actions. A signal expresses directional read; the surface translates to operator action.
- SELL covers both "exit a long" and "open a short" — works in cash-only and 130/30 contexts equally
- SKIP is clearer than HOLD as a "pass" verdict (HOLD reads as "currently holding" not "no action recommended")

PI's surface keeps its action vocabulary (ADD / HOLD / TRIM / EXIT) but maps from direction primitives:
- BUY + no position → ADD
- BUY + already held → HOLD (don't add more)
- SELL + currently long → TRIM or EXIT (PI sizes the difference)
- SELL + no position → SHORT (only in 130/30 mode; otherwise WATCH)
- WATCH → no action; surface in watchlist
- SKIP → no action; surface dismissed

Per-ticker calls inside PI's Research tab switch to direction primitives (BUY/SELL/WATCH/SKIP) since they describe signal direction, not position action.

## 7. Workflow integration: how the two surfaces talk

The intended cross-surface flow:

1. **Morning, mobile.** Operator opens Capex Scout. Sees overnight signals. Taps WATCH on a PLTR insider buy. → Core stores `feedback{signal_id, from_status='new', to_status='watchlist', timestamp}`.

2. **At desk, desktop.** Operator opens PI. PI's Action Queue queries core for `signals where status='watchlist'` joined against current positions. PLTR appears in the queue with sizing context: "PLTR — 0% of book, suggested entry $2K, will bring AI software exposure to 8%."

3. **Operator hits Yes.** PI executes the trade (paper or IBKR), writes to its own `action_history`, and pushes a `feedback{signal_id, from_status='watchlist', to_status='acted', note=...}` event back to core.

4. **Core uses the feedback for calibration.** Across all signals where status='acted', track 30/90/180-day forward returns. This is the calibration backtest that grounds confidence scores. Both products benefit.

5. **Skipped/dismissed signals downweight in core.** If the operator dismisses Cooling Bottleneck signals three times, core lowers the cluster confidence threshold for that theme. Both products see the recalibrated weights.

This loop is what makes the products genuinely co-evolutionary rather than redundant.

## 8. Build sequence and dependencies

**Capex Scout state (May 9, 2026):**
- ✅ Pipeline architecture stable
- ✅ Pydantic contracts in place
- ✅ Real Anthropic LLM wired and validated
- ✅ Real EDGAR fetcher live
- ✅ SQLite persistence + status field shipped (Section 5b)
- ✅ LLM cost tracking with input-hash dedup foundation
- ⏳ Provenance UI design + implementation
- ⏳ Theme canonicalization (handle LLM ontology drift)
- ⏳ Ticker hallucination fix (constrain LLM to source-text or canonical theme tickers)
- ⏳ Wire Finnhub for NewsAnalyst real data
- ⏳ Crude backtest harness for calibration

**Portfolio Intelligence's near-term (parallel work):**
- Continue React mockup polish (positions table, action queue, research tab depth, search)
- Add `signal_id: str | None` field to PI's idea shape — null until core API exists, non-null after
- Confidence stored as float 0..1 internally; "high/med/low" is a display transform only
- Shape PI's data layer so its hardcoded constants could be replaced by API calls. Don't actually replace yet. Just keep the data access pattern abstract enough that the swap is doable in v2.0.

**Convergence point (v2.0, both products):**
- `capex-core` extracted as a Python package (or a FastAPI service)
- PI's React frontend ships to Vercel; backend in Python on Vercel/Supabase
- PI's backend imports from `capex-core` (or HTTPs to it)
- CS's existing Python pipeline stays as-is; just imports from the now-extracted `capex-core` instead of having the code inline
- Both products read and write to the same Postgres (Supabase) database — table shapes from Section 5b
- IBKR integration only on PI's surface

## 9. What this alignment doc commits each product to

**Capex Scout commits to:**
- Keep `contracts.py`, `reasoner.py`, `agents.py`, `scorer.py`, `synthesizer.py`, `basket.py` framework-agnostic — no UI imports, no surface-specific logic
- Keep persistence access through `db/store.py`; all SQL stays there
- When extracting `capex-core` in v2.0, preserve the Section 5b table shapes as the API surface
- Surface PI-relevant fields in API responses (signal_id, evidence, provenance)
- Coordinate any change to signal scoring weights, theme ontology, or persistence schema with PI

**Portfolio Intelligence commits to:**
- Treat agent ideas as views over core signals — don't reimplement source-agent extraction
- Preserve `signal_id` as the link back to core
- Don't define a new signal kind without checking core's kind list
- Coordinate any change to verdict semantics with CS
- When backend lands, mirror CS's "all DB access through a thin store module" discipline

**Both products commit to:**
- This doc is the source of truth on the architecture
- Significant divergences from this doc require an explicit Richard-approved update to this doc, not silent drift in code
- Reconcile theme names, agent names, and verdict names early — every week of drift makes the eventual merge more expensive
- UTC for all "today's data" comparisons (Section 5b)

## 10. Open questions Richard hasn't decided

These are flagged so neither Claude chat invents answers:

- **Customer model.** Is Capex Scout a future Atomic Insights product (subscription, multi-tenant) or a personal companion to PI (single-user, Tailscale-on-Mac)? This determines auth, billing, infrastructure scaling. Defer answering. Don't make decisions today that foreclose either path.
- **Verdict naming canonical.** ~~BUY/SHORT/WATCH/SKIP vs BUY/SELL/HOLD/WATCH — pick one before merge work.~~ **Resolved May 9, 2026: BUY / SELL / WATCH / SKIP.** See section 6 for surface mapping.
- **Theme canonicalization.** PI uses "AI Capex Beneficiaries"; CS uses "AI Infrastructure". Same theme, different label. Both products need to migrate to one canonical name. CS is doing canonicalization work first (theme ontology v2 in progress); PI should adopt CS's canonical labels once landed. Open question on hierarchy: flat vs two-tier, broader-than-AI coverage, agent-emergent themes vs strict closed-set.
- **Per-ticker scoring weights.** PI is iterating on per-ticker BUY/HOLD/SELL conviction logic. CS's scorer uses 25/20/20/15/10/10 composite. These need to converge into one shared scoring function in core.
- **Macro Lens and Quant Signals agents.** PI references both as named agents; CS hasn't built either yet. Build location: core. Both products consume.
- **Crypto.** PI has crypto pending (Robinhood Crypto export). CS has not addressed crypto. Out of scope for v1 alignment; revisit when PI integrates it.
- **Caching layer for LLM calls.** CS shipped `llm_calls.input_hash` as a dedup foundation but the cache lookup function is a stub returning None. Real caching not yet wired. Not blocking; revisit when ingestion volume grows.

## 11. The merge fear, addressed

A specific concern from Richard worth surfacing: he has fine-tuned indicators, scoring details, and prompts independently in both products and is worried that consolidating into core will clobber the work he's done.

**This is a valid fear and the architecture above addresses it explicitly.**

Consolidation is not "pick one and delete the other." It is, per indicator/prompt/threshold:
1. Inventory both versions
2. Mark which is more battle-tested (PI's are likely further along — more iteration than CS's mock-stage prompts)
3. Decide canonical version per item: sometimes PI's, sometimes CS's, sometimes a synthesis
4. Move to core, delete duplicates
5. Re-test both surfaces against the canonical version

This is **deliberate, reversible, per-indicator** work, not a bulk merge. Estimate: weeks of work spread over several sessions when both products hit v1.x maturity. Not days. Not v1.

The cost of *not* doing this consolidation is permanent ontology drift — same theme means two different things in two products forever. That breaks the promise of "one workflow across two surfaces."

---

## How to use this doc in either project

If you are a Claude chat in either Capex Scout's project or Portfolio Intelligence's project:

- Treat sections 3-9 as binding architecture. Do not propose changes to the boundary between core and surfaces without flagging it as a deviation.
- Treat section 5b as the canonical persistence schema. Surface-specific code never reads/writes raw SQL against core tables — always through a store layer.
- Treat section 10 as known-unknowns. Do not invent answers; ask Richard.
- When Richard says "sync from alignment doc" or similar, re-read this in full before responding.
- When you make a change in either product that affects the boundary (a new signal kind, a renamed agent, a changed scoring weight, a new persisted column), flag it and recommend updating this doc.

End of alignment doc.
