# Module Map

Built by hand rather than with `codemap` (see [README § Method](README.md#method-and-skills-used)). 21 tracked files.

## Directory tree

```
stock-deepdive/                                  (repo root = marketplace root)
|-- .claude-plugin/marketplace.json      14   Claude Code marketplace "boonchu-tools" → ./plugins/stock-deepdive-lite
|-- .agents/plugins/marketplace.json     19   Codex marketplace (same name, local source, policy flags)
|-- AGENTS.md                            58   Instructions for agents working in the repo; invariants for editing
|-- README.md                           156   User docs (Thai): install, usage, limits, timings
|-- LICENSE                              21   MIT, © 2026 boonchu
|-- .gitignore                           11   /data/ /paper/ /deepdive/ anchored to root
|-- stock-deepdive-guide.pptx          (bin)  34-slide guide; byte-identical to docs/ copy
|-- docs/stock-deepdive-guide.pptx     (bin)  same SHA-1 397bf2b3…
`-- plugins/stock-deepdive-lite/                 (= <ราก>)
    |-- .claude-plugin/plugin.json       21   Claude Code manifest (no skills/agents keys → default dirs)
    |-- .codex-plugin/plugin.json        38   Codex manifest (skills: ./skills/, UI metadata, default prompts)
    |-- ROOT.md                          43   How each host resolves <ราก>; mandatory --help smoke check
    |-- skills/
    |   |-- deepdive/SKILL.md           254   Orchestration playbook: steps 0–4, lanes, CSS vocabulary, editorial rules
    |   `-- fetch/SKILL.md               81   Fetch playbook: market choice, US/TH commands, post-fetch extract
    |-- agents/
    |   |-- factcheck.md                 59   Number auditor (tools: Read, Grep, Glob, Bash, Write)
    |   |-- writer.md                    61   Report writer, 11 topics (tools: Read, Write, Edit, Grep, Glob, Bash)
    |   `-- critic.md                    50   Logic auditor (tools: Read, Write); never invoked
    `-- bin/
        |-- fetch_us.py                 221   EDGAR: ticker→CIK, filings, EX-99 exhibits, XBRL companyfacts
        |-- fetch_th.py                 360   iDisc: zip discovery/download/extract; docx/xlsx→text library
        |-- extract.py                  202   Re-convert Office, PDF→paged text, scan detection, MANIFEST.md
        |-- build_paper.py              211   Assemble <section> parts into offline HTML; structure checks
        `-- check_numbers.py            151   Screen report numbers against FACTS.md
```

## Script modules: public surface

### `P/bin/fetch_us.py`
| Symbol | Lines | Role |
|---|---|---|
| `TICKERS_URL`, `SUBMISSIONS_URL`, `FACTS_URL`, `ARCHIVE_URL`, `INDEX_URL` | 36-40 | EDGAR endpoints |
| `get(url, ua)` | 45-54 | GET with UA, 0.2 s sleep, gzip sniffing |
| `resolve_cik(ticker, ua)` | 57-63 | Exact ticker match in `company_tickers.json`, returning (CIK10, title) |
| `pick_filings(recent, plan)` | 66-80 | First-N per form from the `recent` arrays, de-duplicated by accession |
| `fetch_exhibits(...)` | 83-104 | Picks EX-99 from the filing index, else other `EX-*` except 101/104 |
| `save_facts(...)` | 107-133 | Writes `companyfacts.json` and `companyfacts_index.txt` |
| `main()` | 136-217 | CLI. Flags: `--out --contact --annuals --quarters --events --no-exhibits --no-facts` |

### `P/bin/fetch_th.py`
| Symbol | Lines | Role |
|---|---|---|
| `INDEX_FS`, `INDEX_561`, `DOWNLOAD`, `UA` | 41-45 | iDisc endpoints and fixed UA |
| `get(url, timeout)` | 53-57 | GET with 0.4 s sleep |
| `find_company(html)` | 60-64 | `<title>` text, shown to the user for the collision check |
| `find_zips(html)` | 67-75 | Regex for `FILEID=…zip`, keeping order and de-duplicating |
| `label_from_fileid(fid)` | 78-83 | `dat/news/YYYYMM/NAME.zip` → `YYYYMM_<NAME[:20]>` |
| `_xlsx_sheets(z)` | 87-129 | Reads sheets: shared strings, inline strings, trailing-blank trimming |
| `xlsx_to_text(path)` | 132-141 | Tab-separated sheet dump |
| `_para_text`, `_table_lines` | 144-158 | docx paragraph and table linearisation |
| `_docx_embed_rels`, `_embedded_table_lines` | 161-188 | Resolve and unpack OLE-embedded workbooks |
| `docx_to_text(path)` | 191-232 | Body walk with embedded tables inlined in place, plus a leftovers section |
| `convert_folder(folder)` | 235-252 | Converts a period folder and reports `.doc/.xls` as skipped |
| `main()` | 255-356 | CLI. Flags: `--out --periods --no-text --no-561` |

### `P/bin/extract.py`
| Symbol | Lines | Role |
|---|---|---|
| `SCAN_REL`, `SCAN_FLOOR` | 44-45 | Scan heuristic constants (0.15, 150) |
| `_pdf_pages(path)` | 48-72 | PyMuPDF → pypdf → explicit error |
| `pdf_to_text(path)` | 75-88 | `[[หน้า N]]` page markers plus scan flags |
| `ranges(nums)` | 91-101 | `[3,4,5,9]` → `"3-5, 9"` |
| `main()` | 104-198 | Recursive walk, writes `MANIFEST.md`. Flag: `--no-pdf` |

### `P/bin/build_paper.py`
| Symbol | Lines | Role |
|---|---|---|
| `CSS` | 34-105 | Dark theme and the full class vocabulary used by the playbook |
| `DISCLAIMER` | 107-111 | Fixed "not investment advice" footer |
| `STRIP`, `clean_fragment` | 113-130 | Remove document-level tags and scripts from fragments |
| `check(doc)` | 133-146 | Tag balance for 7 tags, external `src`, inline `style=` |
| `main()` | 149-207 | CLI. Flags: `--parts --out --title --subtitle --tag* --sources` |

### `P/bin/check_numbers.py`
| Symbol | Lines | Role |
|---|---|---|
| `NUM`, `CALC_MARK`, `SCALES`, `REL_TOL`, `MIN_MAGNITUDE` | 32-36 | Matching parameters |
| `parse_nums`, `is_noise`, `matches` | 39-68 | Tokenise, filter, match |
| `main()` | 71-147 | CLI: `report facts [--strict] [--out]` |

## Prompt modules: inputs and outputs

| Module | Invoked by | Reads | Writes |
|---|---|---|---|
| `skills/deepdive/SKILL.md` | User | MANIFEST, claims, check output | `_parts/05-front.html`; drives everything else |
| `skills/fetch/SKILL.md` | User or deepdive step 0.2 | Script output | `data/<T>/` (via scripts) |
| `agents/factcheck.md` | deepdive step 1 | `data/<T>/*` | `paper/<T>-FACTS.md` (format given by orchestrator) |
| `agents/writer.md` ×3 | deepdive step 2 | `FACTS.md`, `MANIFEST.md` (and, per its own text, source files) | `_parts/{10-a,20-b,30-c}.html`, `claims-{a,b,c}.jsonl` |
| `agents/critic.md` | Nobody | (report) | (verdict) |

## Host wiring

| Host | Marketplace | Plugin manifest | Skill discovery | Agent discovery |
|---|---|---|---|---|
| Claude Code | `.claude-plugin/marketplace.json` → `source: ./plugins/stock-deepdive-lite` | `P/.claude-plugin/plugin.json` | Default `skills/` dir (INFERRED from the missing key) | Default `agents/` dir. Invoked as `stock-deepdive-lite:factcheck`, etc. |
| Codex | `.agents/plugins/marketplace.json` → `{source: local, path}` | `P/.codex-plugin/plugin.json` | `"skills": "./skills/"` (`:20`) | None declared. The orchestrator reads `agents/*.md` as plain Markdown (`SKILL.md:80-81`) |
