# stock-deepdive-pro — rebuild kit

This folder holds everything needed to rebuild the stock deep-dive pipeline as
`plugins/stock-deepdive-pro/`, one phase at a time, with Claude Code.

| File | Purpose | Read when |
|---|---|---|
| [SPEC.md](SPEC.md) | What we are building and why: goals, architecture, components, invariants | Always first |
| [CONTRACTS.md](CONTRACTS.md) | Exact formats: directory layout, schemas, IDs, CLI, reference syntax, gates | Before writing any code or prompt |
| [ACCEPTANCE.md](ACCEPTANCE.md) | How each phase is proven done (commands and expected results) | Before opening a PR |
| [PHASES.md](PHASES.md) | Phase order, dependencies, deliverables, branch names | When choosing what to do next |
| [DECISIONS.md](DECISIONS.md) | The decisions agreed in the design interview, with reasons | When something in the spec looks arbitrary |
| [prompts/](prompts/) | [`_COMMON.md`](prompts/_COMMON.md) (rules shared by every phase) and one prompt per phase (`phase-0.md` … `phase-7.md`) | To start a phase |

Background analysis of the legacy plugin: [`../reverse-engineering/`](../reverse-engineering/README.md)
(available on `develop` once PR #1 is merged).

## How to run a phase

```sh
git switch develop && git pull
git switch -c feature/pN-<short-name>
claude                                 # fresh session in the repo root
> Read docs/rebuild/prompts/phase-N.md and execute it.
```

Each prompt ends by running its acceptance commands and opening a PR into `develop`.
A human reviews and merges before the next phase starts. Do not run two phases in one branch.

## Ground rules for every phase

1. The spec is the source of truth. If code and spec disagree, stop and ask. Do not silently "fix" the spec.
2. Never modify `plugins/stock-deepdive-lite/`. It stays as the comparison baseline.
3. Never commit fetched raw PDFs or personal data. Fixtures follow [CONTRACTS.md §9](CONTRACTS.md#9-fixtures).
4. Network calls (iDisc, TradingView MCP) are only allowed in tests marked `live` and in the run pipeline itself.
