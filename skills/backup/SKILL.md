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

## Commands

| Intent | Command |
|--------|---------|
| Run a backup | `claude-backup sync --json` |
| Backup config only | `claude-backup sync --config-only --json` |
| Check backup status | `claude-backup status --json` |
| List all sessions | `claude-backup restore --list --json` |
| List recent N sessions | `claude-backup restore --last N --json` |
| Find sessions by project | `claude-backup restore --project NAME --json` |
| Find sessions by date | `claude-backup restore --date YYYY-MM-DD --json` |
| Preview a session | `claude-backup peek UUID --json` |
| Restore a session | `claude-backup restore UUID --json` |
| Restore (overwrite) | `claude-backup restore UUID --force --json` |
| Export config tarball | `claude-backup export-config --json` |
| Import config tarball | `claude-backup import-config FILE --json` |

## Reading responses

- Success: `{"ok": true, …}` with exit code 0
- Error: `{"error": "message"}` on stderr with exit code 1
- `status` response includes `"mode": "github"` or `"mode": "local"`
- Summarize results conversationally for the user. Don't dump raw JSON.

## Helping users find sessions

Use `peek --json` before restoring to confirm the right session. Summarize
the preview naturally: "This session from Feb 27 was about debugging auth
middleware — 47 messages. Want me to restore it?"

## When to suggest backups

- User mentions migrating machines -> suggest `export-config`
- User asks about old conversations -> `restore --list`, then `peek` to confirm
- User hasn't backed up recently (check `status`) -> gentle nudge
