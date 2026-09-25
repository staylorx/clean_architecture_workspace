# Changelog

All notable changes to this workspace are recorded here.
Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/). Per-package release notes stay
in each package's own `CHANGELOG.md` (`packages/*/CHANGELOG.md`); this file covers the repo root, its
docs and cross-package changes.

## [Unreleased]

### Added

- `BACKLOG.md` (root) — records 13 open build/analysis problems (section A) and all 104 recorded
  deviations from `staylorx/dart-flutter-bible` @ `d4d2ff1` plus 1 applicability item (section B),
  each with file:line evidence from real tool runs at HEAD `12e164bc`: `flutter pub get` exit 0,
  `dart analyze` exit 3 (3825 issues / 2452 errors), all three `dart test` suites exit 1, and
  `dart analyze` disagreeing with `dart run custom_lint` (exit 0, "No issues found!").
- `CHANGELOG.md` (root, this file) — the repo previously had no root changelog.

### Changed

- No source change. The four files dirtied by the audit's `flutter pub get` (`pubspec.lock` and the
  three `analysis_options.yaml`) were reverted, so this commit contains documentation only.

### Decisions

- Record, don't fix: the deviations are flag-for-review items. `clean_architecture_workspace` is a
  linter/codegen toolkit, so the bible's §01–§08 application rules (domain/usecases/datasource package
  rings, a UI ring, persistence) have no target here; that applicability caveat is carried in the
  backlog rather than silently dropping those rules.
- Backlog holds open/pending problems only; everything already decided lives in this file.
- Default branch is `master` (a task card referred to `main`); the docs commit goes to `master`, the
  branch `origin/HEAD` points at.
- No `AGENTS.md` exists in this repo, so there are no repo-local conventions to follow beyond the
  house style of the existing package docs (GitHub-flavored markdown, LF endings). The missing
  `AGENTS.md` stays a backlog item (§01) instead of being invented here.
- No generated/bot blob was hand-edited.
