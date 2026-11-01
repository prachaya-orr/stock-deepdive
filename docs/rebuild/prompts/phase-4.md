# Phase 4 — market data, assumptions, valuation

Branch: `feature/p4-valuation`. First follow `docs/rebuild/prompts/_COMMON.md`. Requires P3 merged.

## Goal
Compute every football-field bar and the reverse-DCF result from the model, market data and explicit assumptions.

## Tasks
1. `agents/market-collector.md` (tools: Read, Write, and the TradingView MCP tools; loading their schemas via ToolSearch is part of the prompt):
   - Collect the fields in CONTRACTS §5.1 for the ticker, the peers and the 10Y Thai government bond.
   - Write `market.json`. Missing fields are `null` with a note. Never estimate.
   - Every value is `secondary_market` with `as_of`.
2. `agents/assumptions-proposer.md` (tools: Read, Write, TradingView MCP read-only):
   - Propose peers (at least 3, with a rationale each) and any non-default DCF inputs, with a rationale.
   - Write `assumptions.proposed.yaml` in the run dir. Never write to `config/`.
3. `config/defaults.yaml` and `config/tickers/BGRIM.yaml` (peers GULF, GPSC, EGCO, RATCH, BPP as the initial proposal, `exclude_perpetual_from_equity: true`).
   - Resolve the final assumptions as defaults ⊕ ticker ⊕ accepted proposals and write `assumptions.resolved.yaml`.
   - The conductor asks the user to accept or reject proposals. P6 wires this in, and P4 provides the merge function.
4. `sdp market-validate` checks the schema and staleness. It never fills values.
5. `sdp valuation` follows SPEC §5.8:
   - 52-week range, P/E band, P/BV band (equity basis per `equity_adjustments`), peer EV/EBITDA, DCF (FCFF) with a sensitivity grid, and reverse DCF (solved numerically). Peer P/E and DDM are opt-in.
   - Write `valuation.json`, with `unavailable` and reasons.
6. G5 range checks.

## Acceptance
Every bullet under "P4 valuation" in ACCEPTANCE.md.

## Stop and ask if
- TradingView MCP cannot provide multiples history. Propose either deriving it from price history plus reported EPS/BVPS in `model.json`, or dropping the band methods for v1.
