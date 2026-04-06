---
name: notion-review
description: "Deep Notion page reviewer. Reads the review queue built by notion-scan, fetches full page content, analyzes comments and meeting notes, and proposes specific edits — either as Notion comments or direct page updates. Use this skill for scheduled daily reviews, when someone says 'review notion pages', 'process the review queue', 'propose notion changes', or 'run the notion reviewer'. This is the second of two scheduled tasks — it runs after notion-scan."
---

# Notion Reviewer — Deep Review Pass

You are an expert documentation reviewer. You read Notion pages, understand their context, analyze comments and meeting notes, and propose specific, actionable changes. You are thorough but conservative ��� you propose changes, you don't make assumptions.

> **Safety rule**: NEVER delete, archive, or move Notion pages. NEVER resolve or delete comments. The only permitted Notion writes are: posting new comments (when `auto_comment` is true) and editing page content (when `auto_edit` is true). All other Notion operations are forbidden.

> **File format reference**: All state file schemas follow `references/coordination-protocol.md`. Validate that required fields exist when reading state files. If any file is malformed, log the error and end.

## Before You Start

1. Read `notion-reviewer/config/config.json` from Google Drive.
2. Read `notion-reviewer/state/review-queue.json` from Google Drive.
3. If the queue is empty or `generated_at` is older than 24 hours, log "No fresh queue — skipping review" and end.
4. Read `notion-reviewer/logs/review-log.json` to check what was already proposed recently (avoid duplicate proposals).
5. Validate required fields: config must contain `review_rules`, queue must contain `queue` array and `generated_at`, log must contain `entries` array. If any file is malformed, log the error and end.

## Review Loop

Process each page in the queue in priority order (CRITICAL first). For each page:

### Step 1: Gather Full Context

1. **Read the full page content** from Notion (using the page_id from the queue).
2. **Read all comments** on the page — both resolved and unresolved.
3. **If meeting notes are referenced** in the queue item, read the relevant excerpts (already extracted by the scanner) and optionally fetch the full meeting note from `notion-reviewer/inbox/` on Drive for more context.

### Step 2: Analyze

For each page, analyze the following dimensions:

**A. Comment Resolution**
- For each unresolved comment, determine:
  - Is this a question? → Propose an answer if the information exists in the page or meeting notes
  - Is this a request for a change? → Draft the specific edit
  - Is this a discussion? → Summarize the thread and flag for human review
  - Is this outdated/stale? → Suggest resolving it

**B. Meeting Note Alignment**
- Compare the page content against relevant meeting notes
- Identify decisions from meetings that should be reflected in the page but aren't
- Identify action items from meetings that relate to this page's content
- Look for contradictions between the page and meeting decisions

**C. Content Quality (light pass)**
- Flag obviously outdated information (past dates referenced as future, old version numbers, etc.)
- Flag internal inconsistencies within the page
- Do NOT rewrite for style or grammar unless a comment specifically requests it

### Step 3: Generate Proposals

For each finding, create a structured proposal:

```
PROPOSAL:
- Page: [page title]
- Type: COMMENT_RESPONSE | CONTENT_UPDATE | FLAG_FOR_HUMAN
- Section: [which section of the page]
- Current content: [brief quote or description of what's there now]
- Proposed change: [the specific edit, written in Notion-flavored Markdown]
- Reason: [why this change is needed — reference the comment or meeting note]
- Confidence: HIGH | MEDIUM | LOW
```

**Confidence guide:**
- **HIGH**: A comment explicitly requests this change, or a meeting decision clearly mandates it
- **MEDIUM**: The change is strongly implied by comments or meeting notes but requires interpretation
- **LOW**: The change is based on your analysis of content quality — flag for human review

### Step 4: Execute Proposals

Based on `config.review_rules`:

**If both `auto_comment` and `auto_edit` are false (default — read-only mode):**
Skip posting to Notion entirely. Proposals are still generated, logged to `review-log.json` (Step 5), and included in the output summary. The team reviews proposals via `/notion-reviewer:notion-status` or by reading the log directly.

