# Invariants

MUST = breaking it breaks a host or the core guarantee. SHOULD = documented convention.

## Packaging (MUST)
- INV-P1 Plugin content stays under `plugins/stock-deepdive-lite/` — Codex returns `plugin not found` for root-level paths (AGENTS.md:49-51, commit 0cb491d).
- INV-P2 Manifest pairs change together (AGENTS.md:42-47). Names: marketplace `boonchu-tools`, plugin `stock-deepdive-lite`, version kept equal in both `plugin.json`.
- INV-P3 Skill command blocks show two lines — `${CLAUDE_PLUGIN_ROOT}` (Claude Code) and `<ราก>` (Codex) — with "run only one" at the top of each skill (skills/deepdive/SKILL.md:45-50; commit 3749845). *AGENTS.md:39-40 still says the opposite; treat the commit as authoritative and fix AGENTS.md.*
- INV-P4 `<ราก>` resolution is documented only in `ROOT.md`.

## Number pipeline (MUST)
- INV-N1 Writers take numbers only from FACTS.md; computed values carry `[คำนวณ]` with the method (AGENTS.md:21-22; SKILL.md:130-131).
- INV-N2 FACTS rows: `# | รายการ | ค่า | หน่วย | งวด | ไฟล์ + ตำแหน่ง | ชั้น`; raw value and unit as written; tier on every row (SKILL.md:100-112).
- INV-N3 Conflicting source values are both recorded (SKILL.md:114).

## Presentation (MUST)
- INV-H1 Agents emit `<section>` fragments only — no doctype/html/head/style/body, no `style=` (SKILL.md:133); classes only from the whitelist (SKILL.md:144-150), which must match the CSS in `bin/build_paper.py`.
- INV-H2 Output works offline — no external `src` (build_paper.py:140-143).
- INV-H3 Part file names sort in section order: `05-front`, `10-a`, `20-b`, `30-c`.

## Editorial (MUST, prompt-level)
- INV-E1 No buy/sell advice, no steering via scenario order (AGENTS.md:28).
- INV-E2 Missing data stated as missing (AGENTS.md:26).
- INV-E3 Anomaly ≠ intent (AGENTS.md:27).
- INV-E4 No writing without source documents (AGENTS.md:23-24).

## Scripts (SHOULD)
- INV-S1 Python stdlib only; PDF libs optional with explicit failure text.
- INV-S2 Force UTF-8 on stdout/stderr (Windows Thai console).
- INV-S3 Identifiable User-Agent; EDGAR contact required; throttle (0.2 s EDGAR, 0.4 s iDisc).
- INV-S4 Free, public, no-login sources only (skills/fetch/SKILL.md:79).
