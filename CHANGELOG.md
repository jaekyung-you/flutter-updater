# Changelog

## 1.1.0 — 2026-04-14

### Performance improvements

- **Pre-flight impact mapping (Phase 3b)**: Before the per-package breaking change loop,
  a single `grep -rl` pass collects all affected files upfront. This eliminates repeated
  file discovery during Phase 4 and can reduce token usage by 30–50% on projects with
  many widget or service files sharing the same API pattern.

- **Mechanical vs. complex fix branching (Phase 4f)**: If the same substitution pattern
  appears in 5+ files, Phase 4f now applies a `sed -i` batch replacement via `xargs`
  instead of reading and editing each file individually. Falls back to Read+Edit only
  for files where sed leaves residual errors.

- **Filtered analyze output**: All `dart analyze` and `flutter analyze` calls are now
  piped through `grep -E "^\s+error "` (or equivalent). Only error lines are loaded
  into context — warning counts are captured separately with `grep -c`. Eliminates
  loading 10k+ character raw analyze output on each iteration.

- **Session checkpoint (TaskCreate)**: A task checklist is created at the start of every
  run. On session resume, `TaskList` immediately restores which phases are complete,
  avoiding redundant re-scanning of already-processed files.

## 1.0.0 — 2026-04-13

Initial release.

### Features
- Flutter SDK version monitoring via GitHub Releases API
- `dart pub outdated --json` based dependency analysis with safe/breaking classification
- Safe updates applied automatically via `dart pub upgrade`
- Breaking change updates: changelog fetch, codebase impact analysis, user confirmation, auto-fix, rollback
- `dart fix --dry-run` preview + `dart fix --apply`
- QA pipeline: `flutter analyze` → `flutter test` → `flutter build` (platform auto-detected)
- Structured Markdown result report
- Flags: `--sdk-only`, `--deps-only`, `--qa-only`
- Path/git dependencies are never modified
- Automatic rollback per-package if breaking change fix fails after 3 attempts
