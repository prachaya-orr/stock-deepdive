# Business Logic

Extracted following the Rosetta skill: capture the **what and why**, consolidate rules that are scattered across files, and say for each rule
**what actually enforces it**.

Enforcement levels:
- **CODE**: a script refuses or flags the condition.
- **PROMPT**: only an LLM instruction asks for it.
- **NONE**: stated as a principle, but no instruction or code makes it happen.

## 1. Domain vocabulary

| Term (canonical) | Synonyms found | Meaning |
|---|---|---|
| **Source document** | เอกสารต้นฉบับ, filing, งบที่ยื่น | A document the company or regulator published itself (10-K, งบการเงิน, 56-1) |
| **Fact** | ตัวเลข, row in FACTS | One number with item, value, unit, period, location and tier |
| **Tier** | ชั้น, ชั้นความน่าเชื่อถือ | Reliability class of a fact or claim. **Defined three different ways** (see §5) |
| **Lane** | สาย | One parallel writer job (A, B or C) |
| **Section** | ส่วน, หัวข้อ | Numbered article section. **10 in the playbook, 11 in the writer prompt** |
| **Claim** | ข้ออ้าง | A debatable assertion recorded in `claims-*.jsonl` |
| **Orphan** | ตัวหลุด | A report number not traceable to FACTS |
| **Computed figure** | `[คำนวณ]` | A ratio or difference that the writer derived. It must show its method |
| **Scenario** | ฉาก | One of three forward 1–3 year cases |

Duplicate terminology is a **blocking finding** under Rosetta rule 13. *Section/topic* and *tier* have competing definitions. See §5.

## 2. Business rules

### BR-01 Single path for numbers
- **Rule:** Every number in the article comes from `FACTS.md` or is marked `[คำนวณ]` with its method. Writers never read source documents.
- **Sources:** `AGENTS.md:21-22`, `P/skills/deepdive/SKILL.md:15-17,130-131`, `README.md:134`.
- **Why:** Number verification happens once, in one place (`SKILL.md:16-17`).
- **Enforcement:** PROMPT for writer isolation. **Contradicted** by `P/agents/writer.md:14-19`, which tells the writer to open `companyfacts.json` and `NOTES.txt`. Writer tools include `Read, Grep, Bash` (`writer.md:4`), so nothing prevents access. Traceability is checked by BR-02.

### BR-02 Number screen
- **Rule:** After assembly, every number must trace to FACTS. Orphans are added to FACTS, marked `[คำนวณ]`, or removed (`SKILL.md:182-190`).
- **Enforcement:** CODE (`P/bin/check_numbers.py`), but **advisory** (exit 0) and **weak**:

| Weakness | Mechanism | Evidence |
|---|---|---|
| Downscaling with an absolute floor | Scales include 1e-3 and 1e-6, and the tolerance is never below 1.0 after scaling | `:34`, `:64`. Probe P1: fabricated values up to ~10⁸ match row numbers 1…100 |
| Polluted fact pool | Every numeric token in FACTS counts, including `#`, period, page and line numbers | `:87`. Probe P1b: a line reference matched 3,141,592 |
| Values < 100 never checked | No comma and below 100 → noise | `:53-54`. Probe P2: `45.3`, `12.5`, `0.87` skipped |
| Line-granular exemption | One `[คำนวณ]` exempts every number on that physical line | `:97-99`. HTML fragments may put a whole table row, or more, on one line (INFERRED) |
| Year filter | Bare 1900–2100 / 2400–2700 is skipped, even when it is an amount | `:55-57` |
| Not gating | `--strict` is never used by the playbook | `SKILL.md:185-186` |

The script itself says it is "a sieve, not proof" (`:117-120`). The README states it more strongly (`README.md:20,134`).

### BR-03 Raw units, no silent conversion
- **Rule:** FACTS stores the value and unit as written in the document. Agents must not convert units (`SKILL.md:111-112`).
- **Why:** Annual and quarterly statements of the same company often use different units (`:111`).
- **Enforcement:** PROMPT. The screen deliberately tolerates ×1000 and ×1e6 unit changes (`check_numbers.py:10-11,34`), which is also the cause of BR-02's weakness.

### BR-04 Conflicting values are both recorded
- **Rule:** If a document contradicts itself, record both values and flag the conflict. Never pick one silently (`SKILL.md:114`).
- **Enforcement:** PROMPT.

