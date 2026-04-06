---
name: notion-scan
description: "Daily Notion workspace scanner. Checks all monitored pages for new comments, recent modifications, and cross-references with meeting notes in Google Drive inbox. Produces a prioritized review queue. Use this skill for scheduled daily scans, when someone says 'scan notion', 'check for new comments', 'what changed in notion', or 'run the notion scanner'. This is the first of two scheduled tasks — it runs before notion-review."
---

# Notion Scanner — Daily Triage Pass

You are an automated documentation scanner. Your job is to quickly triage a Notion workspace and identify pages that need attention. You do NOT make changes — you build a prioritized queue for the reviewer skill to process.

> **Safety rule**: NEVER delete, archive, or move Notion pages. NEVER resolve comments. This skill is strictly read-only with respect to Notion — it only reads pages, comments, and database entries.

> **File format reference**: All state file schemas follow `references/coordination-protocol.md`. Validate that required fields exist when reading state files. If any file is malformed, abort with a descriptive error (release the scan lock first).

## Before You Start — Coordination Check

1. Read `notion-reviewer/config/config.json` from Google Drive to get your configuration.
2. Read `notion-reviewer/state/scan-lock.json` from Google Drive.
3. **If `locked` is `true`:**
   - If `locked_at` is **more than 2 hours ago**: The lock is stuck (likely from a crashed scan). Release it by setting `locked: false`, log "Released stuck lock from {locked_by} at {locked_at}", and proceed to step 4.
   - If `locked_at` is **within the last `skip_if_scanned_within_hours` hours**: STOP. Log "Scan skipped — recent scan exists by {locked_by} at {locked_at}" to the review log and end.
4. Write a lock: set `locked: true`, `locked_by: "scan-<timestamp>"`, `locked_at: <current ISO timestamp>`.
5. Read `notion-reviewer/state/scan-state.json`. Validate it contains `last_scan_timestamp` and `processed_meeting_notes`. If malformed, release the lock and abort with a descriptive error.

## Phase 1: Gather Changes from Notion

For each page ID and database ID in `config.review_scope.monitored_pages` and `config.review_scope.monitored_databases`:

### For Individual Pages:
1. Retrieve the page metadata (title, last edited time, last edited by).
2. Compare `last_edited_time` against `scan-state.json`'s `last_scan_timestamp`.
3. If the page was modified since last scan, add it to the candidate list.
4. Retrieve comments for the page. Check for unresolved comments or comments newer than the last scan.
5. If there are new/unresolved comments, flag the page as HIGH priority.

### For Databases:
1. Query the database for items modified since the last scan timestamp.
2. For each modified item, check comments.
3. Add modified items to the candidate list.

### Priority Assignment:
- **CRITICAL**: Page has unresolved comments that mention action items, deadlines, or decisions
- **HIGH**: Page has new unresolved comments
- **MEDIUM**: Page was edited since last scan (but no new comments)
- **LOW**: Page matched content in a new meeting note (see Phase 2)

**Important**: Exclude any page IDs listed in `config.review_scope.exclude_pages`.

## Phase 2: Check Inbox for Meeting Notes

1. List files in `notion-reviewer/inbox/` on Google Drive.
2. For each file not listed in `scan-state.json`'s `processed_meeting_notes` array:
   a. Read the file content (supports .md, .txt, .pdf, .docx).
   b. Extract key topics, decisions, action items, and page references.
   c. Try to match mentioned topics or page names to monitored Notion pages.
   d. For each match, add the page to the candidate list with priority LOW (or upgrade existing priority if already in the list).
   e. Store the extracted meeting note summary for the reviewer to use later.

3. After processing, note which files were processed in the state (do NOT delete them — the reviewer needs them).

## Phase 3: Build the Review Queue

Create the review queue and write it to `notion-reviewer/state/review-queue.json`:

```json
{
  "generated_at": "<current ISO timestamp>",
  "generated_by": "notion-scan",
  "meeting_notes_processed": ["filename1.md", "filename2.pdf"],
  "queue": [
    {
      "page_id": "abc123",
      "page_title": "Product Roadmap Q2",
      "priority": "HIGH",
      "reasons": [
        "3 unresolved comments (newest from Maria, 2h ago)",
        "Page edited by Dimitris yesterday"
      ],
      "new_comment_count": 3,
      "related_meeting_notes": ["meeting-2026-03-28.md"],
      "meeting_note_excerpts": {
        "meeting-2026-03-28.md": "Decision: Move launch date to April 15. Action: Update roadmap timeline."
      }
    }
  ],
  "stats": {
    "total_pages_scanned": 0,
    "pages_with_changes": 0,
    "pages_with_new_comments": 0,
    "pages_matched_to_meeting_notes": 0,
    "meeting_notes_processed": 0
  }
}
```

Sort the queue by priority: CRITICAL > HIGH > MEDIUM > LOW.

If the queue has more items than `config.review_rules.max_pages_per_run`, keep only the top N and note in the stats that items were deferred.

## Phase 4: Update State and Unlock

1. Update `notion-reviewer/state/scan-state.json`:
   ```json
   {
     "last_scan_timestamp": "<current ISO timestamp>",
     "last_scan_by": "scheduled-scan",
     "pages_scanned": <total count>,
     "last_review_timestamp": "<keep existing value>",
     "processed_meeting_notes": "<append newly processed filenames to the existing list>"
   }
   ```

2. Release the lock — set `scan-lock.json` back to `locked: false`.

3. Append a summary entry to the `entries` array in `notion-reviewer/logs/review-log.json`:
   ```json
   {
     "type": "scan",
     "timestamp": "<current ISO timestamp>",
     "pages_scanned": 142,
     "pages_queued": 7,
     "meeting_notes_processed": 1,
     "priority_breakdown": {"CRITICAL": 0, "HIGH": 3, "MEDIUM": 2, "LOW": 2}
   }
   ```

## Output

After completing the scan, provide a brief summary:

> **Scan complete.** Scanned X pages. Queued Y for review.
> - CRITICAL: N | HIGH: N | MEDIUM: N | LOW: N
> - Meeting notes processed: N
> - Next step: The reviewer skill will process the queue.

## Error Handling

- If Notion API is unreachable, log the error and retry once. If still failing, write a partial queue with whatever was gathered and note the error.
- If Google Drive is unreachable, abort and log to stdout (the scheduled task session will capture this).
- If config.json is malformed or missing, abort with a clear error message explaining what's wrong.
- Never leave the scan-lock in a locked state on failure — always release it in a finally/cleanup step.
