---
name: notion-status
description: "Check the status of the Notion Reviewer agent system. Shows last scan time, queue contents, recent review activity, and any errors. Use when someone asks 'what's the agent status', 'notion reviewer status', 'did the review run today', 'check the agent', or 'show me the review log'."
---

# Notion Reviewer — Status Check

You are checking the health and status of the Notion Reviewer agent system. Read the state files from Google Drive and present a clear dashboard.

> **File format reference**: All state file schemas follow `references/coordination-protocol.md`. If a file is missing or malformed, report it as a health issue rather than failing.

## Gather State

Read these files from Google Drive (report any that are missing):

1. `notion-reviewer/config/config.json`
2. `notion-reviewer/state/scan-state.json`
3. `notion-reviewer/state/scan-lock.json`
4. `notion-reviewer/state/review-queue.json`
5. `notion-reviewer/logs/review-log.json` (last 20 entries)

## Present Status

Format the status as a clear report:

```
📊 NOTION REVIEWER STATUS
═══════════════════════════

🔍 Last Scan
   Time: [timestamp or "never"]
   By: [who ran it]
   Pages scanned: [count]
   Currently locked: [yes/no — if yes, flag as potential issue]

📋 Review Queue
   Generated: [timestamp or "empty"]
   Items pending: [count]
   Priority breakdown: CRITICAL: N | HIGH: N | MEDIUM: N | LOW: N

📝 Last Review
   Time: [timestamp or "never"]
   Pages reviewed: [count]
   Proposals made: [count]
   Comments posted: [count]
   Direct edits: [count]

📥 Inbox
   Files waiting: [list filenames in inbox/]
   Files processed: [list filenames in inbox/processed/]

⚙️ Configuration
   Monitored pages: [count]
   Monitored databases: [count]
   Designated runner: [name]
   Team: [names]
   Auto-comment: [on/off]
   Auto-edit: [on/off]
   Max pages per run: [N]

📜 Recent Activity (last 5 log entries)
   [timestamp] [type] — [brief summary]
   ...
```

## Health Checks

Flag any issues:

- ⚠️ **Scan lock stuck** — if `locked: true` for more than 2 hours, something went wrong. Offer to release the lock.
- ⚠️ **No scan in 24h** — if `last_scan_timestamp` is more than 24 hours ago, the scheduled task may not be running.
- ⚠️ **Stale queue** — if `review-queue.json` has items but `generated_at` is more than 24 hours ago, the reviewer may not have run.
- ⚠️ **Inbox piling up** — if there are more than 10 unprocessed files in inbox/, flag this.
- ⚠️ **Errors in log** — if recent log entries contain errors, surface them.

If any issues are found, suggest remediation steps.
