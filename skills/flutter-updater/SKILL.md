---
description: |
  Auto-update Flutter/Dart projects. Detects new Flutter/Dart SDK releases,
  updates pubspec.yaml dependencies with safe breaking-change handling (reads
  changelogs, auto-fixes code, rolls back per-package if unfixable), runs dart fix,
  and performs full QA (flutter analyze → flutter test → flutter build).
  Saves a dated Markdown report to .flutter_updater/reports/.
  Flags: --sdk-only, --deps-only, --qa-only
---

You are executing the `flutter-updater` skill. The plugin's bin/ scripts are at
`${CLAUDE_PLUGIN_ROOT}/bin/`. If `${CLAUDE_PLUGIN_ROOT}` is not set, locate them by running:
```bash
find ~/.claude/plugins/cache -name "flutter-updater-config" -type f 2>/dev/null | head -1 | xargs -I{} dirname {}
```
Store this path as `_BIN` for all subsequent script calls.

---

## Preamble (run first, always)

```bash
# Resolve bin/ path
if [ -n "${CLAUDE_PLUGIN_ROOT:-}" ]; then
  _BIN="${CLAUDE_PLUGIN_ROOT}/bin"
else
  _CONFIG=$(find ~/.claude/plugins/cache -name "flutter-updater-config" -type f 2>/dev/null | head -1)
  _BIN=$(dirname "$_CONFIG" 2>/dev/null || echo "")
fi

[ -z "$_BIN" ] && echo "ERROR: flutter-updater bin/ not found. Is the plugin installed?" && exit 1
echo "PLUGIN_BIN: $_BIN"

# Self-update check (silent if up to date)
"$_BIN/flutter-updater-version-check" 2>/dev/null || true

# Load config
_INTERVAL=$("$_BIN/flutter-updater-config" get check_interval_hours 2>/dev/null || echo "12")
_IGNORED=$( "$_BIN/flutter-updater-config" get ignored_packages      2>/dev/null || echo "")
_AUTO_SAFE=$("$_BIN/flutter-updater-config" get auto_apply_safe       2>/dev/null || echo "true")
_CHANNEL=$(  "$_BIN/flutter-updater-config" get track_channel         2>/dev/null || echo "stable")
echo "CONFIG: interval=${_INTERVAL}h channel=${_CHANNEL} auto_safe=${_AUTO_SAFE}"

# Project state
_LAST_CHECK=$("$_BIN/flutter-updater-state" get-last-check 2>/dev/null || echo "")
_SLUG=$(      "$_BIN/flutter-updater-state" slug           2>/dev/null || echo "unknown")
echo "PROJECT: $_SLUG"
[ -n "$_LAST_CHECK" ] && echo "LAST_CHECK: $_LAST_CHECK" || echo "LAST_CHECK: never"

# Detect build targets
_TARGETS=$("$_BIN/flutter-updater-detect-targets" 2>/dev/null || echo "")
echo "TARGETS: ${_TARGETS:-none}"

# Parse flags
_FLAG="${1:-}"
echo "FLAG: ${_FLAG:-none}"
```

After the preamble: store `$_BIN`, `$_IGNORED`, `$_AUTO_SAFE`, `$_CHANNEL`, `$_TARGETS`, `$_FLAG`.
If `$_LAST_CHECK` is within `$_INTERVAL` hours AND `$_FLAG` is empty, ask the user:
"Already checked ${_INTERVAL}h ago. Run again?" before proceeding.

---

## PHASE 0: Environment Validation

```bash
[ -f pubspec.yaml ] || { echo "ERROR: No pubspec.yaml found. Run /flutter-updater from Flutter/Dart project root."; exit 1; }
dart pub outdated --json 2>/dev/null
```

If `pubspec.yaml` is missing, stop. Store the `dart pub outdated --json` output as `$OUTDATED_JSON`.

---

## PHASE 1: Flutter SDK Update Check

Skip if `$_FLAG` is `--deps-only` or `--qa-only`.

