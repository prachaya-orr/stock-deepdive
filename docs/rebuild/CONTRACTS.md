# CONTRACTS — formats every component must honour

Changing anything here requires a decision entry in [DECISIONS.md](DECISIONS.md) and a schema version bump.
Machine-readable schemas live in `plugins/stock-deepdive-pro/schemas/*.schema.json` (created in P0). This document is their human description.
If the two disagree, the JSON Schema wins and this document is fixed.

## 1. Directory layout

```
<workspace>/                          user's cwd
  data/<TICKER>/                      raw cache, reusable across runs (git-ignored)
    fetch.json                        fetch ledger (§10.2)
    FS_<period_end>_<filing_id>/      one financial-statement package per filing
    R561_FY<year>_<filing_id>/        one 56-1 One Report per fiscal year
    MANIFEST.md                       human index (all files, incl. unconvertible)
  runs/<TICKER>/<run_id>/             one directory per run (git-ignored); run_id = UTC yyyymmddThhmmssZ
    facts.jsonl                       §3
    model.json                        §4
    market.json                       §5.1 (optional)
    assumptions.resolved.yaml         §5.2 (defaults ⊕ ticker file ⊕ accepted proposals)
    assumptions.proposed.yaml         agent proposals with rationale (never auto-applied to config/)
    valuation.json                    §5.3
    writer/                           ONLY these inputs are given to writers:
      model.json  valuation.json  facts.jsonl  sections.md
    parts/10-a.md 20-b.md 30-c.md     writer output (§8.1)
    claims/claims-a.jsonl …           §6
    critic.json                       §6.2
    report.html  model.xlsx           deliverables
    run.json                          §11
plugins/stock-deepdive-pro/config/
    defaults.yaml                     global assumption defaults
    tickers/<TICKER>.yaml             per-ticker overrides (peers, DCF inputs) — user-owned
```

## 2. Common enums

| Enum | Values |
|---|---|
| `tier` | `filed` · `management_statement` · `headline` · `secondary_market` · `computed` · `derived` |
| `basis` | `consolidated` · `separate` |
| `statement` | `bs` · `is` · `cf` · `se` · `note` · `annual_report` · `market` |
| `period_type` | `instant` · `duration` |
| `extraction` | `parser` · `agent` · `script` (computed/derived) · `user` |
| `currency` | ISO 4217, v1 always `THB` |

`tier` meaning: `filed` = number in a regulator-filed document · `management_statement` = management's words/targets even if inside a filing (MD&A guidance, press-release quotes) · `headline` = news · `secondary_market` = market data vendors · `computed` = model metric · `derived` = period arithmetic on filed values.

## 3. `facts.jsonl`

### 3.1 Record

```json
{
  "id": "is.net_profit_parent.c.2025-12-31.12m",
  "entity": "BGRIM",
  "statement": "is",
  "item": "net_profit_parent",
  "label_raw": "Profit attributable to owners of the parent",
  "value": 1675487076,
  "value_text": null,
  "currency": "THB",
  "scale": 1,
  "period_type": "duration",
  "period_start": "2025-01-01",
  "period_end": "2025-12-31",
  "basis": "consolidated",
  "tier": "filed",
  "extraction": "parser",
  "source": {"file": "FS_2025-12-31_1339FIN2402202618465/FINANCIAL_STATEMENTS.xlsx",
             "sheet": "EN_P&L9", "cell": "G41", "page": null, "line": null, "note": null},
  "audited": true,
  "unit_corrected": false,
  "conflicts_with": [],
  "inputs": [],
  "notes": null
}
```

Rules:
- `value` is the number **as printed**; `scale` is the printed unit multiplier (1, 1000, 1000000). Absolute = `value × scale`. Never pre-convert.
- Exactly one of `value` / `value_text` is non-null (`value_text` for qualitative facts e.g. audit opinion type).
- `source` must locate the fact: xlsx → `sheet`+`cell`; text → `file`+`line`; PDF → `file`+`page`; note → `note` number.
- `derived`/`computed` facts list their `inputs` (fact IDs) and have `source.file = null`.

### 3.2 IDs

`<statement>.<item>.<basis-letter>.<period_end>[.<months>m]` — basis `c`/`s`; months only for durations.
Raw (unmapped) rows: `item = "raw.<slug-of-label>"`. Notes: `note.<topic>.<slug>.<period_end>`. Market: `mkt.<field>.<as_of>`.
Collisions (same ID from two sources) → keep both with suffix `#2`, set `conflicts_with` on each, and the validator reports them.

### 3.3 Canonical items (v1 taxonomy)

Defined in `taxonomy/items.yaml` (key, statement, sign convention, description) and `taxonomy/synonyms.yaml` (EN/TH label patterns). v1 keys:

