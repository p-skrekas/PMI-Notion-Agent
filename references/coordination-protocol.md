# Coordination Protocol Reference

## Overview

The Notion Reviewer plugin uses Google Drive as a lightweight coordination layer between team members. This document explains the file formats, locking mechanism, and state management.

## File Formats

### config.json

Central configuration shared by all team members. Read by all skills (scan, review, status). Any team member can edit it to change what pages are monitored.

| Field | Type | Description |
|-------|------|-------------|
| `version` | string | Config format version |
| `designated_runner` | string | Name of the person running scheduled tasks |
| `team` | string[] | All team member names |
| `scan_schedule` | string | "daily", "twice-daily", "weekdays" |
| `review_scope.monitored_pages` | string[] | Notion page IDs to scan |
| `review_scope.monitored_databases` | string[] | Notion database IDs to scan |
| `review_scope.exclude_pages` | string[] | Page IDs to skip |
| `review_rules.auto_comment` | boolean | Post proposed changes as Notion comments |
| `review_rules.auto_edit` | boolean | Apply HIGH confidence edits directly |
| `review_rules.flag_for_human_review` | boolean | Flag LOW confidence proposals |
| `review_rules.max_pages_per_run` | integer | Cap per review cycle |
| `review_rules.skip_if_scanned_within_hours` | integer | Prevent duplicate scans |
| `notifications.summary_page_id` | string/null | Notion page to post daily summaries |

### scan-state.json

Tracks when the last scan and review happened, and which meeting notes have been processed. Written by the scan and review skills.

**Default values:**

```json
{
  "last_scan_timestamp": null,
  "last_scan_by": null,
  "pages_scanned": 0,
  "last_review_timestamp": null,
  "processed_meeting_notes": []
}
```

| Field | Type | Default | Written by |
|-------|------|---------|------------|
| `last_scan_timestamp` | string/null | `null` | scan |
| `last_scan_by` | string/null | `null` | scan |
| `pages_scanned` | integer | `0` | scan |
| `last_review_timestamp` | string/null | `null` | review |
| `processed_meeting_notes` | string[] | `[]` | scan (append-only) |

### scan-lock.json

Simple mutex to prevent simultaneous scans. The scanner sets `locked: true` at start and `locked: false` at end. Stuck locks (locked for more than 2 hours) are auto-released by the scan skill. The status skill can also detect and offer to release them.

**Default values:**

```json
{
  "locked": false,
  "locked_by": null,
  "locked_at": null
}
```

| Field | Type | Default | Written by |
|-------|------|---------|------------|
| `locked` | boolean | `false` | scan |
| `locked_by` | string/null | `null` | scan |
| `locked_at` | string/null | `null` | scan |

### review-queue.json

Output of the scan skill, input of the review skill. Contains a prioritized list of pages that need attention, along with reasons and meeting note references.

**Default values:**

```json
{
  "generated_at": null,
  "generated_by": null,
  "meeting_notes_processed": [],
  "queue": [],
  "stats": {
    "total_pages_scanned": 0,
    "pages_with_changes": 0,
    "pages_with_new_comments": 0,
    "pages_matched_to_meeting_notes": 0,
    "meeting_notes_processed": 0
  }
}
```

**Queue item fields:**

| Field | Type | Description |
|-------|------|-------------|
| `page_id` | string | Notion page ID |
| `page_title` | string | Page title for display |
| `priority` | string | `CRITICAL`, `HIGH`, `MEDIUM`, or `LOW` |
| `reasons` | string[] | Human-readable reasons for queuing |
| `new_comment_count` | integer | Number of new/unresolved comments |
| `related_meeting_notes` | string[] | Filenames of relevant meeting notes |
| `meeting_note_excerpts` | object | `{ filename: excerpt }` map |

### review-log.json

Append-only log of all scan and review activity. All entries go in the `entries` array. Used for:
- Preventing duplicate proposals (reviewer checks recent log before proposing)
- Team visibility into what the agent did
- Debugging issues

**Default values:**

```json
{
  "entries": []
}
```

**Entry types:**

*Scan summary* (appended by scan skill):

```json
{
  "type": "scan",
  "timestamp": "<ISO timestamp>",
  "pages_scanned": 0,
  "pages_queued": 0,
  "meeting_notes_processed": 0,
  "priority_breakdown": { "CRITICAL": 0, "HIGH": 0, "MEDIUM": 0, "LOW": 0 }
}
```

*Page review* (appended by review skill, one per page):

```json
{
  "type": "review",
  "timestamp": "<ISO timestamp>",
  "page_id": "<string>",
  "page_title": "<string>",
  "proposals_made": 0,
  "proposals_by_type": { "COMMENT_RESPONSE": 0, "CONTENT_UPDATE": 0, "FLAG_FOR_HUMAN": 0 },
  "proposals_by_confidence": { "HIGH": 0, "MEDIUM": 0, "LOW": 0 },
  "comments_posted": 0,
  "direct_edits_made": 0,
  "errors": []
}
```

*Review summary* (appended by review skill, once per run):

```json
{
  "type": "review_summary",
  "timestamp": "<ISO timestamp>",
  "pages_reviewed": 0,
  "total_proposals": 0,
  "total_comments_posted": 0,
  "total_direct_edits": 0,
  "meeting_notes_fully_processed": 0
}
```

*Scan skipped* (appended by scan skill when skipping due to recent scan or releasing a stuck lock):

```json
{
  "type": "scan_skipped",
  "timestamp": "<ISO timestamp>",
  "reason": "recent_scan | stuck_lock_released",
  "detail": "<descriptive message>"
}
```

## Locking Protocol

The scan-lock is intentionally simple because:
- Daily cadence means collisions are rare
- A small team (2-5 people) with one designated runner means typically only one process writes
- The status skill can detect and fix stuck locks

For teams that need stronger guarantees, consider using a Notion database as the lock instead of a Drive file — Notion provides last-edited metadata that serves as a natural timestamp.

## Adding Pages to Monitor

Any team member can add pages to monitor by:
1. Editing `config.json` directly in Google Drive
2. Or running `/notion-reviewer:notion-setup` and following the prompts

The next scan will pick up the changes automatically.

## Meeting Notes Inbox

Drop any file into `notion-reviewer/inbox/` on Google Drive:
- `.md`, `.txt` — read directly
- `.pdf` — text extracted during scan
- `.docx` — text extracted during scan

The scanner attempts to match meeting note content to monitored pages by topic and keyword matching. After the reviewer processes all pages referencing a meeting note, the note is moved to `inbox/processed/`.

## Handling Large Workspaces (400+ pages)

The system is designed for large workspaces through two mechanisms:

1. **Delta scanning** — only pages modified since the last scan are queued. For a 400-page workspace, a typical daily delta is 5-20 pages.
2. **max_pages_per_run** — caps the review pass to prevent runaway token usage. Overflow pages are automatically picked up in the next cycle.

If the workspace grows beyond 500+ pages, consider:
- Splitting into multiple config files (one per team/domain)
- Using database queries with filters instead of individual page IDs
- Increasing the scan frequency to keep deltas small
