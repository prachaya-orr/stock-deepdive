# Acceptance criteria

All commands are run from `plugins/stock-deepdive-pro/` unless stated otherwise. "Golden" means `tests/fixtures/<T>/facts.golden.jsonl`
rows with a non-null `verified_by`. Monetary comparisons are exact after `value × scale`, unless a tolerance is given.

## Seed golden values (observed in the iDisc xlsx dumps on 2026-11-01; human verification still required)

| Ticker | Item | Basis | Period end | Printed value | Printed unit | Source |
|---|---|---|---|---|---|---|
| BGRIM | `cash` | consolidated | 2025-12-31 | 23,080,057,507 | Baht | FS filed 2026-02, sheet `EN_BS6` |
| BGRIM | `cash` | consolidated | 2024-12-31 | 18,784,932,518 | Baht | same sheet, comparative |
| BGRIM | `cash` | separate | 2025-12-31 | 4,509,202,589 | Baht | same sheet |
| BGRIM | `cash` | consolidated | 2026-06-30 | 19,476,474 | Baht ’000 | FS filed 2026-08, sheet `BS2-4 (EN)` |
| BGRIM | `cash` | consolidated | 2025-12-31 | 23,080,058 | Baht ’000 | same sheet, comparative (must reconcile with the row above within ±1 thousand) |
| DITTO | `cash` | consolidated | 2026-06-30 | 228,145 | Thousand Baht | FS filed 2026-08, sheet `BS-T` |
| DITTO | `cash` | consolidated | 2025-12-31 | 260,958 | Thousand Baht | same sheet, comparative |

P0 extends this to at least **25 verified rows per ticker**, covering every statement type.

---

## P0 scaffold
```sh
uv sync
uv run sdp --help                                  # lists every command in CONTRACTS §10.1
uv run pytest -m "not live" tests/test_schemas.py  # every schema loads; example docs in CONTRACTS validate
uv run python -m sdp.tools.check_fixtures          # fixtures present, ≤ 2 MB/ticker, no PDFs
```
- `claude plugin validate .` (repo root) passes with both plugins listed.
- `git diff --stat develop -- plugins/stock-deepdive-lite` is empty.

## P1 parser
```sh
uv run pytest -m "not live" tests/p1
```
These must pass:
- **Coordinate fidelity:** for every fixture xlsx, the dumped text has the same number of columns per row as the sheet's used range. A regression test covers the legacy empty-cell shift.
- **Statement and unit detection:** every fixture sheet gets the right statement type, basis columns, periods and unit (BGRIM FY2025 = `Baht`, BGRIM 6M/2026 = `Baht ’000`, DITTO = `Thousand Baht`).
- **Golden match:** 100% of verified golden rows for statements `bs`/`is`/`cf` are reproduced exactly.
- **Identities:** BGRIM (6 periods) and DITTO (2 periods) pass G2 for consolidated and separate.
- **Unit sanity:** a test that injects the BGRIM Q1/2026 case (IS header `Baht`, values in thousands) is detected, corrected, and recorded with `unit_corrected: true`.
- **Scan detection:** a synthetic fully scanned PDF (all pages < 150 chars) is flagged as a whole document. The mixed BGRIM-like case flags only the short pages.
- **Ledger:** `fetch.json` for the DITTO fixture reports the 56-1 FY2021/FY2025 gap.
- Live (optional, marked `live`): `uv run sdp fetch BGRIM` produces 6 FS periods plus 56-1 with fiscal years, and the identity is printed.

## P2 notes extractor
- Running the agent on the BGRIM fixture yields ≤ 80 facts, 100% schema-valid after at most 1 repair round, and every fact has `source.page|line|note`.
- It covers every required topic in CONTRACTS §3.4, or the topic is listed in `unavailable` with a reason.
- `company_defined.*` facts have tier `management_statement`.
- A reviewed output is committed as `tests/fixtures/BGRIM/notes.golden.jsonl`.

## P3 model
```sh
uv run pytest -m "not live" tests/p3
```
- Derived Q2-2026 values equal the 6M/2026 value minus the Q1/2026 value for every IS/CF item (exact).
- TTM-2026Q2 equals FY2025 − 6M/2025 + 6M/2026 (exact).
- `common_earnings.FY2025` for BGRIM equals `net_profit_parent − perpetual interest` from facts. The expected value comes from golden facts, not a hard-coded constant.
- Every registry metric is present for every period. Missing inputs give `value: null` plus a caveat.
- `model.xlsx` opens. A formula-recalculation check (openpyxl, with cached values cleared and recomputed by LibreOffice when available, otherwise formula-string assertions) matches `model.json` within 0.01%.

## P4 valuation
```sh
uv run pytest -m "not live" tests/p4
```
- Using `market.sample.json` and `config/tickers/BGRIM.yaml`, `valuation.json` validates. Every enabled bar has `low ≤ mid ≤ high`, and every method that cannot be computed appears in `unavailable` with a reason.
- **DCF correctness:** a unit test against a hand-computed 3-year example matches to 0.01%.
- **Reverse DCF:** feeding its implied growth back into the DCF reproduces the input price within 0.5%.
- **Equity adjustment:** with `exclude_perpetual_from_equity: true`, BVPS falls by `perpetual_debentures / shares`, which is asserted against facts.
- G5 rejects `terminal_growth ≥ risk_free + 0.01` and a peer list shorter than 3.

## P5 dashboard and renderer
```sh
uv run pytest -m "not live" tests/p5
```
- SVG snapshot tests for the football field, the line/bar chart and the heatmap are byte-stable.
- Reference substitution: every `fmt` in CONTRACTS §8.2 is covered, and an unknown ID raises G6.
- **Bare-number gate:** a fragment containing `กำไร 98,765 ล้านบาท` (no reference) fails G6. The same text with `{{m:…}}` passes. This mirrors the legacy fabricated-number probe.
- `report.html` built from fixtures is < 2 MB, has no external `src`, both tabs work with JS disabled except for tab switching (the content stays visible), and the provenance appendix lists every cited fact.

## P6 writers, critic, conductor
- On the BGRIM fixture plus `market.sample.json`, `/stock-deepdive-pro:deepdive BGRIM --offline-fixtures` completes with G0–G8 passing.
- The critic runs, `critic.json` exists, and if there was a revision `run.json.critic.revisions == 1`.
- G0 catches a deliberately injected duplicate section list in a test prompt.
- Writers' working directory contains only `writer/`. A test asserts that the agent call passes no `data/` path.

## P7 release
- GitHub Actions workflow runs `uv run pytest -m "not live"` on push and PR to `develop`, and it is green.
- `run.json` contains stage timings, input hashes, package versions, the LLM id per agent step, and gate results.
- `docs/rebuild/AB-REPORT.md` compares a `lite` run and a `pro` run on BGRIM: wall time per stage, number of facts, number of unverifiable numbers (`lite` probe vs `pro` G6), and the critic verdict.
- Plugin version `1.0.0`. The README covers install, usage, outputs and limits.
