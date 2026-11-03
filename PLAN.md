# PLAN.md — Promo Pricing Agent

## Status (session handoff — update at end of each session)
- **Done:** CLAUDE.md, PLAN.md drafted. Open decisions (§4) accepted as defaults: Claude via Anthropic SDK with hand-written loop; maximize margin; abstention key by scenario design; pandas + statsmodels; constant elasticity for v1.
- **Done (Phase 0, 2026-11-03):** git repo (main), `.venv` (Python 3.14.4), editable install of `promo_agent` (src layout, hatchling), dev group (ruff, pytest) via `pip install -e . --group dev`. `ruff check`, `ruff format --check`, `pytest` (1 smoke test) all pass. Runtime deps intentionally empty; add per phase. Committed (627f11b, amended). PostToolUse hook in `.claude/settings.json` runs ruff format on edited .py files.
- **Next:** add GitHub remote + push, then Phase 1 (synthetic data generator).
- **Phase 4 note:** turn on prompt caching for the system prompt + tool definitions in the agent's API calls; log cache hits alongside the other token counts.
- **Open issues:** none.

## 1. Core design principles

1. **Tools are deterministic; the LLM only orchestrates and explains.**
   All the statistics (elasticity estimation, simulation, optimization) live in plain
   Python functions that are unit-tested without any LLM. The agent decides *which*
   tools to call, with *what* arguments, and how to explain the result.
   *Why:* you can verify the econometrics separately from the agent, and when the
   agent gets an answer wrong you can tell whether the math or the orchestration failed.

2. **Guardrails go in code first, and the prompt repeats them.**
   - `estimate_elasticity` returns an explicit `reliable: bool` plus the reasons
     (CI too wide, too few distinct price points, too little price variation).
   - `optimize_discount` only searches discounts within the observed range, so it
     cannot extrapolate even if the LLM asks it to. It reports when the optimum
     sits at that boundary.
   - The system prompt tells the agent to abstain and state its assumptions.
     Evals check whether it actually does.
   *Why:* a prompt rule alone is not a guarantee. The interview answer is "the
   guardrails are enforced in code, and the evals measure how well the agent follows them."

3. **Synthetic data gives exact ground truth.**
   The generator knows each SKU's true elasticity, so it also knows the true
   profit-maximizing discount within the allowed range. Every eval question has an answer key.

4. **A non-agent baseline ("oracle pipeline").**
   A fixed script that calls the tools in the correct order. This splits the
   agent's error into two parts:
   - *estimation error*: oracle vs truth (a statistics problem)
   - *orchestration error*: agent vs oracle (an agent problem)

   It also answers the question "why do you need an LLM here at all?"

## 2. Proposed repo structure

```
pricing-agent/
├── CLAUDE.md
├── PLAN.md
├── README.md                  # results only from code that ran
├── pyproject.toml             # deps, ruff, pytest config
├── .gitignore  .env.example   # API key goes in .env (never committed)
├── src/promo_agent/
│   ├── data/
│   │   ├── synthetic.py       # demand generator + ground-truth optimal discount
│   │   └── scenarios.py       # SKU archetypes (well-identified, sparse, noisy, inelastic…)
│   ├── tools/
│   │   ├── sales.py           # get_sales_history
│   │   ├── elasticity.py      # estimate_elasticity (log-log OLS, robust CI, diagnostics)
│   │   ├── simulate.py        # simulate_promo
│   │   ├── optimize.py        # optimize_discount (bounded to observed depths)
│   │   └── registry.py        # tool JSON schemas exposed to the LLM
│   ├── agent/
│   │   ├── loop.py            # hand-written tool-use loop (no framework)
│   │   ├── prompts.py         # system prompt: assumptions, abstention policy
│   │   └── answer.py          # structured final answer schema (pydantic)
│   └── oracle.py              # deterministic baseline pipeline
├── evals/
│   ├── build_eval_set.py      # generates questions + answer keys from scenarios
│   ├── run_evals.py           # runs oracle and/or agent, writes traces + metrics
│   ├── metrics.py
│   └── results/               # dated run outputs (summaries committed, raw traces ignored)
└── tests/                     # pytest unit tests for data + tools
```

## 3. Phased milestones

Each phase ends with a short review before the next one starts.

### Phase 0 — Scaffolding
- `git init`, check git identity, venv, `pyproject.toml`, ruff, pytest, `.gitignore`.
- **Done when:** `pytest` runs (with zero tests) and ruff passes.

### Phase 1 — Synthetic data generator with ground truth
- Weekly sales per SKU: `log Q = α + β·log P + seasonality + ε`, with list price,
  unit cost, and a promo calendar of discount depths.