```bash
"$_BIN/flutter-updater-sdk-check" "$_CHANNEL" 2>&1
```

Parse output:
- `CURRENT:<v>` → current version
- `LATEST:<v>` → latest
- `UP_TO_DATE` → skip, note in report
- `UPDATE_AVAILABLE` → ask user via AskUserQuestion: "Flutter vCURRENT → vLATEST. Upgrade?"

If user confirms: `flutter upgrade`

---

## PHASE 2: Dependency Classification

Skip if `$_FLAG` is `--sdk-only` or `--qa-only`.

```bash
echo "$OUTDATED_JSON" | "$_BIN/flutter-updater-classify" pubspec.yaml "$_IGNORED"
```

Parse each line:
- `SAFE:<name>:<current>:<target>` → safe list
- `BREAKING:<name>:<current>:<latest>` → breaking list
- `UP_TO_DATE:<name>:<v>` → already current
- `SKIP_*:<name>` → skip

Print summary table. If all up to date, jump to Phase 5.

---

## PHASE 3: Safe Updates

Skip if `$_FLAG` is `--sdk-only` or `--qa-only`, or no SAFE packages found.

If `$_AUTO_SAFE` is `true`, apply automatically; else ask user first.

```bash
dart pub upgrade 2>&1
dart analyze 2>&1 | head -60
```

Fix any `dart analyze` errors with Edit tool before proceeding.

---

## PHASE 4: Breaking Change Updates (one package at a time)

Skip if `$_FLAG` is `--sdk-only` or `--qa-only`, or no BREAKING packages.

For each BREAKING package:

### 4a — Backup (once, before first breaking update)
```bash
cp pubspec.yaml pubspec.yaml.fu_bak
cp pubspec.lock pubspec.lock.fu_bak
```

### 4b — Fetch Changelog
```bash
"$_BIN/flutter-updater-changelog" "<PACKAGE>" "<TARGET_VERSION>" 2>&1
```
Read output. Identify: removed/renamed classes, changed signatures, migration guide URL.

### 4c — Scan Codebase
Use Grep to find usages of removed/changed APIs in `lib/`, `test/`, `bin/`.

### 4d — Confirm with User (AskUserQuestion)
Show: package name, version bump, breaking changes summary, affected files.
Options: a) Yes, migrate   b) Skip this package

### 4e — Apply Update
```bash
dart pub upgrade --major-versions "<PACKAGE>" 2>&1
```
If resolution fails (non-zero exit), skip to next package.

### 4f — Fix Errors (up to 3 rounds)
```bash
dart analyze 2>&1
```
For each error: Read file → cross-reference changelog → Edit fix → re-run analyze.

### 4g — Rollback If Unfixable
Read `pubspec.yaml.fu_bak`, restore original constraint for this package via Edit, then:
```bash
dart pub get 2>&1
```
Report: what failed, which files need manual fix, migration guide URL.

### 4h — Continue + Cleanup
After all breaking packages:
```bash
rm -f pubspec.yaml.fu_bak pubspec.lock.fu_bak
```

---

## PHASE 5: dart fix

Skip if `$_FLAG` is `--sdk-only` or `--qa-only`.

```bash
dart fix --dry-run 2>&1
```

If fixes available, ask user (AskUserQuestion). If confirmed:
```bash
dart fix --apply 2>&1
dart analyze 2>&1 | head -40
```

---

## PHASE 6: QA

Skip if `$_FLAG` is `--sdk-only` or `--deps-only`.

### 6a — flutter analyze
```bash
flutter analyze 2>&1
```
Errors: attempt Edit fixes, re-run (up to 2 rounds). Warnings: note, don't block.

### 6b — flutter test
```bash
flutter test 2>&1
```
Failures from API changes: read test file, update to new API, re-run once.
Unrelated regressions: record but do NOT silently fix test assertions.

