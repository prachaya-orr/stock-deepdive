# Phase 1 — fetch, extract, statement parser, identities

Branch: `feature/p1-parser`. First follow `docs/rebuild/prompts/_COMMON.md`. Requires P0 merged.

## Goal
Turn iDisc filings into `facts.jsonl` for the primary statements **without any LLM**, with accounting-identity validation.

## Tasks
1. `sdp new-run <T>` creates `runs/<T>/<run_id>/` and prints the run_id.
2. `sdp fetch <T>` uses a source-adapter interface (`resolve`, `list_filings`, `download`) with an iDisc implementation ported from legacy `fetch_th.py`. Port it, do not copy-paste blindly.
   - Parse the **fiscal year** for each 56-1 from the index row (legacy derived nothing). Folder names follow CONTRACTS §1.
   - Write `fetch.json` (§10.2) with `gaps` and the registered name. G1 requires `confirmed: true`, which the conductor sets after the user confirms.
   - Retry with backoff on network errors (3 tries). One failed document must not abort the others.
3. `sdp extract <T>`:
   - The docx converter (port, keeping the embedded-workbook inlining) and the PDF per-page text both stay as they are.
   - Scan detection uses an absolute floor of 150 chars **or** relative 0.15 × median. If more than 50% of pages are flagged, flag the whole document.
   - The **xlsx text dump honours cell coordinates.** This is a regression test for the legacy empty-cell shift.
   - `MANIFEST.md` lists every file, including unconvertible ones and their reasons.
4. `sdp parse <T> --run <id>` follows SPEC §5.3:
   - Read xlsx with openpyxl using coordinates.
   - Detect statement type, basis columns, period headers (instant or duration, months) and the unit, from whichever of the three placements is used.
   - Map labels with `taxonomy/synonyms.yaml`. Unmapped rows become `raw.<slug>`.
   - Emit facts per CONTRACTS §3. IDs are deterministic.
5. Unit sanity cross-check against the prior filing's comparatives (SPEC §5.3). Record corrections.
6. `sdp validate --run <id>` runs G1–G3, including the G2 identities in CONTRACTS §7. Write results to `run.json`.

## Acceptance
Every bullet under "P1 parser" in ACCEPTANCE.md.

## Stop and ask if
- A fixture statement fails an identity for a reason that is not a parser bug (for example the filing itself does not balance). Report it with evidence. Do not loosen tolerances.