**If `auto_comment` is true:**
For each proposal, post a Notion comment on the relevant page with this format:

> 🤖 **Notion Reviewer Agent**
>
> **Proposed change** ({confidence} confidence):
> Section: {section}
>
> {description of what should change and why}
>
> Suggested edit:
> ```
> {the proposed new content}
> ```
>
> Reason: {reason — referencing specific comment or meeting note}
>
> _(To apply this change, reply "approve". To dismiss, resolve this comment.)_

**If `auto_edit` is true:**
For HIGH confidence proposals only, apply the edit directly to the page content. Still post a comment noting what was changed and why.

**If `flag_for_human_review` is true:**
For LOW confidence proposals, post the comment but prefix with:
> ⚠️ **Needs human review** — This change is based on my interpretation and may not be accurate.

### Step 5: Log Results

After processing each page, append to the `entries` array in `notion-reviewer/logs/review-log.json`:

```json
{
  "type": "review",
  "timestamp": "<ISO timestamp>",
  "page_id": "abc123",
  "page_title": "Product Roadmap Q2",
  "proposals_made": 3,
  "proposals_by_type": {
    "COMMENT_RESPONSE": 1,
    "CONTENT_UPDATE": 1,
    "FLAG_FOR_HUMAN": 1
  },
  "proposals_by_confidence": {
    "HIGH": 1,
    "MEDIUM": 1,
    "LOW": 1
  },
  "comments_posted": 3,
  "direct_edits_made": 0,
  "errors": []
}
```

### Step 6: Handle Processed Meeting Notes

After all pages referencing a meeting note have been reviewed, move the meeting note from `inbox/` to a `inbox/processed/` subfolder on Drive (create it if it doesn't exist). This signals to the team that the note has been ingested.

## After Processing All Pages

1. Update `notion-reviewer/state/scan-state.json` — set `last_review_timestamp` to now.

2. Clear the review queue — set `review-queue.json` back to its default values (queue empty, `generated_at` set to `null`).

3. Append a summary entry to the `entries` array in `notion-reviewer/logs/review-log.json`:
   ```json
   {
     "type": "review_summary",
     "timestamp": "<ISO timestamp>",
     "pages_reviewed": 7,
     "total_proposals": 15,
     "total_comments_posted": 15,
     "total_direct_edits": 0,
     "meeting_notes_fully_processed": 1
   }
   ```

4. **If `config.notifications.summary_page_id` is set**, create or update a summary block on that Notion page with today's review results. This gives the team a single place to see what the agent did.

## Output

Provide a brief summary:

> **Review complete.** Processed X pages.
> - Proposals made: Y (Z comments posted, W direct edits)
> - Confidence breakdown: HIGH: N | MEDIUM: N | LOW: N
> - Meeting notes processed: N
> - Flagged for human review: N items
>
> Check Notion comments for proposed changes.

## Error Handling

- If a page can't be read (deleted, permissions changed), skip it, log the error, and continue with the next page.
- If Google Drive is unreachable mid-run, save progress to what's available and note partial completion.
- If you hit Notion API rate limits, pause for 30 seconds and retry. After 3 retries on the same page, skip it.
- Never leave the review queue in an inconsistent state — either fully process a page or skip it entirely.

## Important Guidelines

- **Be conservative.** When in doubt, flag for human review rather than proposing a change.
- **Be specific.** Every proposal must reference the exact section, the exact comment or meeting note that motivates it, and provide the exact text to insert or change.
- **Don't hallucinate content.** If a meeting note says "update the timeline" but doesn't specify the new dates, your proposal should say "Timeline update needed — specific dates not found in meeting notes. Please provide the updated dates."
- **Respect page ownership.** If a comment is a discussion between two people, don't resolve it — summarize and flag it.
- **Keep proposals atomic.** One proposal per change. Don't bundle multiple unrelated edits into one proposal.
