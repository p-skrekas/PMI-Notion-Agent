---
name: notion-setup
description: "Initial setup for the Notion Reviewer agent. Creates the shared Google Drive folder structure and configuration files. Use this skill when setting up the Notion reviewer for the first time, when onboarding a new team member to the reviewer system, or when someone says 'set up notion reviewer', 'configure notion agent', or 'initialize the review system'."
---

# Notion Reviewer — Initial Setup

You are helping the user set up the Notion Reviewer agent for the first time (or onboard to an existing setup). Follow these steps carefully.

> **File format reference**: All state file schemas and default values are defined in `references/coordination-protocol.md`. Use those defaults when initializing files.

## Step 1: Check Prerequisites

Before anything else, verify the user has the required connectors:

1. **Notion connector** — Ask the user to confirm it's connected. The agent needs read AND write access (to post comments and update pages).
2. **Google Drive connector** — Ask the user to confirm it's connected. The agent needs read AND write access to a shared folder.

If either connector is missing, guide the user to Settings > Connectors to add them.

## Step 2: Determine Setup Mode

Ask the user:

> Are you the **designated runner** (the person whose account runs the daily scheduled tasks)?
> Or are you **joining an existing setup** (a colleague already runs the agent and you want to use the plugin for on-demand reviews)?

### If DESIGNATED RUNNER — Create the Drive Structure

Create the following folder structure in Google Drive. Ask the user for their preferred shared Drive location (e.g., a team shared drive or a specific folder).

```
notion-reviewer/
├── config/
│   └── config.json            # Pages to monitor, team assignments
├── state/
│   ├── scan-state.json        # Last scan timestamp, processed meeting notes
│   ├── scan-lock.json         # Coordination lock for multi-runner prevention
│   └── review-queue.json      # Pages queued for deep review
├── logs/
│   └── review-log.json        # History of all proposed changes
└── inbox/
    └── (drop meeting notes and files here)
```

Initialize `config/config.json` with this template:

```json
{
  "version": "1.0",
  "designated_runner": "<user-name>",
  "team": ["<user-name>"],
  "scan_schedule": "daily",
  "review_scope": {
    "monitored_pages": [],
    "monitored_databases": [],
    "exclude_pages": []
  },
  "review_rules": {
    "auto_comment": false,
    "auto_edit": false,
    "flag_for_human_review": true,
    "max_pages_per_run": 30,
    "skip_if_scanned_within_hours": 12
  },
  "notifications": {
    "summary_page_id": null
  }
}
```

Initialize the remaining state and log files with the default values from `references/coordination-protocol.md`:
- `state/scan-state.json`
- `state/scan-lock.json`
- `state/review-queue.json`
- `logs/review-log.json`

Then help the user populate `config.json` with the Notion pages and databases they want to monitor. Walk them through it:

1. Ask: "Which Notion pages or databases should the agent monitor? You can provide page URLs, page IDs, or describe them and I'll search Notion."
2. Use Notion search to find the pages and get their IDs.
3. Add them to `monitored_pages` or `monitored_databases` in config.json.

### If JOINING EXISTING SETUP

Ask for the shared Google Drive folder path. Verify the folder structure exists and read config.json. Confirm the user's name is in the `team` array. If not, add them.

## Step 3: Verify Connectivity

Run a quick test:

1. Read one page from the Notion workspace to confirm Notion access works.
2. Read config.json from Drive to confirm Drive access works.
3. List any existing items in the inbox/ folder.

Report results to the user.

## Step 4: Guide Scheduled Task Creation (Designated Runner Only)

Tell the user:

> Now let's create two scheduled tasks. You can do this by going to **Scheduled** in the left sidebar and clicking **+ New task**.
>
> **Task 1 — Notion Scanner** (runs daily, e.g., 7:00 AM)
> Use the skill `/notion-reviewer:notion-scan` as your prompt.
> Set frequency to: Daily
> Ensure both Notion and Google Drive connectors are included.
>
> **Task 2 — Notion Reviewer** (runs daily, e.g., 7:30 AM)
> Use the skill `/notion-reviewer:notion-review` as your prompt.
> Set frequency to: Daily
> Ensure both Notion and Google Drive connectors are included.

## Step 5: Confirm Setup

Summarize what was configured:
- Number of pages/databases being monitored
- Designated runner
- Team members
- Scheduled task cadence
- Google Drive folder location

Remind the user:
- Drop meeting notes (any format) into the `inbox/` folder on Drive — the agent will pick them up
- Review proposed changes in Notion comments
- Run `/notion-reviewer:notion-status` anytime to check the system's state