### BR-05 Trust tiers are always labelled
- **Rule:** Filed documents ≠ management statements ≠ headlines ≠ secondary market data (`AGENTS.md:25`, `README.md:135`). Every FACTS row carries a tier (`SKILL.md:103`). Management quotes go in `box quote` (`SKILL.md:248`).
- **Enforcement:** PROMPT. The tier scheme is inconsistent; see §5.

### BR-06 Absence is stated, never filled
- **Rule:** Missing data is written as missing (`AGENTS.md:26`). Market size and competitors are stated as unavailable when filings lack them (`SKILL.md:211`). A valuation angle that cannot be done is stated as such (`SKILL.md:233`).
- **Enforcement:** PROMPT. Partially supported by the scripts printing "what the source lacks" checklists (`fetch_us.py:211-216`, `fetch_th.py:351-355`).

### BR-07 No writing without sources
- **Rule:** Stop and ask if documents cannot be fetched. Never write from model memory (`AGENTS.md:23-24`, `SKILL.md:73-74`).
- **Enforcement:** PROMPT. Weakened by "folder exists ⇒ fetched" (`SKILL.md:54`).

### BR-08 No investment advice
- **Rule:** Never say buy or sell, and do not steer through scenario ordering (`AGENTS.md:28`, `SKILL.md:252-255`).
- **Enforcement:** PROMPT, plus CODE for the fixed disclaimer (`build_paper.py:107-111,186`).

### BR-09 Anomaly ≠ intent
- **Rule:** Report what is anomalous and why. Never attribute intent (`AGENTS.md:27`, `SKILL.md:249`).
- **Enforcement:** PROMPT. `critic` is the natural enforcer, but it is never invoked.

### BR-10 Valuation frame
- Use at least 3 angles and compare them (`SKILL.md:226`). Disagreements between angles get their own subsection and are never averaged (`:227`).
- Frame the question as "what does this price force you to believe", not "cheap vs expensive" (`:229`).
- Analyst targets are not a proxy for the market (`:231`).
- Price is secondary-tier data with a date (`SKILL.md:70-71`). Without a price, the front page switches to "three core claims of this business" (`:159`).
- **Enforcement:** PROMPT. `writer.md:32` says "2-3 angles", which conflicts with "at least 3".

### BR-11 Scenario validity (three scenarios over 1–3 years)
| Rule | Source |
|---|---|
| Conditions must reference a metric the company has actually disclosed | `SKILL.md:236` |
| Never require two rare events at once | `:237` |
| A condition already true today is not a condition | `:238` |
| The bad case must be worse than the worst real historical year | `:239` |
| All three on the same scale and strictness | `:240` |
- **Enforcement:** PROMPT. Mirrored in the unused `critic.md:25-27`.

### BR-12 Earnings quality
- Compare operating cash flow with net income **for every period**. If earnings do not become cash, say where the cash is stuck (`SKILL.md:107,214`).
- **Enforcement:** PROMPT. FACTS must include OCF–NI pairs for every period (`:107`).

### BR-13 FACTS size cap
- 80–120 facts, only those actually used (`SKILL.md:106`). Reason: "the bigger the table, the weaker the sieve" (`:106`, `check_numbers.py:118-119`). The script header says 100–150 (`check_numbers.py:119`), which is inconsistent.
- **Enforcement:** PROMPT.

### BR-14 Length caps and no full rewrites
- Every agent prompt carries `เพดาน N ตัวอักษร · เขียนรอบเดียวจบ…` (`SKILL.md:29-31`). Caps per lane are 12k, 14k and 14k (`:124-126`).
- **Why:** Agents otherwise overshoot and rewrite, which costs more than the first draft (`:31`).
- **Enforcement:** PROMPT. `build_paper` prints visible character counts but does not compare them to caps (`build_paper.py:194-198`).

### BR-15 Presentation contract
- Fragments contain only `<section>`, with no `style=` and only whitelisted classes (`SKILL.md:133-134,144-150`). The article must work offline (`build_paper.py:17-18`).
- **Enforcement:** CODE for `style=`, tag balance (7 tags) and external `src`. PROMPT for the class whitelist.

### BR-16 Cross-lane consistency
- Each lane emits 8–15 debatable claims as JSONL (`SKILL.md:136-141`). The orchestrator looks for same-topic, opposite-direction pairs and for strong claims weakened elsewhere, then fixes them in the text (`:192-194`).
- **Enforcement:** PROMPT. No schema validation and no script.

