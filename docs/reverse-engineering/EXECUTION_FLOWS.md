# Execution Flows

`P/` = `plugins/stock-deepdive-lite/`. All script execution is **synchronous and single-threaded**. The only concurrency
is the three writer agents launched "together in one message" (`P/skills/deepdive/SKILL.md:84,120`).

## F1. `/stock-deepdive-lite:deepdive <TICKER>`: end to end

```mermaid
sequenceDiagram
    autonumber
    actor User
    participant O as Orchestrator LLM<br/>(deepdive/SKILL.md)
    participant F as fetch skill + fetch_*.py
    participant X as extract.py
    participant FC as factcheck agent
    participant WA as writer A
    participant WB as writer B
    participant WC as writer C
    participant B as build_paper.py
    participant C as check_numbers.py

    User->>O: deepdive TICKER
    O-->>User: "takes ~35-45 min" (SKILL.md:8)
    O->>O: 0.0 choose command line per host (SKILL.md:45-50)
    alt market ambiguous
        O-->>User: ask which market (SKILL.md:52)
    end
    alt ./data/TICKER/ missing (SKILL.md:54)
        O->>F: fetch skill
        F-->>O: files + printed company name
    end
    O->>X: extract.py ./data/TICKER (always, SKILL.md:56)
    X-->>O: MANIFEST.md
    O->>O: read MANIFEST (SKILL.md:65)
    alt no source documents
        O-->>User: stop, say where it failed, ask (SKILL.md:73-74)
    end
    O->>O: can host spawn sub-agents? (SKILL.md:78-91)
    O->>FC: build paper/TICKER-FACTS.md (80-120 rows, raw units, tier 1/2/3)
    FC-->>O: FACTS.md
    par lanes in one message (SKILL.md:120)
        O->>WA: sections 1-3, cap 12,000 chars
        O->>WB: sections 4-6, cap 14,000 chars
        O->>WC: sections 7-10, cap 14,000 chars
    end
    WA-->>O: _parts/10-a.html + claims-a.jsonl
    WB-->>O: _parts/20-b.html + claims-b.jsonl
    WC-->>O: _parts/30-c.html + claims-c.jsonl
    O->>O: write _parts/05-front.html itself (SKILL.md:156)
    O->>B: --parts ./paper/_parts --out ... --title ...
    alt exit 2 (structure problem)
        B-->>O: problem list (file already written)
        O->>O: fix fragments, rebuild (SKILL.md:180)
    else exit 0
        B-->>O: OK + stats
    end
    O->>C: check_numbers.py paper.html FACTS.md
    C-->>O: checked / orphans (exit 0 unless --strict)
    O->>O: per orphan: add to FACTS, mark [คำนวณ], or delete (SKILL.md:189)
    O->>O: 3.4 merge claims-*.jsonl, find opposite-direction pairs, fix (SKILL.md:192-194)
    O-->>User: path, sections, time, check stats, gaps, one standout finding (SKILL.md:200-204)
```

Source: [diagrams/deepdive-sequence.md](diagrams/deepdive-sequence.md)

**Degraded mode** (OBSERVED, `SKILL.md:86-91`). If the host cannot spawn sub-agents, the orchestrator runs lanes A → B → C itself,
reading `agents/writer.md` as its own instructions. It must still write all `claims-*.jsonl` files, and it tells the user the run will take 60–75 min.

**Not in the flow** (OBSERVED, bounded search): the `critic` agent. The factcheck agent is used once, to *produce* FACTS. It is never used to
audit the finished article, even though its prompt is written for that (`P/agents/factcheck.md:15,52-59`).

### Implicit run state machine

The run has no explicit state. Its state is implied by which artefacts exist (Rosetta "implicit state machine", INFERRED):

```mermaid
stateDiagram-v2
    [*] --> NoData
    NoData --> Fetched: fetch_* wrote data/T/ (even partially)
    NoData --> Blocked: fetch failed / no docs
    Blocked --> [*]: ask user (SKILL.md:73)
    Fetched --> Indexed: extract.py → MANIFEST.md
    Indexed --> FactsReady: factcheck → paper/T-FACTS.md
    FactsReady --> Drafted: 3 parts + 3 claims files
    Drafted --> Fronted: 05-front.html
    Fronted --> Assembled: build_paper exit 0
    Fronted --> Fronted: build_paper exit 2 → fix
    Assembled --> Screened: check_numbers (always exit 0)
    Screened --> Reconciled: claims conflicts fixed
    Reconciled --> Delivered
    Delivered --> [*]
```

