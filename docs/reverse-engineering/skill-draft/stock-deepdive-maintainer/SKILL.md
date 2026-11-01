---
name: stock-deepdive-maintainer
description: Use when changing, debugging or reviewing the stock-deepdive plugin (plugins/stock-deepdive-lite) — its skills/agents prompts, bin/*.py scripts, or the Claude Code/Codex manifests. Covers the invariants that must hold across both hosts, the pipeline contracts between stages, and the known defects to avoid re-introducing or to fix. Not for running a stock analysis (use the plugin's own deepdive/fetch skills for that).
---

# Stock DeepDive — maintainer guide

DRAFT generated from a static reverse-engineering pass (docs/reverse-engineering/, commit e71a2c2).
Verify line references before relying on them; they drift with edits.

## Mental model (read first)

The plugin is a **Markdown playbook run by an LLM** plus **five stdlib Python CLIs**.
Control flow lives in `skills/deepdive/SKILL.md`; workers are `agents/*.md`; anything deterministic is a script in `bin/`.
Numbers reach the article only through `paper/<T>-FACTS.md`. Details: [references/architecture.md](references/architecture.md).

## Before any change

1. Identify which layer you touch: playbook · role prompt · script · manifest.
2. Read [references/invariants.md](references/invariants.md) — MUST rules that break one host or the number pipeline if violated.
3. Check [references/known-defects.md](references/known-defects.md) — do not "fix" behaviour other parts depend on without updating them together.

## Change procedures

**Editing a prompt (`skills/*/SKILL.md`, `agents/*.md`)**
- Keep the 10-section contract, tier scheme and lane split identical everywhere they appear (playbook, writer, factcheck, README, slides). Grep before and after: `grep -rn "หัวข้อ\|ส่วน\|ชั้น" plugins/ README.md`.
- Command blocks keep **two lines**: the `${CLAUDE_PLUGIN_ROOT}` line for Claude Code and the `<ราก>` line for Codex, plus the "run only one" warning. Do not collapse to a placeholder (commit 3749845 explains why).

**Editing a script (`bin/*.py`)**
- Stdlib only; optional imports must degrade with an explicit message (pattern in `extract.py` `_pdf_pages`).
- Keep the UTF-8 stdout/stderr reconfigure block at the top.
- Preserve file-name and marker formats consumed downstream (see [references/configuration.md](references/configuration.md)): `FS_<YYYYMM>_…`, `R561_…`, `[[หน้า N]]`, `----- ตารางฝัง #n …`, `MANIFEST.md` columns, part prefixes `05/10/20/30`.
- `extract.py` imports converters from `fetch_th.py` — renaming them breaks extraction.
- Do not run fetchers in tests without a contact (`SEC_CONTACT`) and without the user's consent: they hit public regulator APIs.

**Editing manifests**
- Change pairs together: `P/.claude-plugin/plugin.json` ↔ `P/.codex-plugin/plugin.json`; `.claude-plugin/marketplace.json` ↔ `.agents/plugins/marketplace.json`.
- Never move content out of `plugins/stock-deepdive-lite/` (Codex cannot load a root-level plugin).
- Validate both: `claude plugin validate .` and the Codex validator in `AGENTS.md`.

## Verifying a change without the network

- `python plugins/stock-deepdive-lite/bin/<script>.py --help` for each touched script (the playbook's own smoke check).
- For converters: build a small `.docx` with an embedded `.xlsx` and confirm the `ตารางฝัง` block appears in place.
- For `build_paper.py`: feed a parts dir with an unbalanced `<div>` → expect exit 2; with `style=` → exit 2.
- For `check_numbers.py`: use the probe cases in [references/examples.md](references/examples.md); a correct fix must reject fabricated numbers P1.

## Output when reviewing

Report findings as OBSERVED / DERIVED / INFERRED / UNKNOWN with `path:line`, and state which invariant or defect ID each relates to.
