# Worked examples

## E1. Regression cases for any `check_numbers.py` change

Synthetic FACTS: 100 rows, `#` = 1…100, values ≈ 1.2 M (unit พันบาท), location column `NOTES.txt บรรทัด <n>`.

| Report token | Should be | Current behaviour (probe) |
|---|---|---|
| `1,197 ล้านบาท` where FACTS has `1196589` พันบาท | match | match |
| `123` (not in FACTS) | orphan | **match** (via ×1e-3 → row #0/1) |
| `98,765` | orphan | **match** |
| `77,777,777` | orphan | **match** (via ×1e-6) |
| `45.3%` | checked | **skipped as noise** |
| `2568` alone | skip (year) | skip |

A correct fix turns the bold rows into the "should be" column.

## E2. Scan detection

| Document | Expected flags | Current |
|---|---|---|
| 20 pages × 2,500 chars + 3 pages of 40–108 chars | 3 pages | 3 pages |
| 6 pages, all 40–108 chars (fully scanned) | all 6 / whole-doc warning | **none** |

## E3. Adding a new section field to the article

1. Edit the section list in `skills/deepdive/SKILL.md` (sections and lane table).
2. Mirror in `agents/writer.md` topic list.
3. Update README "10 ส่วน" mentions and the slide deck.
4. If a new CSS class is needed, add it to `build_paper.py` CSS **and** the class list in SKILL.md.
5. Re-run `build_paper.py --help` / a local parts build to confirm exit 0.

## E4. Adding a third market

1. New `bin/fetch_<mkt>.py` following the pattern: identifiable UA, throttle, per-document failure tolerance, closing "what this source lacks" checklist.
2. Ensure its outputs are formats `extract.py` lists (or extend `extract.py`).
3. Add the command (two-line form) to `skills/fetch/SKILL.md` and market disambiguation text.
4. Update both `plugin.json` descriptions/keywords and both marketplace descriptions.
