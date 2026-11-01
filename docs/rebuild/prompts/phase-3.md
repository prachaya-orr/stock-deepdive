# Phase 3 — period engine, metrics, model export

Branch: `feature/p3-model`. First follow `docs/rebuild/prompts/_COMMON.md`. Requires P1 and P2 merged.

## Goal
A deterministic financial model built from facts. Every metric is traceable to its input facts and exported as `model.json` and as `model.xlsx` with live formulas.

## Tasks
1. **Period engine:**
   - Normalise facts to periods.
   - Derive discrete quarters from year-to-date values by subtraction, and derive TTM.
   - Derived facts get tier `derived`, `extraction: script`, and `inputs`.
   - Handle a missing Q1 (no derivation, with a caveat) and non-December fiscal years.
2. **Sign normalisation** uses `taxonomy/items.yaml`.
3. **Metric registry** `src/sdp/metrics.py` covers every metric in CONTRACTS §4. Each has a formula string, unit, required inputs and a pure function. A metric with missing inputs gives `value: null` plus a caveat.
   - `net_debt` and `net_debt_incl_perpetual` both exist.
   - `common_earnings`, `eps_common`, `bvps_ex_perpetual` and `payout_ratio_common` capture the BGRIM finding: perpetual debenture interest is deducted from parent earnings.
4. `sdp model --run <id>` writes `model.json`, and G4 checks that derivations reconcile.
5. **`model.xlsx`** has sheets `facts`, `periods`, `metrics`, `assumptions` (placeholder), `valuation` (placeholder). Metric cells are **formulas referencing fact cells**, not pasted values. Add a test that recomputes them (LibreOffice headless if available, otherwise formula-string assertions) and compares to `model.json`.

## Acceptance
Every bullet under "P3 model" in ACCEPTANCE.md.

## Stop and ask if
- A metric's definition in CONTRACTS is ambiguous for a real fixture case (for example whether `finance_costs` should include FX). Propose a definition and add a DECISIONS entry in the PR.
