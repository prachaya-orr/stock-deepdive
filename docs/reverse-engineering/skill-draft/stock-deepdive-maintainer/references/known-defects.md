# Known defects (as of e71a2c2)

Severity is the analyst's judgement. Evidence: docs/reverse-engineering/BUSINESS_LOGIC.md.

| ID | Sev | Defect | Where | Fix direction |
|---|---|---|---|---|
| D-01 | High | Number screen matches fabricated values: ×1e-3/×1e-6 scales + absolute tolerance floor 1.0 + FACTS row numbers/line refs in the pool | `bin/check_numbers.py:34,64,87` | Parse only the value column with its unit; relative tolerance only; or move to fact-ID references |
| D-02 | High | Values < 100 without comma never checked (%, margins, EPS) | `check_numbers.py:53` | Check decimals and `%`-suffixed values |
| D-03 | High | Writer prompt tells writers to open source files, contradicting isolation | `agents/writer.md:14-19` | Remove; restate FACTS-only rule |
| D-04 | Med | Writer prompt has 11 topics vs playbook's 10; topic 8 asks for dodged questions (no transcripts) | `agents/writer.md:3,21-37` | Align to SKILL.md:208-219 |
| D-05 | Med | factcheck prompt describes auditing a report, but is used to produce FACTS | `agents/factcheck.md:15,52-59` vs `SKILL.md:97` | Separate producer prompt, or rewrite |
| D-06 | Med | Tier schemes disagree (3 in FACTS, 4 in policy, 4 icons in writer); no tier for management statements | `SKILL.md:104`, `AGENTS.md:25`, `writer.md:47-51` | One enum incl. management_statement |
| D-07 | Med | Fully/mostly scanned PDF flagged as zero scanned pages (`min` makes floor a ceiling) | `bin/extract.py:82` | Absolute floor OR relative; whole-doc flag |
| D-08 | Med | MANIFEST omits `.htm`/`.json` → empty for US tickers | `bin/extract.py:123-146` | List all files; convert htm |
| D-09 | Med | "Fetched" = folder exists; partial failures invisible next run | `SKILL.md:54` | Write a fetch ledger / status in MANIFEST |
| D-10 | Low | `critic` agent never invoked | `agents/critic.md`; `SKILL.md` | Wire into step 3 or delete |
| D-11 | Low | `build_paper.py` writes output before validating; title/subtitle/sources unescaped in body | `build_paper.py:176,179,185,190-205` | Escape; validate then write |
| D-12 | Low | One 56-1 download failure skips remaining 56-1 files | `bin/fetch_th.py:328,346` | Per-item try |
| D-13 | Low | `paper/_parts/` not namespaced by ticker | `SKILL.md:124-126` | `paper/<T>/_parts/` |
| D-14 | Low | AGENTS.md forbids `${CLAUDE_PLUGIN_ROOT}` though 3749845 restored it deliberately | `AGENTS.md:39-40` | Update AGENTS.md |
| D-16 | Med | 56-1 fetch takes the first 2 listed, which can be years apart (DITTO: FY2025 + FY2021), with no year in the output | `bin/fetch_th.py:324` | Parse the year from the index row; put it in the folder name and MANIFEST; warn on gaps |
| D-17 | Low | 56-1 FILEIDs (`dat/f56/…`) miss the `YYYYMM/` pattern, so folder labels fall back to `R561_dat_f56_…_zip` | `bin/fetch_th.py:78-83` | Separate label rule for 56-1 |
| D-15 | Low | Minor drift: FACTS size 80-120 vs 100-150; valuation ≥3 vs 2-3 angles; writer tables ≤3 cols; per-lane summary + disclaimer | see BUSINESS_LOGIC.md §5 | Single contract file |
