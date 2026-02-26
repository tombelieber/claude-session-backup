# claude-backup v3 Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Replace UUID-only restore with a rich session index — enabling `--list`, `--last N`, `--date`, `--project`, and `--force` restore modes.

**Architecture:** Two new helper functions (`build_session_index`, `query_session_index`) added to `cli.sh`. `build_session_index` scans `~/.claude-backup/projects/` on every sync and writes `~/.claude-backup/session-index.json`. `query_session_index` uses `python3` (always available on macOS) to filter and return pipe-delimited rows. `cmd_restore` is fully rewritten to consume these. All other commands unchanged.

**v4/Team extensibility hooks:** `session-index.json` is a separate file from `manifest.json` (so v4 can encrypt sessions while keeping manifest readable). The index has a `version` field for future format changes. `build_session_index` only handles `*.jsonl.gz` — a `# v4: also scan *.jsonl.age.gz` comment marks the extension point. `query_session_index` is a standalone function, easily swappable.

**Tech Stack:** Bash, python3 (macOS built-in), stat, find, gzip

---

## Rollback

If anything goes wrong mid-implementation:

```bash
# Undo the last commit (keeps changes staged)
git reset HEAD~1

# Restore cli.sh to last known-good state
git checkout HEAD -- cli.sh

# Remove partial index file (safe — backup data untouched)
rm -f ~/.claude-backup/session-index.json
```

---

### Task 1: Add `build_session_index` function

**Files:**
- Modify: `cli.sh` — insert after the closing `}` of `write_manifest` and before `cmd_sync()`

**Step 1: Add the function**

Insert between `write_manifest`'s closing `}` and `cmd_sync() {` (find by string anchor `}\n\ncmd_sync()`):

```bash
build_session_index() {
  # Scans ~/.claude-backup/projects/ and writes session-index.json.
  # One entry per *.jsonl.gz file. Sorted newest-first by backedUpAt.
  # v4: also scan *.jsonl.age.gz when encryption is added.
  local index_file="$BACKUP_DIR/session-index.json"
  local entries=""
  local count=0

  while IFS= read -r -d '' gz_file; do
    local filename project_hash uuid size_bytes mod_time
    filename=$(basename "$gz_file")
    project_hash=$(basename "$(dirname "$gz_file")")
    uuid="${filename%.jsonl.gz}"
    size_bytes=$(stat -f %z "$gz_file" 2>/dev/null || echo 0)
    mod_time=$(stat -f "%Sm" -t "%Y-%m-%dT%H:%M:%SZ" "$gz_file" 2>/dev/null || echo "unknown")

    [ $count -gt 0 ] && entries="${entries},"
    entries="${entries}
    {\"uuid\":\"${uuid}\",\"projectHash\":\"${project_hash}\",\"sizeBytes\":${size_bytes},\"backedUpAt\":\"${mod_time}\"}"
    ((count++)) || true
  done < <(find "$DEST_DIR" -name "*.jsonl.gz" -type f -print0 2>/dev/null)

  cat > "$index_file" <<INDEX
{
  "version": "$VERSION",
  "generatedAt": "$(date -u '+%Y-%m-%dT%H:%M:%SZ')",
  "sessions": [${entries}
  ]
}
INDEX
}

# stdout contract: prints pipe-delimited rows (uuid|projectHash|sizeBytes|backedUpAt) — sorted newest first.
# Args: <mode> <arg>
#   mode=all     arg=ignored      → all sessions
#   mode=last    arg=N            → last N sessions
#   mode=date    arg=YYYY-MM-DD   → sessions where backedUpAt starts with arg (UTC)
#   mode=project arg=partial      → sessions where projectHash contains arg (case-insensitive)
query_session_index() {
  local mode="${1:-all}"
  local arg="${2:-}"
  local index_file="$BACKUP_DIR/session-index.json"

  [ -f "$index_file" ] || { warn "No session index. Run: claude-backup sync"; return 1; }

  python3 - "$mode" "$arg" "$index_file" <<'PYEOF'
import json, sys

mode = sys.argv[1]
arg  = sys.argv[2]
path = sys.argv[3]

with open(path) as f:
    data = json.load(f)

sessions = data.get("sessions", [])
sessions.sort(key=lambda s: s.get("backedUpAt", ""), reverse=True)

if mode == "last":
    sessions = sessions[:int(arg) if arg else 10]
elif mode == "date":
    sessions = [s for s in sessions if s.get("backedUpAt", "").startswith(arg)]
elif mode == "project":
    sessions = [s for s in sessions if arg.lower() in s.get("projectHash", "").lower()]

for s in sessions:
    print(f"{s['uuid']}|{s['projectHash']}|{s['sizeBytes']}|{s['backedUpAt']}")
PYEOF
}

```