### BR-17 Market disambiguation and identity check
- Ask when the market is unclear, because tickers collide across markets (`SKILL.md:52`, `P/skills/fetch/SKILL.md:23-25`).
- For TH, always check the printed company name, because iDisc includes non-listed bond issuers (`fetch/SKILL.md:54-55`, `fetch_th.py:276-279`).
- For US, the exact ticker match prints the SEC title (`fetch_us.py:61-62,164`).
- **Enforcement:** PROMPT plus CODE (printing only).

### BR-18 Free, public sources only
- No paid or login sources (`fetch/SKILL.md:79`, `README.md:150`). Third-party analysis is a lead, not a source (`fetch/SKILL.md:81`).
- **Enforcement:** CODE by construction, since only two hosts are hard-coded.

## 3. Algorithms

### A1. Filing selection (`fetch_us.py:66-80, 172-179`)
Iterate EDGAR's `recent` parallel arrays in their order, which is most recent first (external EDGAR convention, INFERRED). Take the first *N* of each form
in the plan and de-duplicate by accession number. Matching is exact, so amendments such as `10-K/A` are excluded (DERIVED). If no 10-K or 10-Q was picked,
the company is treated as a foreign private issuer and the selection is redone with the 20-F/6-K plan.

### A2. Exhibit selection (`fetch_us.py:83-104`)
From the filing's `index.json`, keep readable extensions (`.htm .html .pdf .txt`) other than the primary document. Classify an item as EX-99 if its type contains "99" or
its name contains `ex99`/`ex-99`. Return the EX-99 items. **If there are none, return every other `EX-*` except XBRL (101) and cover page (104)**, which can pull in large
material-contract exhibits (INFERRED).

### A3. XBRL index (`fetch_us.py:107-133`)
Save raw `companyfacts.json`. Write a human index with one line per `taxonomy/tag`, followed by its label, so agents can find tags before opening the big JSON.

### A4. iDisc discovery (`fetch_th.py:67-83`)
Regex `FILEID=(…\.zip)` over the index HTML, preserving order and de-duplicating. Folder label = `YYYYMM_` + first 20 chars of the zip basename.

### A5. Office → text, the embedded-table recovery (`fetch_th.py:87-232`)
The core domain insight: **Thai financial-statement notes keep their numbers in OLE-embedded Excel workbooks**, which ordinary converters drop (`:195-197`, `README.md:17`).
1. Map relationship IDs to `word/embeddings/*` (`:161-172`).
2. Walk the top-level body children in order. Paragraphs become text and tables become tab-joined rows (cell paragraphs joined by spaces, trailing empty cells trimmed).
3. For each `o:OLEObject` inside that element, resolve `r:id` and unpack the nested `.xlsx` **in memory**. Emit its sheets between `----- ตารางฝัง #n -----` markers at that position (`:216-221`).
4. Emit any embedded workbooks never referenced in a separate "unplaced" section (`:223-231`).
5. XLSX reading handles shared strings, inline strings and raw values. Formulas are read as their cached `<v>` values (DERIVED). Number formats are **not** applied, so values appear raw, for example as serial dates (INFERRED).

### A6. Scan-page detection (`extract.py:41-45, 75-88`)
`cut = min(150, median(page_lengths) × 0.15)`. A page is "scanned" if its stripped text length is below `cut`.
**Intent** (comment `:41-43`): be relative to the document, with a floor for thin documents. **Behaviour:** 150 is a *ceiling*. A document that is mostly or entirely scanned has
a tiny median, so nothing is flagged (probe P3b). That is exactly the case where flagging matters most.

### A7. Page-range compression (`extract.py:91-101`)
Collapse consecutive integers into `a-b` runs.

### A8. Number screen (`check_numbers.py:39-68`)
See BR-02.

### A9. Fragment sanitising and structure check (`build_paper.py:113-146`)
Regex strip of document-level tags. Open/close count parity for `section table div ol ul tbody thead`. External `src="http…"` detection, while external `href` links are allowed.
Any ` style=` is rejected.

## 4. Verdict state models

