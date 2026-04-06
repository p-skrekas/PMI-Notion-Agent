# Notion Reviewer Plugin

A Claude Code plugin with four skills that automate Notion documentation review using Google Drive as a coordination layer.

## Structure

- `skills/` — Skill definitions (SKILL.md files), each in its own subdirectory
- `references/` — Technical specs for file formats, locking, and state management
- `README.md` — User-facing documentation and quick start guide

## Skills

| Skill | Purpose |
|-------|---------|
| `notion-setup` | First-time setup, onboarding, Drive folder creation |
| `notion-scan` | Daily triage — builds prioritized review queue |
| `notion-review` | Deep review — analyzes pages and posts proposals |
| `notion-status` | Health dashboard and stuck-lock detection |

## State Files (on Google Drive)

All state lives in a shared `notion-reviewer/` folder on Google Drive:

- `config/config.json` — Central config (pages, rules, team)
- `state/scan-state.json` — Last scan/review timestamps, processed meeting notes
- `state/scan-lock.json` — Simple mutex for scan coordination
- `state/review-queue.json` — Scanner output, reviewer input
- `logs/review-log.json` — Append-only log (`entries` array)
- `inbox/` — Drop meeting notes here; moved to `inbox/processed/` after review

## Key Conventions

- The scan skill acquires and releases `scan-lock.json`. Stuck locks (>2h) are auto-released.
- Log entries are appended to the `entries` array in `review-log.json`.
- The `processed_meeting_notes` array in `scan-state.json` tracks which inbox files have been scanned.
- Proposals use confidence levels: HIGH (direct evidence), MEDIUM (interpretation), LOW (flag for human).

## Maintaining File Formats

`references/coordination-protocol.md` is the **canonical schema definition** for all state files. When changing any file format:

1. Update the schema in `coordination-protocol.md` first
2. Sync all skills that read or write that file:

| File | Read by | Written by |
|------|---------|------------|
| `config.json` | scan, review, status | setup |
| `scan-state.json` | scan, review, status | scan, review |
| `scan-lock.json` | scan, status | scan |
| `review-queue.json` | review, status | scan, review |
| `review-log.json` | review, status | scan, review |