Source: [diagrams/pipeline-state.md](diagrams/pipeline-state.md). Two consequences follow:

- `Fetched` is entered even after partial failure, and later runs skip fetching because the folder exists (`SKILL.md:54`).
- `Screened` cannot fail unless `--strict` is passed, and the playbook never passes it (`SKILL.md:185-186`, `check_numbers.py:147`).

## F2. `fetch_th.py <SYMBOL>`: Thai filings from iDisc

```mermaid
sequenceDiagram
    autonumber
    participant CLI as fetch_th.py
    participant ID as market.sec.or.th (iDisc)
    participant FS as data/SYMBOL/

    CLI->>ID: GET /idisc/en/Viewmore/fs-full?searchSymbol=SYM (sleep 0.4s)
    alt network error
        CLI-->>CLI: print, exit 1 (fetch_th.py:272-274)
    end
    CLI->>CLI: find_company(page title tag) → print + collision warning (:276-279)
    CLI->>CLI: find_zips(html) → FILEIDs in page order (:281)
    alt none
        CLI-->>CLI: exit 1 (:282-285)
    end
    loop first --periods (default 6) zips
        CLI->>ID: GET /idisc/Download?FILEID=...
        alt error
            CLI-->>CLI: print FAILED, continue (:295-297)
        end
        CLI->>FS: FS_<label>/package.zip → extractall → delete zip
        CLI->>FS: convert_folder: *.docx/*.xlsx → *.txt (embedded workbooks inlined)
    end
    opt not --no-561
        CLI->>ID: GET fs-r561?searchSymbol=SYM
        loop first 2 zips
            CLI->>ID: GET Download (timeout 90s)
            Note over CLI: no per-item try: one failure skips the rest (:328, :346)
            CLI->>FS: R561_<label>/ extract + convert
        end
    end
    CLI-->>CLI: print "what iDisc lacks" checklist, exit 0
```

Source: [diagrams/fetch-th-sequence.md](diagrams/fetch-th-sequence.md)

## F3. `fetch_us.py <TICKER>`: US filings from EDGAR

```mermaid
sequenceDiagram
    autonumber
    participant CLI as fetch_us.py
    participant SEC as www.sec.gov / data.sec.gov
    participant FS as data/TICKER/

    CLI->>CLI: contact = --contact or $SEC_CONTACT, empty → exit 2 (:150-155)
    CLI->>SEC: GET files/company_tickers.json
    CLI->>CLI: exact ticker match → CIK10 + title, none → SystemExit exit 1 (:57-63)
    opt not --no-facts
        CLI->>SEC: GET api/xbrl/companyfacts/CIK##########.json
        CLI->>FS: companyfacts.json + companyfacts_index.txt (errors → skip)
    end
    CLI->>SEC: GET submissions/CIK##########.json
    CLI->>CLI: pick domestic plan {10-K:2, 10-Q:4, 8-K:6, DEF 14A:1}
    alt no 10-K/10-Q picked
        CLI->>CLI: switch to foreign plan {20-F:2, 6-K:max(events,quarters)} (:176-179)
    end
    loop each picked filing
        CLI->>SEC: GET Archives/edgar/data/CIK/ACCN/primaryDoc
        CLI->>FS: FORM_DATE_ACCN.ext (error → print, continue)
        opt form in {8-K, 6-K} and exhibits enabled
            CLI->>SEC: GET .../ACCN/index.json
            loop EX-99 items (or other EX-* if no EX-99)
                CLI->>SEC: GET exhibit
                CLI->>FS: FORM_DATE_ACCN_TYPE.ext
            end
        end
    end
    CLI-->>CLI: print "what EDGAR lacks", exit 0
```

Source: [diagrams/fetch-us-sequence.md](diagrams/fetch-us-sequence.md)

## F4. `extract.py <folder>`