**Balance sheet (instant):** `cash` · `st_investments` · `restricted_deposits` · `trade_receivables` · `inventories` · `current_assets` · `ppe` · `rou_assets` · `intangibles` · `investments_assoc_jv` · `loans_to_related` · `total_assets` · `st_borrowings` · `current_ltd` · `trade_payables` · `current_liabilities` · `lt_borrowings` · `debentures` · `lease_liabilities` · `total_liabilities` · `share_capital_paid` · `perpetual_debentures` · `equity_parent` · `nci` · `total_equity` · `shares_outstanding`

**Income statement (duration):** `revenue` · `revenue_sales` · `cost_of_sales` · `gross_profit` · `sga` · `other_income` · `fx_gain_loss` · `finance_costs` · `share_of_assoc_profit` · `ebt` · `income_tax` · `net_profit` · `net_profit_parent` · `net_profit_nci` · `eps_basic`

**Cash flow (duration):** `cfo` · `interest_paid` · `tax_paid` · `capex` · `acquisitions` · `cfi` · `dividends_paid_parent` · `dividends_paid_nci` · `perpetual_interest_paid` · `borrowings_proceeds` · `borrowings_repaid` · `cff` · `net_change_cash`

Sign convention: store as printed (outflows negative if printed negative); `items.yaml` declares expected sign so the model normalises.

### 3.4 Note items (agent-extracted)

Required topics (cap 80 facts total): `segment.revenue` (by segment/customer, e.g. EGAT vs industrial) · `segment.volume` · `segment.price` · `related_party.*` (balances & transactions, top 10 by size) · `commitments.*` · `contingencies.*` · `guarantees.*` · `covenants.*` (definition text + threshold) · `shareholders.top` (top 10 %) · `remuneration.board` · `remuneration.executives` · `capacity.installed` · `capacity.attributable` · `capacity.pipeline` · `audit.opinion` · `audit.kam` · `perpetual.*` (amount, coupon, call date) · `company_defined.ebitda` / `company_defined.core_profit` (tier `management_statement`).

## 4. `model.json`

```json
{
  "schema": "model/1",
  "entity": "BGRIM",
  "periods": [{"key": "FY2025", "start": "2025-01-01", "end": "2025-12-31", "kind": "annual"},
              {"key": "Q2-2026", "start": "2026-04-01", "end": "2026-06-30", "kind": "quarter", "derived": true},
              {"key": "TTM-2026Q2", "start": "2025-07-01", "end": "2026-06-30", "kind": "ttm", "derived": true}],
  "metrics": [
    {"id": "m.common_earnings.FY2025", "metric": "common_earnings", "period": "FY2025",
     "value": 755487076, "unit": "THB", "formula": "net_profit_parent - perpetual_interest",
     "inputs": ["is.net_profit_parent.c.2025-12-31.12m", "note.perpetual.interest.2025-12-31"],
     "caveats": []}
  ]
}
```

Metric registry (`src/sdp/metrics.py`, each with formula string + unit + required inputs). v1 set:
`revenue_growth_yoy` · `gross_margin` · `ebit_margin` · `net_margin` · `cfo_to_net_profit` · `fcf` (cfo + capex) · `fcf_to_equity` (fcf − interest_paid − perpetual_interest − dividends_nci) · `net_debt` (borrowings + debentures + leases − cash − st_investments) · `net_debt_incl_perpetual` · `net_debt_to_equity` · `net_debt_to_ebitda` · `interest_coverage` · `common_earnings` · `eps_common` · `bvps` · `bvps_ex_perpetual` · `payout_ratio_common` · `roe_common` · `ebitda_reported` (from `company_defined.ebitda`, flagged) · `ebitda_computed` (if derivable).
A metric whose inputs are missing is emitted with `value: null` and `caveats: ["missing: <ids>"]` — never omitted.

## 5. Market, assumptions, valuation

### 5.1 `market.json` (agent-written, script-validated)

```json
{"schema": "market/1", "as_of": "2026-10-30", "source": "TradingView MCP", "symbol": "SET:BGRIM",
 "price": 18.80, "currency": "THB", "shares_outstanding": 2606900000, "market_cap": 49009716175,
 "price_history": [{"date": "2021-11-01", "close": 0.0}],
 "multiples_history": {"pe": [{"date": "…", "value": 0.0}], "pbv": []},
 "peers": [{"symbol": "SET:GPSC", "price": 0, "market_cap": 0, "ev": 0, "ev_ebitda": 0, "pe": 0, "pbv": 0, "as_of": "…"}],
 "risk_free": {"value": 0.0, "tenor": "10Y", "symbol": "…", "as_of": "…"},
 "notes": []}
```
All values are tier `secondary_market`. Missing fields are `null` with a note — never estimated.