| Actor | States | Transition rules | Source |
|---|---|---|---|
| factcheck, per number | 🟢 matches · 🟡 unverifiable · 🔴 contradicts | 🟡 can never be promoted to 🟢 without evidence. "Probably right" stays 🟡 | `P/agents/factcheck.md:23-30` |
| factcheck, overall | pass · conditional pass · fail | Not specified | `:54` |
| critic, overall | pass · conditional pass · fail | Uncertain issues are reported as uncertain, not as fail | `P/agents/critic.md:36,50` |
| build_paper | ok (0) · input error (1) · structure error (2) | — | `build_paper.py:160-206` |
| check_numbers | ok (0) · orphans (0, or 1 if strict) · input error (2) | — | `check_numbers.py:82-147` |

## 5. Inconsistencies (hidden assumptions that break)

| # | Topic | Version A | Version B | Version C |
|---|---|---|---|---|
| I-1 | Section count | 10 sections (`SKILL.md:208-219`, `README.md:20`, slide 26) | **11 topics** (`writer.md:3,21-37`) | — |
| I-2 | Section content | §8 "growth drivers and investor communication" (`SKILL.md:217`) | §8 "what management **dodged** when asked" (`writer.md:34`) | Transcripts are unavailable (`README.md:104`), so B is unanswerable |
| I-3 | Trust tiers | FACTS: 1 filed · 2 secondary market · 3 headline (`SKILL.md:104`) | 4 layers: filed ≠ mgmt ≠ headline ≠ market (`AGENTS.md:25`) | writer: 📄 filed · 🗣 mgmt · 🟨 others' opinion · 🧮 computed (`writer.md:47-51`) |
| I-4 | Writer source access | FACTS only (`AGENTS.md:21`, slide 24) | Open companyfacts/NOTES (`writer.md:14-19`) | — |
| I-5 | factcheck purpose | Produce FACTS (`SKILL.md:97`) | Audit a report (`factcheck.md:15`) | — |
| I-6 | Valuation angles | ≥ 3 (`SKILL.md:226`) | 2–3 (`writer.md:32`) | — |
| I-7 | FACTS size | 80–120 (`SKILL.md:106`) | 100–150 (`check_numbers.py:119`) | — |
| I-8 | Tables | Any width, with `tw` wrapper (`SKILL.md:149`) | ≤ 3 columns (`writer.md:60`) | — |
| I-9 | Opening and disclaimer | Front page by orchestrator, disclaimer by script (`SKILL.md:156`, `build_paper.py:186`) | Each writer opens with a 3–5 line summary and closes with a disclaimer (`writer.md:59-61`) | Applied to 3 lanes, this gives 3 summaries and 3 disclaimers (INFERRED) |
| I-10 | Host-specific vars in shared files | Forbidden (`AGENTS.md:39-40`) | Present on purpose since `3749845` (`SKILL.md:47,59,167,185`) | — |

Management statements have no FACTS tier (I-3). A number quoted from a CEO in an 8-K EX-99 press release would therefore be stored as
tier 1 (it is in a filing) or tier 3 (a headline). Either way it violates BR-05 (INFERRED).

## 6. Edge cases

| Case | Behaviour | Evidence |
|---|---|---|
| Same stem for `.docx` and `.pdf` in one folder | Both write `<stem>.txt`; the last one processed wins | `extract.py:125,134` (DERIVED) |
| Old `.doc` auditor report | Listed as unconvertible. MANIFEST points to the 56-1 scan pages instead | `extract.py:145-146,172-175` |
| Ticker collides with a bond issuer (TH) | Script continues. Only the user or LLM can stop | `fetch_th.py:276-279` |
| US company files only 8-K and no 10-K/10-Q | The foreign plan is tried and finds nothing, so it exits "no supported filings", even if 8-Ks exist | `fetch_us.py:176-181` (DERIVED) |
| `DEF 14A` file names | Contain a space, e.g. `DEF 14A_2026-…htm` | `fetch_us.py:188` (DERIVED) |
| Fully scanned PDF | Zero scanned pages reported | P3b |
| US run | MANIFEST lists no filings (only `.htm`/`.json` exist) | `extract.py:123-146` |
| Fragment contains `<body>` | Everything before it, and after `</body>`, is dropped | `build_paper.py:114-115,126-127` |
| `--title` contains `<` or `&` | Raw in `<h1>`, escaped in `<title>` | `build_paper.py:173,176` |
| FACTS without numbers | exit 2 | `check_numbers.py:88-90` |
| Negative numbers in parentheses `(1,234)` | Parsed as positive 1234. Accounting negatives therefore match their positive FACTS entry, which is acceptable for a screen | `check_numbers.py:32` (DERIVED) |
