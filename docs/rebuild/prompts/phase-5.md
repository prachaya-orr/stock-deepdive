# Phase 5 — SVG charts, dashboard, renderer, bare-number gate

Branch: `feature/p5-dashboard`. First follow `docs/rebuild/prompts/_COMMON.md`. Requires P4 merged.

## Goal
Produce the offline single-file `report.html` with the Dashboard and Article tabs. Every number in it is rendered from data, and G6 proves it.

## Tasks
1. `src/sdp/svg/` is a small chart library with no dependencies. It covers:
   - a horizontal range-bar **football field** with a price line, labels and tooltips (`<title>`)
   - a line plus bar combo chart
   - grouped bars
   - a heatmap
   Output is deterministic (fixed number formatting and ordering).
2. The dashboard layout follows CONTRACTS §8.4 exactly (8 blocks).
   - It uses CSS variables and a light/dark theme via `prefers-color-scheme`.
   - The colour palette must be readable for colour-blind users. Use more than hue alone: patterns or labels on the bars.
   - Tab switching uses about 30 lines of inline JS. With JS off, both tabs render stacked.
3. **Reference substitution** follows CONTRACTS §8.1–8.2: `{{f:…}}`, `{{m:…}}`, `{{v:…}}` with formats.
   - Each rendered number is wrapped in `<span class="ref" data-ref="…" title="source …">` and links to the provenance appendix.
   - An unknown reference is a G6 failure.
4. A Markdown-to-HTML converter for writer parts handles the restricted subset defined in `sections.md`. Write a minimal parser; do not add a dependency.
5. **G6:**
   - no unresolved references
   - the bare-number allow-list from CONTRACTS §8.3
   - balanced HTML
   - no external `src`
   - size under 2 MB
6. The provenance appendix lists every cited fact: id, value with unit, file and sheet!cell or page.
7. Add `model.xlsx` styling (header formats and number formats). Formulas stay.

## Acceptance
Every bullet under "P5 dashboard and renderer" in ACCEPTANCE.md. Use placeholder writer parts made of fixture Markdown with references to build the test report.

## Stop and ask if
- The bare-number allow-list blocks legitimate Thai phrasing found in the placeholder parts. Propose an allow-list addition with an example.