1. Recursively walk `folder` in sorted order (`extract.py:118`).
2. `.docx`/`.xlsx`: convert with `fetch_th`'s converters, overwrite `<stem>.txt`, and count embedded tables by marker (`:123-130`).
3. `.pdf` (case-insensitive, unless `--no-pdf`): extract text per page with PyMuPDF, falling back to pypdf. Prefix `[[หน้า N]]`. Flag pages whose length is below `min(150, median×0.15)` (`:82-83`).
4. `.doc`/`.xls`: list as unconvertible (`:145-146`).
5. **Everything else (`.htm`, `.json`, `.txt`) is ignored** and does not appear in MANIFEST (OBSERVED).
6. Write `MANIFEST.md`: a table, plus a "scanned pages" section and an "unconvertible" section (`:148-188`). Exit 0. Exit 1 if the folder is missing.

## F5. `build_paper.py`

1. Collect `--parts/*.html` and sort by filename (`:163`). Ordering is a naming convention: `05-front`, `10-a`, `20-b`, `30-c`.
2. Header: tags (escaped), `<title>` (escaped), `<h1>` and subtitle (**raw**) (`:168-179`).
3. Body: each fragment passes through `clean_fragment`, which strips doctype/html/head/style/script and anything outside `<body>` (`:124-130`).
4. Footer: sources (raw), date and fixed disclaimer (`:183-187`).
5. **Write the file**, print stats, **then** `check()`. Problems give exit 2 (`:190-205`).

## F6. `check_numbers.py report facts`

1. Tokenise **all** of FACTS.md, including the `#`, period, file and line-reference columns, into a float list (`:87`).
2. For each report line, skip headings, separator rows and blank lines (`:95-96`). Skip the whole line if it contains `[คำนวณ]` (`:97-99`).
3. For each number token: skip noise, which is anything below 100 without a comma, or a bare year 1900–2100 / 2400–2700 (`:50-58`).
4. Match under scales ×{1, 1e3, 1e-3, 1e6, 1e-6} with tolerance `max(|t|×0.6%, 1.0)` (`:61-68`).
5. Print the summary and up to 15 orphans. Exit 0, or exit 1 only with `--strict` (`:147`).

When run on HTML, lines are raw markup. Inline CSS lives in `<head>` as one `<style>` element whose content spans
many lines (`build_paper.py:174`), so hex colours such as `#141f2b` yield the token `141`. These are then "matched" via the ×1e-3 scale (DERIVED).

## Error handling and exit codes (consolidated)

| Script | Condition | Behaviour | Line |
|---|---|---|---|
| `fetch_us.py` | No contact | stderr message, **exit 2** | 151-155 |
| | Ticker not in SEC list | `SystemExit(msg)` → **exit 1** | 63 |
| | No supported filings | `SystemExit(msg)` → exit 1 | 180-181 |
| | companyfacts fetch fails | Warn, continue | 110-114 |
| | Filing or exhibit fetch fails | Warn, continue | 193-194, 206-207 |
| | `index.json` fails | Warn, no exhibits | 85-89 |
| `fetch_th.py` | Index page fails | **exit 1** | 272-274 |
| | No zips | exit 1 | 282-285 |
| | Period download fails | Warn, continue | 295-297 |
| | Zip extraction fails | Keep raw zip, still try conversion | 307-308 |
| | Any 56-1 error | Abort all 56-1, warn | 346-347 |
| `extract.py` | Folder missing | exit 1 | 111-113 |
| | No PDF library | Recorded in MANIFEST, warn | 64-68, 138-139 |
| `build_paper.py` | No parts dir / no `.html` | exit 1 | 160-166 |
| | Structure problems | **exit 2** (output already written) | 200-205 |
| `check_numbers.py` | Missing file / no numbers in FACTS | exit 2 | 82-90 |
| | Orphans | exit 0, or 1 with `--strict` | 147 |

**Retries:** none anywhere. **Timeouts:** EDGAR 30 s (`fetch_us.py:49`), iDisc 40 s, or 90 s for 56-1 (`fetch_th.py:53,328`).
**State changes:** only file writes. No deletions apart from the downloaded `package.zip` after a successful extract (`fetch_th.py:303,334`).
