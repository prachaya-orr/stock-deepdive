# Phase 0 — scaffold, contracts as schemas, fixtures

Branch: `feature/p0-scaffold` (from `develop`). First follow `docs/rebuild/prompts/_COMMON.md`.

## Goal
Create the empty but runnable `stock-deepdive-pro` plugin, encode every contract as JSON Schema, and build the golden fixtures that all later phases test against.

## Tasks
1. **Skeleton** under `plugins/stock-deepdive-pro/`:
   - `.claude-plugin/plugin.json` (name `stock-deepdive-pro`, version `0.0.0`, MIT, homepage of this fork).
   - `pyproject.toml` (Python ≥ 3.11, deps pinned: `openpyxl`, `pypdf`, `pyyaml`, `jsonschema`; dev: `pytest`, `ruff`), `uv.lock`.
   - `src/sdp/__init__.py` and `src/sdp/cli.py`, with every command in CONTRACTS §10.1 registered as a stub that exits 3 "not implemented".
   - `schemas/`: `fact`, `model`, `market`, `assumptions`, `valuation`, `claims`, `critic`, `fetch`, `run` (`*.schema.json`), plus `tests/test_schemas.py` that validates the examples in CONTRACTS.
   - `taxonomy/items.yaml` (every key in CONTRACTS §3.3 with statement, sign and description) and `taxonomy/synonyms.yaml` (EN labels seen in the fixtures, with TH where known).
   - Add the plugin to `.claude-plugin/marketplace.json` next to `stock-deepdive-lite`. Do not touch `.agents/plugins/marketplace.json` (D4).
   - Add `/runs/` to `.gitignore`.
2. **Fixtures** (CONTRACTS §9):
   - Obtain BGRIM and DITTO filings. If `data/BGRIM` or `data/DITTO` are absent, run the legacy fetcher once (network allowed for this step only): `python3 plugins/stock-deepdive-lite/bin/fetch_th.py <T> --out data/<T>`.
   - Copy each FS period's `FINANCIAL_STATEMENTS.xlsx` and the minimal `NOTES` / 56-1 text excerpts that later tests need into `tests/fixtures/<T>/`. Keep each ticker ≤ 2 MB and include no PDFs.
   - Create `facts.golden.jsonl` with **at least 25 candidate rows per ticker** across `bs`/`is`/`cf`, starting from the seed table in ACCEPTANCE.md. Fill `source.sheet` + `source.cell` by opening the xlsx with openpyxl. Set `verified_by: null`. Humans verify in PR review.
   - `tests/fixtures/BGRIM/market.sample.json`: a schema-valid **synthetic** sample, clearly labelled `"notes": ["synthetic fixture"]`.
3. `src/sdp/tools/check_fixtures.py` implements the ACCEPTANCE P0 check.

## Stop and ask if
- A fixture sheet uses a layout that the canonical items cannot represent.
- Any contract example in CONTRACTS fails its own schema. Propose the fix.

## PR note for the reviewer
List the golden rows that need human verification as a checklist (ticker, id, value, sheet!cell), so the reviewer can tick them off against the xlsx.
