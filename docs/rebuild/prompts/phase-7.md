# Phase 7 — run manifest, CI, documentation, 1.0.0

Branch: `feature/p7-release`. First follow `docs/rebuild/prompts/_COMMON.md`. Requires P6 merged.

## Goal
Make the pipeline reproducible and continuously tested, prove it beats `lite`, and release 1.0.0.

## Tasks
1. **`run.json` (CONTRACTS §11) is complete.**
   - stage start and end times
   - input file sha256
   - Python and package versions from `uv`
   - the LLM model id per agent step, recorded by the conductor
   - parser fallbacks and unit corrections
   - gates
   - critic verdict and revisions
2. **GitHub Actions** `.github/workflows/pro-tests.yml`:
   - Install uv and run `uv sync`.
   - Run `uv run ruff check` and `uv run pytest -m "not live"`.
   - Trigger on push and PR to `develop` and `master`, with paths filtered to `plugins/stock-deepdive-pro/**`.
3. **Plugin README** (`plugins/stock-deepdive-pro/README.md`, in Thai for users):
   - install and usage
   - outputs, with a screenshot of the dashboard
   - data tiers
   - limits (what is unavailable)
   - difference from `lite`
   - disclaimer
4. **A/B report** `docs/rebuild/AB-REPORT.md`: run `lite` and `pro` on BGRIM on the same day and compare:
   - wall time per stage
   - fact counts
   - for `lite`: the number of article numbers that pass only through its loose matching (re-use the probe in `docs/reverse-engineering/README.md`)
   - for `pro`: the G6 result
   - critic verdict
   - what each version could not answer
5. Set the version to `1.0.0`, add a CHANGELOG entry, and update the root `README.md` to mention both plugins.

## Acceptance
Every bullet under "P7 release" in ACCEPTANCE.md. CI must be green on the PR.

## Stop and ask if
- The A/B comparison shows `pro` is worse on any dimension the owner cares about (time, coverage, readability). Report it honestly before releasing.
