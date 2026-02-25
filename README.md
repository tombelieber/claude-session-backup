# Claude Backup

Back up your Claude Code chat sessions to a private GitHub repo.

## Quick Start

```bash
npx claude-backup
```

That's it. The interactive setup checks requirements, creates a private repo, runs the first backup, and schedules daily automatic backups.

## What It Does

- **Compresses and backs up** all Claude Code sessions (`~/.claude/projects/`) to a private GitHub repo
- **Schedules daily automatic backups** via macOS launchd (3:00 AM)
- **Restores any session** from backup when needed

## Requirements

- **macOS** (Linux/Windows coming soon)
- **git**
- **gh** ([GitHub CLI](https://cli.github.com), authenticated via `gh auth login`)
- **gzip** (built-in on macOS)

## Commands

| Command | Description |
|---|---|
| `claude-backup` | Interactive first-time setup |
| `claude-backup sync` | Run backup now |
| `claude-backup status` | Show backup status |
| `claude-backup restore <UUID>` | Restore a session |
| `claude-backup uninstall` | Remove scheduler |

## How It Works

1. Compresses `.jsonl` session files with gzip (typically 3-5x compression ratio)
2. Commits to a private GitHub repo (`github.com/<you>/claude-backup-data`)
3. Incremental -- only processes files that changed since the last backup
4. Non-JSONL files (e.g. metadata) are copied as-is

The daily scheduler runs at 3:00 AM via macOS launchd. Backups are pushed automatically.

## Restoring Sessions

List available backups:

```bash
ls ~/.claude-backup/projects/*/
```

Restore a specific session:

```bash
claude-backup restore <UUID>
```

The restore command decompresses the `.gz` file back to its original location under `~/.claude/projects/`. It will not overwrite existing files.

## Uninstall

```bash
claude-backup uninstall
```

This removes the daily scheduler and optionally deletes local backup data. Your GitHub repo must be deleted separately:

```bash
gh repo delete claude-backup-data
```

## Storage

Session data is stored at:

| Location | Contents |
|---|---|
| `~/.claude-backup/` | Local compressed backups + git repo |
| `github.com/<you>/claude-backup-data` | Remote private repo |
| `~/Library/LaunchAgents/com.claude-backup.plist` | macOS scheduler |

## Future Plans

- Linux support (systemd timer)
- iCloud/Dropbox alternative backends
- Session browser and search

## License

[MIT](LICENSE)