### 6c — Build (first available target from `$_TARGETS`)
| Target | Command |
|--------|---------|
| android | `flutter build apk --debug 2>&1 \| tail -30` |
| web | `flutter build web 2>&1 \| tail -30` |
| macos | `flutter build macos --debug 2>&1 \| tail -30` |
| linux | `flutter build linux --debug 2>&1 \| tail -30` |
| ios | `flutter build ios --debug --no-codesign 2>&1 \| tail -30` |
| windows | `flutter build windows --debug 2>&1 \| tail -30` |
| dart_only | `dart compile exe bin/main.dart -o /tmp/fu_build_check 2>&1 \| tail -10` |

Fix loop same as 6a (up to 2 rounds) if build fails.

---

## PHASE 7: Save Report

### 7a — Get current Flutter version
```bash
_FLUTTER_VER=$(flutter --version --machine 2>/dev/null | python3 -c "import json,sys; print(json.load(sys.stdin).get('frameworkVersion','unknown'))" 2>/dev/null || echo "unknown")
```

### 7b — Compose and save report
Write the full Markdown report (template below) to `/tmp/fu_report_$$.md`, then:

```bash
_REPORT_DIR="${FLUTTER_UPDATER_REPORT_DIR:-.flutter_updater/reports}"
"$_BIN/flutter-updater-save-report" "$_FLUTTER_VER" "/tmp/fu_report_$$.md" "$_REPORT_DIR"
rm -f /tmp/fu_report_$$.md
```

Tell the user the exact saved file path, then print the report to the conversation.

### 7c — Update state
```bash
"$_BIN/flutter-updater-state" set-last-check
"$_BIN/flutter-updater-state" log-update '{"sdk_updated":"<v>","updated":[...],"skipped":[...],"dart_fix_applied":true,"qa_analyze":"pass","qa_test":"47/47","qa_build":"pass"}'
```

### Report Template

```markdown
# Flutter Update — v<FLUTTER_VERSION> (<YYYY-MM-DD>)

Project: <name from pubspec.yaml>
Run at: <ISO timestamp>

---

## Flutter SDK
Before: v<OLD> → After: v<NEW>   (or: Already current / Skipped)

## Dependency Updates

| Package | Before | After | Type | Result |
|---------|--------|-------|------|--------|
| http | 0.13.5 | 1.2.2 | Breaking | ✅ Updated + auto-fixed (3 files) |
| provider | 6.1.2 | 6.1.4 | Safe | ✅ Updated |
| dio | 4.0.6 | 5.4.0 | Breaking | ⚠️ Rolled back — see below |

## Code Changes Applied

### dart fix
N issues fixed  (or: No fixes needed / Skipped)

### Breaking Change Migrations
- **http** v0.13.5 → v1.2.2
  - `BaseClient` → `Client` in lib/api/service.dart:34, lib/api/service.dart:89

## QA Results

| Check | Result | Details |
|-------|--------|---------|
| flutter analyze | ✅ Pass | 0 errors, 2 warnings |
| flutter test | ✅ Pass | 47/47 passed |
| flutter build (android) | ✅ Pass | Build succeeded |

## Manual Action Required

### dio — Could Not Auto-Migrate
- Version: v4.0.6 → v5.4.0 (rolled back to v4.0.6)
- Breaking change: `Options` removed; use `RequestOptions`
- Affected files: lib/api/client.dart:34, test/api_test.dart:12
- Migration guide: https://pub.dev/packages/dio/changelog#500

---
*Generated by flutter-updater v1.0.0*
```

---

## Error Recovery Rules

- **Network failures**: Retry once. If still failing, skip that step, note in report. Never abort full run.
- **Resolution failure**: Do not touch pubspec.yaml. Report conflict, skip package.
- **Fix loop cap**: Max 3 rounds per package. After 3, rollback.
- **Flaky test**: Re-run once. If fails again, record as failed.
- **Unrelated build failure**: Report, do not fix issues unrelated to the update.
- **git**: Do NOT run any git commands. Use only file backup/restore strategy.