**Step 2: Verify syntax**

Run: `bash -n cli.sh`
Expected: No output

**Step 3: Commit**

```bash
git add cli.sh
git commit -m "feat: add build_session_index and query_session_index"
```

---

### Task 2: Integrate `build_session_index` into `cmd_sync`

**Files:**
- Modify: `cli.sh` — `cmd_sync` function

**Step 1: Add `build_session_index` call after `write_manifest`**

Find the exact line:
```bash
  # Write manifest
  write_manifest
```

Replace with:
```bash
  # Write manifest and session index
  write_manifest
  build_session_index
```

**Step 2: Verify syntax**

Run: `bash -n cli.sh`
Expected: No output

**Step 3: Smoke test**

Run: `./cli.sh sync`
Expected: Runs normally. Check that `~/.claude-backup/session-index.json` exists and contains valid JSON:
```bash
python3 -m json.tool ~/.claude-backup/session-index.json | head -20
```
Expected: Valid JSON with `version`, `generatedAt`, `sessions` array.

**Step 4: Commit**

```bash
git add cli.sh
git commit -m "feat: write session-index.json on every sync"
```

---

### Task 3: Rewrite `cmd_restore` with full flag support

**Files:**
- Modify: `cli.sh` — replace existing `cmd_restore` function entirely

**Step 1: Replace `cmd_restore`**

Find the function by its opening line `cmd_restore() {` and replace the entire function (through the closing `}` before `cmd_uninstall`) with:

```bash
cmd_restore() {
  local mode="uuid"
  local uuid=""
  local last_n=10
  local filter_date=""
  local filter_project=""
  local force=false

  # Parse args — supports both "--last 10" (two args) and UUID with optional --force
  local args=("$@")
  local i=0
  while [ $i -lt ${#args[@]} ]; do
    case "${args[$i]}" in
      --list)    mode="list" ;;
      --force)   force=true ;;
      --last)    mode="last";    ((i++)) || true; last_n="${args[$i]:-10}" ;;
      --date)    mode="date";    ((i++)) || true; filter_date="${args[$i]:-}" ;;
      --project) mode="project"; ((i++)) || true; filter_project="${args[$i]:-}" ;;
      *)         [ "$mode" = "uuid" ] && uuid="${args[$i]}" ;;
    esac
    ((i++)) || true
  done

  if [ ! -d "$DEST_DIR" ]; then
    fail "No backups found. Run: claude-backup sync"
  fi

  # ── Listing modes ────────────────────────────────────────────────────────────
  if [ "$mode" != "uuid" ]; then
    printf "\n${BOLD}Claude Code Sessions${NC}\n\n"
    printf "  %-38s %-36s %6s  %s\n" "PROJECT" "UUID" "SIZE" "DATE (UTC)"
    printf "  %-38s %-36s %6s  %s\n" "--------------------------------------" \
      "------------------------------------" "------" "----------"

    local query_mode query_arg
    case "$mode" in
      list)    query_mode="all";     query_arg="" ;;
      last)    query_mode="last";    query_arg="$last_n" ;;
      date)    query_mode="date";    query_arg="$filter_date" ;;
      project) query_mode="project"; query_arg="$filter_project" ;;
    esac

    local shown=0
    while IFS='|' read -r s_uuid s_hash s_size s_date; do
      local display_hash display_size display_date
      # Show last 38 chars of project hash (most specific part)
      if [ ${#s_hash} -gt 38 ]; then
        display_hash="...${s_hash: -35}"
      else
        display_hash="$s_hash"
      fi
      display_size=$(( s_size / 1024 ))
      display_date="${s_date%T*}"  # strip time, show date only
      printf "  %-38s %-36s %5sK  %s\n" "$display_hash" "$s_uuid" "$display_size" "$display_date"
      ((shown++)) || true
    done < <(query_session_index "$query_mode" "$query_arg")

    if [ $shown -eq 0 ]; then
      printf "  ${DIM}No sessions found matching your filter.${NC}\n"
    fi

    printf "\n  ${DIM}Restore: claude-backup restore <uuid>${NC}\n"
    printf "  ${DIM}Force:   claude-backup restore <uuid> --force${NC}\n\n"
    return 0
  fi

  # ── UUID restore mode ─────────────────────────────────────────────────────────
  if [ -z "$uuid" ]; then
    printf "\n${BOLD}Usage:${NC}\n"
    printf "  claude-backup restore <uuid>                  Restore by UUID\n"
    printf "  claude-backup restore <uuid> --force          Overwrite existing\n"
    printf "  claude-backup restore --list                  List all sessions\n"
    printf "  claude-backup restore --last N                List last N sessions\n"
    printf "  claude-backup restore --date YYYY-MM-DD       Sessions from date (UTC)\n"
    printf "  claude-backup restore --project PARTIAL       Filter by project name\n\n"
    return 1
  fi

  if [[ ! "$uuid" =~ ^[a-zA-Z0-9._-]+$ ]]; then
    fail "Invalid session identifier: $uuid"
  fi

  local matches
  matches=$(find "$DEST_DIR" -name "*${uuid}*.gz" -type f 2>/dev/null)

  if [ -z "$matches" ]; then
    fail "No backup found matching: $uuid"
  fi

  local match_count
  match_count=$(echo "$matches" | wc -l | tr -d ' ')

  if [ "$match_count" -gt 1 ]; then
    printf "\n${YELLOW}Multiple matches found:${NC}\n"
    echo "$matches" | while read -r f; do printf "  %s\n" "$f"; done
    printf "\nProvide a more specific UUID.\n\n"
    return 1
  fi

  local gz_file="$matches"
  local filename project_dir
  filename=$(basename "$gz_file" .gz)
  project_dir=$(basename "$(dirname "$gz_file")")

  local target_dir="$SOURCE_DIR/$project_dir"
  local target_file="$target_dir/$filename"

  printf "\n${BOLD}Restoring session:${NC}\n"
  printf "  ${DIM}From:${NC} $gz_file\n"
  printf "  ${DIM}To:${NC}   $target_file\n\n"

  if [ -f "$target_file" ] && [ "$force" = false ]; then
    warn "File already exists. Use --force to overwrite."
    return 1
  fi

  mkdir -p "$target_dir"
  gzip -dkc "$gz_file" > "$target_file"
  info "Session restored: $target_file"
  printf "\n"
}
```

