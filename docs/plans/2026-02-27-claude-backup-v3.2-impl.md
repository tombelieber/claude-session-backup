# claude-backup v3.2 Implementation Plan

**Goal:** Ship multi-machine backup support, backend switching, configurable schedule, and sync hardening — making claude-backup safe for multi-device users and ready for the claude-view backup settings screen.

**Architecture:** Eight tasks in dependency order. Foundation (machine slug, SERVICE_LABEL, mode guards) lands first since every subsequent feature needs it. Multi-machine namespacing second (directory layout, migration, dual-write manifest). Then consumers (cross-machine search in index/restore/peek), restore --all, backend set, schedule command, sync improvements, and finally version bump + skill sync. All changes in `cli.sh` (~2057 lines after), plus updates to `skills/backup/SKILL.md`, `test/skill-sync.sh`, `package.json`, `.claude-plugin/plugin.json`.

**Tech Stack:** Bash, python3 (macOS built-in), gzip, git, launchd (macOS), GitHub CLI (optional)

**Design doc:** `docs/plans/2026-02-27-claude-backup-v3.2-design.md`

---

## Implementation Summary

### Task 1: Foundation
- `machine_slug()` — computes filesystem-safe slug from `hostname -s`, lowercased, sanitized, truncated to 64 chars, empty fallback to "unknown"
- `get_machine_slug()` — reads `machineSlug` from manifest (stable across renames), falls back to `machine_slug()`
- `SERVICE_LABEL="com.claude-backup"` replacing `PLIST_NAME` (removed `.plist` from label)
- `PLIST_PATH` as hardcoded string path
- All `bootout/bootstrap` replacing deprecated `load/unload`
- Mode guards: `!= "local"` instead of `= "github"` to handle both `github` and `git`
- `cmd_status_json()`: added `machine` and `machineSlug` fields with `json_escape`

### Task 2: Multi-machine namespacing
- `resolve_dest_dir()` — sets `DEST_DIR` to `machines/<slug>/projects` if `machines/` exists, else flat `projects/` (v3.0 compat)
- `migrate_to_namespaced()` — one-time v3.0→v3.2 migration: creates machine dir, moves projects, patches manifest, commits in subshell
- `write_manifest()` dual-write: per-machine manifest at `machines/$slug/manifest.json` + root manifest with backward-compat fields and `machines` array aggregation via python3

### Task 3: Cross-machine consumers
- `build_session_index()` scans `$BACKUP_DIR/machines` (all machines) instead of just `DEST_DIR`, adds `machine` field per session entry
- `cmd_restore()` UUID mode searches across `$BACKUP_DIR/machines`
- `cmd_peek()` searches across `$BACKUP_DIR/machines`
- `query_session_index()` outputs machine field in pipe-delimited format
- JSON and human-readable restore listings include machine field

### Task 4: restore --all
- `cmd_restore_all()` — iterates all `.jsonl.gz` across machines, skip-by-default collision handling
- `--all` and `--machine` flag parsing in `cmd_restore()`
- Three-way dispatch: all → `cmd_restore_all`, listing → query, uuid → find+decompress
- `--machine` flag validates against path traversal (`/` and `..` rejected)

### Task 5: backend set
- `cmd_backend()` — `backend set <github|git|local>` subcommand
- GitHub pre-flight: gh CLI check, auth check, repo create/verify, remote set, test push
- Git pre-flight: `git ls-remote` verify, remote set, test push
- Local: remove git remote
- Updates both root and per-machine manifests

### Task 6: schedule command
- `generate_plist()` — extracted from `schedule_launchd()`, parameterized with frequency and cli_path
- `cmd_schedule()` — full 5-step launchd lifecycle: bootout → generate plist → plutil lint → bootstrap → verify
- Frequencies: off (remove plist), daily (StartCalendarInterval 3:00 AM), 6h (StartInterval 21600), hourly (StartInterval 3600)
- Stores `schedule` field in both root and per-machine manifests
- `schedule_launchd()` updated to use `generate_plist()` + `bootout/bootstrap`

### Task 7: Sync improvements
- `git pull --rebase` before push with conflict recovery (abort rebase, retry with merge)
- `git gc --auto` after push
- 2GB size warning for remote modes

### Task 8: Version bump + Skill sync
- Version bumped to 3.2.0 in cli.sh, package.json, plugin.json
- `show_help()` updated with all new commands and `--backend` flag
- `skills/backup/SKILL.md` updated with all new commands and flags
- `test/skill-sync.sh` updated with new subcommands and flags
- `--backend <mode>` global flag on init (replaces `--local`, kept as alias)
- Schedule field preserved across `write_manifest()` rebuilds
- Schedule frequency displayed in both `status --json` and human-readable status

---

## Shippable Audit Results

| Pass | Status | Issues |
|------|--------|--------|
| Plan Compliance | 21/21 PASS | 0 |
| Wiring Integrity | 12/12 PASS | 0 |
| Prod Hardening | Clean | blockers: 0, warnings: 0 |
| Build & Test | All pass | syntax, skill-sync, versions green |

**Verdict: SHIP IT**

---

## Files Changed

| File | Change |
|------|--------|
| `cli.sh` | +625 lines — all 8 tasks (foundation, namespacing, consumers, restore-all, backend, schedule, sync, version) |
| `skills/backup/SKILL.md` | Updated command table, reading responses, init docs |
| `test/skill-sync.sh` | Added `backend`, `schedule` subcommands; `--all`, `--machine`, `--remote`, `--backend` flags |
| `package.json` | Version 3.1.0 → 3.2.0 |
| `.claude-plugin/plugin.json` | Version 3.1.0 → 3.2.0 |
| `docs/plans/2026-02-27-claude-backup-v3.2-design.md` | Status: Draft → Implemented (CLI scope) |

---

## What's NOT in v3.2 (deferred to future)

- **claude-view backup settings screen** — requires claude-view integration (separate repo)
- **Relay server metadata** — requires relay implementation
- **Mobile app status display** — requires mobile app
- **Encryption** — design doc mentions `.jsonl.age.gz`, deferred
- **Pruning** (`prune --older-than`) — mentioned in size warning, not yet implemented
- **Linux/systemd support** — launchd only for now
- **Cross-machine session deduplication** — by design, both copies kept
