# Reverse Engineering Report — `stock-deepdive`

Static analysis of `BoonchuTummeng/stock-deepdive` at commit `e71a2c2` (2026-08-16), produced 2026-11-01.
Nothing in the repository was executed, installed or modified; all output lives under `docs/reverse-engineering/`.

## Short answer

`stock-deepdive` is not an application in the usual sense. It is an **LLM-agent plugin** (Claude Code + Codex) whose
"runtime" is an AI orchestrator following a Markdown playbook. It has three layers:

1. **Playbooks**: `skills/deepdive/SKILL.md` and `skills/fetch/SKILL.md`. These are the control flow.
2. **Role prompts**: `agents/factcheck.md`, `writer.md` and `critic.md`. These are the workers.
3. **Five deterministic Python CLIs** (`bin/*.py`, stdlib only). They fetch filings from SEC EDGAR and SEC Thailand iDisc, convert Office/PDF to text, assemble the HTML and screen numbers.

The central design idea is a **single-path number pipeline**: one agent builds `FACTS.md`, writers may only quote it, and a script
checks the article against it. The idea is sound. Its two load-bearing guarantees are weaker than the documentation claims:

- **The number screen is close to non-functional** (DERIVED; reproduced with a probe). In `check_numbers.py` the scale list includes ×1e-3 and ×1e-6 (`check_numbers.py:34`), and after rescaling the tolerance floor is an absolute 1.0 (`:64`). The required `#` row-number column of `FACTS.md` (`skills/deepdive/SKILL.md:100`) puts 1…N into the fact pool. Together these let any invented number up to roughly 10⁸ "match". In addition, every value below 100 without a thousands comma is skipped (`:53`), so percentages, margins and EPS are never checked.
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

## Coverage

| Area | Coverage |
|---|---|
| `P/bin/*.py` (5 files, 1,145 lines) | **Complete**. Every line was read |
| `P/skills/*/SKILL.md`, `P/agents/*.md`, `P/ROOT.md`, `AGENTS.md`, `README.md` | **Complete** |
| Manifests (4 JSON files), `.gitignore`, `LICENSE` | **Complete** |
| `docs/stock-deepdive-guide.pptx` (binary; identical SHA-1 to the root copy) | **Partial**. Slide text was extracted with a read-only parser. 19 images were not inspected |
| Git history (13 commits) | Messages read. Diffs read only for `3749845` (stat) |
| Runtime behaviour against SEC/iDisc | **None**. Network calls are forbidden by the task rules, so the HTML structure of iDisc pages and the real `<title>` content are UNKNOWN |

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

## Unresolved questions

- UNKNOWN: Does iDisc's `<title>` actually contain the company name? `find_company` (`P/bin/fetch_th.py:60-64`) relies on it, and the README's main ticker-collision safeguard (`README.md:108`) depends on that output.
- UNKNOWN: Does the iDisc index list zips newest-first? `find_zips` preserves page order (`fetch_th.py:67-75`), but the docstring asserts newest→oldest (`:68`).
- UNKNOWN: Is the 40.6-minute timing reproducible? It is REPORTED (`README.md:7-8,114-122`). No logs or benchmark harness are in the repository.
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