**Step 2: Verify syntax**

Run: `bash -n cli.sh`
Expected: No output

**Step 3: Commit**

```bash
git add cli.sh
git commit -m "feat: rewrite cmd_restore with --list --last --date --project --force"
```

---

### Task 4: Update case statement, `show_help`, and `cmd_status`

**Files:**
- Modify: `cli.sh` — three locations

**Step 1: Update case statement**

Find:
```bash
  restore)       cmd_restore "${2:-}" ;;
```

Replace with:
```bash
  restore)       shift; cmd_restore "$@" ;;
```

**Step 2: Update `show_help`**

Find the `restore` line in `show_help`:
```bash
  claude-backup restore ID     Restore a session by UUID
```

Replace with:
```bash
  claude-backup restore --list              List all backed-up sessions
  claude-backup restore --last N            List last N sessions
  claude-backup restore --date YYYY-MM-DD   Sessions from date (UTC)
  claude-backup restore --project NAME      Filter by project name
  claude-backup restore <uuid>              Restore a session
  claude-backup restore <uuid> --force      Overwrite existing session
```

**Step 3: Update `cmd_status` to show index stats**

In `cmd_status`, find:
```bash
  # Config backup
  if [ -d "$CONFIG_DEST" ]; then
```

Insert before it:
```bash
  # Session index
  local index_file="$BACKUP_DIR/session-index.json"
  if [ -f "$index_file" ]; then
    local index_count
    index_count=$(python3 -c "import json; d=json.load(open('$index_file')); print(len(d.get('sessions',[])))" 2>/dev/null || echo "?")
    printf "  ${BOLD}Index:${NC}       $index_count sessions indexed\n"
  fi

```

**Step 4: Update `cmd_init` success message**

Find:
```bash
  printf "    claude-backup restore ID       Restore a session\n"
```

Replace with:
```bash
  printf "    claude-backup restore --list   List and restore sessions\n"
```

**Step 5: Verify syntax**

