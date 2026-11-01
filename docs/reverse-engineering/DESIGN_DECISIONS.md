# Design Decisions (recovered ADRs)

These decisions were recovered from code comments, playbook text and commit history (`git log`, 13 commits from 2026-08-10 to 2026-08-16).
None are written down as ADRs in the repository. "Evidence" cites where each one is visible.

---

### ADR-01 Deterministic work belongs in scripts, judgement in agents
- **Decision:** Unpacking, PDF conversion, scan detection, HTML assembly and number screening are scripts (`AGENTS.md:30-34`, `P/skills/deepdive/SKILL.md:19-21`).
- **Rationale (REPORTED):** The same output every time, in seconds instead of "tens of minutes" (`README.md:127-128`).
- **Alternatives:** Let the agent parse files ad hoc (the rejected status quo). A single monolithic Python pipeline with LLM calls via API (not chosen; it would lose host-native sub-agents).
- **Consequence:** Scripts are the only mechanically reliable layer. Everything else depends on prompt compliance.

### ADR-02 Numbers enter the article by one path (FACTS.md)
- **Decision:** One factcheck pass builds a fact table, and writers quote it (`SKILL.md:15-17`).
- **Rationale:** Verify once, centrally, and screen the result by script (`:16-17`).
- **Alternatives:** Post-hoc auditing of every number in the finished article. That is what `factcheck.md` was originally written for (`P/agents/factcheck.md:15`), and `check_numbers.py:5-8` describes it as the slow approach being replaced.
- **Consequence:** The factcheck prompt was never rewritten for its new producer role (finding 5). The screen was designed to be tolerant (`check_numbers.py:10-11`), which undermines it (BR-02).

### ADR-03 Parallel lanes split by workload, not by topic count
- **Decision:** 3 writer lanes. Financials, valuation and scenarios are separated because they are heaviest (`SKILL.md:33-34,122-126`).
- **Rationale (REPORTED):** Wall time equals the slowest lane. The B lane is about 4× the others (`README.md:124-125`).
- **Alternatives:** One writer (serial, 60–75 min, now the fallback mode). Ten writers, one per section (more cross-lane contradictions, more orchestration).
- **Consequence:** Lanes cannot see each other, which needs ADR-04.

### ADR-04 Cross-lane contradiction detection through claim logs
- **Decision:** Each lane emits debatable claims with a direction. The orchestrator reconciles them (`SKILL.md:136-142,192-194`).
- **Rationale:** "Lanes that cannot see each other contradict each other unknowingly" (`:142`). In serial mode this matters more, because a later lane "vaguely remembers" an earlier one and smooths conflicts over (`:89-90`).
- **Alternatives:** A final critic pass over the whole article (the `critic` agent exists but is not wired in).

### ADR-05 Scripts own the HTML shell and CSS
- **Decision:** Agents write only `<section>` with a whitelisted class vocabulary. `build_paper.py` owns the doctype, CSS and disclaimer (`SKILL.md:39,133-134`, `build_paper.py:5-7`).
- **Rationale:** Agents reinvent CSS each run and sometimes over-draft and rewrite (`build_paper.py:5-6`).
- **Consequence:** Offline single-file output, enforced by the external-`src` check (`:140-143`).

### ADR-06 Explicit length caps and "write once"
- **Decision:** Every prompt carries a numeric character cap plus "no full rewrite" (`SKILL.md:29-31`).
- **Rationale:** Rewrites cost more than the original draft (`:31`).
- **Note:** Paired with "AI will always do more than asked, so budget 2–3× and don't forbid it" (`:36-37`). This is a deliberate choice to trade time for thoroughness.

### ADR-07 Stdlib-only Python, with optional PDF libraries
- **Decision:** Office parsing by hand (zip + XML), and PDF only if `pymupdf` or `pypdf` exists (`fetch_th.py:11-12`, `extract.py:17-19,48-72`).
- **Rationale:** Zero install friction for plugin users (`README.md:59-60`).
- **Alternatives:** `python-docx` and `openpyxl`. Neither exposes OLE-embedded workbooks inline by default (INFERRED), so a custom walk would be needed anyway.
- **Consequence:** `.doc`/`.xls` cannot be converted (`README.md:107`).