- Scenario archetypes:
  - well-identified elastic
  - inelastic (|β| < 1, so the true optimum is no discount)
  - sparse price variation (e.g. only 1–2 distinct depths)
  - high noise
  - optimum outside the observed discount range (tests the "no extrapolation" rule)
- Ground truth: the true profit-maximizing discount within `[0, max observed depth]`,
  computed from the true β. For constant elasticity this has a closed form
  (Lerner rule, p* = c·β/(1+β)), clamped to the allowed range. A grid search
  cross-checks it.
- **Done when:** unit tests confirm the generator reproduces its own parameters
  and the closed-form optimum matches the grid search.

### Phase 2 — Deterministic tools
- `get_sales_history(sku, start, end)`
- `estimate_elasticity(sku)` returns:
  - β̂ and its 95% CI (heteroskedasticity-robust standard errors)
  - diagnostics: number of distinct price points, std of log price, share of weeks on promo
  - `reliable` flag and the reasons behind it
- `simulate_promo(sku, discount, elasticity)` returns expected units, revenue,
  and margin, with bands carried over from the elasticity CI.
- `optimize_discount(sku, ...)` searches only `[0, max observed depth]` and
  returns the optimum, the profit curve, and a `hit_boundary` flag.
- **Statistical validation (Monte Carlo):** across many simulated datasets,
  check that β̂ is unbiased and the 95% CI covers the true β about 95% of the time.
- **Done when:** the tests pass and the Monte Carlo coverage numbers come from a real run.

### Phase 3 — Oracle pipeline and eval set
- `oracle.py` calls the tools in a fixed order with fixed abstention rules.
- `build_eval_set.py`: 30–50 questions across the scenario types. Each answer key
  holds the true optimal discount (or "should abstain") and the tools that must be called.
  - Plain-language variety, e.g. "what discount for SKU 12?" or "is 30% off on SKU 7 a good idea?"
  - A few adversarial questions, e.g. "what about 60% off?" when 60% was never observed.
- `run_evals.py` + `metrics.py`, scored on the oracle first:
  - |recommended − true optimal discount|
  - abstention precision and recall
  - tool-call correctness: required tools called, arguments valid
  - cost and latency per query (zero for the oracle)
- **Done when:** the oracle baseline metrics exist from a real run.

### Phase 4 — The agent
- Hand-written tool-use loop on the Anthropic Messages API: one function, readable
  end to end, with a cap on iterations.
- System prompt covers the role, the abstention policy, the no-extrapolation rule,
  and a required assumptions section.
- Structured final answer: `{decision: recommend|abstain, discount, ci, rationale, assumptions[]}`.
- Full trace logging: every tool call, its arguments, token usage, and latency.
- **Done when:** the agent answers a handful of questions end to end and the traces are readable.

### Phase 5 — Agent evals and error analysis
- Run the full eval set and compare the agent against the oracle and the ground truth.
- Classify failures: wrong tool, bad arguments, ignored the `reliable` flag,
  hallucinated a number, missing assumptions.
- Make one prompt or tool-description change at a time and re-run the evals after each.
- Optional: compare models on accuracy vs. cost and latency.
- Run every configuration with repeated seeds, since LLM outputs vary from run to run.
- **Done when:** a results table produced by `run_evals.py` is checked into `evals/results/`.

### Phase 6 — Real public data (stretch)
- Candidate: Dominick's Finer Foods scanner data (Chicago Booth, public), or a
  Kaggle retail dataset. To be confirmed.
- Real data has no ground-truth elasticity, so evals change to holdout demand
  prediction, abstention behavior, and qualitative review.
- Discuss price endogeneity: retailers set promos based on expected demand.
  The synthetic data is exogenous by design. Optionally add an endogenous scenario to show the bias.

### Phase 7 — README and write-up
- Architecture diagram, design decisions, eval results (only numbers that ran), and limitations.

## 4. Open decisions for you

1. **LLM provider.** I assumed Claude via the Anthropic Python SDK, with a
   hand-written loop rather than LangChain or another framework.
   *Why:* every step stays visible and defensible in an interview.
2. **Objective.** Maximize gross margin (needs unit cost), revenue, or units?
   I assumed margin, because it gives a well-defined optimum.
3. **How the "should abstain" answer key is defined.** Option (a): by how the
   scenario was built (e.g. sparse prices means abstain). Option (b): by a rule
   applied to the data (e.g. fewer than k distinct prices, or CI wider than w).
   I lean toward (a) and report how well (b) matches it.
4. **Data stack.** I assumed pandas + statsmodels; the data is small. PySpark would
   show your background, but at this scale it adds overhead without adding to what we learn.
5. **Promo model scope.** Should v1 use constant elasticity only, or also
   include promo-specific effects (display/feature lift, post-promo dip)?
   I suggest constant elasticity for v1 and adding these later.
