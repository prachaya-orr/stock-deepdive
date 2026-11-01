# Reverse Engineering Report — `stock-deepdive`

Static analysis of `BoonchuTummeng/stock-deepdive` at commit `e71a2c2` (2026-08-16), produced 2026-11-01.
The first pass was static: nothing in the repository was executed, installed or modified. A later live check, approved by the user, ran the scripts against iDisc ([Live verification](#live-verification-2026-11-01)). All output lives under `docs/reverse-engineering/`.

## Short answer

`stock-deepdive` is not an application in the usual sense. It is an **LLM-agent plugin** (Claude Code + Codex) whose
"runtime" is an AI orchestrator following a Markdown playbook. It has three layers:

1. **Playbooks**: `skills/deepdive/SKILL.md` and `skills/fetch/SKILL.md`. These are the control flow.
2. **Role prompts**: `agents/factcheck.md`, `writer.md` and `critic.md`. These are the workers.
3. **Five deterministic Python CLIs** (`bin/*.py`, stdlib only). They fetch filings from SEC EDGAR and SEC Thailand iDisc, convert Office/PDF to text, assemble the HTML and screen numbers.

The central design idea is a **single-path number pipeline**: one agent builds `FACTS.md`, writers may only quote it, and a script
checks the article against it. The idea is sound. Its two load-bearing guarantees are weaker than the documentation claims:

- **The number screen is close to non-functional** (OBSERVED: the real script accepted fabricated numbers in a live run; see [Live verification](#live-verification-2026-11-01)). In `check_numbers.py` the scale list includes ×1e-3 and ×1e-6 (`check_numbers.py:34`), and after rescaling the tolerance floor is an absolute 1.0 (`:64`). The required `#` row-number column of `FACTS.md` (`skills/deepdive/SKILL.md:100`) puts 1…N into the fact pool. Together these let any invented number up to roughly 10⁸ "match". In addition, every value below 100 without a thousands comma is skipped (`:53`), so percentages, margins and EPS are never checked.
- **Writer isolation is contradicted by the writer's own prompt** (OBSERVED). `agents/writer.md:14-19` tells the writer to open `companyfacts.json` and `NOTES.txt`. That violates the rule that writers may only read `FACTS.md` (`AGENTS.md:21`, `skills/deepdive/SKILL.md:130`).

## Documents

| File | Contents |
|---|---|
| [ARCHITECTURE.md](ARCHITECTURE.md) | Layers, boundaries, external services, Mermaid architecture diagram, tradeoffs |
| [MODULE_MAP.md](MODULE_MAP.md) | Every file, its responsibility, dependencies, size |
| [EXECUTION_FLOWS.md](EXECUTION_FLOWS.md) | End-to-end `/deepdive`, `fetch_th`, `fetch_us`, `extract`, `build_paper`, `check_numbers`; error handling and exit codes; sequence diagrams |
| [BUSINESS_LOGIC.md](BUSINESS_LOGIC.md) | Business rules (with enforcement level), algorithms, state machines, edge cases |
| [DATA_MODELS.md](DATA_MODELS.md) | File-system "schema", FACTS/claims/MANIFEST formats, filing records |
| [DESIGN_DECISIONS.md](DESIGN_DECISIONS.md) | 15 implicit ADRs recovered from code, prompts and commit history |
| [REIMPLEMENTATION_BLUEPRINT.md](REIMPLEMENTATION_BLUEPRINT.md) | Clean-room blueprint, essential vs incidental, fixes, alternatives, licensing |
| [diagrams/](diagrams/) | Standalone Mermaid sources |
| [skill-draft/](skill-draft/) | Draft agent skill for maintaining this plugin |

## Evidence labels

| Label | Meaning |
|---|---|
| **OBSERVED** | Read directly in a source file at the cited line |
| **DERIVED** | Follows mechanically from observed code. Probes P1–P3 are reproduced with a re-typed copy of the logic (see below) |
| **REPORTED** | Claimed by the README, slides or a commit message, and not verifiable from the repository alone |
| **INFERRED** | The analyst's interpretation. It could be wrong |
| **UNKNOWN** | Cannot be answered without running the code or the network |

Citations are `path:line`. `P/` is shorthand for `plugins/stock-deepdive-lite/`.

## Method and skills used

All four requested skills were installed, and their `SKILL.md` files were read before use.

| Skill | How it was used | What was not done |
|---|---|---|
| `software-architecture-analysis` | Phase 1–3 structure: key-file reading, architecture mapping, external dependencies, Mermaid-only diagrams, clean-room rule for the blueprint, observed/inferred/reported/unknown labelling | It suggests `git clone` into `/tmp`. Not needed, because the repository is local |
| `reverse-engineering-investigator` (T-REX) | Evidence discipline (OBSERVED / DERIVED / INFERRED / UNKNOWN), small decisive probes, bounded absence searches (for example, "is `critic` invoked anywhere?") | Its Ghidra/Rizin providers and `INVESTIGATION.json` state files are for binary targets. They were not used, because this target is Python and Markdown |
| `reverse-engineering` (Rosetta) | Phase 4: intent over mechanism, implicit state machines, consolidating scattered rules, flagging duplicate terminology, "would we rebuild this?" filter | Its prerequisite `codemap` writes `CODEMAP.md` at the workspace root. That would break the write-only-under-`docs/` rule, so the module map was built by hand ([MODULE_MAP.md](MODULE_MAP.md)) |
| `codebase-constitution` (installed name of **repo-to-skill**) | Phase 6: the skill-draft layout follows its output contract (`SKILL.md`, `references/`, `evals/evals.json`) | Its pipeline is `scripts/orchestrate.py`. Running it would execute scripts, which the task rules forbid without permission, so the draft was written by hand from this analysis |

### Probes (DERIVED evidence)

Repository code was **not** imported or run. The matching rule (`P/bin/check_numbers.py:32-68`) and the scan cutoff
(`P/bin/extract.py:44-83`) were re-typed into a scratch script and run against synthetic inputs:

| Probe | Input | Result |
|---|---|---|
| P1 | FACTS with 100 rows (`#` = 1…100). Report contains fabricated 123 · 4,567 · 98,765 · 3,141,592 · 77,777,777 · 101,000,000 · 250,000,000 | **All except 250,000,000 "matched"** |
| P1b | Same facts with the `#` column removed | 3,141,592 still matched, against a line-number reference (`NOTES.txt บรรทัด 3145`) |
| P2 | `45.3`, `12.5`, `99`, `0.87` | All classified as noise and never checked |
| P3a | PDF with text pages of 2,500 chars plus 3 scan pages of 40–108 chars | Scans detected (cutoff 150) |
| P3b | **Entirely scanned** PDF (all pages 40–108 chars) | **Zero pages flagged**, because the cutoff collapses to 9.75 |
| P3c | Median 600 chars, one 100-char scan page | Not flagged (cutoff 90) |

### Live verification (2026-11-01)

The user approved running the real scripts and making network calls. Everything was written to a temporary session folder; nothing was written to the repository.

| Test | Command / input | Result |
|---|---|---|
| L1 iDisc title | GET `fs-full?searchSymbol=DITTO` / `TACC` | `<title>` = `Financial Statement (Full version) - DITTO (THAILAND) PUBLIC COMPANY LIMITED` and `… - T.A.C.CONSUMER PUBLIC COMPANY LIMITED`. **The company name is present**, so the collision check works |
| L2 Unknown symbol | `searchSymbol=QQZZX` | Title without a company name, 0 zips. `fetch_th.py` exits 1 (`:282-285`) |
| L3 Index order | FILEIDs on the DITTO and TACC pages | **Newest first** (`202608 → 202411`, 11 periods each) |
| L4 `fetch_th.py DITTO --periods 2` | Real script | 10.5 s total. 2 FS periods + two 56-1 PDFs (17.4 MB + 14.1 MB). `AUDITOR_REPORT.DOC` is legacy `.doc` in both periods, so it cannot be converted |
| L5 Embedded tables | `NOTES.txt` after conversion | **47 and 49** `ตารางฝัง` blocks. The OLE-embedded recovery (ADR-08) works on real filings |
| L6 56-1 index | GET `fs-r561?searchSymbol=DITTO` | Only 2 entries: **FY2025** (filed 31/03/2026) and **FY2021** (filed 29/03/2022). FILEIDs are `dat/f56/…zip`, giving fallback labels `R561_dat_f56_F1640E21_zip` (findings 11, 12) |
| L7 `extract.py`, no PDF library | Real script | Clear install hint printed. PDFs listed as "อ่านไม่ได้" (unreadable). It does not fail silently |
| L8 `extract.py` with `pypdf==5.1.0` (temporary venv) | Real script, 4.3 s | 415-page One Report: 120 pages flagged as scanned (including 308–413). 343-page 2021 report: 102 flagged. Scan detection **works on mixed documents**. The fully scanned case (P3b) has not been seen live |
| L9 `check_numbers.py` | Real script. FACTS with 100 rows, plus a report with 1 genuine and 6 fabricated figures | `ตรวจ 5 ตัว · หลุด 1 ตัว (80.0%)`, exit 0. **98,765 · 4,567 · 77,777,777 passed**. Only 250,000,000 was caught. `45.3%` and `0.87` were not checked. This confirms P1/P2 against the real code |

## Coverage

| Area | Coverage |
|---|---|
| `P/bin/*.py` (5 files, 1,145 lines) | **Complete**. Every line was read |
| `P/skills/*/SKILL.md`, `P/agents/*.md`, `P/ROOT.md`, `AGENTS.md`, `README.md` | **Complete** |
| Manifests (4 JSON files), `.gitignore`, `LICENSE` | **Complete** |
| `docs/stock-deepdive-guide.pptx` (binary; identical SHA-1 to the root copy) | **Partial**. Slide text was extracted with a read-only parser. 19 images were not inspected |
| Git history (13 commits) | Messages read. Diffs read only for `3749845` (stat) |
| Runtime behaviour against SEC/iDisc | **Partial**. With the user's permission, a live run was done for iDisc (DITTO, TACC) plus local runs of `extract.py` and `check_numbers.py`. EDGAR has **not** been run yet (it needs the user's contact e-mail). See [Live verification](#live-verification-2026-11-01) |

## Key findings (ranked)

1. **Number screen is nearly vacuous** (DERIVED, P1/P2). See [BUSINESS_LOGIC.md § BR-02](BUSINESS_LOGIC.md#br-02-number-screen). The README claims the script checks "that no number escapes the table" (`README.md:20`). The mechanism cannot support that claim.
2. **Writer prompt contradicts the isolation rule** (OBSERVED). `P/agents/writer.md:14-19` versus `AGENTS.md:21`. The slides even claim "the writer never opens the statements" (slide 24, REPORTED).
3. **Topic list drift.** `P/agents/writer.md:21-37` defines **11** topics, including "what management dodged when asked" (`:34`). The playbook, README and slides define **10** sections (`P/skills/deepdive/SKILL.md:208-219`). The README itself says transcripts are unavailable, so that question cannot be answered (`README.md:104`).
4. **`critic` agent is dead code** (OBSERVED, bounded search). It is defined (`P/agents/critic.md`) and listed (`AGENTS.md:13`, `SKILL.md:80`) but never invoked. Slide 22 confirms "4 agents per run" (factcheck ×1 + writer ×3).
5. **`factcheck` role mismatch** (OBSERVED). Its prompt is written to *audit an existing report* (`P/agents/factcheck.md:15`, verdict format `:52-59`). The playbook uses it to *produce* `FACTS.md` before any report exists (`SKILL.md:95-100`). The FACTS table schema exists only in the orchestrator's prompt.
6. **Trust-tier model cannot represent management statements** (OBSERVED). `FACTS.md` has 3 tiers (filed / secondary market / headline; `SKILL.md:103-104`). The policy names 4 layers, with management statements separate (`AGENTS.md:25`). `writer.md:47-51` uses yet another 4-icon scheme.
7. **Fully scanned PDFs are reported as fully readable** (DERIVED, P3b). `min()` turns the 150-char "floor" into a ceiling (`P/bin/extract.py:82`). The comment says it is a floor (`:43`).
8. **`MANIFEST.md` is blind to US filings** (OBSERVED). `extract.py` lists only `.docx/.xlsx/.pdf` (`:123-146`), while `fetch_us.py` saves `.htm` filings and JSON (`:188`, `:115`). The playbook requires every agent to work from MANIFEST (`SKILL.md:65`).
9. **`AGENTS.md` is stale against a deliberate later decision** (OBSERVED). It forbids `${CLAUDE_PLUGIN_ROOT}` in shared files (`AGENTS.md:39-40`). Commit `3749845` intentionally restored it in both skills (`SKILL.md:47,59,167,185`), with the rationale that a model copying `<ราก>` literally makes the shell read `<` as a redirect.
10. **"Already fetched" means "folder exists"** (OBSERVED, `SKILL.md:54`). A partially failed fetch (failures are logged and skipped: `fetch_th.py:295-297`, `fetch_us.py:193-194`) is indistinguishable from a complete one at the next run.
11. **The second 56-1 may be years old and its folder name does not show the year** (OBSERVED live). iDisc listed only the 2025 and 2021 One Reports for DITTO. `fetch_th.py` takes the first two (`:324`), so a 2021 report lands next to the 2025 one with nothing marking its age.
12. **56-1 folder labels are unreadable** (OBSERVED live). 56-1 FILEIDs look like `dat/f56/<name>.zip`, with no `YYYYMM/` part, so `label_from_fileid` (`fetch_th.py:78-83`) takes its fallback branch and produces names like `R561_dat_f56_F1640E21_zip`.

## Unresolved questions

- ~~Does iDisc's `<title>` contain the company name?~~ **Answered: yes** (live). See Live verification.
- ~~Does the iDisc index list zips newest-first?~~ **Answered: yes** for the financial-statement index (live, 2 tickers).
- UNKNOWN: Is the 40.6-minute timing reproducible? It is REPORTED (`README.md:7-8,114-122`). Only the fetch stage was timed live: 10.5 s versus the reported 4.2 min. A full `/deepdive` run (4 agents) has not been done.
- UNKNOWN: Is `MANIFEST.md` empty for US tickers in practice (finding 8)? This waits on an EDGAR run.
- INFERRED: `check_numbers.py`'s docstring examples use `./deepdive/*.md` (`:17`). `.gitignore` still ignores `/deepdive/` (`.gitignore:5`). An earlier Markdown-report version probably existed and the HTML path was added later. The CLI help still says "report .md" (`:74`).
- UNKNOWN: Codex manifest declares `capabilities: ["Read","Write"]` (`P/.codex-plugin/plugin.json:27-30`), while the skills need shell and network. Whether Codex enforces this field could not be tested.

## Suggested next steps

1. Fix `check_numbers.py` matching. Parse only the `ค่า` (value) column of FACTS. Drop the absolute floor after rescaling, or only rescale by declared units. Check numbers below 100 that have decimals or a `%`. (Blueprint §4.)
2. Rewrite `agents/writer.md` to the 10-section contract and remove the instructions that open source files.
3. Either wire `critic` into Step 3 or delete it. Split `factcheck` into a FACTS producer prompt and keep the audit prompt.
4. Change the scan cutoff to `max(floor, median × rel)` capped sensibly, or flag documents where most pages fall under an absolute threshold.
5. Make `extract.py` list `.htm`/`.json` and record fetch completeness, for example a `FETCH_STATUS` line in MANIFEST.
6. Update `AGENTS.md` to the "two lines, run one" convention introduced by `3749845`.
7. With permission: run `fetch_*` against one TH and one US ticker to settle the iDisc UNKNOWNs.