### 5.2 `assumptions.yaml`

```yaml
schema: assumptions/1
ticker: BGRIM
peers: [SET:GULF, SET:GPSC, SET:EGCO, SET:RATCH, SET:BPP]   # rationale required when proposed
equity_adjustments: {exclude_perpetual_from_equity: true}
bands: {lookback_years: 5, percentiles: [25, 50, 75]}
wacc: {risk_free: from_market, erp: 0.07, beta: 0.9, cost_of_debt: 0.045, tax_rate: 0.20, debt_weight: 0.55}
dcf: {horizon_years: 5, fcff_growth: [0.03, 0.03, 0.03, 0.03, 0.03], terminal_growth: 0.02,
      sensitivity: {wacc_step: 0.005, g_step: 0.005, size: 5}}
football_field: {methods: [range_52w, pe_band, pbv_band, peer_ev_ebitda, dcf], optional: [peer_pe, ddm]}
rationale: {beta: "…", erp: "…"}            # one entry per non-default value
```
Range checks (G5): `0.04 ≤ wacc ≤ 0.15` · `terminal_growth < risk_free + 0.01` · `0 ≤ debt_weight ≤ 0.9` · `0.3 ≤ beta ≤ 2.0` · peers ≥ 3.

### 5.3 `valuation.json`

```json
{"schema": "valuation/1", "price": 18.80, "as_of": "2026-10-30",
 "bars": [{"method": "pe_band", "low": 0, "mid": 0, "high": 0, "unit": "THB/share",
           "inputs": ["m.eps_common.TTM-2026Q2", "mkt.pe_history"], "caveats": []}],
 "reverse_dcf": {"implied_fcff_growth": 0.0, "wacc": 0.0, "terminal_growth": 0.0, "inputs": []},
 "sensitivity": {"wacc": [], "g": [], "value_per_share": [[]]},
 "unavailable": [{"method": "peer_ev_ebitda", "reason": "market.json missing peers"}]}
```

## 6. Claims and critic

### 6.1 `claims-<lane>.jsonl`
`{"section": "5.2", "claim": "…", "direction": "positive|negative|neutral", "refs": ["m.cfo_to_net_profit.FY2025"], "strength": "strong|moderate|weak"}` — 8–15 per lane, debatable claims only.

### 6.2 `critic.json`
`{"verdict": "pass|conditional|fail", "issues": [{"severity": "high|medium|low", "section": "7.6", "quote": "…", "problem": "…", "fix": "…", "lane": "C"}], "cross_lane_conflicts": [{"claims": ["a:3.3", "b:5.5"], "resolution": "…"}], "checked_ok": ["…"]}`
Critic checklist (in its prompt): overconfidence · unsupported management praise · leaps · cherry-picking · moat without mechanism · unfair peer comparison · bad case not worse than worst history · unusable scenario conditions · intent attribution · implicit buy/sell steering · cross-lane contradictions.

## 7. Gates

| Gate | Checks | Fails when |
|---|---|---|
| G0 lint | prompts/agents don't restate enums/sections/caps from this file (they must link) | duplicate definitions found |
| G1 fetch | `fetch.json` complete; identity confirmed; 56-1 fiscal years known | missing required docs without reason, unconfirmed identity |
| G2 parse | per statement: `total_assets = total_liabilities + total_equity` (±1 unit of scale); CF: `cfo + cfi + cff ≈ net_change_cash` (±0.5% or ±1 unit, FX line allowed); IS→CF net profit consistency; comparatives match prior filing (after unit sanity) | any identity off without an agent-fallback fact set |
| G3 facts | JSON-schema valid; IDs unique or flagged conflicts; caps (notes ≤ 80); every fact locatable | any violation |
| G4 model | quarterly derivations sum back to YTD; no metric silently missing | mismatch > 0.1% |
| G5 assumptions | §5.2 range checks; rationale present for non-defaults | out of range |
| G6 render | no unresolved `{{…}}`; **no bare number** in article/dashboard text outside the allow-list (§8.3); HTML balanced; no external `src`; size < 2 MB | any |
| G7 critic | final verdict ≠ `fail` | `fail` after the single revision |
| G8 claims | every claim's `refs` resolve; critic addressed every cross-lane conflict | unresolved |

Exit code 2 on gate failure, with a report at `runs/<T>/<run_id>/gates.md`.

## 8. Renderer and dashboard

### 8.1 Writer fragment format
Writers output **Markdown** with a restricted HTML subset for boxes/tags (list in `sections.md`), not raw HTML. Numbers appear only as references:
`{{f:<fact_id>|<fmt>}}` or `{{m:<metric_id>|<fmt>}}` or `{{v:<valuation_path>|<fmt>}}` (e.g. `v:bars.dcf.mid`).

