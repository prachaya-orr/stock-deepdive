# Phase 6 — writers, critic, conductor skill, remaining gates

Branch: `feature/p6-writers`. First follow `docs/rebuild/prompts/_COMMON.md`. Requires P5 merged.

## Goal
The end-to-end `/stock-deepdive-pro:deepdive <T>` experience: a thin conductor runs scripts and agents in order, and writers produce reference-only Markdown. A critic reviews the draft, gets at most one revision round, and every gate blocks.

## Tasks
1. `sections.md` (owned by CONTRACTS §8.5) contains:
   - the 10 section definitions (port the intent of the legacy `skills/deepdive/SKILL.md` §"10 ส่วน" and its editorial rules: valuation frame, scenario validity, plain-language, anomaly ≠ intent, no advice)
   - the lane split A/B/C with caps
   - the Markdown subset
   - the claims format
2. `agents/writer.md` is **one** prompt, with the lane passed at call time.
   - Inputs: only `runs/<T>/<id>/writer/`. Tools: Read and Write only. No Grep, Glob or Bash on `data/`.
   - Numbers must be references. Each lane writes `parts/<nn>-<lane>.md` and `claims/claims-<lane>.jsonl`.
   - The article is in Thai. Instructions are in English.
3. `agents/critic.md` uses the checklist in CONTRACTS §6.2 and writes `critic.json`. It never rewrites the article.
4. `skills/deepdive/SKILL.md` is the conductor.
   - Steps: new-run → fetch (confirm identity with the user) → extract → parse → notes-extractor (+1 repair) → validate → model → market-collector → assumptions-proposer (ask the user to accept) → valuation → prepare-writers → 3 writers **in one message** → render → critic → (≤ 1 revision of the flagged lanes → render → critic) → gate → report.
   - The conductor writes the article front ("what does THB X force you to believe", three beliefs with references).
   - It stops on any exit code 2 and shows `gates.md`.
   - It supports `--offline-fixtures`, which uses the test fixtures and `market.sample.json` instead of the network.
5. `skills/fetch/SKILL.md` covers fetch + extract only.
6. Gates:
   - G0: lint that prompts link to CONTRACTS rather than restating enums, section lists or caps
   - G7: critic verdict
   - G8: claims references resolve, and every cross-lane conflict is addressed
7. The final user report (conductor) lists:
   - the file paths
   - stage timings from `run.json`
   - gate results
   - the critic verdict
   - what was unavailable
   - one standout finding

## Acceptance
Every bullet under "P6 writers, critic, conductor" in ACCEPTANCE.md, plus one live run on BGRIM (with user confirmation at the identity and assumptions prompts). Attach `report.html` screenshots to the PR.

## Stop and ask if
- Writers cannot express a needed idea without a bare number (for example a count from the filings not present as a fact). Propose adding a fact type instead of widening the allow-list.
