# common-repo

Rust CLI for managing repo config files as dependencies: inheritance, merge,
version pinning across multiple Git repositories. Full CI/CD automation
(conventional commits, semantic-release via release-please).

## Layout

- `src/` — CLI binary `common-repo` (`cli.rs`, `commands/`, `merge/`, `phases/`)
- `xtask/` — dev tasks (`cargo xtask coverage`, `cargo xtask check-prose`)
- `script/` — Scripts-to-Rule-Them-All (`setup`, `test`, `ci`, `cibuild`)
- `tests/` — `cli_e2e_*.rs` integration/E2E; unit tests live beside modules
- `context/` — LLM task tracking (`current-task.json`, plans, `traceability-map.md`,
  `design.md`, `purpose.md`); `context/completed/` archives finished plans
- `docs/src/` — mdBook user guides (published to common-repo.github.io)
- `spec/` — behavior specs; `benches/` — benchmarks
- `.pre-commit-config.yaml` — fmt/check/clippy + action-validator + whitespace

## Required

- **TDD for all feature work, bug fixes, and refactors** — see `TDD.md`. Write
  one failing test → make it pass → refactor. No implementation without a
  failing test first.
- **Conventional commits**: `<type>(<scope>): <description>`, max 100 chars.
  Types: `feat`, `fix`, `docs`, `style`, `refactor`, `perf`, `test`, `build`,
  `ci`, `chore`, `revert`. Breaking: `feat!:` or `BREAKING CHANGE:` footer.
- **`./script/ci` before every commit** (fmt, clippy `-D warnings`, pre-commit,
  security audits, prose check). Env: `SKIP_SECURITY=1`, `SKIP_PROSE=1`,
  `OFFLINE=1`.
- Hooks: `prek run --all-files` (or Python `pre-commit`). Clippy warnings are
  errors.
- **Backwards compatibility is required** — do not break existing CLI flags,
  public APIs, or documented behavior unless explicitly requested; rewire old
  flags to new paths instead of dropping them.
- Before pushing: rebase on main and resolve conflicts. Never commit/push
  without explicit user approval (this repo's own rule).
- Documentation is part of implementation — module docs, function docs, and
  `docs/src/` updates land in the same change.
- Avoid AI writing patterns (`context/ai-writing-patterns.md`); the prose
  linter rejects them.

### Session / task tracking

Work is tracked in JSON under `context/` (not YAML/markdown). Session startup:

1. Verify clean `git status` / stash / unpushed commits
2. Branch `<type>/<description>-<session-id>` from updated main
3. Baseline `./script/test` in background (skip for docs-only changes)
4. Read `context/current-task.json`; pick first `status=pending` with
   `blocked_by=null`
5. Stash stack for interrupts: rename `current-task.json` →
   `current-task-stash{N}.json`; pop highest N when done
6. Archive finished plans: `git mv context/<plan>.json context/completed/`,
   commit `chore(context): archive completed <plan-name>`

Plans may split into sub-plans (`sub_plan` field) when a task would read too
much; archive sub-plans independently.

Feature completion criteria: E2E tests exist and pass (unit tests alone are
insufficient), all acceptance criteria met, `./script/test` actually run,
docs updated.

## Test / build

```bash
./script/setup                  # first-time setup
./script/test                   # cargo-nextest (QUICK=1 / SKIP_UPDATE=1 / CI=1)
./script/ci                     # checks only (before every commit)
./script/cibuild                # full CI (update + checks + tests)

cargo nextest run               # unit (fast, no network)
cargo nextest run --features integration-tests   # integration (network)
SKIP_NETWORK_TESTS=1 cargo nextest run --features integration-tests
cargo nextest run --profile ci  # identify slow tests

cargo xtask coverage            # HTML; --format json | --fail-under 80 | --open
cargo xtask check-prose .       # AI-writing-pattern lint (exit 1 on hits)
cargo build --release
```

- Integration tests are feature-gated off by default; datatest schema tests
  auto-discover from YAML. Slow-test config: `.config/nextest.toml`.
- E2E CLI tests use `assert_cmd::cargo::cargo_bin_cmd!` — not the deprecated
  `Command::cargo_bin` (see `tests/cli_e2e_*.rs`).

## Key libraries

- `serde` / `serde_yaml` / `serde_json` / `toml` — config formats
- `pulldown-cmark`, `xot`, `rust-ini`, `taplo` — markdown/XML/INI/TOML merge
- `rayon` — parallel walks; `semver` — version pinning
- `assert_cmd` + `predicates` + `insta` (tests); workspace member `xtask`

## Navigation

- `README.md` — user-facing overview and CLI usage
- `TDD.md` — required TDD workflow
- `context/design.md`, `context/purpose.md`, `context/traceability-map.md`
- `docs/src/` — mdBook user documentation
- `DEVIN.md` / `GEMINI.md` — sibling agent guides (other tools; not Claude)

## Maintaining this index

- Update the affected `AGENTS.md` files in the same change when paths,
  responsibilities, commands, dependencies, or conventions change.
- Keep indexes brief: record semantic entry points and non-obvious
  constraints; link to existing documentation instead of duplicating it.
- Add a directory index only when it provides useful navigation beyond its
  parent; omit generated, vendored, and fixture trees.
- Do not add `CLAUDE.md` files; Claude Code reads `AGENTS.md` natively, and
  a `CLAUDE.md` anywhere in the tree stops every `AGENTS.md` from loading.
