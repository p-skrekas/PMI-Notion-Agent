# Notion Reviewer Plugin

An automated Notion documentation reviewer for teams managing large page sets. Scans for new comments, cross-references meeting notes from Google Drive, and proposes specific page edits — all running on a daily schedule via Claude Code.

## How It Works

```
 ┌─────────────┐     ┌──────────────────┐     ┌──────────────┐
 │  SCHEDULED   │     │   Google Drive    │     │    Notion    │
 │   7:00 AM    │────▶│                  │     │              │
 │  notion-scan │     │  config.json     │     │  400+ pages  │
 │              │     │  scan-state.json │◀───▶│  comments    │
 └──────┬───────┘     │  review-queue    │     │  databases   │
        │             │  review-log      │     │              │
        ▼             │  inbox/          │     │              │
 ┌─────────────┐     │   └─meeting notes │     │              │
 │  SCHEDULED   │     │                  │     │              │
 │   7:30 AM    │────▶│                  │────▶│  💬 posts    │
 │notion-review │     │                  │     │  comments    │
 │              │     │                  │     │  with edits  │
 └──────────────┘     └──────────────────┘     └──────────────┘

 ┌──────────────┐
 │  ON DEMAND   │     Any team member can run these anytime:
 │notion-status │     • Check what the agent did
 │notion-setup  │     • Add pages to monitor
 │notion-review │     • Run a manual review on specific pages
 └──────────────┘
```

**One person** (the designated runner) runs the daily scheduled tasks.
**Everyone** sees proposed changes as Notion comments and can run skills on-demand.

## Prerequisites

- Claude Desktop app (macOS or Windows) with a paid plan (Pro/Max)
- Notion connector configured in Claude (Settings > Connectors)
- Google Drive connector configured in Claude (Settings > Connectors)
- A shared Google Drive folder accessible to the team

## Quick Start

### 1. Install the Plugin

```
# In Claude Code CLI:
claude plugin install /path/to/notion-reviewer-plugin

# Or add --plugin-dir when testing locally:
claude --plugin-dir /path/to/notion-reviewer-plugin
```

Alternatively, install from the team's Git repo once published.

### 2. Run Setup

In Claude Code or Claude Desktop, type:

```
/notion-reviewer:notion-setup
```

This will walk you through:
- Creating the shared Google Drive folder structure
- Adding Notion pages to monitor
- Configuring review rules
- Setting up scheduled tasks (designated runner only)

### 3. Daily Operation

Once set up, the system runs automatically:

| Time | What Happens |
|------|-------------|
| 7:00 AM | **Scanner** checks all monitored pages for changes and comments |
| 7:30 AM | **Reviewer** reads flagged pages, analyzes comments + meeting notes, posts proposed edits as Notion comments |
| Anytime | Team reviews proposed changes in Notion (approve, reject, or discuss) |

### 4. Feeding Meeting Notes

Drop meeting notes into the shared `notion-reviewer/inbox/` folder on Google Drive. Supported formats: `.md`, `.txt`, `.pdf`, `.docx`.

The scanner will automatically:
- Extract decisions and action items
- Match them to relevant monitored pages
- Queue those pages for review with meeting context

Processed notes are moved to `inbox/processed/` after the reviewer handles them.

## Available Skills

| Skill | Trigger | Purpose |
|-------|---------|---------|
| `/notion-reviewer:notion-setup` | First-time setup or onboarding | Creates Drive structure, configures pages to monitor |
| `/notion-reviewer:notion-scan` | Daily scheduled (or manual) | Scans workspace, builds prioritized review queue |
| `/notion-reviewer:notion-review` | Daily scheduled (or manual) | Deep-reviews queued pages, posts proposed changes |
| `/notion-reviewer:notion-status` | On demand | Dashboard showing agent health and recent activity |

## Configuration

Edit `notion-reviewer/config/config.json` in Google Drive:

```jsonc
{
  "review_rules": {
    "auto_comment": false,      // Post proposals as Notion comments (off by default)
    "auto_edit": false,          // Apply HIGH-confidence edits directly (use with caution)
    "flag_for_human_review": true, // Flag LOW-confidence proposals
    "max_pages_per_run": 30,     // Cap pages per review cycle
    "skip_if_scanned_within_hours": 12  // Prevent duplicate scans
  }
}
```

### Adding/Removing Pages

Either:
- Edit `config.json` directly (add page IDs to `monitored_pages` or `monitored_databases`)
- Run `/notion-reviewer:notion-setup` again and follow the prompts

### Enabling Direct Edits

By default, the agent only posts comments with proposals. To let it edit pages directly:

1. Set `auto_edit: true` in config.json
2. Only HIGH confidence changes will be applied directly
3. The agent always posts a comment noting what it changed

**Recommendation**: Start with both `auto_comment: false` and `auto_edit: false` (the defaults). Review proposals in the review log and `/notion-reviewer:notion-status` output. Once you trust the agent's proposals, enable `auto_comment: true` to post them as Notion comments. Only enable `auto_edit: true` after sustained confidence in the proposals.

## Team Roles

| Role | Who | What They Do |
|------|-----|-------------|
| **Designated Runner** | One person (has Pro or Max plan) | Runs scheduled tasks, manages config, monitors health |
| **Team Member** | Everyone else | Reviews proposals in Notion, drops meeting notes in inbox, runs on-demand reviews |

### Onboarding a New Team Member

1. They install the plugin from the Git repo
2. They run `/notion-reviewer:notion-setup` and select "joining existing setup"
3. They point it to the shared Google Drive folder
4. Done — they can now run on-demand reviews and see all activity

### Handing Off the Runner Role

1. New runner creates scheduled tasks on their account using the scan and review skills
2. Old runner disables their scheduled tasks
3. Update `designated_runner` in config.json

## Troubleshooting

### "Scan skipped — recent scan exists"
Normal. The scan-lock prevents duplicate runs within the `skip_if_scanned_within_hours` window.

### Scan lock appears stuck
Run `/notion-reviewer:notion-status`. If it detects a stuck lock (locked > 2 hours), it will offer to release it.

### Agent proposes changes that don't make sense
1. Check if the relevant meeting notes in `inbox/` were clearly written
2. Lower `max_pages_per_run` to give the agent more tokens per page
3. Consider setting pages with complex context to `exclude_pages` and reviewing them manually

### Hitting usage limits
Each scheduled task run is a full Claude session. On a Pro plan, daily scan + review of 20-30 pages is manageable. For larger workspaces, a Max plan gives more headroom.

### Agent isn't picking up my meeting notes
- Verify files are in the `inbox/` folder, not a subfolder
- Supported formats: .md, .txt, .pdf, .docx
- Check that the content mentions topics related to monitored pages

## Architecture Reference

See `references/coordination-protocol.md` for detailed file format specs, locking mechanism, and scaling guidance.

## License

MIT
