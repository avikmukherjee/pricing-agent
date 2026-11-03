# CLAUDE.md

## About me
Senior Data Scientist (~9 years) in pricing, promotion, and decision science: causal inference, price/promo elasticity, optimization, PySpark. M.S. Operations Research. New to agentic AI. This project is how I'm learning it properly.

## Goal
A portfolio project I can defend line by line in senior DS / applied AI interviews. Understanding matters more than speed.

## How to work with me
- Use plan mode for anything non-trivial; propose, then wait for my approval.
- Explain design decisions and trade-offs, especially agent design, evals, and guardrails.
- Small, reviewable changes. No large code dumps.
- Never invent results or numbers. Every metric in the README must come from code that actually ran.
- Environment: WSL2 Ubuntu, Python with venv, git with my personal identity.

## Efficiency
- One phase/task per session; I /clear between. PLAN.md "Status" section is the handoff.
- Concise by default; I'll ask for depth.
- Read only needed files/ranges; keep command output minimal; no subagents.
- Agent API calls are billed separately: dev on small subsets + cheap model.

## Project: promo pricing agent
An LLM agent that answers "what discount should we run on SKU X?" by calling my tools: get_sales_history, estimate_elasticity (log-log with CI and a sparse-price-variation diagnostic), simulate_promo, optimize_discount.
Guardrails: abstain when elasticity is too uncertain or price variation is sparse; never extrapolate beyond observed discount depths; always state assumptions.
Data: synthetic generator with known true elasticities (ground truth for evals), with real public data later.
Evals: 30-50 questions with known answers, measuring distance from the true optimal discount, correct abstention rate, tool-call correctness, cost/latency per query.
