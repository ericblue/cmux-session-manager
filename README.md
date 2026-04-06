# cmux-session-manager

Snapshot and restore cmux workspaces with Claude Code session resumption.

When cmux crashes or is restarted, this tool recreates your workspace layout — splits, panels, working directories, Claude sessions, and optionally terminal commands — and resumes everything where it left off.

## How It Works

1. **Snapshot** reads cmux's session state file to capture all windows, workspaces, panels, and their layout (splits, orientations, working directories).

2. It cross-references running Claude processes (`ps`) with their `--session-id` flags, mapping each Claude panel to its active session. For non-Claude terminal panels, it detects running foreground commands (dev servers, watchers, etc.) via process inspection.

3. **Restore** recreates the workspace structure using `cmux` CLI commands, targeting each panel by surface ref to ensure commands land in the correct split. Claude sessions resume with `claude --resume <session-id>`, and terminal panels `cd` to their original directories. With `--run-commands`, captured terminal commands are re-launched automatically.

## Quick Start

```bash
# Take a snapshot of all workspaces
make snapshot

# List active Claude sessions
make list-active

# Show detailed info for a workspace
make show W=myproject

# After a crash — preview the restore plan
make restore-dry-run W=myproject

# Restore a workspace (auto-executes inside cmux, prompts for confirmation)
make restore W=myproject

# Respawn: snapshot + kill + restore in one step
make respawn W=myproject
```

## Common Workflows

### Recover after a cmux crash

```bash
# See what you had running
make list-snapshots

# Inspect a specific snapshot to verify it has what you need
make show F=cmux-20260405-161401

# Preview the restore plan
make restore-dry-run

# Restore everything (requires typing 'yes' for multi-workspace)
make restore
```

### Respawn a misbehaving workspace

```bash
# Check current state
make show W=devops-work

# Snapshot, kill, and restore in one step
make respawn W=devops-work
```

### Restore a single workspace from an older snapshot

```bash
# List available snapshots
make list-snapshots

# Preview what it would do
make restore-dry-run W=devops-work F=cmux-20260405-161401

# Restore it
make restore W=devops-work F=cmux-20260405-161401
```

### Restore with dev servers and watchers

```bash
# Take a snapshot (captures running terminal commands)
make snapshot W=vibefocus

# Check what commands were captured
make show W=vibefocus F=latest

# Restore with commands auto-launched
make restore W=vibefocus RC=1
```

### Audit what's running across all workspaces

```bash
# Quick overview — Claude sessions and panel counts
make list-active

# Deep dive into a specific workspace
make show W=vibefocus
```

### Safe teardown before OS restart

```bash
# Snapshot everything
make snapshot

# Verify it captured your workspaces
make list-snapshots

# After reboot, restore from inside cmux
make restore
```

## Make Targets

```
make help               Show all available targets
make list-active        List active Claude sessions with git branches
make show               Show detailed workspace info (W= workspace, F= snapshot)
make snapshot           Capture state (W= workspace, N= name)
make list-snapshots     List all saved snapshots with workspace names
make validate           Check snapshot health before restoring (F= snapshot, W= workspace)
make prune              Delete old snapshots, keep last N (KEEP=10)
make restore-dry-run    Preview restore (W= workspace, F= snapshot, RC=1 for commands)
make restore            Restore from snapshot (W= workspace, F= snapshot, RC=1 for commands)
make kill               Close a workspace with confirmation (requires W=)
make respawn            Snapshot, kill, and restore a workspace (requires W=)
make install            Symlink cmux-sessions into ~/bin
```

### Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `W=`     | Workspace name filter (case-insensitive substring) | `W=devops-work` |
| `F=`     | Snapshot file (bare name, with .json, or full path) | `F=cmux-20260405-161401` |
| `N=`     | Named snapshot (instead of timestamp) | `N=before-refactor` |
| `RC=1`   | Re-run captured terminal commands on restore | `RC=1` |
| `KEEP=`  | Number of snapshots to keep when pruning (default: 10) | `KEEP=5` |

## CLI Commands

```
cmux-sessions list                            Show Claude sessions with git branches
cmux-sessions show -w myproject               Show detailed workspace info (live)
cmux-sessions show -w myproject -f snap.json  Show detailed workspace info (snapshot)
cmux-sessions snapshot                        Save current state (all workspaces)
cmux-sessions snapshot -w myproject           Snapshot only matching workspace
cmux-sessions snapshot -n before-refactor     Save with a name instead of timestamp
cmux-sessions snapshots                       List all saved snapshots
cmux-sessions validate                        Check snapshot health before restoring
cmux-sessions validate -f snap.json -w proj   Validate specific snapshot/workspace
cmux-sessions prune                           Delete old snapshots (keep last 10)
cmux-sessions prune --keep 5                  Keep last 5 snapshots
cmux-sessions restore --dry-run               Preview what would be restored
cmux-sessions restore -w myproject            Restore only matching workspace
cmux-sessions restore --run-commands          Re-run saved terminal commands
cmux-sessions kill -w myproject               Close a workspace (with confirmation)
cmux-sessions kill -w myproject -y            Close without confirmation
cmux-sessions respawn -w myproject            Snapshot, kill, and restore in one step
```

