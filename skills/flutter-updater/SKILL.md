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

### Session Checkpoint (TaskCreate)

Immediately after the preamble, create a TaskCreate checklist so that if the session is
interrupted, TaskList can restore progress without re-scanning files:

Create tasks for each applicable phase (mark N/A phases as already complete):
- `[flutter-updater] Phase 0: Environment validation`
- `[flutter-updater] Phase 1: Flutter SDK check`
- `[flutter-updater] Phase 2: Dependency classification`
- `[flutter-updater] Phase 3: Safe updates`
- `[flutter-updater] Phase 3b: Pre-flight impact mapping`
- `[flutter-updater] Phase 4: Breaking change updates`
- `[flutter-updater] Phase 5: dart fix`
- `[flutter-updater] Phase 6: QA`
- `[flutter-updater] Phase 7: Save report`

Mark each task complete (TaskUpdate) as soon as its phase finishes.
On session resume, call TaskList first — skip phases already marked complete.

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
dart analyze 2>&1 | grep -E "^\s+error " | head -40
```

Fix any `dart analyze` errors with Edit tool before proceeding.
Only read the error lines — do not load the full analyze output into context.

---

## PHASE 3b: Pre-flight Impact Mapping

Skip if `$_FLAG` is `--sdk-only` or `--qa-only`, or no BREAKING packages.

**Run this once before the per-package loop.** Collect all affected files and patterns
in a single grep pass rather than discovering them file-by-file during Phase 4.

For each BREAKING package, extract the changed/removed API names from its changelog
(Phase 4b will still fetch the changelog, but use these grep results to plan work):

```bash
# Example: scan all source dirs for known breaking API names at once
grep -rl "<API_PATTERN_1>\|<API_PATTERN_2>" lib/ test/ bin/ 2>/dev/null \
  | sort -u > /tmp/fu_affected_$$.txt
wc -l /tmp/fu_affected_$$.txt
cat /tmp/fu_affected_$$.txt
```

Store the affected file list. Count the hits per pattern:
```bash
grep -rc "<API_PATTERN>" lib/ test/ bin/ 2>/dev/null | grep -v ":0$"
```

**Decision gate — mechanical vs. complex:**
- If a single pattern appears in N≥5 files with an identical replacement rule → mark as **mechanical** (use sed in Phase 4f)
- Otherwise → mark as **complex** (use Read+Edit per file in Phase 4f)

Save this classification per package so Phase 4f can branch immediately.

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
Use the pre-flight results from Phase 3b (already computed). Only run additional Grep
queries for patterns not captured upfront (e.g., patterns only visible after reading
the full changelog).

### 4d — Confirm with User (AskUserQuestion)
Show: package name, version bump, breaking changes summary, affected files count,
and whether the fix path is **mechanical (sed)** or **complex (manual edit)**.
Options: a) Yes, migrate   b) Skip this package

### 4e — Apply Update
```bash
dart pub upgrade --major-versions "<PACKAGE>" 2>&1
```
If resolution fails (non-zero exit), skip to next package.

### 4f — Fix Errors (up to 3 rounds)

**First:** get the error list without loading full output:
```bash
dart analyze 2>&1 | grep -E "^\s+error " | head -60
```

**Branch on fix strategy (set in Phase 3b):**

#### Mechanical path (sed-first):
If the same substitution pattern repeats across many files, apply with one command:
```bash
# Example: rename an API across all affected files
grep -rl "<OLD_PATTERN>" lib/ test/ bin/ \
  | xargs sed -i 's/<OLD_PATTERN>/<NEW_PATTERN>/g'
```
Then re-run analyze (filtered) to confirm the fix landed. Only fall back to
Read+Edit for files where sed did not resolve the error.

#### Complex path (Read+Edit per file):
For each error: Read file → cross-reference changelog → Edit fix → re-run analyze (filtered).

After each fix round, re-run:
```bash
dart analyze 2>&1 | grep -E "^\s+error " | head -60
```
Stop when zero errors or 3 rounds exhausted.

### 4g — Rollback If Unfixable
Read `pubspec.yaml.fu_bak`, restore original constraint for this package via Edit, then:
```bash
dart pub get 2>&1
```
Report: what failed, which files need manual fix, migration guide URL.

### 4h — Continue + Cleanup
After all breaking packages:
```bash
rm -f pubspec.yaml.fu_bak pubspec.lock.fu_bak /tmp/fu_affected_$$.txt
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
dart analyze 2>&1 | grep -E "^\s+error " | head -40
```

---

## PHASE 6: QA

Skip if `$_FLAG` is `--sdk-only` or `--deps-only`.

### 6a — flutter analyze
```bash
flutter analyze 2>&1 | grep -E "error •|• error|^\s+error " | head -60
```
If zero error lines → pass. Otherwise attempt Edit fixes (errors only, not warnings),
re-run filtered analyze (up to 2 rounds). Warnings: note in report, don't block.

To get a full warning summary without loading all output:
```bash
flutter analyze 2>&1 | grep -cE "warning •|• warning" || echo "0 warnings"
```

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
| http | 0.13.5 | 1.2.2 | Breaking | ✅ Updated + auto-fixed (3 files, sed) |
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
*Generated by flutter-updater v1.1.0*
```

---

## Error Recovery Rules

- **Network failures**: Retry once. If still failing, skip that step, note in report. Never abort full run.
- **Resolution failure**: Do not touch pubspec.yaml. Report conflict, skip package.
- **Fix loop cap**: Max 3 rounds per package. After 3, rollback.
- **Flaky test**: Re-run once. If fails again, record as failed.
- **Unrelated build failure**: Report, do not fix issues unrelated to the update.
- **git**: Do NOT run any git commands. Use only file backup/restore strategy.
- **Large analyze output**: Always pipe `flutter analyze` / `dart analyze` through grep filters.
  Never load the raw full output into context — it can exceed 10k characters.
- **sed failure**: If sed produces a syntax error or zero replacements, fall back to Read+Edit
  for that file. Never apply sed without re-running analyze to confirm correctness.
