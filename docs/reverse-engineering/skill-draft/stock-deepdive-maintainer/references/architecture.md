# Architecture (condensed)

Full version: `docs/reverse-engineering/ARCHITECTURE.md`. `P/` = `plugins/stock-deepdive-lite/`.

## Pipeline

| Step | Actor | Reads | Writes |
|---|---|---|---|
| 0.2 fetch | `bin/fetch_us.py` (EDGAR) or `bin/fetch_th.py` (iDisc) | network | `data/<T>/…` |
| 0.3 extract | `bin/extract.py` | `data/<T>/**` | `*.txt`, `data/<T>/MANIFEST.md` |
| 1 facts | `factcheck` agent | `data/<T>/` | `paper/<T>-FACTS.md` |
| 2 write ×3 parallel | `writer` agent lanes A/B/C | FACTS + MANIFEST | `paper/_parts/{10-a,20-b,30-c}.html`, `claims-*.jsonl` |
| 3.1 front | orchestrator | all parts | `paper/_parts/05-front.html` |
| 3.2 build | `bin/build_paper.py` | `_parts/*.html` (name order) | `paper/<T>-deepdive-<date>.html` |
| 3.3 screen | `bin/check_numbers.py` | article + FACTS | console |
| 3.4 reconcile | orchestrator | `claims-*.jsonl` | edits to parts |

`critic` is defined but not invoked.

## Lanes

| Lane | Sections | Cap |
|---|---|---|
| A | 1 business · 2 market & competitors · 3 advantage | 12,000 chars |
| B | 4 management & related parties · 5 financials & earnings quality · 6 risks | 14,000 |
| C | 7 valuation · 8 drivers & IR · 9 timeline · 10 scenarios & limits | 14,000 |

## Enforcement reality

Mechanically enforced: offline output, no `style=`, tag balance, scan-page flags (partially), number screen (weakly).
Everything else — writer isolation, tiers, no advice, scenario validity — is prompt-only.