## Features

### Workspace Show

`make show W=myproject` displays every panel with its type, working directory, and session/command info:

```
Workspace: myproject
Directory: ~/Development/myproject
Panels:    3

  Panel 1: [claude] Implement auth middleware
    cwd: ~/Development/myproject
    session: a1b2c3d4-5678-90ab-cdef-1234567890ab
    pid: 12345
    status: running

  Panel 2: [terminal] Terminal
    cwd: ~/Development/myproject
    command: npm run dev

  Panel 3: [terminal] Terminal
    cwd: ~/Development/myproject/docs
```

Works with `-f` to inspect snapshot contents: `make show W=myproject F=cmux-20260405-161401`

### Terminal Command Capture

Snapshots detect foreground processes running in non-Claude terminal panels (dev servers, watchers, REPLs, etc.) and save them as `lastCommand`. On restore:

- **Default**: commands are shown as hints but not executed — panels just `cd` to the correct directory
- **`RC=1`** / `--run-commands`: commands are re-launched automatically

### Smart Restore

Restore auto-detects whether you're inside cmux:

- **Inside cmux**: shows the plan, prompts for confirmation, then executes directly
- **Outside cmux**: generates a restore shell script to run from a cmux terminal

Safety checks before restoring:

- **Duplicate detection**: warns if target workspaces are already open, requires typing `force` to continue
- **Multi-workspace guard**: restoring all workspaces (no `W=` filter) requires typing `yes`
- **Single workspace**: standard `y/N` confirmation

### Respawn

`make respawn W=myproject` is the all-in-one workflow:

1. Snapshots the workspace
2. Prompts for confirmation
3. Kills it via `cmux close-workspace`
4. Restores from the fresh snapshot

### Session Discovery

Claude session IDs are resolved in priority order:

1. Running process with `--session-id` or `--resume` flag
2. `sessions-index.json` in `~/.claude/projects/`
3. Most recent `.jsonl` session file by modification time (fallback when index is missing)

### Named Snapshots

Give snapshots meaningful names instead of timestamps:

```bash
make snapshot W=devops N=before-refactor
# Saves as ~/.cmux-snapshots/cmux-before-refactor.json

# Restore from it later
make restore W=devops F=before-refactor
```

### Pre-restore Validation

Check that a snapshot is still valid before restoring:

```bash
make validate F=cmux-20260405-161401
```

Output shows PASS/FAIL for each panel's directory and Claude session:

```
WORKSPACE  TYPE        DIRECTORY                   DIR   SESSION
---------  ----------  --------------------------  ----  -------
myproject  [claude]    ~/Development/myproject      PASS  PASS
myproject  [terminal]  ~/Development/myproject      PASS  -
myproject  [terminal]  ~/Development/myproject/api  PASS  -
```

### Snapshot Pruning

Snapshots accumulate over time. Clean up old ones:

```bash
make prune              # Keep last 10
make prune KEEP=5       # Keep last 5
```

### Git Branch Display

`make list-active` includes a BRANCH column showing the current git branch for each Claude session's working directory.

## Installation

```bash
make install    # Symlinks to ~/bin/cmux-sessions
```

Or add to your PATH manually:
```bash
export PATH="$HOME/Development/cmux-sessions:$PATH"
```

## File Locations

| Path | Purpose |
|------|---------|
| `~/.cmux-snapshots/` | Snapshot storage directory |
| `~/.cmux-snapshots/latest.json` | Most recent snapshot (used by default restore) |
| `~/.cmux-snapshots/restore.sh` | Generated restore script (outside cmux fallback) |
| `~/Library/Application Support/cmux/session-com.cmuxterm.app.json` | cmux's live session state (read-only) |
| `~/.claude/projects/` | Claude Code session index and history files |

## Automating Snapshots

To take periodic snapshots, add a cron job or launchd plist:

```bash
# Every 30 minutes
*/30 * * * * python3 ~/Development/cmux-sessions/cmux-sessions.py snapshot 2>/dev/null
```

## Limitations

- **Kill/respawn must be run from inside cmux** — `cmux close-workspace` requires a socket connection only available to cmux child processes. Snapshot, list, and show work from anywhere.
- **Layout is approximate** — complex nested splits are flattened into sequential split operations. Split directions and panel order are preserved, but exact divider positions may shift.
- **Terminal command capture is best-effort** — it detects foreground child processes of shell sessions. Idle shells (at a prompt) have no command to capture. Background jobs and piped commands may not be detected.
- **Session IDs are point-in-time** — if a Claude session is ended and a new one started between snapshot and restore, the old session ID will be used. Take fresh snapshots regularly.

## Requirements

- Python 3.8+
- cmux (macOS)
- Claude Code CLI