### 8.2 Formats
`mb` million THB 1 dp (`1,675.5 ล้านบาท`) · `bb` billion THB 2 dp · `thb` THB 2 dp · `pct1` · `pct0` · `x1` multiple (`32.9 เท่า`) · `x2` · `int` · `mw` · `raw`. Renderer converts using `scale`; writers never convert.

### 8.3 Bare-number allow-list (G6)
Allowed digits outside references: section numbers in headings (`^#+ \d+(\.\d+)?`), years `19xx/20xx/25xx` adjacent to period words, quarter/period tokens (`Q[1-4]`, `[369]M`, `12M`, `FY\d{4}`), list ordinals, counts ≤ 10 written next to `ข้อ|ส่วน|สาย|ฉาก|มุม`. Everything else fails.

### 8.4 Dashboard (tab 1) layout
1. Header: name, ticker, as-of dates (filings, market), data-tier legend.
2. KPI tiles (8): price & market cap (tier badge) · TTM revenue · TTM common earnings · CFO/NI TTM · net debt/EBITDA · net debt/equity (both definitions) · payout (common) · dividend yield.
3. Football field (SVG) with current-price line; bar tooltips list inputs; unavailable methods listed beneath.
4. Reverse-DCF box: "at THB X the market implies FCFF growth of Y% p.a. (WACC Z%, g W%)".
5. Charts: revenue & gross margin (quarterly + annual) · CFO vs net profit · net debt & finance cost · capacity operating vs pipeline (if facts exist).
6. Sensitivity heatmap (WACC × g → value/share).
7. Assumptions panel (resolved values + rationale + source).
8. Data-quality panel: parser fallbacks, unit corrections, conflicts, scanned pages unread.

### 8.5 Article (tab 2)
Front (conductor-written from model: "what does THB X force you to believe" — three beliefs with refs), TOC, sections 1–10 (definitions in `sections.md`, owned by this contract), provenance appendix (every cited fact: id, value, file, sheet/cell/page), fixed disclaimer.

## 9. Fixtures

`plugins/stock-deepdive-pro/tests/fixtures/<TICKER>/`: `FINANCIAL_STATEMENTS.xlsx` for each FS period, `NOTES.txt` / 56-1 excerpts needed by tests (≤ 2 MB per ticker), `facts.golden.jsonl` (human-verified subset, `verified_by` + date on each row), `market.sample.json`. No PDFs. v1 tickers: **BGRIM** (no embedded tables, perpetual debentures, Q1/2026 unit mislabel) and **DITTO** (47–49 embedded tables, `.DOC` auditor report, 56-1 year gap).

## 10. CLI

### 10.1 Commands (`uv run --project <plugin> sdp <cmd>`)
| Command | Reads | Writes | Exit |
|---|---|---|---|
| `fetch <T>` | network | `data/<T>/`, `fetch.json` | 0 ok · 1 not found · 2 G1 |
| `extract <T>` | `data/<T>/` | `*.txt`, `MANIFEST.md` | 0 · 1 |
| `parse <T> --run <id>` | xlsx | `facts.jsonl` (parser part) | 0 · 2 G2 |
| `validate --run <id>` | facts | report | 0 · 2 G3 |
| `model --run <id>` | facts | `model.json` | 0 · 2 G4 |
| `market-validate --run <id>` | market.json | report | 0 · 1 missing (non-fatal) |
| `valuation --run <id>` | model, market, assumptions | `valuation.json` | 0 · 2 G5 |
| `prepare-writers --run <id>` | model, valuation, facts | `writer/` | 0 |
| `render --run <id>` | parts, model, valuation | `report.html`, `model.xlsx` | 0 · 2 G6 |
| `gate --run <id> [--only Gn]` | everything | `gates.md`, `run.json` | 0 · 2 |
| `new-run <T>` | — | prints run_id, creates dir | 0 |

### 10.2 `fetch.json`
`{"entity": {"symbol": "BGRIM", "registered_name": "B.GRIMM POWER PUBLIC COMPANY LIMITED", "confirmed": false}, "filings": [{"kind": "fs|r561", "period_end": "2026-06-30", "fiscal_year": null, "filing_id": "…", "status": "ok|failed|absent", "reason": null, "files": []}], "gaps": ["r561 FY2022–FY2024 not listed"]}`

## 11. `run.json`
`{"run_id", "ticker", "plugin_version", "python", "packages": {…}, "model": "<LLM id per agent step>", "started", "ended", "stages": [{"name", "start", "end", "status", "notes"}], "inputs": [{"path", "sha256"}], "parser_fallbacks": [], "unit_corrections": [], "gates": {"G1": "pass", …}, "critic": {"verdict", "revisions": 0|1}}`