### ADR-08 OLE-embedded workbooks are first-class content
- **Decision:** Unpack embedded `.xlsx` inside `.docx` and inline it at its position (`fetch_th.py:191-232`).
- **Rationale:** Thai notes put the critical numbers there, sometimes in more than 100 tables (`:195-197`, `README.md:17`). This is the most domain-specific insight in the repository.

### ADR-09 Prefer structured XBRL over HTML for US numbers
- **Decision:** Always fetch `companyfacts.json` plus a tag index. Agents should use it rather than reading statement HTML (`P/skills/fetch/SKILL.md:40-42`, `fetch_us.py:107-133`).
- **Rationale:** "Cannot be misread", and periods and units are labelled.

### ADR-10 Regulator portals as the only sources
- **Decision:** EDGAR for US and iDisc for TH. No news, analysis, paid or login sources (`P/skills/fetch/SKILL.md:8-9,79-81`).
- **Rationale:** Mixing reliability tiers is "the origin of wrong analysis" (`:9`). iDisc is "the only free source of notes without login or Cloudflare" (`fetch_th.py:5`).
- **Consequence:** No prices, transcripts or market size (`README.md:102-108`). These gaps are stated as gaps (BR-06).

### ADR-11 One content set, two host manifests
- **Decision:** Shared `skills/`, `agents/` and `bin/`, with separate `.claude-plugin` and `.codex-plugin` manifests and marketplaces (commit `0cb491d`, `AGENTS.md:38-47`).
- **Alternatives:** Two forks (drift), or a generator that emits both manifests (not done; parity is maintained by hand).

### ADR-12 `plugins/<name>/` layout
- **Decision:** Plugin content moved under `plugins/stock-deepdive-lite/` (commit `0cb491d`).
- **Rationale (REPORTED, tested by the author):** Codex returned `plugin not found` for `path: "./"` and `path: "."`. Claude Code accepts both (`AGENTS.md:49-51`).

### ADR-13 Host-specific path: two literal lines instead of a placeholder
- **History:** `0cb491d` replaced `${CLAUDE_PLUGIN_ROOT}` with `<ราก>` at 6 sites. `3749845` reverted to showing **two lines**: the original Claude Code string, plus a Codex line with `<ราก>` and a "run only one" warning (`SKILL.md:45-50`).
- **Rationale (commit `3749845`):** A placeholder adds a decision where none existed. If the model copies `<ราก>` literally, the shell reads `<` as a redirect and the run breaks mid-way through a 40-minute job.
- **Debt:** `AGENTS.md:39-40` still states the superseded rule.

### ADR-14 Valuation as "what must you believe at this price"
- **Decision:** Reverse-engineer the implied expectations instead of giving a cheap/expensive verdict (`SKILL.md:229`, `README.md:137`).
- **Rationale:** The former is checkable, the latter is opinion (`:229`). It is consistent with BR-08 (no advice).

### ADR-15 Plugin renamed `stock-deepdive-lite`
- **Decision:** Commit `8e86421` renamed the plugin to avoid colliding with a same-named skill users may already have (`README.md:52-53`).
- **Consequence:** Commands are namespaced `/stock-deepdive-lite:*`. The repository name stays `stock-deepdive`.

### Minor recorded decisions
- `.gitignore` anchors with `/` so `skills/deepdive/` is not ignored (commit `ff56d20`, `.gitignore:2`).
- UTF-8 is forced on stdout and stderr in every script, for the Windows Thai console (`fetch_us.py:28-34`, repeated in all five).
- iDisc rejects generic browser UAs, so a fixed, identifiable UA is used (`fetch_th.py:14,45`). EDGAR needs a real contact (`fetch_us.py:12-13`).
- Rate limits: EDGAR at 5 requests/s, against SEC's 10/s guidance (`fetch_us.py:48`). iDisc at 2.5 requests/s (`fetch_th.py:55`).
- Timings in the README were corrected to measured values (commits `aabc0cc`, `a4de8e4`), which shows a stated preference for measured over aspirational claims.