Run: `bash -n cli.sh`
Expected: No output

**Step 6: Test help and status**

Run: `./cli.sh --help` — verify new restore flags shown
Run: `./cli.sh status` — verify "Index: N sessions indexed" line shown

**Step 7: Commit**

```bash
git add cli.sh
git commit -m "feat: update help, status, init for v3 restore commands"
```

---

### Task 5: Bump version to 3.0.0

**Files:**
- Modify: `cli.sh` — `VERSION` constant
- Modify: `package.json` — `version` field

**Step 1: Bump in cli.sh**

Find:
```bash
VERSION="2.0.0"
```

Replace with:
```bash
VERSION="3.0.0"
```

**Step 2: Bump in package.json**

Change `"version": "2.0.0"` to `"version": "3.0.0"`.

**Step 3: Verify syntax**

Run: `bash -n cli.sh`

**Step 4: Commit**

```bash
git add cli.sh package.json
git commit -m "chore: bump version to 3.0.0"
```

---

### Task 6: Update README.md

**Files:**
- Modify: `README.md`

**Step 1: Add session restore section**

Update the README to document:
- New `restore` subcommands (`--list`, `--last`, `--date`, `--project`, `--force`)
- Explain that `session-index.json` is auto-generated on every sync
- Updated command table
- Note about `--date` using UTC

Keep it concise. Reference the design doc at `docs/plans/2026-02-25-claude-backup-v2-design.md` for background context but write README in user-facing style.

**Step 2: Commit**

```bash
git add README.md
git commit -m "docs: update README for v3 selective restore"
```

---

### Task 7: End-to-end verification

**No file changes — verification only.**

**Step 1: Check initialization**

Run: `./cli.sh status`
If not initialized: run `./cli.sh init` first.

**Step 2: Run sync to generate index**

Run: `./cli.sh sync`
Verify `~/.claude-backup/session-index.json` exists:
```bash
python3 -m json.tool ~/.claude-backup/session-index.json | head -30
```
Expected: Valid JSON with `version: "3.0.0"`, `sessions` array with entries.

**Step 3: Verify `--list`**

Run: `./cli.sh restore --list`
Expected: Table of sessions with project hash, UUID, size, date columns. No errors.

**Step 4: Verify `--last N`**

Run: `./cli.sh restore --last 3`
Expected: At most 3 rows shown (newest first).

**Step 5: Verify `--project` filter**

Run: `./cli.sh restore --project claude-backup`
Expected: Only sessions from the claude-backup project shown (or empty if no sessions from this project).

**Step 6: Verify `--date` filter**

Run: `./cli.sh restore --date $(date -u '+%Y-%m-%d')`
Expected: Sessions backed up today (UTC) shown. May be empty if sync happened yesterday UTC.

**Step 7: Verify UUID restore still works**

Pick a UUID from `--list` output. Run:
```bash
./cli.sh restore <uuid>
```
Expected: Either restores successfully OR warns "File already exists. Use --force to overwrite."

**Step 8: Verify `--force` flag**

If previous step warned about existing file:
```bash
./cli.sh restore <uuid> --force
```
Expected: Overwrites and confirms restore.

**Step 9: Verify security exclusions still intact**

```bash
find ~/.claude-backup/ -name ".credentials.json" -o -name ".encryption_key" | wc -l
```
Expected: 0

**Step 10: Verify version**

Run: `./cli.sh --version`
Expected: `claude-backup v3.0.0`

---

## Changelog of Fixes Applied (Audit → Final Plan)

| # | Issue | Severity | Fix Applied |
|---|-------|----------|-------------|
| 1 | v4 extensibility: session-index separate from manifest | Design | `session-index.json` is a distinct file; `manifest.json` unchanged |
| 2 | v4 extensibility: encryption hook in `build_session_index` | Design | Added `# v4: also scan *.jsonl.age.gz` comment at extension point |
| 3 | `query_session_index` stdout contract | Warning | Documented in comment above function: pipe-delimited rows, sorted newest first |
| 4 | `--date` uses UTC — potential user confusion | Warning | Documented in `show_help` and README |
| 5 | `python3` inline script receives `index_file` as arg, not shell expansion | Security | `path = sys.argv[3]` — no shell injection risk |
| 6 | `shown` counter in while loop subshell | Correctness | `while ... done < <(...)` runs in main shell in bash — `shown` increments correctly |
