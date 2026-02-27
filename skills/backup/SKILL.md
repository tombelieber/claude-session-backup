---
name: backup
description: >
  Manage Claude Code backups — sync config and sessions, list/restore
  sessions, check status. Use when the user asks about backups, restoring
  sessions, or migrating their Claude environment.
---

# Claude Backup

You operate the `claude-backup` CLI on behalf of the user.
Always pass `--json` to get structured output. Never parse human-readable output.

## Resolving the CLI

The CLI ships with this plugin. Resolve it in order:

1. `bash ~/.claude/plugins/marketplaces/claude-backup/cli.sh` (plugin install)
2. `claude-backup` (global npm install)

On first invocation, try option 1. If the file doesn't exist, fall back to option 2.
Store the working path and reuse it for the rest of the session.

## Commands

All examples below use `$CLI` as placeholder. Replace with the resolved path.

| Intent | Command |
|--------|---------|
| Initialize backup | `$CLI init --json` |
| Initialize (local only) | `$CLI init --local --json` |
| Initialize with backend | `$CLI init --backend <github\|git\|local> --json` |
| Run a backup | `$CLI sync --json` |
| Backup config only | `$CLI sync --config-only --json` |
| Backup sessions only | `$CLI sync --sessions-only --json` |
| Check backup status | `$CLI status --json` |
| List all sessions | `$CLI restore --list --json` |
| List recent N sessions | `$CLI restore --last N --json` |
| Find sessions by project | `$CLI restore --project NAME --json` |
| Find sessions by date | `$CLI restore --date YYYY-MM-DD --json` |
| Preview a session | `$CLI peek UUID --json` |
| Restore a session | `$CLI restore UUID --json` |
| Restore (overwrite) | `$CLI restore UUID --force --json` |
| Export config tarball | `$CLI export-config --json` |
| Import config tarball | `$CLI import-config FILE --json` |
| Switch backend mode | `$CLI backend set <github\|git\|local> --json` |
| Switch to custom remote | `$CLI backend set git --remote URL --json` |
| Set backup schedule | `$CLI schedule <off\|daily\|6h\|hourly> --json` |
| Restore all sessions | `$CLI restore --all --json` |
| Restore all (overwrite) | `$CLI restore --all --force --json` |
| Restore from machine | `$CLI restore --all --machine SLUG --json` |
| Uninstall scheduler | `$CLI uninstall` |

## Reading responses

- Success: `{"ok": true, …}` with exit code 0
- Error: `{"error": "message"}` on stderr with exit code 1
- `status` response includes `"mode": "github"`, `"mode": "git"`, or `"mode": "local"`, plus `"machine"`, `"machineSlug"`, and `"schedule"` fields
- Summarize results conversationally for the user. Don't dump raw JSON.
- Pass `--backend <mode>` during `init` to set backend (github, git, local). `--local` is kept as alias for `--backend local`

## Helping users find sessions

Use `peek --json` before restoring to confirm the right session. Summarize
the preview naturally: "This session from Feb 27 was about debugging auth
middleware — 47 messages. Want me to restore it?"

## When to suggest backups

- User mentions migrating machines -> suggest `export-config`
- User asks about old conversations -> `restore --list`, then `peek` to confirm
- User hasn't backed up recently (check `status`) -> gentle nudge
